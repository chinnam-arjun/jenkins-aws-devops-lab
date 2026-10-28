pipeline {
    agent {
        label 'aws-docker-agent'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        IMAGE_NAME = 'jenkins-aws-docker-lab'
        CONTAINER_NAME = 'jenkins-ci-test'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Verify Agent') {
            steps {
                sh '''
                    echo "Running on: $(hostname)"
                    whoami
                    git --version
                    docker --version
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build \
                      -t ${IMAGE_NAME}:${BUILD_NUMBER} \
                      -t ${IMAGE_NAME}:latest .
                '''
            }
        }

        stage('Run Container Test') {
            steps {
                sh '''
                    set -eu

                    docker rm -f ${CONTAINER_NAME} 2>/dev/null || true

                    docker run -d \
                      --name ${CONTAINER_NAME} \
                      -p 127.0.0.1:3000:3000 \
                      ${IMAGE_NAME}:${BUILD_NUMBER}

                    trap 'docker rm -f ${CONTAINER_NAME} >/dev/null 2>&1 || true' EXIT

                    for i in $(seq 1 15); do
                        if curl -fsS http://127.0.0.1:3000/; then
                            echo "Application health check passed"
                            exit 0
                        fi

                        sleep 2
                    done

                    echo "Application health check failed"
                    docker logs ${CONTAINER_NAME}
                    exit 1
                '''
            }
        }
    }

    post {
        always {
            sh '''
                docker rm -f ${CONTAINER_NAME} 2>/dev/null || true
            '''
        }

        success {
            echo 'Docker CI pipeline completed successfully.'
        }

        failure {
            echo 'Docker CI pipeline failed. Review the console output.'
        }
    }
}