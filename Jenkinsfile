pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
metadata:
  labels:
    jenkins: agent
spec:
  securityContext:
    runAsUser: 0
  dnsPolicy: None
  dnsConfig:
    nameservers:
      - 8.8.8.8
      - 1.1.1.1
  containers:
  - name: jnlp
    image: nocnex/jenkins-agent:nerdctlv4
    args: ['$(JENKINS_SECRET)', '$(JENKINS_NAME)']
    tty: true
    securityContext:
      privileged: true
    volumeMounts:
      - name: workspace-volume
        mountPath: /home/jenkins/agent
      - name: buildkit-cache
        mountPath: /tmp/buildkit-cache
  - name: node
    image: node:18-slim
    command: ['cat']
    tty: true
    volumeMounts:
      - name: workspace-volume
        mountPath: /home/jenkins/agent
  - name: kubectl
    image: bitnami/kubectl:1.29
    command: ['cat']
    tty: true
    volumeMounts:
      - name: workspace-volume
        mountPath: /home/jenkins/agent
  - name: docker
    image: docker:20.10-dind
    command: ['cat']
    tty: true
    securityContext:
      privileged: true
    volumeMounts:
      - name: workspace-volume
        mountPath: /home/jenkins/agent
      - name: docker-sock
        mountPath: /var/run/docker.sock
  volumes:
    - name: workspace-volume
      emptyDir: {}
    - name: buildkit-cache
      emptyDir: {}
    - name: docker-sock
      hostPath:
        path: /var/run/docker.sock
'''
        }
    }

    environment {
        DOCKER_IMAGE = "nocnex/nodejs-app-v2"
        REGISTRY = "docker.io"
        KUBE_NAMESPACE = "devops-tools"
        IMAGE_TAG = "${BUILD_TAG}"
        WORKER_NODE = "worker-node-1"
        CI = 'true'
        NODE_HOME = "/home/jenkins/agent/.nodejs"
        PATH = "${NODE_HOME}/bin:${PATH}"
    }

    stages {
        stage('Verify Kubernetes Access') {
            steps {
                container('kubectl') {
                    withCredentials([file(credentialsId: 'kubernetes_file', variable: 'KUBECONFIG_FILE')]) {
                        sh '''
                            export KUBECONFIG=${KUBECONFIG_FILE}
                            echo "=== Cluster Information ==="
                            kubectl cluster-info
                            echo "=== Node Information ==="
                            kubectl get nodes -o wide
                            echo "=== Verify Worker Node ==="
                            kubectl describe node ${WORKER_NODE} || { echo "Worker node not found"; exit 1; }
                            echo "=== Current Context ==="
                            kubectl config current-context
                            echo "=== Namespace Verification ==="
                            kubectl get ns ${KUBE_NAMESPACE} || kubectl create ns ${KUBE_NAMESPACE}
                            echo "=== RBAC Verification ==="
                            kubectl auth can-i create deployments --namespace ${KUBE_NAMESPACE} || \\
                            { echo "Missing deployment permissions"; exit 1; }
                        '''
                    }
                }
            }
        }

        stage('Setup Node Configuration') {
            steps {
                container('kubectl') {
                    withCredentials([file(credentialsId: 'kubernetes_file', variable: 'KUBECONFIG_FILE')]) {
                        sh '''
                            export KUBECONFIG=${KUBECONFIG_FILE}
                            echo "=== Labeling Worker Node ==="
                            kubectl label nodes ${WORKER_NODE} dedicated=devops-tools --overwrite || \\
                            { echo "Failed to label node"; exit 1; }
                            echo "=== Node Labels Verification ==="
                            kubectl get node ${WORKER_NODE} --show-labels
                            echo "=== PriorityClass Setup ==="
                            kubectl apply -f - <<EOF
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: devops-tools-priority
value: 1000000
globalDefault: false
description: "Priority class for devops-tools namespace"
EOF
                        '''
                    }
                }
            }
        }

        stage('Checkout Code') {
            steps {
                container('node') {
                    git url: 'https://github.com/nocnexhamza/nodejs.git', branch: 'main'
                }
            }
        }

        stage('Setup Node.js') {
            steps {
                container('node') {
                    sh """
                        mkdir -p ${NODE_HOME}
                        curl -fsSL https://nodejs.org/dist/v18.20.0/node-v18.20.0-linux-x64.tar.gz -o /home/jenkins/agent/node.tar.gz
                        tar -xzf /home/jenkins/agent/node.tar.gz -C ${NODE_HOME} --strip-components=1
                        rm -f /home/jenkins/agent/node.tar.gz
                        node --version
                        npm --version
                    """
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                container('node') {
                    sh 'npm ci --prefer-offline --audit false'
                }
            }
        }

        stage('Test') {
            steps {
                container('node') {
                    sh 'npm test'
                }
            }
        }

        stage('Build & Push Image') {
            steps {
                container('docker') {
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhublogin',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh '''
                            # Verify Docker credentials
                            echo "=== Docker Credentials Test ==="
                            docker login -u ${DOCKER_USER} -p ${DOCKER_PASS} docker.io || \\
                            { echo "Docker login failed"; exit 1; }

                            # Build and push with verification
                            echo "=== Building Image ==="
                            docker build -t docker.io/${DOCKER_IMAGE}:${IMAGE_TAG} .
                            docker push docker.io/${DOCKER_IMAGE}:${IMAGE_TAG} || \\
                            { echo "Image push failed"; exit 1; }

                            echo "=== Image Verification ==="
                            docker pull docker.io/${DOCKER_IMAGE}:${IMAGE_TAG} || \\
                            { echo "Image pull verification failed"; exit 1; }
                        '''
                    }
                }
            }
        }

        stage('Deploy Application') {
            steps {
                container('kubectl') {
                    withCredentials([file(credentialsId: 'kubernetes_file', variable: 'KUBECONFIG_FILE')]) {
                        sh '''
                            export KUBECONFIG=${KUBECONFIG_FILE}
                            echo "=== Applying Deployment ==="
                            kubectl apply -n ${KUBE_NAMESPACE} -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-app
  namespace: ${KUBE_NAMESPACE}
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nodejs
  template:
    metadata:
      labels:
        app: nodejs
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: dedicated
                operator: In
                values:
                - "devops-tools"
      containers:
      - name: nodejs-app
        image: docker.io/${DOCKER_IMAGE}:${IMAGE_TAG}
        ports:
        - containerPort: 3000
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "1000m"
            memory: "1024Mi"
