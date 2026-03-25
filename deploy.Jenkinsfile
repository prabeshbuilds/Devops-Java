```groovy
pipeline {
    agent any

    triggers {
        githubPush()
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timestamps()
        timeout(time: 45, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    environment {
        APP_NAME      = 'devops-java'
        IMAGE_TAG     = "${env.GIT_COMMIT.take(7)}"

        DEPLOY_SERVER = '185.199.53.175'
        DEPLOY_USER   = 'prabesh'
        DEPLOY_PORT   = '22'
        APP_PORT      = '8080'

        // IMPORTANT: must exist on remote server
        ENV_FILE      = '/home/prabesh/.env'
    }

    stages {

        stage('📋 Pipeline Info') {
            steps {
                echo """
╔══════════════════════════════════════════════╗
║           🚀 CD PIPELINE START               ║
╚══════════════════════════════════════════════╝
Build   : #${env.BUILD_NUMBER}
Branch  : ${env.BRANCH_NAME}
Commit  : ${IMAGE_TAG}
Server  : ${DEPLOY_SERVER}
══════════════════════════════════════════════
"""
            }
        }

        stage('🚀 Deploy to Production') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USERNAME',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {

                    sshagent(['deployment-server-ssh']) {
                        sh '''
                        set -e

                        echo "🚀 Starting Deployment..."

                        ssh -o StrictHostKeyChecking=accept-new -p $DEPLOY_PORT $DEPLOY_USER@$DEPLOY_SERVER "
                            set -e

                            echo 'Connected to server'

                            # Ensure Docker network exists
                            docker network inspect private-net >/dev/null 2>&1 || \
                            docker network create private-net

                            # Login to DockerHub
                            echo \"$DOCKER_PASSWORD\" | docker login -u \"$DOCKER_USERNAME\" --password-stdin

                            # Pull latest image
                            docker pull $DOCKER_USERNAME/$APP_NAME:$IMAGE_TAG

                            # Stop & remove old container
                            docker stop $APP_NAME 2>/dev/null || true
                            docker rm   $APP_NAME 2>/dev/null || true

                            # Run new container
                            docker run -d \
                                --name $APP_NAME \
                                --restart unless-stopped \
                                --network private-net \
                                --env-file $ENV_FILE \
                                -p $APP_PORT:8080 \
                                $DOCKER_USERNAME/$APP_NAME:$IMAGE_TAG

                            # Verify container
                            sleep 5
                            docker ps | grep $APP_NAME

                            # Show logs
                            docker logs --tail 20 $APP_NAME

                            # Cleanup old images (keep last 5)
                            docker images | grep $DOCKER_USERNAME/$APP_NAME | \
                            tail -n +6 | awk '{print \\$3}' | xargs -r docker rmi || true

                            docker logout

                            echo '✅ Deployment successful!'
                        "
                        '''
                    }
                }
            }
        }

        stage('🏥 Health Check') {
            steps {
                sh '''
                set -e

                echo "🏥 Checking application health..."

                # Install curl if missing (Ubuntu/Debian)
                if ! command -v curl >/dev/null 2>&1; then
                    sudo apt-get update && sudo apt-get install -y curl
                fi

                # Retry mechanism (max 10 attempts)
                for i in $(seq 1 10); do
                    if curl -f http://$DEPLOY_SERVER:$APP_PORT; then
                        echo "✅ Application is healthy!"
                        echo "🌐 Live: http://$DEPLOY_SERVER:$APP_PORT"
                        exit 0
                    fi

                    echo "⏳ Waiting for app... ($i/10)"
                    sleep 5
                done

                echo "❌ Health check failed!"
                exit 1
                '''
            }
        }

        stage('🧹 Cleanup Jenkins') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USERNAME',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
                    sh '''
                    IMAGE="$DOCKER_USERNAME/$APP_NAME"

                    docker rmi $IMAGE:$IMAGE_TAG || true
                    docker rmi $IMAGE:latest || true
                    docker image prune -f || true

                    echo "🧹 Cleanup completed"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo """
╔══════════════════════════════════════════════╗
║        🎉 DEPLOYMENT SUCCESSFUL              ║
╚══════════════════════════════════════════════╝
App URL : http://${DEPLOY_SERVER}:${APP_PORT}
Image   : ${IMAGE_TAG}
══════════════════════════════════════════════
"""
        }

        failure {
            echo """
╔══════════════════════════════════════════════╗
║            ❌ DEPLOYMENT FAILED              ║
╚══════════════════════════════════════════════╝
Build : #${env.BUILD_NUMBER}
Logs  : ${env.BUILD_URL}
══════════════════════════════════════════════
"""
        }

        always {
            sh 'docker image prune -f || true'
        }
    }
}
```
