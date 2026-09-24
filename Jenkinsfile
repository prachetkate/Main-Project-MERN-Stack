pipeline {

    agent any

    environment {
        SONAR_PROJECT_KEY = 'wonderla'

        BACKEND_IMAGE = 'prachetkate111/wanderlust-backend'
        FRONTEND_IMAGE = 'prachetkate111/wanderlust-frontend'

        IMAGE_TAG = "${BUILD_NUMBER}"

        // CHANGE THIS to your actual Jenkins DockerHub credential ID
        DOCKERHUB_CREDENTIALS = 'dockerhub-credentials'
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
                '''
            }
        }

        stage('Install Backend Dependencies') {
            steps {
                dir('backend') {
                    sh '''
                        if [ -f package-lock.json ]; then
                            npm ci
                        else
                            npm install
                        fi
                    '''
                }
            }
        }

        stage('Install Frontend Dependencies') {
            steps {
                dir('frontend') {
                    sh '''
                        if [ -f package-lock.json ]; then
                            npm ci
                        else
                            npm install
                        fi
                    '''
                }
            }
        }

        stage('Application Test') {
            parallel {

                stage('Backend Test') {
                    steps {
                        dir('backend') {
                            sh 'npm test --if-present'
                        }
                    }
                }

                stage('Frontend Test') {
                    steps {
                        dir('frontend') {
                            sh 'npm test --if-present'
                        }
                    }
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
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

                timeout(time: 10, unit: 'MINUTES') {

                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                sh '''
                    trivy fs \
                      --scanners vuln,secret \
                      --severity HIGH,CRITICAL \
                      --ignore-unfixed \
                      .
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    echo "Building Backend Docker Image..."

                    docker build \
                      -t ${BACKEND_IMAGE}:${IMAGE_TAG} \
                      ./backend

                    echo "Building Frontend Docker Image..."

                    docker build \
                      -t ${FRONTEND_IMAGE}:${IMAGE_TAG} \
                      ./frontend

                    echo "Docker images created:"
                    docker images | grep wanderlust
                '''
            }
        }

        stage('Trivy Docker Image Scan') {
            steps {
                sh '''
                    echo "Scanning Backend Image..."

                    trivy image \
                      --severity HIGH,CRITICAL \
                      --ignore-unfixed \
                      ${BACKEND_IMAGE}:${IMAGE_TAG}

                    echo "Scanning Frontend Image..."

                    trivy image \
                      --severity HIGH,CRITICAL \
                      --ignore-unfixed \
                      ${FRONTEND_IMAGE}:${IMAGE_TAG}
                '''
            }
        }

        stage('DockerHub Push') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: "${DOCKERHUB_CREDENTIALS}",
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {

                    sh '''
                        echo "$DOCKER_PASSWORD" | docker login \
                            -u "$DOCKER_USERNAME" \
                            --password-stdin

                        docker push ${BACKEND_IMAGE}:${IMAGE_TAG}

                        docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}

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

            Backend:
            ${BACKEND_IMAGE}:${IMAGE_TAG}

            Frontend:
            ${FRONTEND_IMAGE}:${IMAGE_TAG}

            Build Number:
            ${BUILD_NUMBER}

            ==========================================
            """
        }

        failure {
            echo "Wanderlust CI Pipeline Failed."
        }

        always {
            archiveArtifacts(
                artifacts: '**/dependency-check-report.xml,**/dependency-check-report.html',
                allowEmptyArchive: true
            )
        }
    }
}
