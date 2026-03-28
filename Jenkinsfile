pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_IMAGE = "aniketwakekar/ecommerce:latest"
    }

    stages {

        stage('Git Checkout') {
            steps {
                git 'https://github.com/Wakekar/Ecommerce-App-Kastro.git'
            }
        }

        stage('Maven Compile') {
            steps {
                sh "mvn compile"
            }
        }

        stage('Maven Test') {
            steps {
                sh "mvn test -DskipTests=true"
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
                    sh '''
                    $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=ECommerce \
                    -Dsonar.projectKey=ECommerce \
                    -Dsonar.java.binaries=target/classes
                    '''
                }
            }
        }

        stage('Maven Build') {
            steps {
                sh "mvn package -DskipTests=true"
            }
        }

        stage('Publish to Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'maven-setting', jdk: 'jdk17', maven: 'maven') {
                    sh "mvn deploy -DskipTests=true"
                }
            }
        }

        stage('Docker Build & Tag') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker build -t ${DOCKER_IMAGE} ."
                    }
                }
            }
        }

        stage('Docker Image Scan') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html ${DOCKER_IMAGE}"
                archiveArtifacts artifacts: 'trivy-image-report.html', fingerprint: true
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker push ${DOCKER_IMAGE}"
                    }
                }
            }
        }

        stage('Deploy to Container') {
            steps {
                script {
                    sh "docker stop ecommerce-container || true && docker rm ecommerce-container || true"
                    sh "docker run -d --name ecommerce-container -p 8083:8080 ${DOCKER_IMAGE}"
                }
            }
        }

        // ✅ NOW correctly inside stages
        stage('Deploy To Kubernetes') {
            steps {
                withKubeConfig(
                    clusterName: 'aniket-eks',
                    credentialsId: 'k8-token',
                    namespace: 'webapps',
                    serverUrl: 'https://2BB7BD2C5ADBA2B3087DEB638955CF81.sk1.ap-south-1.eks.amazonaws.com'
                ) {
                    sh "kubectl apply -f deployment-service.yaml -n webapps"
                }
            }
        }

        stage('Verify the Deployment') {
            steps {
                withKubeConfig(
                    clusterName: 'aniket-eks',
                    credentialsId: 'k8-token',
                    namespace: 'webapps',
                    serverUrl: 'https://2BB7BD2C5ADBA2B3087DEB638955CF81.sk1.ap-south-1.eks.amazonaws.com'
                ) {
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
                def pipelineStatus = currentBuild.result ?: 'SUCCESS'
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
                    to: 'aniket241192@gmail.com',
                    mimeType: 'text/html'
                )
            }
        }
    }
}
