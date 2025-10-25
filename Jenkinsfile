pipeline {
    agent { label 'linux' }

    tools {
        maven 'maven3'
        jdk 'jdk17'
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
                        docker build -t boardgame .
                        docker tag boardgame siva937/boardgame:latest
                        '''
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                echo "trivy image --format table -o trivy-image-report.html siva937/boardgame:latest"
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

        stage('Deploy to Kubernetes') {
            steps {
                withKubeCredentials(kubectlCredentials: [[caCertificate: '', clusterName: 'lke527832', contextName: '', credentialsId: 'k8-cred', namespace: 'webapps', serverUrl: 'https://518ce15f-0f44-4434-aed2-ffb0f1d7c3ef.ap-west-1-gw.linodelke.net:443']]) {

                    sh "kubectl apply -f deployment-service.yaml" 
                }
            }
        }

        stage('verify Kubernetes') {
            steps {
                withKubeCredentials(kubectlCredentials: [[caCertificate: '', clusterName: 'lke527832', contextName: '', credentialsId: 'k8-cred', namespace: 'webapps', serverUrl: 'https://518ce15f-0f44-4434-aed2-ffb0f1d7c3ef.ap-west-1-gw.linodelke.net:443']]) {

                     sh "kubectl get pods -n webapps"
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
            def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'

            def body = """
                <html>
                <body>
                <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                <h2>${jobName} - Build ${buildNumber}</h2>
                <div style="background-color: ${bannerColor}; padding: 10px;">
                <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                </div>
                <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                </div>
                </body>
                </html>
            """

            emailext (
                subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                body: body,
                to: 'smanikanta937@gmail.com',
                from: 'jenkins@example.com',
                replyTo: 'jenkins@example.com',
                mimeType: 'text/html',
                attachmentsPattern: 'trivy-image-report.html'
            )
            }
        }
    }
}

   


