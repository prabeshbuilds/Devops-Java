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

        ENV_FILE      = '/home/jenkins/.env'
    }

    stages {

        stage('📋 Info') {
            steps {
                echo """
🚀 Build #${env.BUILD_NUMBER}
Branch: ${env.BRANCH_NAME}
Commit: ${IMAGE_TAG}
Deploy: ${DEPLOY_SERVER}
"""
            }
        }
stage('🚀 Deploy') {
    steps {
        withCredentials([usernamePassword(
            credentialsId: 'dockerhub-credentials',
            usernameVariable: 'DOCKER_USERNAME',
            passwordVariable: 'DOCKER_PASSWORD'
        )]) {
            sshagent(['deployment-server-ssh']) {
                sh '''
                ssh -o StrictHostKeyChecking=no -p $DEPLOY_PORT $DEPLOY_USER@$DEPLOY_SERVER "
                    set -e

                    echo '🚀 Deploying application...'

                    # Ensure network
                    sudo docker network create private-net || true

                    # Login DockerHub
                    echo '$DOCKER_PASSWORD' | sudo docker login -u '$DOCKER_USERNAME' --password-stdin

                    # Pull image
                    sudo docker pull $DOCKER_USERNAME/$APP_NAME:$IMAGE_TAG

                    # Stop old container
                    sudo docker stop $APP_NAME || true
                    sudo docker rm $APP_NAME || true

                    # Run container
                    sudo docker run -d \
                      --name $APP_NAME \
                      --restart unless-stopped \
                      --network private-net \
                      --env-file $ENV_FILE \
                      -p $APP_PORT:8080 \
                      $DOCKER_USERNAME/$APP_NAME:$IMAGE_TAG

                    # Verify
                    sleep 5
                    sudo docker ps | grep $APP_NAME

                    sudo docker logout

                    echo '✅ Deployment done!'
                "
                '''
            }
        }
    }
}

        stage('🏥 Health Check') {
            steps {
                sh '''
                    echo "Checking application health..."

                    # Install curl if missing (Ubuntu)
                    if ! command -v curl >/dev/null 2>&1; then
                      sudo apt-get update && sudo apt-get install -y curl
                    fi

                    sleep 20

                    curl -f http://$DEPLOY_SERVER:$APP_PORT || exit 1

                    echo "✅ App is healthy!"
                '''
            }
        }

        stage('🧹 Cleanup') {
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
                        docker image prune -f
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "✅ SUCCESS: http://${DEPLOY_SERVER}:${APP_PORT}"
        }

        failure {
            echo "❌ FAILED: ${env.BUILD_URL}"
        }

        always {
            sh 'docker image prune -f || true'
        }
    }
}