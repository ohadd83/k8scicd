pipeline {
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: node
    image: node:18
    command:
    - cat
    tty: true

  - name: docker
    image: docker:24
    command:
    - cat
    tty: true
    env:
    - name: DOCKER_HOST
      value: tcp://localhost:2375

  - name: docker-daemon
    image: docker:24-dind
    securityContext:
      privileged: true
    env:
    - name: DOCKER_TLS_CERTDIR
      value: ""
"""
    }
  }

  stages {
    stage('Install') {
      steps {
        container('node') {
          sh 'npm install'
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

    stage('Build Docker') {
      steps {
        container('docker') {
          sh '''
            echo "Waiting for Docker daemon to become ready..."
            # Loop up to 10 times checking if the daemon is reachable
            for i in {1..10}; do
              if docker info >/dev/null 2>&1; then
                echo "Docker daemon is ready!"
                break
              fi
              echo "Daemon not ready yet, retrying in 3 seconds... ($i/10)"
              sleep 3
            done
            docker build -t simple-app:latest .'''
        }
      }
    }

    stage('Deploy') {
      steps {
        container('docker') {
          sh '''
            # 1. Install kubectl binary on the fly
            echo "Installing kubectl..."
            curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
            chmod +x kubectl
            mv kubectl /usr/local/bin/
            docker stop simple-app || true
            docker rm simple-app || true
            docker run -d -p 8087:3000 --name simple-app simple-app:latest
            
# 2. Dynamically create a NodePort service
            cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: sandbox-nodeport-svc
  namespace: jenkins
spec:
  type: NodePort
  selector:
    jenkins/jenkins-jenkins-agent: "true" 
  ports:
    - protocol: TCP
      port: 8087
      targetPort: 8087
      nodePort: 32000 # Hardcoding a specific port makes firewall rules easier
EOF


            echo "============================================="
            echo "Application is running! Keeping pod alive for 60 seconds..."
            echo "============================================="
            
            sleep 180
          '''
         
        }
      }
    }
  }

  post {
    always {
      deleteDir() 
    }
  }
}
