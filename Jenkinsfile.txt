pipeline {
    agent any

    environment {
        DOCKER_CREDENTIALS_ID = '5dd6653b-26fa-486e-8503-2af4a5605588'
        CONTAINER_NAME_FRONTEND = 'manyok007/frontend'
        CONTAINER_NAME_BACKEND = 'manyok007/beck'
        CONTAINER_NAME_DB = 'manyok007/mssql-server'
    }

    stages {
        stage('Login to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: DOCKER_CREDENTIALS_ID, passwordVariable: 'slon68766', usernameVariable: 'manyok007')]) {
                        sh 'docker login --username $DOCKER_USERNAME --password $DOCKER_PASSWORD'
                    }
                }
            }
        }

        stage('Build Frontend Docker Image') {
            steps {
                script {
                    sh 'cd /home/ubuntu/diplom/FrontEnd/my-app'
                    sh 'docker build -t manyok007/frontend:version${BUILD_NUMBER} .'
                    sh 'docker push manyok007/frontend:version${BUILD_NUMBER}'
                }
            }
        }

        stage('Build Backend Docker Image') {
            steps {
                script {
                    sh 'cd /home/ubuntu/diplom/BackEnd/Amazone-clone'
                    sh 'docker build -t manyok007/beck:version${BUILD_NUMBER} .'
                    sh 'docker push manyok007/beck:version${BUILD_NUMBER}'
                }
            }
        }

        stage('Tagging images') {
            steps {
                script {
                    sh 'docker tag manyok007/frontend:version${BUILD_NUMBER} manyok007/frontend:latest'
                    sh 'docker tag manyok007/beck:version${BUILD_NUMBER} manyok007/beck:latest'
                }
            }
        }

        stage('Push Backend Docker Image') {
            steps {
                script {
                    sh 'docker push manyok007/beck:version${BUILD_NUMBER}'
                    sh 'docker push manyok007/beck:latest'
                }
            }
        }

        stage('Push Frontend Docker Image') {
            steps {
                script {
                    sh 'docker push manyok007/frontend:version${BUILD_NUMBER}'
                    sh 'docker push manyok007/frontend:latest'
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
                    sh 'docker run -d -p 8080:80 --name manyok007/frontend manyok007/frontend:version${BUILD_NUMBER}'
                    sh 'docker run -d -p 5034:5034 --name manyok007/beck manyok007/beck:version${BUILD_NUMBER}'
                    sh 'docker run -e "ACCEPT_EULA=Y" -e "MYSQL_SA_PASSWORD=Qwerty-1" -p 1433:1433 --name sql111 --hostname sql1 -d manyok007/mssql-server:version${BUILD_NUMBER}'
                }
            }
        }
    }
}
