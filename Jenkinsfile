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

                    java -version
                    node --version
                    npm --version
                    git --version
                    docker --version
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
                        echo "Installing backend dependencies..."
                        npm install --no-audit --no-fund
                    '''
                }
            }
        }

        stage('Install Frontend Dependencies') {
            steps {
                dir('frontend') {
                    sh '''
                        echo "Installing frontend dependencies..."
                        npm install --no-audit --no-fund
                    '''
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {

                catchError(
                    buildResult: 'SUCCESS',
                    stageResult: 'UNSTABLE'
                ) {

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

                echo "OWASP Dependency Check completed."
                echo "Vulnerabilities, if any, are being reported."
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

                catchError(
                    buildResult: 'SUCCESS',
                    stageResult: 'UNSTABLE'
                ) {

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

                echo "Trivy filesystem vulnerabilities are being reported."
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

                catchError(
                    buildResult: 'SUCCESS',
                    stageResult: 'UNSTABLE'
                ) {

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

                echo "Trivy image vulnerabilities are being reported."
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
