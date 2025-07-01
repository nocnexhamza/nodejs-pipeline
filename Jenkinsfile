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
  volumes:
    - name: workspace-volume
      emptyDir: {}
    - name: buildkit-cache
      emptyDir: {}
'''
    }
  }
environment {
    DOCKER_IMAGE = "nocnex/nodejs-app-v2"
    REGISTRY = "docker.io"
    KUBE_NAMESPACE = "devops-tools"
    IMAGE_TAG = "${BUILD_TAG}"
    WORKER_NODE = "worker-node-1"
  }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Build & Push Image') {
      steps {
        container('jnlp') {
          withCredentials([usernamePassword(
            credentialsId: 'dockerhublogin',
            usernameVariable: 'DOCKER_USER',
            passwordVariable: 'DOCKER_PASS'
          )]) {
            sh '''
              # Verify Docker credentials
              echo "=== Docker Credentials Test ==="
              docker login -u ${DOCKER_USER} -p ${DOCKER_PASS} docker.io || \
                { echo "Docker login failed"; exit 1; }
              
              # Build and push with verification
              echo "=== Building Image ==="
              buildctl build \
                --frontend dockerfile.v0 \
                --local context=. \
                --local dockerfile=. \
                --output type=image,name=docker.io/${DOCKER_IMAGE}:${IMAGE_TAG},push=true \
                --opt registry.username=${DOCKER_USER} \
                --opt registry.password=${DOCKER_PASS} || \
                { echo "Image build failed"; exit 1; }
              
              echo "=== Image Verification ==="
              docker pull docker.io/${DOCKER_IMAGE}:${IMAGE_TAG} || \
                { echo "Image pull verification failed"; exit 1; }
            '''
          }
        }
      }
    }
        stage('Test') {
            steps {
                script {
                    dockerImage.inside {
                         // Set execute permissions for node_modules/.bin
                        sh 'chmod +x node_modules/.bin/mocha'
                        sh 'npm test'
                    }
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                script {
                  
                      kubeconfig(credentialsId: 'kubeconfig', serverUrl: 'https://127.0.0.1:32769') {
                        sh 'kubectl apply -f kubernetes-deployment.yml'
                    }
                    
              
                    
                }
            }
        }
    }
}
