
pipeline {
    agent any

    environment {
        JAVA_HOME = '/usr/lib/jvm/java-21-openjdk-amd64'
        PATH = "${JAVA_HOME}/bin:${env.PATH}"
        WEBGOAT_HOME = '/home/devops/WebGoat'
        DOCKER_NETWORK = 'jenkins-network'
        SERVER_IP = '192.168.18.215'  // Your server's IP address
    }

    stages {
        stage('Prepare Environment') {
            steps {
                script {
                    sh '''
                        docker network inspect ${DOCKER_NETWORK} || \
                        docker network create ${DOCKER_NETWORK}
                    '''
                }
            }
        }

        stage('Verify Source Code') {
            steps {
                script {
                    sh '''
                        ls -la ${WEBGOAT_HOME}
                        cd ${WEBGOAT_HOME}
                        git status
                        git pull origin develop
                    '''
                }
            }
        }

        stage('Build WebGoat Application') {
            steps {
                script {
                    sh '''
                        cd ${WEBGOAT_HOME}
                        chmod +x mvnw
                        ./mvnw clean package -DskipTests
                    '''
                }
            }
        }

        stage('Static Security Testing (SAST)') {
            parallel {
                stage('Dependency Vulnerability Scan') {
                    steps {
                        script {
                            sh '''
                                docker run --rm \
                                    -v ${WEBGOAT_HOME}:/src \
                                    owasp/dependency-check:latest \
                                    --scan /src \
                                    --format HTML \
                                    --out /src/dependency-check-report
                            '''
                        }
                    }
                }

                stage('Code Quality Scan') {
                    steps {
                        script {
                            sh '''
                                cd ${WEBGOAT_HOME}
                                mvn clean verify sonar:sonar \
                                  -Dsonar.projectKey=webgoat \
                                  -Dsonar.projectName='WebGoat Security Lab' \
                                  -Dsonar.host.url=http://${SERVER_IP}:9000 \
                                  -Dsonar.token=your_sonar_token
                            '''
                        }
                    }
                }
            }
        }

        stage('Containerize Application') {
            steps {
                script {
                    sh '''
                        cd ${WEBGOAT_HOME}
                        docker build -t webgoat-app:${BUILD_NUMBER} .
                        docker run -d --name webgoat-container-${BUILD_NUMBER} \
                            --network ${DOCKER_NETWORK} \
                            -p 8080:8080 \
                            webgoat-app:${BUILD_NUMBER}
                    '''
                }
            }
        }

        stage('Dynamic Security Testing (DAST)') {
            steps {
                script {
                    sh '''
                        docker run --rm \
                            --network ${DOCKER_NETWORK} \
                            -v ${WEBGOAT_HOME}:/zap/wrk:rw \
                            owasp/zap2docker-stable \
                            zap-baseline.py \
                            -t http://${SERVER_IP}:8080 \
                            -r /zap/wrk/zap_scan_report.html
                    '''
                }
            }
        }

        stage('Generate Security Report') {
            steps {
                script {
                    sh '''
                        mkdir -p ${WEBGOAT_HOME}/security-reports
                        cp ${WEBGOAT_HOME}/dependency-check-report/dependency-check-report.html ${WEBGOAT_HOME}/security-reports/
                        cp ${WEBGOAT_HOME}/zap_scan_report.html ${WEBGOAT_HOME}/security-reports/
                    '''
                    archiveArtifacts artifacts: 'security-reports/**/*', allowEmptyArchive: true
                }
            }
        }
    }

    post {
        always {
            script {
                sh '''
                    docker stop webgoat-container-${BUILD_NUMBER} || true
                    docker rm webgoat-container-${BUILD_NUMBER} || true
                    docker rmi webgoat-app:${BUILD_NUMBER} || true
                    docker network rm ${DOCKER_NETWORK} || true
                '''
            }

            publishHTML([
                allowMissing: false,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: 'security-reports',
                reportFiles: 'dependency-check-report.html,zap_scan_report.html',
                reportName: 'Security Scan Reports'
            ])
        }

        failure {
            mail to: 'zeeshan.khaliq@cydea.tech',
                 subject: "Security Scan Failed: ${currentBuild.fullDisplayName}",
                 body: "Security scanning failed for WebGoat. Please check the attached reports."
        }
    }
}
