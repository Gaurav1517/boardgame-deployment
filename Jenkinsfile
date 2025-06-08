pipeline {
    agent any

    tools {
        jdk 'jdk-17'
        dockerTool 'docker'
        maven 'maven'
    }

    environment {
        SCANNER_HOME = tool "sonar-scanner"
        DOCKER_USERNAME = "gchauhan1517"
    }

    stages {
        stage('Set Job Name Lower') {
            steps {
                script {
                    env.JOB = env.JOB_NAME.toLowerCase()
                }
            }
        }

        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/Gaurav1517/boardgame-deployment.git'
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('Maven Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('File System Scan') {
            steps {
                sh "trivy fs --format table -o trivy-${env.JOB}-fs-report.html ."
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh """
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectName=${env.JOB} \
                        -Dsonar.projectKey=${env.JOB} \
                        -Dsonar.java.binaries=.
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'    
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn package'
            }
        }

        stage('Publish to Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk-17', maven: 'maven', traceability: true) {
                    sh 'mvn deploy'
                }
            }
        }

        stage('Docker Build & Tag') {
            steps {
                script {
                    sh "docker build -t ${env.JOB}:${BUILD_NUMBER} ."
                    sh "docker tag ${env.JOB}:${BUILD_NUMBER} ${DOCKER_USERNAME}/${env.JOB}:v${BUILD_NUMBER}"
                    sh "docker tag ${env.JOB}:${BUILD_NUMBER} ${DOCKER_USERNAME}/${env.JOB}:latest"
                }
            }
        }

        stage('Docker Image Scan') {
            steps {
                sh "trivy image --format table -o trivy-${env.JOB}-image-report.html ${DOCKER_USERNAME}/${env.JOB}:v${BUILD_NUMBER}"
            }
        }

        stage('Docker Image Push') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-creds', passwordVariable: 'docker_pass', usernameVariable: 'docker_user')]) {
                        sh "docker login -u '${docker_user}' -p '${docker_pass}'"
                        sh "docker push ${docker_user}/${env.JOB}:v${BUILD_NUMBER}"
                        sh "docker push ${docker_user}/${env.JOB}:latest"
                    }
                }
            }
        }

        stage('Deploy on Kubernetes') {
            steps {
                withKubeConfig(
                    caCertificate: '',
                    clusterName: 'kubernetes',
                    contextName: '',
                    credentialsId: 'k8s-cred',
                    namespace: 'webapps',
                    restrictKubeConfigAccess: false,
                    serverUrl: 'https://192.168.70.130:6443'
                ) {
                    sh "kubectl apply -f deployment-service.yaml"
                }
            }
        }

        stage('Verify the Deployment') {
            steps {
                withKubeConfig(
                    caCertificate: '',
                    clusterName: 'kubernetes',
                    contextName: '',
                    credentialsId: 'k8s-cred',
                    namespace: 'webapps',
                    restrictKubeConfigAccess: false,
                    serverUrl: 'https://192.168.70.130:6443'
                ) {
                    sh "kubectl get pod -n webapps"
                    sh "kubectl get svc -n webapps"
                }
            }
        }
    }

    post {
        always {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
                def bannerColor = pipelineStatus == 'SUCCESS' ? 'green' : 'red'

                def body = """<html>
                                <body>
                                    <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                                        <h2>${jobName} - Build ${buildNumber}</h2>
                                        <div style="background-color: ${bannerColor}; padding: 10px;">
                                            <h3 style="color: white;">Pipeline Status: ${pipelineStatus}</h3>
                                        </div>
                                        <p>Check the <a href="${env.BUILD_URL}">console output</a> for more details.</p>
                                        <p><strong>Build Summary:</strong></p>
                                        <p>${pipelineStatus == 'SUCCESS' ? 'The build completed successfully!' : 'The build failed. Please check the logs for errors.'}</p>
                                    </div>
                                </body>
                              </html>"""

                echo "Sending email to: gaurav.mau854@gmail.com"
                emailext(
                    subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus}",
                    body: body,
                    to: 'gaurav.mau854@gmail.com',
                    from: 'gaurav.mau854@gmail.com',
                    replyTo: 'gaurav.mau854@gmail.com',
                    mimeType: 'text/html',
                    attachmentsPattern: "trivy-${env.JOB}-*.html"
                )
            }
        }
    }
}
