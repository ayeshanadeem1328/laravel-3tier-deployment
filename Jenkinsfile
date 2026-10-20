pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    triggers {
        pollSCM('H/2 * * * *')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                echo 'Starting Docker image build...'
                sh '''
                    set -eu
                    docker build -t laravel-3tier-backend -f Dockerfile .
                    docker build -t laravel-3tier-frontend -f nginx/Dockerfile .
                '''
            }
        }

        stage('Test') {
            steps {
                echo 'Running PHP syntax tests...'
                sh '''
                    set -eu
                    docker run --rm \
                      -v "$WORKSPACE/app:/src:ro" \
                      -w /src \
                      php:8.4-cli \
                      sh -ec 'find app bootstrap config database public routes tests -type f -name "*.php" -print0 | xargs -0 -r -n1 php -l'
                '''
                echo 'Build and tests successful!'
            }
        }

        stage('Deploy') {
            steps {
                echo 'Deploying Laravel to Ubuntu...'
                sh 'sudo -n /usr/local/sbin/deploy-laravel-3tier'

                sh '''
                    set -eu
                    for attempt in $(seq 1 15); do
                        if curl -fsS http://127.0.0.1:8081/ >/dev/null; then
                            echo "Laravel HTTP health check: PASS"
                            exit 0
                        fi
                        sleep 2
                    done
                    echo "Laravel HTTP health check: FAIL"
                    exit 1
                '''
            }
        }
    }

    post {
        success {
            echo 'Build and tests successful!'
            echo 'Deployment completed successfully!'
        }
        failure {
            echo 'Build, tests or deployment failed. Check Console Output.'
        }
        always {
            echo 'Pipeline execution finished.'
        }
    }
}
