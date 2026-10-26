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
        stage('Build Images') {
            steps {
                sh '''
                    set -eu
                    docker build -t laravel-3tier-backend:${BUILD_NUMBER} -f Dockerfile .
                    docker build -t laravel-3tier-frontend:${BUILD_NUMBER} -f nginx/Dockerfile .
                '''
            }
        }
        stage('Test') {
            steps {
                sh '''
                    set -eu
                    docker run --rm -v "$WORKSPACE/app:/src:ro" -w /src php:8.4-cli \
                      sh -ec 'find app bootstrap config database public routes tests -type f -name "*.php" -print0 | xargs -0 -r -n1 php -l'
                '''
            }
        }
        stage('Load Images into Kubernetes') {
            steps {
                sh '''
                    set -eu
                    minikube image load laravel-3tier-backend:${BUILD_NUMBER}
                    minikube image load laravel-3tier-frontend:${BUILD_NUMBER}
                '''
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    set -eu
                    kubectl apply -f k8s/mysql.yaml
                    kubectl rollout status deployment/mysql --timeout=240s
                    kubectl apply -f k8s/backend.yaml
                    kubectl apply -f k8s/frontend.yaml
                    kubectl set image deployment/backend backend=laravel-3tier-backend:${BUILD_NUMBER}
                    kubectl set image deployment/frontend frontend=laravel-3tier-frontend:${BUILD_NUMBER}
                    kubectl rollout status deployment/backend --timeout=240s
                    kubectl rollout status deployment/frontend --timeout=240s
                '''
            }
        }
        stage('Migrate Database') {
            steps {
                sh 'kubectl exec deploy/backend -- php artisan migrate --force'
            }
        }
        stage('Verify') {
            steps {
                sh '''
                    set -eu
                    kubectl get pods
                    kubectl get svc
                    kubectl exec deploy/frontend -- wget -S -qO /dev/null http://127.0.0.1/
                    echo "Laravel HTTP health check: PASS"
                '''
            }
        }
    }
    post {
        success {
            echo 'Laravel deployed on Kubernetes successfully!'
        }
        failure {
            echo 'Pipeline failed. Check Console Output.'
        }
    }
}
