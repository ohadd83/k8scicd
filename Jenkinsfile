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
          sh 'docker build -t simple-app:latest .'
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
            
            sleep 60
          '''
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
