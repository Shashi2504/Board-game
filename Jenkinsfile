pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-credentials', url: 'https://github.com/' // Replace with your git repository link
            }
        }

        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }

        stage('Test cases') {
            steps {
                sh "mvn test"
            }
        }

        stage('File System Scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=BoardGame -Dsonar.project.key=BoardGame \
                            -Dsonar.java.binaries=. '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'SonarQube-token'
                }
            }
        }

        stage('Build Stage') {
            steps {
                sh "mvn package"
            }
        }

        stage('Publish Artifact to Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk17', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                    sh "mvn deploy"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'Docker-credentials', toolName: 'docker') {
                        sh "docker build -t shashikr2912/simple-boardgame:latest ."
                    }
                }
            }
        }

        stage('Image scan') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html shashikr2912/simple-boardgame:lastest"
            }
        }


        stage('Push Docker image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'Docker-credentials', toolName: 'docker') {
                        sh "docker push -t shashikr2912/simple-boardgame:latest"
                    }
                }
            }
        }

        stage('Deployment') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'Kubernetes-cred', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.29.129:6443') {
                    sh "kubectl apply -f deployment.yaml"
                }
            }
        }

        stage('Verifying Deployments') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'Kubernetes-cred', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.29.129:6443') {
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
                to: 'yourmail@gmail.com',
                from: 'jenkins@example.com',
                replyTo: 'jenkins@example.com',
                mimeType: 'text/html',
                attachmentsPattern: 'trivy-image-report.html'
            )
        }
    }
}

}
