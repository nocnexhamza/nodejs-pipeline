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
    runAsUser: 1000
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
      env:
        - name: BUILDKIT_HOST
          value: "tcp://buildkitd:1234"
      volumeMounts:
        - name: workspace-volume
          mountPath: /home/jenkins/agent
    - name: buildkitd
      image: moby/buildkit:latest
      args: ["--addr", "tcp://0.0.0.0:1234"]
      securityContext:
        privileged: true
      volumeMounts:
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
    
    stages {
        stage('Build & Push Image') {
            steps {
                container('jnlp') {
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhublogin',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh '''
                            buildctl build \
                                --frontend dockerfile.v0 \
                                --local context=. \
                                --local dockerfile=. \
                                --output type=image,name=${REGISTRY}/${DOCKER_IMAGE}:${IMAGE_TAG},push=true \
                                --opt registry.username=${DOCKER_USER} \
                                --opt registry.password=${DOCKER_PASS}
                        '''
                    }
                }
            }
        }
        
        stage('Test') {
            steps {
                container('node') {
                    sh '''
                        echo "=== Installing dependencies ==="
                        npm install
                        
                        echo "=== Running tests ==="
                        npm test || { echo "Tests failed"; exit 1; }
                    '''
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                container('kubectl') {
                    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                        sh '''
                            echo "=== Deploying to Kubernetes ==="
                            kubectl apply -f kubernetes-deployment.yml -n ${KUBE_NAMESPACE} || \
                                { echo "Deployment failed"; exit 1; }
                            
                            echo "=== Verifying deployment ==="
                            kubectl get pods -n ${KUBE_NAMESPACE} -l app=nodejs-app
                        '''
                    }
                }
            }
        }
    }
    
    post {
        always {
            echo "Pipeline completed - cleaning up"
        }
        success {
            echo "Pipeline succeeded!"
        }
        failure {
            echo "Pipeline failed"
        }
    }
}
