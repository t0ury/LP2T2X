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

        // Stage 1: Build
        stage('Build') {
      steps {
          script {
            env.GIT_COMMIT_SHORT = env.GIT_COMMIT.take(7)
          }
          sh '''
                    docker build -t ${IMAGE_NAME}:${GIT_COMMIT_SHORT} -t ${IMAGE_NAME}:latest .
                '''
      }
        }

        // Stage 2: Test
        stage('Test') {
      steps {
          sh '''
                    set -eux
                    docker run --rm ${IMAGE_NAME}:${GIT_COMMIT_SHORT} python -V
                    docker run --rm ${IMAGE_NAME}:${GIT_COMMIT_SHORT} python manage.py test
                '''
      }
        }

        // Stage 3: Code Quality
        stage('Code Quality') {
      steps {
          withCredentials([string(credentialsId: 'SONAR_TOKEN', variable: 'SONAR_TOKEN')]) {
            sh '''
                        set -eux
                        rm -rf sonar-scanner-*
                        curl -sSLo sonar-scanner.zip https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-8.0.1.6346-macosx-aarch64.zip
                        unzip -q sonar-scanner.zip
                        ./sonar-scanner-*/bin/sonar-scanner -Dsonar.token=$SONAR_TOKEN
                    '''
          }
      }
        }

        // Stage 4: Security
        stage('Security') {
      steps {
          sh '''
                    set -eux
                    docker run --rm \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy:0.50.4 \
                        image --severity HIGH,CRITICAL ${IMAGE_NAME}:${GIT_COMMIT_SHORT}
                '''
      }
        }

        // Stage 5: Deploy to Staging
        stage('Deploy to Staging') {
      steps {
          sh '''
                    set -eux
                    echo "Deploying to staging environment..."

                    export GIT_COMMIT="${GIT_COMMIT_SHORT}"

                    # Stop existing staging containers
                    docker compose -f docker-compose.staging.yml down --remove-orphans || true

                    # Deploy to staging
                    docker compose -f docker-compose.staging.yml up -d

                    # Wait for application to start
                    sleep 10

                    # Health check
                    curl -sS http://127.0.0.1:8000 || true
                    echo "Staging deployment completed"
                '''
      }
        }

        // Stage 6: Release to Production
        stage('Release') {
          steps {
            sh """
                set -eux

                # Create release tag
                git config user.name "jenkins"
                git config user.email "jenkins@local"
                git tag -a release-${GIT_COMMIT_SHORT} -m "Release ${GIT_COMMIT_SHORT}" || true
                git push origin release-${GIT_COMMIT_SHORT} || true

                # Deploy to production
                export GIT_COMMIT="${GIT_COMMIT_SHORT}"
                docker compose -f docker-compose.prod.yml down --remove-orphans || true
                docker compose -f docker-compose.prod.yml up -d

                sleep 10
                curl -sS http://127.0.0.1:8080 || true
                echo "Production deployment completed"
              """
      }
        }

        // Stage 7: Monitoring
        stage('Monitoring') {
      steps {
          sh '''
                    set -eux
                    # Start Prometheus
                    docker compose -f docker-compose.monitoring.yml down --remove-orphans || true
                    docker compose -f docker-compose.monitoring.yml up -d
                    sleep 5

                    # Verify Prometheus is healthy
                    curl -sS http://127.0.0.1:9090/-/healthy

                    # Show configured alert rules
                    echo "=== CONFIGURED ALERT RULES ==="
                    curl -sS http://127.0.0.1:9090/api/v1/rules | grep -o '"alertname":"[^"]*"' || true

                    # Show targets status
                    echo "=== MONITORING TARGETS ==="
                    curl -sS http://127.0.0.1:9090/api/v1/targets | grep -o '"health":"[^"]*"' || true

                    echo ""
                    echo "Prometheus Dashboard: http://localhost:9090"
                    echo "Alert Rules: http://localhost:9090/alerts"
                '''
      }
        }
    }

    post {
        always {
      echo "Pipeline finished: ${currentBuild.currentResult}"
        }
    }
}
