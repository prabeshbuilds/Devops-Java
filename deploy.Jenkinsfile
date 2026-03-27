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

                    stage('🏥 Health Check') {
                        steps {
                            script {
                                sh """
                                    set -e
                                    echo "=== Preparing Health Check Environment ==="

                                    # Only install curl if missing
                                    if ! command -v curl &> /dev/null; then
                                        if [ -f /etc/debian_version ]; then
                                            echo "Detected Debian/Ubuntu. Installing curl..."
                                            sudo apt-get update -y
                                            sudo apt-get install -y curl
                                        elif [ -f /etc/alpine-release ]; then
                                            echo "Detected Alpine Linux. Installing curl..."
                                            apk add --no-cache curl
                                        else
                                            echo "❌ Unknown OS. Please install curl manually."
                                            exit 1
                                        fi
                                    else
                                        echo "✅ curl already installed at \$(command -v curl)"
                                    fi

                                    # Health check
                                    APP_URL="http://\$DEPLOY_SERVER:\$APP_PORT/"
                                    echo "=== Checking App on \$APP_URL ==="

                                    RETRIES=12
                                    until curl -f \$APP_URL 2>/dev/null || [ \$RETRIES -le 0 ]; do
                                        echo "⏳ Waiting for app... (\$RETRIES retries left)"
                                        sleep 5
                                        RETRIES=\$((RETRIES - 1))
                                    done

                                    if [ \$RETRIES -le 0 ]; then
                                        echo "❌ Application did not become healthy in time!"
                                        exit 1
                                    fi

                                    echo "✅ Application is healthy!"
                                    echo "🌐 Live at: \$APP_URL"
                                """
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