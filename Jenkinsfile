pipeline {
    agent any

    parameters {
        string(name: 'UI_TAG', description: 'Ui Docker Image Tag', defaultValue: 'latest')
        string(name: 'CART_TAG', description: 'Cart Docker Image Tag', defaultValue: 'latest')
        string(name: 'ORDERS_TAG', description: 'Orders Docker Image Tag', defaultValue: 'latest')
        string(name: 'CATALOG_TAG', description: 'Catalog Docker Image Tag', defaultValue: 'latest')
        string(name: 'CHECKOUT_TAG', description: 'Checkout Docker Image Tag', defaultValue: 'latest')
    }

    environment {
        GITHUB_CREDENTIALS = 'github-cred'
        DOCKER_CREDENTIALS = 'dockerhub-cred'
        DOCKERHUB_USERNAME = 'roshanx'  // ✅ ADD THIS
        SONAR_HOME = tool "SonarQube"
    }

    stages {
        stage("Workspace Cleanup") {
            steps {
                cleanWs()
            }
        }

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    credentialsId: env.GITHUB_CREDENTIALS,
                    url: 'https://github.com/roshanx69/retail-store-sample-app.git'  // ✅ FIX REPO URL
            }
        }

        stage('Security Scans') {
            parallel {
                stage('Trivy Filesystem Scan') {
                    steps {
                        sh '''
                            trivy fs --exit-code 1 --severity CRITICAL --no-progress . || true
                        '''
                    }
                }
                stage('OWASP Dependency Check') {
                    steps {
                        script {
                            // Check if dependency-check.sh exists
                            def exists = sh(script: 'which dependency-check.sh || true', returnStdout: true).trim()
                            if (exists) {
                                sh 'dependency-check.sh --project store-app --scan . || true'
                            } else {  // ✅ FIXED: Proper closing brace
                                echo '⚠️ OWASP Dependency Check skipped: dependency-check.sh not found.'
                            }
                        }
                    }
                }
            }
        }

        stage('Check Sonar') {
            steps {
                sh 'echo SONAR_HOME="$SONAR_HOME"'
                sh '$SONAR_HOME/bin/sonar-scanner --version'
            }
        }

        stage('SonarQube: Code Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        ${SONAR_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=store-app \
                        -Dsonar.projectName=store-app \
                        -Dsonar.sources=.
                    '''
                }
            }
        }

        stage('SonarQube: Quality Gate') {
            steps {
                timeout(time: 1, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Login') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: env.DOCKER_CREDENTIALS,
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    '''
                }
            }
        }

        stage('Build & Push Images') {
            parallel {
                stage('Build & Push Cart Image') {
                    steps {
                        sh '''
                            docker build -t ${DOCKERHUB_USERNAME}/store-app-cart:${CART_TAG} ./src/cart
                            docker push ${DOCKERHUB_USERNAME}/store-app-cart:${CART_TAG}
                        '''
                    }
                }
                stage('Build & Push Catalog Image') {
                    steps {
                        sh '''
                            docker build -t ${DOCKERHUB_USERNAME}/store-app-catalog:${CATALOG_TAG} ./src/catalog
                            docker push ${DOCKERHUB_USERNAME}/store-app-catalog:${CATALOG_TAG}
                        '''
                    }
                }
                stage('Build & Push Checkout Image') {
                    steps {
                        sh '''
                            docker build -t ${DOCKERHUB_USERNAME}/store-app-checkout:${CHECKOUT_TAG} ./src/checkout
                            docker push ${DOCKERHUB_USERNAME}/store-app-checkout:${CHECKOUT_TAG}
                        '''
                    }
                }
                stage('Build & Push Orders Image') {
                    steps {
                        sh '''
                            docker build -t ${DOCKERHUB_USERNAME}/store-app-orders:${ORDERS_TAG} ./src/orders
                            docker push ${DOCKERHUB_USERNAME}/store-app-orders:${ORDERS_TAG}
                        '''
                    }
                }
                stage('Build & Push UI Image') {
                    steps {
                        sh '''
                            docker build -t ${DOCKERHUB_USERNAME}/store-app-ui:${UI_TAG} ./src/ui
                            docker push ${DOCKERHUB_USERNAME}/store-app-ui:${UI_TAG}
                        '''
                    }
                }
            }
        }

        stage('Trigger CD Pipeline') {
            steps {
                build job: 'storeapp-cd', parameters: [
                    string(name: 'UI_TAG', value: params.UI_TAG),
                    string(name: 'ORDERS_TAG', value: params.ORDERS_TAG),
                    string(name: 'CHECKOUT_TAG', value: params.CHECKOUT_TAG),
                    string(name: 'CATALOG_TAG', value: params.CATALOG_TAG),
                    string(name: 'CART_TAG', value: params.CART_TAG)
                ]
            }
        }
    }

    post {
        failure {
            echo "❌ The store-app CI pipeline has failed. Please check Jenkins for errors."
        }
    }
}



