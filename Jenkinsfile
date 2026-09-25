pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '10'))
  }

  triggers {
    githubPush()
  }

  parameters {
    string(
      name: 'DOCKERHUB_NAMESPACE',
      defaultValue: 'hoangquang153636',
      description: 'Docker Hub username or organization'
    )

    string(
      name: 'VITE_API_URL',
      defaultValue: 'http://192.168.100.168.nip.io:5000/api',
      description: 'API URL embedded in the frontend image'
    )

    string(
      name: 'VITE_SOCKET_URL',
      defaultValue: 'http://192.168.100.168.nip.io:5000',
      description: 'Socket.IO URL embedded in the frontend image'
    )

    string(
      name: 'VITE_GOOGLE_WEB_CLIENT_ID',
      defaultValue: '148026452250-h7gmgrskqvt4nao9r95uoplu4tfc9msh.apps.googleusercontent.com',
      description: 'Optional Google OAuth Web Client ID'
    )
  }

  environment {
    BACKEND_IMAGE = "${params.DOCKERHUB_NAMESPACE}/backend"
    FRONTEND_IMAGE = "${params.DOCKERHUB_NAMESPACE}/frontend"
    IMAGE_TAG = "${BUILD_NUMBER}"

    //Update url stagging and production
    STAGING_FRONTEND_TAG = "${BUILD_NUMBER}-staging"
    PRODUCTION_FRONTEND_TAG = "${BUILD_NUMBER}-production"

    STAGING_API_URL = 'http://192.168.100.168.nip.io:5000/api'
    STAGING_SOCKET_URL = 'http://192.168.100.168.nip.io:5000'

    PRODUCTION_API_URL = 'http://192.168.100.168.nip.io:5001/api'
    PRODUCTION_SOCKET_URL = 'http://192.168.100.168.nip.io:5001'
  }

  stages {
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

    // stage('Build frontend') {
    //   steps {
    //     sh '''
    //       docker build \
    //         --file frontend/Dockerfile \
    //         --build-arg "VITE_API_URL=$VITE_API_URL" \
    //         --build-arg "VITE_SOCKET_URL=$VITE_SOCKET_URL" \
    //         --build-arg "VITE_GOOGLE_WEB_CLIENT_ID=$VITE_GOOGLE_WEB_CLIENT_ID" \
    //         --tag "$FRONTEND_IMAGE:$IMAGE_TAG" \
    //         --tag "$FRONTEND_IMAGE:latest" \
    //         frontend
    //     '''
    //   }
    // }
    stage('Build frontend - Staging') {
      steps {
        sh '''
      docker build \
        --file frontend/Dockerfile \
        --build-arg "VITE_API_URL=$STAGING_API_URL" \
        --build-arg "VITE_SOCKET_URL=$STAGING_SOCKET_URL" \
        --build-arg "VITE_GOOGLE_WEB_CLIENT_ID=$VITE_GOOGLE_WEB_CLIENT_ID" \
        --tag "$FRONTEND_IMAGE:$STAGING_FRONTEND_TAG" \
        frontend
    '''
      }
    }

    stage('Build frontend - Production') {
      steps {
        sh '''
      docker build \
        --file frontend/Dockerfile \
        --build-arg "VITE_API_URL=$PRODUCTION_API_URL" \
        --build-arg "VITE_SOCKET_URL=$PRODUCTION_SOCKET_URL" \
        --build-arg "VITE_GOOGLE_WEB_CLIENT_ID=$VITE_GOOGLE_WEB_CLIENT_ID" \
        --tag "$FRONTEND_IMAGE:$PRODUCTION_FRONTEND_TAG" \
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

            docker push "$FRONTEND_IMAGE:$STAGING_FRONTEND_TAG"
            docker push "$FRONTEND_IMAGE:$PRODUCTION_FRONTEND_TAG"

            docker logout
          '''
        }
      }
    }

    stage('Deploy Staging') {
      steps {
        withCredentials([
          file(
            credentialsId: 'staging-env',
            variable: 'STAGING_ENV_FILE'
          )
        ]) {
          sh '''
            cp "$STAGING_ENV_FILE" .env.staging

            
            IMAGE_TAG="$IMAGE_TAG" \
            FRONTEND_TAG="$STAGING_FRONTEND_TAG" \
            docker compose \
              --project-name flowchat-staging \
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
          set -e

          echo "Checking staging backend health..."

          for i in $(seq 1 20); do
            STATUS=$(docker inspect \
              --format='{{.State.Health.Status}}' \
              flowchat-staging-backend 2>/dev/null || true)

            echo "Attempt $i/20 - Backend health: $STATUS"

            if [ "$STATUS" = "healthy" ]; then
              echo "Staging backend is healthy."
              exit 0
            fi

            sleep 3
          done

          echo "Staging backend did not become healthy."

          echo "===== BACKEND LOGS ====="
          docker logs --tail 100 flowchat-staging-backend

          exit 1
        '''
      }
    }

    stage('Manual Approval') {
      steps {
        input(
          message: 'Staging is healthy. Deploy to production?',
          ok: 'Deploy Production'
        )
      }
    }

    stage('Deploy Production') {
      steps {
        withCredentials([
          file(
            credentialsId: 'production-env',
            variable: 'PRODUCTION_ENV_FILE'
          )
        ]) {
          sh '''
            cp "$PRODUCTION_ENV_FILE" .env.production

            

            IMAGE_TAG="$IMAGE_TAG" \
            FRONTEND_TAG="$PRODUCTION_FRONTEND_TAG" \
            docker compose \
              --project-name flowchat-production \
              --env-file .env.production \
              -f compose.production.yml \
              up -d

            rm -f .env.production
          '''
        }
      }
    }

    stage('Production Health Check') {
      steps {
        sh '''
          set -e

          echo "Checking production backend health..."

          for i in $(seq 1 20); do
            STATUS=$(docker inspect \
              --format='{{.State.Health.Status}}' \
              flowchat-production-backend 2>/dev/null || true)

            echo "Attempt $i/20 - Production backend health: $STATUS"

            if [ "$STATUS" = "healthy" ]; then
              echo "Production backend is healthy."
              exit 0
            fi

            sleep 3
          done

          echo "Production backend did not become healthy."

          echo "===== PRODUCTION BACKEND LOGS ====="
          docker logs --tail 100 flowchat-production-backend

          exit 1
        '''
      }
    }
  }
}
