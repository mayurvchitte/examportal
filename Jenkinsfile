pipeline {
  agent any

  environment {
    BACKEND_IMAGE  = "mahesh1925/examportal-backend"
    FRONTEND_IMAGE = "mahesh1925/examportal-frontend"

    SERVER_HOST = "72.60.219.208"
    SERVER_USER = "ubuntu"

    REPO_URL  = "https://github.com/mayurvchitte/examportal.git"
    BRANCH    = "master"
    IMAGE_TAG = "latest"

    DOCKER_COMPOSE_PROJECT = "examportal"
  }

  stages {
    stage('Checkout') {
      steps {
        echo "📥 Checking out source code..."
        checkout([$class: 'GitSCM',
          userRemoteConfigs: [[url: env.REPO_URL]],
          branches: [[name: "*/${env.BRANCH}"]]
        ])
      }
    }

    stage('Build Backend (Spring Boot)') {
      steps {
        echo "⚙️ Building Spring Boot backend Docker image..."
        sh "docker build -t ${BACKEND_IMAGE}:${IMAGE_TAG} ./examportalspring"
      }
    }

    stage('Build Frontend (Angular)') {
      steps {
        echo "⚙️ Building Angular frontend Docker image..."
        sh "docker build -t ${FRONTEND_IMAGE}:${IMAGE_TAG} ./online-exam-portal"
      }
    }

    stage('Push Images to Docker Hub') {
      steps {
        echo "📤 Pushing Docker images to Docker Hub (with retry logic)..."
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKERHUB_USR', passwordVariable: 'DOCKERHUB_PSW')]) {
          sh '''
            echo "$DOCKERHUB_PSW" | docker login -u "$DOCKERHUB_USR" --password-stdin

            echo "🚀 Pushing backend image..."
            for i in 1 2 3; do
              docker push ${BACKEND_IMAGE}:${IMAGE_TAG} && break || \
              echo "❌ Push attempt $i failed. Retrying in 15 seconds..." && sleep 15
            done

            echo "🚀 Pushing frontend image..."
            for i in 1 2 3; do
              docker push ${FRONTEND_IMAGE}:${IMAGE_TAG} && break || \
              echo "❌ Push attempt $i failed. Retrying in 15 seconds..." && sleep 15
            done
          '''
        }
      }
    }

    stage('Deploy to EC2') {
      steps {
        echo "🚀 Deploying to EC2 (${SERVER_HOST}) ..."
        sshagent(credentials: ['prod-ssh']) {
          withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKERHUB_USR', passwordVariable: 'DOCKERHUB_PSW')]) {
            sh """
              ssh -o StrictHostKeyChecking=no ${SERVER_USER}@${SERVER_HOST} '
                set -e

                echo "🧹 Cleaning old containers..."
                sudo docker compose --project-name ${DOCKER_COMPOSE_PROJECT} down || true

                echo "📂 Ensuring latest repo..."
                cd /opt
                if [ ! -d ${DOCKER_COMPOSE_PROJECT} ]; then
                  git clone ${REPO_URL} ${DOCKER_COMPOSE_PROJECT}
                fi
                cd ${DOCKER_COMPOSE_PROJECT}
                git pull origin ${BRANCH}

                echo "🔑 Docker login..."
                echo ${DOCKERHUB_PSW} | sudo docker login -u ${DOCKERHUB_USR} --password-stdin

                echo "📦 Pulling latest images..."
                sudo docker compose --project-name ${DOCKER_COMPOSE_PROJECT} pull

                echo "🚀 Starting updated stack..."
                sudo docker compose --project-name ${DOCKER_COMPOSE_PROJECT} up -d

                echo "🧼 Cleaning unused images..."
                sudo docker image prune -f

                echo "✅ Deployment successful!"
              '
            """
          }
        }
      }
    }
  }

  post {
    success {
      echo "✅ CI/CD pipeline completed successfully."
    }
    failure {
      echo "❌ CI/CD pipeline failed. Check Jenkins console logs."
    }
    always {
      sh 'docker logout || true'
    }
  }
}
