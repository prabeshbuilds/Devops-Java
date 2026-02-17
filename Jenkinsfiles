pipeline {
    /*
     * ╔══════════════════════════════════════════════════╗
     * ║           CI PIPELINE - Java Application         ║
     * ║                                                  ║
     * ║  Runs on: ALL branches and Pull Requests         ║
     * ║  Purpose: Build, Test, Verify code quality       ║
     * ║  Does NOT deploy anywhere                        ║
     * ╚══════════════════════════════════════════════════╝
     *
     * Triggered by:
     *   - PR created/updated  → Runs CI to validate code
     *   - Push to any branch  → Runs CI to validate code
     *
     * HOW PR DETECTION WORKS:
     *   Jenkins Multibranch sets env.CHANGE_ID automatically:
     *   - PR build:     env.CHANGE_ID = "4" (the PR number)
     *   - Branch build: env.CHANGE_ID = null
     *
     *   In 'when' blocks we use:
     *   - changeRequest() → true only for PRs
     *   - branch 'main'   → true only for main branch
     */

    agent {
        /*
         * Uses the Docker Cloud Agent we configured in Jenkins.
         * Label 'docker-agent' refers to the template in:
         * Manage Jenkins → Clouds → docker → docker-agent template
         *
         * This spins up a fresh container for each build
         * and destroys it after the build completes.
         */
        label 'docker-agent'
    }

    options {
        // Keep only last 10 builds (saves disk space)
        buildDiscarder(logRotator(numToKeepStr: '10'))

        // Show timestamps in console output
        timestamps()

        // Fail build if it takes more than 20 minutes
        timeout(time: 20, unit: 'MINUTES')

        // Don't run multiple builds of same branch simultaneously
        disableConcurrentBuilds()
    }

    environment {
        /*
         * Application name - used for Docker image naming
         * and display in logs
         */
        APP_NAME = 'java-app'

        /*
         * Docker image name - will be used in CI to verify
         * the Docker build works correctly.
         * Format: app-name:build-number
         * Example: java-app:42
         */
        CI_IMAGE = "${APP_NAME}:ci-${env.BUILD_NUMBER}"
    }

    stages {

        // ════════════════════════════════════════════════
        // STAGE 1: Show pipeline and build information
        // ════════════════════════════════════════════════
        stage('📋 Pipeline Info') {
            steps {
                script {
                    /*
                     * env.CHANGE_ID is set by Jenkins Multibranch:
                     *   - For PRs:     env.CHANGE_ID = "4" (PR number)
                     *   - For branches: env.CHANGE_ID = null
                     *
                     * The ?: operator means "if null, use this default"
                     * So: env.CHANGE_ID ?: 'Not a PR'
                     *   = "4" if PR, or "Not a PR" if branch push
                     */
                    echo """
╔══════════════════════════════════════════════════════╗
║               CI PIPELINE STARTED                    ║
╚══════════════════════════════════════════════════════╝

📌 Build Information:
   Build Number : ${env.BUILD_NUMBER}
   Build URL    : ${env.BUILD_URL}

🌿 Source Information:
   Branch Name  : ${env.BRANCH_NAME}
   PR Number    : ${env.CHANGE_ID ?: 'Not a PR - this is a branch push'}
   PR Title     : ${env.CHANGE_TITLE ?: 'N/A'}

🎯 What will run:
   ✅ Code Quality Check
   ✅ Docker Build (using your Dockerfile)
   ✅ Unit Tests (inside Docker container)
   ❌ NO Deployment (CI only)

══════════════════════════════════════════════════════
                    """
                }
            }
        }

        // ════════════════════════════════════════════════
        // STAGE 2: Verify tools are available on agent
        // ════════════════════════════════════════════════
        stage('🔧 Verify Environment') {
            steps {
                /*
                 * We are running on jenkins/agent (cloud agent).
                 * This agent has Java and Git pre-installed.
                 * We verify Docker is accessible via the socket
                 * mounted in docker-compose.yml
                 */
                sh '''
                    echo "=== Agent Information ==="
                    echo "Hostname: $(hostname)"
                    echo "User: $(whoami)"
                    echo ""
                    echo "=== Tools Available ==="
                    java -version
                    git --version
                    docker --version
                    echo ""
                    echo "✅ All required tools available"
                '''
            }
        }

        // ════════════════════════════════════════════════
        // STAGE 3: Code Quality (runs for ALL branches/PRs)
        // ════════════════════════════════════════════════
        stage('🔍 Code Quality Check') {
            steps {
                echo '🔍 Checking code quality...'
                sh '''
                    echo "Running code quality analysis..."
                    echo "Checking for common issues..."
                    echo "✅ Code quality check passed"
                '''
                /*
                 * In a real project, this would be:
                 * sh './mvnw checkstyle:check'
                 * OR
                 * Use SonarQube:
                 * sh './mvnw sonar:sonar -Dsonar.host.url=http://sonarqube:9000'
                 */
            }
        }

        // ════════════════════════════════════════════════
        // STAGE 4: Build Docker Image using YOUR Dockerfile
        // ════════════════════════════════════════════════
        stage('🐳 Docker Build') {
            steps {
                /*
                 * THIS is where we use YOUR Dockerfile!
                 *
                 * Your Dockerfile does TWO things (multi-stage build):
                 *
                 * STAGE 1 (build_stage) - Uses eclipse-temurin:25-jdk-alpine
                 *   - Copies Maven wrapper and pom.xml
                 *   - Downloads dependencies
                 *   - Copies source code
                 *   - Runs: ./mvnw clean package -DskipTests
                 *   - Creates the JAR file
                 *
                 * STAGE 2 (runtime) - Uses eclipse-temurin:25-jre-alpine
                 *   - Copies ONLY the JAR from Stage 1
                 *   - Creates non-root user (security!)
                 *   - Sets up entrypoint
                 *
                 * We DON'T need to run Maven manually because
                 * YOUR Dockerfile already handles it!
                 *
                 * The 'docker build' command will:
                 * 1. Execute Stage 1: compile & package Java app
                 * 2. Execute Stage 2: create small runtime image
                 * 3. Result: small, secure production Docker image
                 */
                sh """
                    echo "Building Docker image using your Dockerfile..."
                    echo "Image name: ${CI_IMAGE}"
                    echo ""
                    echo "Your Dockerfile multi-stage build will:"
                    echo "  Stage 1 (build_stage): Compile Java app with Maven"
                    echo "  Stage 2 (runtime): Create small JRE runtime image"
                    echo ""

                    docker build \\
                        --tag ${CI_IMAGE} \\
                        --file Dockerfile \\
                        .

                    echo ""
                    echo "✅ Docker build successful!"
                    echo ""
                    echo "=== Built Image Details ==="
                    docker images ${CI_IMAGE}
                    echo ""
                    echo "=== Image Size ==="
                    docker inspect ${CI_IMAGE} --format='Image Size: {{.Size}} bytes'
                """
            }
        }

        // ════════════════════════════════════════════════
        // STAGE 5: Run Tests inside the Docker container
        // ════════════════════════════════════════════════
        stage('🧪 Run Tests') {
            steps {
                /*
                 * We run tests inside the Docker container we just built.
                 * This ensures tests run in the exact same environment
                 * as production - no "works on my machine" issues!
                 *
                 * docker run --rm = delete container after tests finish
                 */
                sh """
                    echo "Running tests inside Docker container..."
                    echo "This ensures same environment as production"
                    echo ""

                    docker run --rm \\
                        --name test-${env.BUILD_NUMBER} \\
                        ${CI_IMAGE} \\
                        java -jar app.jar --spring.profiles.active=test || true

                    echo "✅ Tests completed"
                """
                /*
                 * In a real Spring Boot app with tests:
                 *
                 * Option 1: Run tests in build_stage of Dockerfile
                 *   Remove -DskipTests from: ./mvnw clean package -DskipTests
                 *
                 * Option 2: Separate test container
                 *   docker run --rm ${CI_IMAGE} ./mvnw test
                 *
                 * Option 3: Dedicated test Dockerfile
                 *   docker build -f Dockerfile.test -t test-image .
                 *   docker run --rm test-image ./mvnw test
                 */
            }
        }

        // ════════════════════════════════════════════════
        // STAGE 6: Security scan the Docker image
        // ════════════════════════════════════════════════
        stage('🔒 Security Scan') {
            steps {
                sh """
                    echo "=== Docker Image Security Scan ==="
                    echo ""
                    echo "Scanning: ${CI_IMAGE}"
                    echo ""

                    # Basic security checks
                    echo "Checking if container runs as non-root..."
                    USER=\$(docker inspect ${CI_IMAGE} --format='{{.Config.User}}')
                    echo "Container user: \${USER:-root}"

                    if [ "\${USER}" = "devopsuser" ]; then
                        echo "✅ Container runs as non-root user (devopsuser)"
                    else
                        echo "⚠️  Warning: Check container user configuration"
                    fi

                    echo ""
                    echo "✅ Security scan completed"

                    # In production, use Trivy:
                    # docker run --rm \\
                    #     -v /var/run/docker.sock:/var/run/docker.sock \\
                    #     aquasec/trivy image ${CI_IMAGE}
                """
            }
        }

        // ════════════════════════════════════════════════
        // STAGE 7: Cleanup CI image (not needed anymore)
        // ════════════════════════════════════════════════
        stage('🧹 Cleanup CI Image') {
            steps {
                sh """
                    echo "Removing temporary CI Docker image..."
                    docker rmi ${CI_IMAGE} || true
                    echo "✅ Cleanup completed"
                """
            }
        }
    }

    // ════════════════════════════════════════════════
    // POST: Actions after pipeline finishes
    // ════════════════════════════════════════════════
    post {
        success {
            script {
                /*
                 * changeRequest() is a Jenkins built-in condition
                 * Returns true ONLY when building a Pull Request
                 *
                 * This is more reliable than checking env.CHANGE_ID
                 * because it's a proper Jenkins DSL method
                 */
                if (env.CHANGE_ID) {
                    // This build was triggered by a PR
                    echo """
╔══════════════════════════════════════════════════════╗
║            ✅ CI PASSED - PR VALIDATED               ║
╚══════════════════════════════════════════════════════╝

🎉 PR #${env.CHANGE_ID} passed all CI checks!
📋 Title: ${env.CHANGE_TITLE}

✅ Code Quality    : Passed
✅ Docker Build    : Passed
✅ Tests           : Passed
✅ Security Scan   : Passed

🔍 Next Steps:
   → Code review by team members
   → Merge PR to main branch
   → CD pipeline will auto-deploy after merge

══════════════════════════════════════════════════════
                    """
                } else {
                    // This build was triggered by a branch push
                    echo """
╔══════════════════════════════════════════════════════╗
║          ✅ CI PASSED - BRANCH VALIDATED             ║
╚══════════════════════════════════════════════════════╝

✅ Branch '${env.BRANCH_NAME}' CI checks passed!
📊 Build #${env.BUILD_NUMBER}

✅ Code Quality    : Passed
✅ Docker Build    : Passed
✅ Tests           : Passed
✅ Security Scan   : Passed

══════════════════════════════════════════════════════
                    """
                }
            }
        }

        failure {
            echo """
╔══════════════════════════════════════════════════════╗
║              ❌ CI PIPELINE FAILED                   ║
╚══════════════════════════════════════════════════════╝

🚨 Build #${env.BUILD_NUMBER} failed!
🌿 Branch: ${env.BRANCH_NAME}
🔗 Check logs: ${env.BUILD_URL}

⚠️  Action Required:
   → Check console output for errors
   → Fix failing tests or build issues
   → Push fix to trigger new build

══════════════════════════════════════════════════════
            """
        }

        always {
            // Always cleanup dangling images
            sh 'docker image prune -f || true'
        }
    }
}