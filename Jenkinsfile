pipeline {
    environment {
        DOCKER_IMAGE_NAME = "kimbobu/sample"
        DOCKER_CREDENTIALS_ID = "dockerhub-credentials"
        KUBE_CREDENTIALS_ID = "KUBECONFIG"
        KUBE_SERVER_URL = "https://kubernetes.default"
    }

    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: jnlp
    image: sheayun/jnlp-agent-sample
    env:
    - name: DOCKER_HOST
      value: "tcp://127.0.0.1:2375"

  - name: dind
    image: docker:24-dind
    args:
    - "dockerd"
    - "-H"
    - "tcp://0.0.0.0:2375"
    - "-H"
    - "unix:///var/run/docker.sock"
    env:
    - name: DOCKER_TLS_CERTDIR
      value: ""
    securityContext:
      privileged: true
            '''
        }
    }

    stages {
        stage('Git SCM Update') {
            steps {
                 checkout scm            }
        }

        stage('Check Docker Daemon') {
            steps {
                script {
                    echo "Checking Docker Daemon..."
                    sh "curl -s http://127.0.0.1:2375/_ping || echo 'Docker Daemon not running'"
                }
            }
        }

        stage('Docker Build and Push') {
            steps {
                script {
                    echo "Building Docker image..."
                    dockerImage = docker.build("${DOCKER_IMAGE_NAME}")

                    echo "Pushing Docker image..."
                    docker.withRegistry('https://registry.hub.docker.com', "${DOCKER_CREDENTIALS_ID}") {
                        dockerImage.push('latest')
                    }
                }
            }
        }

        stage('Deploy on Kubernetes') {
            steps {
                echo "Deploying to Kubernetes cluster..."
                withKubeConfig([credentialsId: "${KUBE_CREDENTIALS_ID}", serverUrl: "${KUBE_SERVER_URL}", namespace: 'default']) {
                    sh '''
                    kubectl apply -f deployment.yaml || true
                    kubectl apply -f service.yaml || true
                    '''
                }
            }
        }
    }

    post {
        always {
            echo "Cleaning up Docker session..."
            sh "docker logout || echo 'Docker logout failed, but continuing...'"
        }
    }
}
