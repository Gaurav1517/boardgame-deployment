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
        // stage('Cleanup Workspace') {
        //     steps {
        //         cleanWs()
        //     }
        // }

        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/Gaurav1517/java-maven-webapp.git'
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('Maven test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('File System Scan') {
            steps {
                sh 'trivy fs --format table -o trivy-fs-report.html .'
            }
        }

        stage('SonarQube Analaysis') {
            steps {
                withSonarQubeEnv('sonar-token') {
                    sh  ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=BoardGame -Dsonar.projectKey=BoardGame \
                             -Dsonar.java.binaries=. '''
                }
            }
        }
        stage('Quality Gate'){
            steps{
                script{
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }
        stage('Build'){
            steps{
                sh "mvn package"
            }
        }
        stage('Publish to nexus'){
            steps{
            
            }
        }  
        
    }
}