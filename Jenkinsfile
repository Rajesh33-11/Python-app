
pipeline {
    agent any

    environment {
        IMAGE_NAME = 'raja3333/python-app'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Validate') {
            steps {
                sh '''
                    set -e
                    test -f Dockerfile
                    test -f app.py
                    docker --version
                '''
            }
        }

        stage('Build') {
            steps {
                sh '''
                    docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} .
                    docker tag ${IMAGE_NAME}:${BUILD_NUMBER} ${IMAGE_NAME}:latest
                '''
            }
        }

        stage('Smoke Test') {
            steps {
                sh '''
                    set -e
                    docker rm -f python-app-smoke 2>/dev/null || true
                    docker run -d --name python-app-smoke -p 127.0.0.1:18080:8080 ${IMAGE_NAME}:${BUILD_NUMBER}

                    for i in $(seq 1 15); do
                        if curl -fsS http://127.0.0.1:18080/; then
                            echo "Smoke test passed"
                            docker rm -f python-app-smoke
                            exit 0
                        fi
                        sleep 2
                    done

                    docker logs python-app-smoke
                    docker rm -f python-app-smoke
                    exit 1
                '''
            }
        }

        stage('Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKERHUB_USER',
                    passwordVariable: 'DOCKERHUB_TOKEN'
                )]) {
                    sh '''
                        set +x
                        echo "$DOCKERHUB_TOKEN" | docker login \
                            --username "$DOCKERHUB_USER" \
                            --password-stdin

                        docker push ${IMAGE_NAME}:${BUILD_NUMBER}
                        docker push ${IMAGE_NAME}:latest
                        docker logout
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'docker rm -f python-app-smoke 2>/dev/null || true'
        }
        success {
            echo 'Python image built, smoke-tested, and pushed successfully.'
        }
        failure {
            echo 'Pipeline failed. Check the stage Console Output.'
        }
    }
}
