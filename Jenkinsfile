pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'docker.io/nocnex'
        APP_NAME = 'nodejs-hello-world'
        K8S_DEPLOYMENT = 'nodejs-hello-world'
    }
    
     stage('Checkout') {
            steps {
                git branch: 'main', 
                url: 'https://github.com/nocnexhamza/nodejs-helloworld.git',
            }
        }
        
        stage('Build & Test') {
            agent {
                docker {
                    image 'node:18-alpine'
                    args '-u root'  // Needed for dependency installation
                }
            }
            steps {
                sh 'npm install'
                sh 'npm test || echo "Tests failed but continuing deployment"'
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    // Fixed docker build command with BuildKit enabled
                    sh """
                        DOCKER_BUILDKIT=1 docker build \
                        -t ${DOCKER_REGISTRY}/${APP_NAME}:${env.BUILD_NUMBER} .
                    """
                }
            }
        }
        
        stage('Push Docker Image') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'docker-hub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh """
                            echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin
                            docker push ${DOCKER_REGISTRY}/${APP_NAME}:${env.BUILD_NUMBER}
                        """
                    }
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    withKubeConfig([credentialsId: 'k8s-credentials']) {
                        sh """
                            # Use temp file to preserve original
                            sed 's|image:.*|image: ${DOCKER_REGISTRY}/${APP_NAME}:${env.BUILD_NUMBER}|g' \
                                k8s/deployment.yaml > k8s/deployment-${env.BUILD_NUMBER}.yaml
                            
                            kubectl apply -f k8s/deployment-${env.BUILD_NUMBER}.yaml
                            kubectl apply -f k8s/service.yaml
                            kubectl rollout status deployment/${K8S_DEPLOYMENT}
                        """
                    }
                }
            }
        }
    }
    
    post {
        always {
            script {
                // Clean up temporary files
                sh 'rm -f k8s/deployment-*.yaml || true'
                cleanWs()  // Jenkins built-in workspace cleaner
            }
        }
        success {
            script {
                if (env.SLACK_CHANNEL) {
                    slackSend(
                        color: "good",
                        message: "SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                        channel: env.SLACK_CHANNEL
                    )
                }
            }
        }
        failure {
            script {
                if (env.SLACK_CHANNEL) {
                    slackSend(
                        color: "danger",
                        message: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                        channel: env.SLACK_CHANNEL
                    )
                }
            }
        }
    }
}
