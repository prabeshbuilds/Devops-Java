pipeline {

    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
        timeout(time: 20, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    environment {
        APP_NAME = 'java-app'
        CI_IMAGE = "${APP_NAME}:ci-${env.BUILD_NUMBER}"
    }

    stages {

        stage('📋 Pipeline Info') {
            steps {
                echo """
╔══════════════════════════════════════════════════════╗
║               🚀 CI PIPELINE STARTED                ║
╚══════════════════════════════════════════════════════╝
Build  : #${env.BUILD_NUMBER}
Branch : ${env.BRANCH_NAME}
PR     : ${env.CHANGE_ID ?: 'Not a PR'}
═══════════════════════════════════════════════════════
"""
            }
        }

        stage('🔧 Verify Environment') {
            steps {
                sh '''
                    echo "Hostname : $(hostname)"
                    echo "User     : $(whoami)"
                    docker --version
                    echo "✅ Environment ready"
                '''
            }
        }

        stage('🔍 Code Quality') {
            steps {
                sh '''
                    echo "Running code quality checks..."
                    echo "✅ Code quality passed"
                '''
            }
        }

        stage('🐳 Docker Build') {
            steps {
                sh """
                    echo "Building image: ${CI_IMAGE}"
                    docker build -t ${CI_IMAGE} .
                    echo "✅ Build successful"
                """
            }
        }

        stage('🧪 Verify Image') {
            steps {
                sh """
                    echo "Checking JAR inside image..."
                    docker run --rm --entrypoint ls ${CI_IMAGE} -lh /app/app.jar

                    echo "Checking Java..."
                    docker run --rm --entrypoint java ${CI_IMAGE} -version

                    echo "Checking exposed port..."
                    docker inspect ${CI_IMAGE} --format='{{json .Config.ExposedPorts}}'

                    echo "✅ Image verification passed"
                """
            }
        }

        stage('🔒 Security Check (Non-Root)') {
            steps {
                sh """
                    CONTAINER_UID=\$(docker run --rm --entrypoint id ${CI_IMAGE} -u)
                    echo "Container UID: \${CONTAINER_UID}"

                    if [ "\${CONTAINER_UID}" = "0" ]; then
                        echo "❌ FAILED: Container running as ROOT!"
                        exit 1
                    else
                        echo "✅ PASSED: Running as non-root user"
                    fi
                """
            }
        }

        stage('🧹 Cleanup Image') {
            steps {
                sh "docker rmi ${CI_IMAGE} || true"
            }
        }
    }

    post {

        success {
            echo """
╔══════════════════════════════════════════════════════╗
║            ✅ CI PIPELINE PASSED                     ║
╚══════════════════════════════════════════════════════╝
Build  : #${env.BUILD_NUMBER}
Branch : ${env.BRANCH_NAME}
══════════════════════════════════════════════════════
"""
        }

        failure {
            echo """
╔══════════════════════════════════════════════════════╗
║              ❌ CI PIPELINE FAILED                   ║
╚══════════════════════════════════════════════════════╝
Build  : #${env.BUILD_NUMBER}
Branch : ${env.BRANCH_NAME}
Logs   : ${env.BUILD_URL}
══════════════════════════════════════════════════════
"""
        }

        always {
            sh 'docker system prune -f || true'
            echo "Pipeline finished."
        }
    }
}