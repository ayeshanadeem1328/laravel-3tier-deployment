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

        stage('PHP Syntax Test') {
            steps {
                sh '''
                    set -eu
                    docker run --rm \
                      -v "$WORKSPACE/app:/src:ro" \
                      -w /src \
                      php:8.4-cli \
                      sh -ec 'find app bootstrap config database public routes tests -type f -name "*.php" -print0 | xargs -0 -r -n1 php -l'
                '''
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                    set -eu
                    docker build -t laravel-3tier-backend -f Dockerfile .
                    docker build -t laravel-3tier-frontend -f nginx/Dockerfile .
                '''
            }
        }

        stage('Deploy to Ubuntu') {
            steps {
                sh 'sudo -n /usr/local/sbin/deploy-laravel-3tier'
            }
        }

        stage('HTTP Health Check') {
            steps {
                sh '''
                    set -eu
                    for attempt in $(seq 1 15); do
                        if curl -fsS http://127.0.0.1:8081/ >/dev/null; then
                            echo "PASS: Laravel is responding on port 8081"
                            exit 0
                        fi
                        sleep 2
                    done
                    echo "FAIL: Laravel health check failed"
                    exit 1
                '''
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: PHP validation, Docker build, deployment and HTTP check completed.'
        }
        failure {
            echo 'FAILED: Check Console Output for the failed stage.'
        }
    }
}
