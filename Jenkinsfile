pipeline {
    agent any

    environment {
        DOCKER_CREDENTIALS_ID = '5dd6653b-26fa-486e-8503-2af4a5605588'
        CONTAINER_NAME_FRONTEND = 'manyok007/diplom'
    }

    stages {
        stage('Login to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: DOCKER_CREDENTIALS_ID, passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                        sh 'docker login --username $DOCKER_USERNAME --password $DOCKER_PASSWORD'
                    }
                }
            }
        }

        stage('Build Frontend Docker Image') {
            steps {
                script {
                    sh 'docker build -t manyok007/diplom:version${BUILD_NUMBER} .'
                }
            }
        }


        stage('Tagging images') {
            steps {
                script {
                    sh 'docker tag manyok007/diplom:version${BUILD_NUMBER} manyok007/diplom:latest'
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    sh 'docker push manyok007/diplom:version${BUILD_NUMBER}'
                    sh 'docker push manyok007/diplom:latest'
                }
            }
        }

        stage('Stop and Remove Old Containers') {
            steps {
                script {
                    def removeContainers = { containerName ->
                        sh """
                        if [ \$(docker ps -aq -f name=${containerName}) ]; then
                            docker stop ${containerName}
                            docker rm ${containerName}
                        else
                            echo "Контейнер ${containerName} не знайдено. Продовжуємо..."
                        fi
                        """
                    }
                    removeContainers(CONTAINER_NAME_FRONTEND)
                    removeContainers(CONTAINER_NAME_BACKEND)
                }
            }
        }

        stage('Run Docker Containers') {
            steps {
                script {
                    sh 'docker run -d -p 8081:80 --name ${CONTAINER_NAME} --health-cmd="curl --fail http://localhost:80 || exit 1" manyok007/diplom:version${BUILD_NUMBER}'
                }
            }
        }
    }
}