EOF

                            echo "=== Deployment Verification ==="
                            kubectl get deployment -n ${KUBE_NAMESPACE} || \\
                            { echo "Deployment not found"; exit 1; }

                            echo "=== Applying Service ==="
                            kubectl apply -n ${KUBE_NAMESPACE} -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: nodejs-service
  namespace: ${KUBE_NAMESPACE}
spec:
  selector:
    app: nodejs
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
  type: LoadBalancer
EOF

                            echo "=== Service Verification ==="
                            kubectl get service -n ${KUBE_NAMESPACE} || \\
                            { echo "Service not found"; exit 1; }

                            echo "=== Waiting for Rollout ==="
                            kubectl rollout status deployment/nodejs-app -n ${KUBE_NAMESPACE} --timeout=300s || \\
                            { echo "Rollout failed"; exit 1; }
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            container('kubectl') {
                withCredentials([file(credentialsId: 'kubernetes_file', variable: 'KUBECONFIG_FILE')]) {
                    sh '''
                        export KUBECONFIG=${KUBECONFIG_FILE}
                        echo "=== Final Resource Status ==="
                        kubectl get all -n ${KUBE_NAMESPACE}
                    '''
                }
            }
            container('docker') {
                sh 'docker logout || true'
            }
        }
        failure {
            container('kubectl') {
                withCredentials([file(credentialsId: 'kubernetes_file', variable: 'KUBECONFIG_FILE')]) {
                    sh '''
                        export KUBECONFIG=${KUBECONFIG_FILE}
                        echo "=== Detailed Diagnostics ==="
                        echo "--- Node Status ---"
                        kubectl describe node ${WORKER_NODE}
                        echo "--- Pods ---"
                        kubectl get pods -n ${KUBE_NAMESPACE} -o wide
                        echo "--- Events ---"
                        kubectl get events -n ${KUBE_NAMESPACE} --sort-by=.metadata.creationTimestamp
                        echo "--- Deployment ---"
                        kubectl describe deployment -n ${KUBE_NAMESPACE}
                        echo "--- Service ---"
                        kubectl describe service -n ${KUBE_NAMESPACE}
                    '''
                }
            }
        }
    }
}
