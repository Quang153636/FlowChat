pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '10'))
  }

  parameters {
    string(
      name: 'DOCKERHUB_NAMESPACE',
      defaultValue: 'hoangquang153636',
      description: 'Docker Hub username or organization'
    )

    string(
      name: 'VITE_API_URL',
      defaultValue: 'http://localhost:5000/api',
      description: 'API URL embedded in the frontend image'
    )

    string(
      name: 'VITE_SOCKET_URL',
      defaultValue: 'http://localhost:5000',
      description: 'Socket.IO URL embedded in the frontend image'
    )

    string(
      name: 'VITE_GOOGLE_WEB_CLIENT_ID',
      defaultValue: '',
      description: 'Optional Google OAuth Web Client ID'
    )
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
          credentialsId: 'dockerhub',
          usernameVariable: 'DOCKERHUB_USERNAME',
          passwordVariable: 'DOCKERHUB_TOKEN'
        )]) {
          sh '''
            echo "$DOCKERHUB_TOKEN" | docker login \
              --username "$DOCKERHUB_USERNAME" \
              --password-stdin

            docker push "$BACKEND_IMAGE:$IMAGE_TAG"
            docker push "$BACKEND_IMAGE:latest"

            docker push "$FRONTEND_IMAGE:$IMAGE_TAG"
            docker push "$FRONTEND_IMAGE:latest"

            docker logout
          '''
        }
      }
    }

    stage('Deploy Staging') {
      steps {
        withCredentials([
          string(
            credentialsId: 'staging-env',
            variable: 'STAGING_ENV'
          )
        ]) {
          sh '''
            printf '%s\\n' "$STAGING_ENV" > .env.staging

            IMAGE_TAG="$IMAGE_TAG" docker compose \
              --env-file .env.staging \
              -f compose.staging.yml \
              up -d

            rm -f .env.staging
          '''
        }
      }
    }

    stage('Staging Health Check') {
      steps {
        sh '''
          echo "Checking staging backend health..."

          curl --fail --retry 10 --retry-delay 3 \
            http://localhost:5000/api/health

          echo ""
          echo "Staging backend is healthy."
        '''
      }
    }
  }
}

