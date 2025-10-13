// Jenkinsfile (Declarative Pipeline, Groovy)
pipeline {
  agent { label 'j_worker' }
  options {
    disableConcurrentBuilds()
    timestamps()
  }

  parameters {
    // за бажанням змініть параметри під свій репо/тести
    string(name: 'IMAGE_NAME', defaultValue: 'nodeapp')
    string(name: 'DOCKERHUB_REPO', defaultValue: 'olka2/nodeapp')
    string(name: 'IMAGE_TAG', defaultValue: '')
    string(name: 'TEST_CMD', defaultValue: 'pytest -q')
    string(name: 'BUILD_CONTEXT', defaultValue: '.')
    string(name: 'DOCKERFILE', defaultValue: 'Dockerfile')
  }

  environment {
    DOCKERHUB_CREDENTIALS_ID = 'dockerhub-creds'
    RESOLVED_TAG = "${params.IMAGE_TAG ?: (env.GIT_COMMIT ? env.GIT_COMMIT.take(7) : env.BUILD_NUMBER)}"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        sh 'git rev-parse --short HEAD || true'
      }
    }

    stage('Build Docker image') {
      steps {
        sh """
          set -eux
          docker build -f '${params.DOCKERFILE}' -t '${params.IMAGE_NAME}:${RESOLVED_TAG}' '${BUILD_CONTEXT}'
        """
      }
    }

    stage('Run tests in container') {
      steps {
        script {
          // Запускаємо контейнер з тестами; в разі ненульового коду — stage впаде
          sh """
            set -eux
            docker run --rm \\
              -e CI=true \\
              '${params.IMAGE_NAME}:${RESOLVED_TAG}' \\
              sh -lc "${params.TEST_CMD}"
          """
        }
      }
    }

    stage('Login & Push to Docker Hub') {
      when { expression { currentBuild.currentResult == null || currentBuild.currentResult == 'SUCCESS' } }
      steps {
        script {
          // Логін через Jenkins credentials і пуш у Docker Hub
          withCredentials([usernamePassword(credentialsId: env.DOCKERHUB_CREDENTIALS_ID,
                                           usernameVariable: 'DOCKERHUB_USER',
                                           passwordVariable: 'DOCKERHUB_PASS')]) {
            sh """
              set -eux
              echo "\$DOCKERHUB_PASS" | docker login -u "\$DOCKERHUB_USER" --password-stdin
              # Перетегнути локальний імедж у формат user/repo:tag
              docker tag '${params.IMAGE_NAME}:${RESOLVED_TAG}' '${params.DOCKERHUB_REPO}:${RESOLVED_TAG}'
              docker push '${params.DOCKERHUB_REPO}:${RESOLVED_TAG}'

              # Додатково пушимо 'latest' (необов’язково; приберіть, якщо не потрібно)
              docker tag '${params.IMAGE_NAME}:${RESOLVED_TAG}' '${params.DOCKERHUB_REPO}:latest'
              docker push '${params.DOCKERHUB_REPO}:latest'

              docker logout
            """
          }
        }
      }
    }
  }

  post {
    failure {
      echo 'Tests failed'
    }
    always {
      // опційне прибирання локальних шарів/контейнерів (обережно в CI)
      sh """
        docker ps -aq --filter "status=exited" | xargs -r docker rm -v || true
        docker images --filter "dangling=true" -q | xargs -r docker rmi || true
      """
    }
  }
}
