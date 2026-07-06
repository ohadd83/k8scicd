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
            docker stop simple-app || true
            docker rm simple-app || true
            docker run -d -p 8087:3000 --name simple-app simple-app:latest
            
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
