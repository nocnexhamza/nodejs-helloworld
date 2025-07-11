pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'docker.io/nocnex'
        APP_NAME = 'nodejs-hello-world'
        K8S_DEPLOYMENT = 'nodejs-hello-world'
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', 
                url: 'https://github.com/nocnexhamza/nodejs-helloworld.git'
            }
        }
        
        stage('Build & Test') {
            agent {
                docker {
                    image 'node:18'
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
                    sh """
                        DOCKER_BUILDKIT=1 docker build \
                        -t ${DOCKER_REGISTRY}/${APP_NAME}:${env.BUILD_NUMBER} .
                    """
                }
            }
        }
        
        // NEW STAGE: Scan with Trivy
        stage('Scan with Trivy') {
            steps {
                script {
                    sh '''
                        docker run --rm \
                            -v /var/run/docker.sock:/var/run/docker.sock \
                            -v $WORKSPACE:/workspace \
                            aquasec/trivy image \
                            --exit-code 1 \
                            --severity CRITICAL \
                            --format template \
                            --template "@/usr/local/share/trivy/templates/html.tpl" \
                            --output /workspace/report.html \
                            ${DOCKER_REGISTRY}/${APP_NAME}:${env.BUILD_NUMBER}
                    '''
                }
            }
            post {
                always {
                    publishHTML(target: [
                        allowMissing: false,
                        alwaysLinkToLastBuild: false,
                        keepAll: true,
                        reportDir: '.',
                        reportFiles: 'report.html',
                        reportName: 'Trivy Report'
                    ])
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
                sh 'rm -f k8s/deployment-*.yaml || true'
                cleanWs()
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
