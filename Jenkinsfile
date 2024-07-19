pipeline {
    agent any
    stages {
        stage('Get Server IP Address') {
            steps {
                script {
                    if (!env.IP_ADDRESS_SERVER) {
                        def ipAddress = sh(script: "curl -s ifconfig.me", returnStdout: true).trim()
                        env.IP_ADDRESS_SERVER = ipAddress
                    } else {
                        println("Geted from parameters")
                    }
                }
            }
        }
        stage('Build MSSQL') {
            steps {
                script {
                    sh "docker build -f Dockerfile.mssql -t manyok007/diplom:mssql.v.${BUILD_NUMBER}.dev --no-cache ."
                }
            }
        }
        stage('Start MSSQL') {
            steps {
                script {
                    sh "docker run -d --name mssql --hostname mssql -p 1433:1433 --restart unless-stopped manyok007/diplom:mssql.v.${BUILD_NUMBER}.dev"
                }
            }
        }
        stage('Check Health - MSSQL') {
            steps {
                script {
                    sleep(100)
                    withCredentials([usernamePassword(credentialsId: 'deb', usernameVariable: 'MSSQL_USER', passwordVariable: 'MSSQL_PASS')]){
                        def mssqlStatus = sh(script: 'docker inspect -f \'{{.State.Status}}\' mssql', returnStdout: true).trim()
                        def mssqlLogs = sh(script: 'docker logs mssql', returnStdout: true).trim()
                        def mssqldb = sh(script: "docker exec mssql /opt/mssql-tools/bin/sqlcmd -S localhost -U '${MSSQL_USER}' -P '${MSSQL_PASS}' -d master -Q 'SELECT name from sys.databases'", returnStdout: true).trim()
                        if (mssqlLogs.contains('Error:')) {
                            if (mssqlStatus == 'running') {
                                println("Warning: MSSQL container is running but contains errors:\n${mssqlLogs}")
                            } else {
                                error "Fatal error found in MSSQL container logs:\n${mssqlLogs}"
                                return
                            }
                        } else if (mssqldb.contains('asdd')) {
                            println("MSSQL container health check: OK")
                        } else {
                            error "Fatal error: database asdd not created! Check logs:\n${mssqlLogs}"
                        }
                    }
                }
            }
        }
        stage('Download Content Backend') {
            steps {
                script {
                    withCredentials([sshUserPrivateKey(credentialsId: 'gitbogdan', keyFileVariable: 'SSH_KEY')]) {
                        if (fileExists('FrontEnd')) {
                            sh 'GIT_SSH_COMMAND="ssh -i $SSH_KEY" git -C BackEnd pull origin BackEnd'
                        } else {
                            sh 'GIT_SSH_COMMAND="ssh -i $SSH_KEY" git clone git@github.com:jj975/project.git -b BackEnd BackEnd'
                        }
                    }
                }
            }
        }
        stage('Prepare Appsettings Backend') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'deb', usernameVariable: 'MSSQL_USER', passwordVariable: 'MSSQL_PASS')]){
                        def settingsTemplate = readFile './BackEnd/Amazon-clone/ShopApi/appsettings.json.j2'
                        settingsTemplate = settingsTemplate.replaceAll('\\{\\{ DB_USER \\}\\}', env.MSSQL_USER)
                        settingsTemplate = settingsTemplate.replaceAll('\\{\\{ DB_PASSWORD \\}\\}', env.MSSQL_PASS)
                        settingsTemplate = settingsTemplate.replaceAll('\\{\\{ IP_ADDRESS_SERVER \\}\\}', env.IP_ADDRESS_SERVER)
                        writeFile file: './BackEnd/Amazon-clone/ShopApi/appsettings.json', text: settingsTemplate
                    }
                }
            }
        }
        stage('Build Backend') {
            steps {
                script {
                    sh "docker build -f Dockerfile.backend -t manyok007/diplom:backend.v.${BUILD_NUMBER}.dev --no-cache ."
                }
            }
        }
        stage('Start Backend') {
            steps {
                script {
                    sh "docker run -d --name backend --hostname backend -p 5034:5034 -p 52119:52119 --restart unless-stopped manyok007/diplom:backend.v.${BUILD_NUMBER}.dev"
                }
            }
        }
        stage('Check Health - Backend') {
            steps {
                script {
                    sleep(60)
                    def backendStatus = sh(script: 'docker inspect -f \'{{.State.Status}}\' backend', returnStdout: true).trim()
                    def backendLogs = sh(script: 'docker logs backend', returnStdout: true).trim()
                    if (backendLogs.contains('error') || backendLogs.contains('Error')) {
                        if (backendStatus == 'running') {
                            println("Warning: Backend container is running but contains errors:\n${backendLogs}")
                        } else {
                            error "Fatal error found in Backend container logs:\n${backendLogs}"
                        }
                    } else {
                        println("Backend container health check: OK")
                    }
                }
            }
        }
        stage('Download Content Frontend') {
            steps {
                script {
                    withCredentials([sshUserPrivateKey(credentialsId: 'gitbogdan', keyFileVariable: 'SSH_KEY')]) {
                        if (fileExists('FrontEnd')) {
                            sh 'GIT_SSH_COMMAND="ssh -i $SSH_KEY" git -C FrontEnd pull origin FrontEnd'
                        } else {
                            sh 'GIT_SSH_COMMAND="ssh -i $SSH_KEY" git clone git@github.com:jj975/project.git -b FrontEnd FrontEnd'
                        }
                    }
                }
            }
        }
        stage('Prepare Appsettings Frontend') {
            steps {
                script {
                    def settingsTemplate = readFile './FrontEnd/my-app/src/api/axios.js.j2'
                    settingsTemplate = settingsTemplate.replaceAll('\\{\\{ IP_ADDRESS_SERVER \\}\\}', env.IP_ADDRESS_SERVER)
                    writeFile file: './FrontEnd/my-app/src/api/axios.js', text: settingsTemplate
                }
            }
        }
        stage('Build Frontend') {
            steps {
                script {
                    sh "docker build -f Dockerfile.frontend -t manyok007/diplom:frontend.v.${BUILD_NUMBER}.dev --no-cache ."
                }
            }
        }
        stage('Start Frontend') {
            steps {
                script {
                    sh "docker run -d --name frontend --hostname frontend -p 81:80 --restart unless-stopped manyok007/diplom:frontend.v.${BUILD_NUMBER}.dev"
                }
            }
        }
        stage('Check Health - Frontend') {
            steps {
                script {
                    sleep(120)
                    def frontendStatus = sh(script: 'docker inspect -f \'{{.State.Status}}\' frontend', returnStdout: true).trim()
                    def frontendLogs = sh(script: 'docker logs frontend', returnStdout: true).trim()
                    if (frontendLogs.contains('error') || frontendLogs.contains('Error')) {
                        if (frontendStatus == 'running') {
                            println("Warning: Frontend container is running but contains errors:\n${frontendLogs}")
                        } else {
                            error "Fatal error found in Frontend container logs:\n${frontendLogs}"
                        }
                    } else {
                        println("Frontend container health check: OK")
                    }
                }
            }
        }
        stage('Push Success Build to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh "docker login -u ${DOCKER_USER} -p ${DOCKER_PASS}"
                        sh "docker stop mssql frontend backend"
                        sh "docker commit mssql manyok007/diplom:mssql.v.${BUILD_NUMBER}.lts"
                        sh "docker push manyok007/diplom:mssql.v.${BUILD_NUMBER}.lts"
                        sh "docker commit backend manyok007/diplom:backend.v.${BUILD_NUMBER}.lts"
                        sh "docker push manyok007/diplom:backend.v.${BUILD_NUMBER}.lts"
                        sh "docker commit frontend manyok007/diplom:frontend.v.${BUILD_NUMBER}.lts"
                        sh "docker push manyok007/diplom:frontend.v.${BUILD_NUMBER}.lts"
                    }
                }
            }
        }
    }
    post {
        always {
            script {
                def containers = ["mssql", "backend", "frontend"]
                containers.each { container ->
                    def containerExists = sh(script: "docker ps -a --format '{{.Names}}' | grep -Eq '^${container}\$'", returnStatus: true) == 0
                    if (containerExists) {
                        sh "docker rm -f ${container}"
                    }
                }
            }
            script {
                sh 'docker image prune -a --filter "until=24h" --force'
            }
        }
    }
}
