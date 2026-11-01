pipeline {
    agent {
        label 'docker'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        IMAGE_NAME = 'jenkins-aws-docker-lab'
        CONTAINER_NAME = 'jenkins-ci-test'
        DEPLOY_USER = 'jenkins'
        DEPLOY_HOST = '44.219.110.233' //elastic ip of ec2 instance
        APP_PORT = '3000'
        APP_NAME = 'node-cicd'
        DOCKERHUB_USERNAME = '354arjun'
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
                      -p 127.0.0.1:${APP_PORT}:${APP_PORT} \
                      ${IMAGE_NAME}:${BUILD_NUMBER}

                    trap 'docker rm -f ${CONTAINER_NAME} >/dev/null 2>&1 || true' EXIT

                    for i in $(seq 1 15); do
                        if curl -fsS http://127.0.0.1:${APP_PORT}/; then
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

        stage('push to docker hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                    sh '''
                        echo "${DOCKER_PASSWORD}" | docker login -u "${DOCKER_USERNAME}" --password-stdin
                        docker tag ${IMAGE_NAME}:${BUILD_NUMBER} ${DOCKER_USERNAME}/${IMAGE_NAME}:${BUILD_NUMBER}
                        docker tag ${IMAGE_NAME}:${BUILD_NUMBER} ${DOCKER_USERNAME}/${IMAGE_NAME}:latest
                        docker push ${DOCKER_USERNAME}/${IMAGE_NAME}:${BUILD_NUMBER}
                        docker push ${DOCKER_USERNAME}/${IMAGE_NAME}:latest
                    '''
                }
            }
        }

        stage('Deploy to AWS EC2'){
            steps{
                sshagent(credentials: ['deployement-instance-ssh']){
                    sh ''' 
                      
                        ssh -o StrictHostKeyChecking=no \
                            -o BatchMode=yes \ 
                            "$DEPLOY_USER@$DEPLOY_HOST" \ 
                            "IMAGE='${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${BUILD_NUMBER}' \
                            PORT='${APP_PORT}' \
                            APP='${APP_NAME}' \
                            bash -s" << 'REMOTE'

                            set -e 

                            echo "Pulling Docker image: $IMAGE"
                            docker pull "$IMAGE"

                            echo "Stopping and removing existing container: $APP"
                            docker rm -f "$APP" 2>/dev/null || true

                            echo "Starting new container: $APP"
                            docker run -d \
                                --name "$APP" \
                                --restart unless-stopped \
                                -p 80:"${PORT}" \
                                "$IMAGE"

                            echo "Deployment completed successfully."

                            docker ps --filter "name=$APP" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

                    REMOTE
                    
                    '''
                }
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