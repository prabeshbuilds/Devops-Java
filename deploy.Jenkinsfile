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
Branch  : ${env.BRANCH_NAME ?: 'N/A'}
Commit  : ${IMAGE_TAG}
Server  : ${DEPLOY_SERVER}
══════════════════════════════════════════════
"""
            }
        }

        stage('🔨 Build Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USERNAME',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
                    sh '''
                    set -e
                    echo "🔨 Building Docker image..."
                    docker build --pull -t $DOCKER_USERNAME/$APP_NAME:$IMAGE_TAG .
                    docker tag $DOCKER_USERNAME/$APP_NAME:$IMAGE_TAG $DOCKER_USERNAME/$APP_NAME:latest
                    '''
                }
            }
        }

        stage('📤 Push Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USERNAME',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
                    sh '''
                    set -e
                    echo "📤 Logging in to DockerHub..."
                    echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin

                    echo "📤 Pushing Docker images..."
                    docker push $DOCKER_USERNAME/$APP_NAME:$IMAGE_TAG
                    docker push $DOCKER_USERNAME/$APP_NAME:latest
                    '''
                }
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
                    echo '✅ Connected to server'

                    # Ensure Docker network exists
                    docker network inspect private-net >/dev/null 2>&1 || docker network create private-net

                    # Login to DockerHub on remote
                    echo '$DOCKER_PASSWORD' | docker login -u '$DOCKER_USERNAME' --password-stdin

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

                    # Verify container is running
                    sleep 5
                    docker ps | grep $APP_NAME

                    # Show last 20 logs
                    docker logs --tail 20 $APP_NAME

                    # Cleanup old images (keep last 5)
                    docker images --format '{{.Repository}} {{.ID}} {{.CreatedAt}}' \
                        | grep $DOCKER_USERNAME/$APP_NAME \
                        | sort -rk3 \
                        | tail -n +6 \
                        | awk '{print \$2}' \
                        | xargs -r docker rmi || true
                "
                '''
            }
        }
    }
}
stage('Health Check') {
    steps {
        script {
            echo "=== Performing Health Check on ${DEPLOY_URL} ==="
            def healthy = false
            for (int i = 1; i <= MAX_RETRIES; i++) {
                try {
                    def response = sh(
                        script: "curl -s -o /dev/null -w '%{http_code}' ${DEPLOY_URL}",
                        returnStdout: true
                    ).trim()
                    
                    if (response == '200') {
                        echo "✅ Deployment is healthy!"
                        healthy = true
                        break
                    } else {
                        echo "⚠️ Attempt ${i}: Service returned ${response}, retrying in ${RETRY_DELAY}s..."
                    }
                } catch (Exception e) {
                    echo "⚠️ Attempt ${i}: Failed to reach service, retrying in ${RETRY_DELAY}s..."
                }
                sleep RETRY_DELAY
            }
            if (!healthy) {
                error "❌ Deployment health check failed after ${MAX_RETRIES} attempts!"
            }
        }
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
                    set -e
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