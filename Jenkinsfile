pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '10'))
  }

  parameters {
    string(name: 'DOCKERHUB_NAMESPACE', defaultValue: 'hoangquang153636', description: 'Docker Hub username or organization')
    string(name: 'VITE_API_URL', defaultValue: 'http://localhost:5000/api', description: 'API URL embedded in the frontend image')
    string(name: 'VITE_SOCKET_URL', defaultValue: 'http://localhost:5000', description: 'Socket.IO URL embedded in the frontend image')
    string(name: 'VITE_GOOGLE_WEB_CLIENT_ID', defaultValue: '', description: 'Optional Google OAuth Web Client ID')
  }

  environment {
    BACKEND_IMAGE = "${params.DOCKERHUB_NAMESPACE}/backend"
    FRONTEND_IMAGE = "${params.DOCKERHUB_NAMESPACE}/frontend"
    IMAGE_TAG = "${BUILD_NUMBER}"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build backend') {
      steps {
        sh '''
          docker build \
            --file backend/Dockerfile \
            --tag "$BACKEND_IMAGE:$IMAGE_TAG" \
            --tag "$BACKEND_IMAGE:latest" \
            backend
        '''
      }
    }

    stage('Build frontend') {
      steps {
        sh '''
          docker build \
            --file frontend/Dockerfile \
            --build-arg "VITE_API_URL=$VITE_API_URL" \
            --build-arg "VITE_SOCKET_URL=$VITE_SOCKET_URL" \
            --build-arg "VITE_GOOGLE_WEB_CLIENT_ID=$VITE_GOOGLE_WEB_CLIENT_ID" \
            --tag "$FRONTEND_IMAGE:$IMAGE_TAG" \
            --tag "$FRONTEND_IMAGE:latest" \
            frontend
        '''
      }
    }

    stage('Docker Hub') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub-credentials',
          usernameVariable: 'DOCKERHUB_USERNAME',
          passwordVariable: 'DOCKERHUB_TOKEN'
        )]) {
          sh '''
            echo "$DOCKERHUB_TOKEN" | docker login --username "$DOCKERHUB_USERNAME" --password-stdin
            docker push "$BACKEND_IMAGE:$IMAGE_TAG"
            docker push "$BACKEND_IMAGE:latest"
            docker push "$FRONTEND_IMAGE:$IMAGE_TAG"
            docker push "$FRONTEND_IMAGE:latest"
            docker logout
          '''
        }
      }
    }
  }
}
