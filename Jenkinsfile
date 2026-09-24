pipeline {

    agent any

    environment {
        SONAR_PROJECT_KEY = 'wonderla'

        BACKEND_IMAGE = 'prachetkate111/wanderlust-backend'
        FRONTEND_IMAGE = 'prachetkate111/wanderlust-frontend'

        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Clean Workspace') {
            steps {
                deleteDir()
            }
        }

        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }

        stage('Verify Tools') {
            steps {
                sh '''
                    echo "========================================"
                    echo "Verifying Required Tools"
                    echo "========================================"

                    echo "===== Java ====="
                    java -version

                    echo "===== Node ====="
                    node --version

                    echo "===== NPM ====="
                    npm --version

                    echo "===== Git ====="
                    git --version

                    echo "===== Docker ====="
                    docker --version

                    echo "===== Trivy ====="
                    trivy --version

                    echo "========================================"
                    echo "All tools verified"
                    echo "========================================"
                '''
            }
        }

        stage('Install Backend Dependencies') {
            steps {
                dir('backend') {
                    sh '''
                        echo "========================================"
                        echo "Installing Backend Dependencies"
                        echo "========================================"

                        npm install --no-audit --no-fund

                        echo "Backend dependencies installed successfully"
                    '''
                }
            }
        }

        stage('Install Frontend Dependencies') {
            steps {
                dir('frontend') {
                    sh '''
                        echo "========================================"
                        echo "Installing Frontend Dependencies"
                        echo "========================================"

                        npm install --no-audit --no-fund

                        echo "Frontend dependencies installed successfully"
                    '''
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {

                echo "========================================"
                echo "Running OWASP Dependency Check"
                echo "========================================"

                dependencyCheck(
                    odcInstallation: 'DependencyCheck',
                    additionalArguments: '''
                        --scan .
                        --format XML
                        --format HTML
                        --prettyPrint
                        --disableNodeAudit
                    '''
                )

                dependencyCheckPublisher(
                    pattern: '**/dependency-check-report.xml'
                )
            }
        }

        stage('SonarQube Analysis') {
            steps {

                script {

                    echo "========================================"
                    echo "Running SonarQube Analysis"
                    echo "========================================"

                    def scannerHome = tool 'SonarQubeScanner'

                    withSonarQubeEnv('SonarQube') {

                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                              -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                              -Dsonar.projectName=Wanderlust \
                              -Dsonar.sources=backend,frontend \
                              -Dsonar.exclusions=**/node_modules/**,**/dist/**,**/build/**
                        """
                    }
                }
            }
        }

        stage('SonarQube Quality Gate') {
            steps {

                echo "========================================"
                echo "Waiting for SonarQube Quality Gate"
                echo "========================================"

                timeout(time: 10, unit: 'MINUTES') {

                    waitForQualityGate abortPipeline: true
                }

                echo "SonarQube Quality Gate Passed"
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {

                sh '''
                    echo "========================================"
                    echo "Running Trivy Filesystem Scan"
                    echo "========================================"

                    trivy fs \
                      --scanners vuln,secret \
                      --severity HIGH,CRITICAL \
                      --ignore-unfixed \
                      .

                    echo "========================================"
                    echo "Trivy Filesystem Scan Completed"
                    echo "========================================"
                '''
            }
        }

        stage('Docker Build') {
            steps {

                sh '''
                    echo "========================================"
                    echo "Building Backend Docker Image"
                    echo "========================================"

                    docker build \
                      -t ${BACKEND_IMAGE}:${IMAGE_TAG} \
                      ./backend

                    echo "========================================"
                    echo "Building Frontend Docker Image"
                    echo "========================================"

                    docker build \
                      -t ${FRONTEND_IMAGE}:${IMAGE_TAG} \
                      ./frontend

                    echo "========================================"
                    echo "Docker Images Created"
                    echo "========================================"

                    docker images | grep wanderlust
                '''
            }
        }

        stage('Trivy Docker Image Scan') {
            steps {

                sh '''
                    echo "========================================"
                    echo "Scanning Backend Docker Image"
                    echo "========================================"

                    trivy image \
                      --severity HIGH,CRITICAL \
                      --ignore-unfixed \
                      ${BACKEND_IMAGE}:${IMAGE_TAG}

                    echo "========================================"
                    echo "Scanning Frontend Docker Image"
                    echo "========================================"

                    trivy image \
                      --severity HIGH,CRITICAL \
                      --ignore-unfixed \
                      ${FRONTEND_IMAGE}:${IMAGE_TAG}

                    echo "========================================"
                    echo "Docker Image Security Scan Completed"
                    echo "========================================"
                '''
            }
        }

        stage('DockerHub Push') {
            steps {

                echo "========================================"
                echo "Pushing Images to DockerHub"
                echo "========================================"

                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-cred',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {

                    sh '''
                        echo "Logging in to DockerHub..."

                        echo "$DOCKER_PASSWORD" | docker login \
                            -u "$DOCKER_USERNAME" \
                            --password-stdin

                        echo "========================================"
                        echo "Pushing Backend Image"
                        echo "========================================"

                        docker push ${BACKEND_IMAGE}:${IMAGE_TAG}

                        echo "========================================"
                        echo "Pushing Frontend Image"
                        echo "========================================"

                        docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}

                        echo "========================================"
                        echo "DockerHub Push Completed"
                        echo "========================================"

                        docker logout
                    '''
                }
            }
        }
    }

    post {

        success {

            echo """
            ==========================================
                  WANDERLUST CI SUCCESS
            ==========================================

            Backend Image:
            ${BACKEND_IMAGE}:${IMAGE_TAG}

            Frontend Image:
            ${FRONTEND_IMAGE}:${IMAGE_TAG}

            Build Number:
            ${BUILD_NUMBER}

            ==========================================
                  CI PIPELINE COMPLETED
            ==========================================
            """
        }

        failure {

            echo """
            ==========================================
                  WANDERLUST CI PIPELINE FAILED
            ==========================================

            Check the failed stage in Console Output.

            ==========================================
            """
        }

        always {

            archiveArtifacts(
                artifacts: '**/dependency-check-report.xml,**/dependency-check-report.html',
                allowEmptyArchive: true
            )
        }
    }
}
