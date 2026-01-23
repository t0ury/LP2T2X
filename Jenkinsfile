pipeline {
  agent any

  environment {
    PATH = '/opt/homebrew/bin:/usr/local/bin:/bin:/usr/bin:/sbin:/usr/sbin'
    IMAGE_NAME = 'lp2t2x-app'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Init') {
      steps {
        script {
          env.GIT_COMMIT = env.GIT_COMMIT.take(7)
          echo "IMAGE_NAME=${env.IMAGE_NAME}, GIT_COMMIT=${env.GIT_COMMIT}"
        }
      }
    }

    stage('Build') {
      steps {
        sh '''
          set -eux
          echo "Building ${IMAGE_NAME}:${GIT_COMMIT} and ${IMAGE_NAME}:latest"
          docker build --progress=plain -t ${IMAGE_NAME}:${GIT_COMMIT} -t ${IMAGE_NAME}:latest .
        '''
      }
    }

    stage('Test') {
      steps {
        sh '''
          set -eux
          docker run --rm ${IMAGE_NAME}:${GIT_COMMIT} python -V
          docker run --rm ${IMAGE_NAME}:${GIT_COMMIT} python manage.py test
        '''
      }
    }

    stage('Code Quality') {
      steps {
        withCredentials([string(credentialsId: 'SONAR_TOKEN', variable: 'SONAR_TOKEN')]) {
          sh '''
            set -eux
            rm -rf sonar-scanner-cli sonar-scanner-*.zip sonar-scanner-*
            curl -sSLo sonar-scanner.zip https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-8.0.1.6346-macosx-aarch64.zip
            unzip -q sonar-scanner.zip
            ./sonar-scanner-*/bin/sonar-scanner -Dsonar.token=$SONAR_TOKEN
          '''
        }
      }
    }

    stage('Security') {
      steps {
        sh '''
          set -eux
          docker run --rm \
            -v /var/run/docker.sock:/var/run/docker.sock \
            -v "$PWD:/work" -w /work \
            aquasec/trivy:0.50.4 \
            image --no-progress --format table --severity HIGH,CRITICAL \
            --exit-code 0 ${IMAGE_NAME}:${GIT_COMMIT} > trivy-report.txt

          head -n 40 trivy-report.txt || true
        '''
      }
      post {
        always {
          archiveArtifacts artifacts: 'trivy-report.txt', allowEmptyArchive: false
        }
      }
    }

    stage('Deploy to Staging') {
      steps {
        sh '''
          set -eux
          export IMAGE_NAME="${IMAGE_NAME}"
          export GIT_COMMIT="${GIT_COMMIT}"

          docker compose -f docker-compose.staging.yml up -d
          sleep 10
          curl -sS http://127.0.0.1:8000 > /dev/null
          echo "Staging healthcheck OK"
        '''
      }
    }

    stage('Approval to Promote') {
      steps {
        input message: 'Promote to production?'
      }
    }

    stage('Deploy to Production') {
      steps {
        sh '''
          set -eux
          export IMAGE_NAME="${IMAGE_NAME}"
          export GIT_COMMIT="${GIT_COMMIT}"
          docker compose -f docker-compose.prod.yml up -d
        '''
      }
    }
  }
}
