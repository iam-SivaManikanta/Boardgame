pipeline {
    agent { label 'linux' }

    tools {
        maven 'maven3'
        jdk 'JDK17'
    }

    environment {
        SONARQUBE_SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'Github-Token-PAT', url: 'https://github.com/iam-SivaManikanta/Boardgame.git'
            }
        }

        stage('Compile Code') {
            steps {
                sh "mvn compile"
            }
        }

        stage('Test Code') {
            steps {
                sh "mvn test"
            }
        }
    
        stage('Trivy Repo Scan') {
            steps {
                echo "trivy fs --format table -o trivy-report.html . "
            }
        }

        stage('SonarQube Scan') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''${SONARQUBE_SCANNER_HOME}/bin/sonar-scanner \
                    -Dsonar.projectName=Boardgame \
                    -Dsonar.projectKey=Boardgame \
                    -Dsonar.java.binaries=target/classes 
                    '''
             }
            }
        }

        stage('Code Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                waitForQualityGate abortPipeline: true, credentialsId: 'sonar-token'
                }
            }
        }

        stage('Package Code') {
            steps {
                sh "mvn package"
            }
        }

        stage('Artifact Upload') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk17', maven: 'maven3', traceability: true) {
                    sh "mvn deploy"
                }
            }
        }

        stage('Docker Build and Tag') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub_creds', toolName: 'docker') {
                        sh '''
                        docker build -t siva937/boardgame:latest .
                        '''
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                echo "trivy image --format table -o trivy-report.html siva937/boardgame:latest"
            }
        }

        stage('Docker Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub_creds', toolName: 'docker') {
                        sh '''
                        docker push siva937/boardgame:latest
                        '''
                    }
                }
            }
        }

        




}

