pipeline {
    agent any  // This defines the agent where the pipeline will run.
    
    tools {
        jdk 'jdk17'  // Specifies the JDK to be used for the build (from the Eclipse Temurin plugin).
        maven 'maven3'  // Specifies the Maven installation to be used.
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'  // Defines the location of SonarQube Scanner.
    }

    stages {
        // Git Checkout Stage
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/jaiswaladi246/Boardgame.git'
            }
        }

        // Compile Stage (Using Maven)
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }

        // Test Stage (Using Maven)
        stage('Test') {
            steps {
                sh "mvn test"
            }
        }

        // File System Scan using Trivy
        stage('File System Scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }

        // SonarQube Analysis Stage
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=BoardGame -Dsonar.projectKey=BoardGame -Dsonar.java.binaries=.'''
                }
            }
        }

        // Quality Gate Stage (Wait for the SonarQube analysis result)
        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }

        // Build Stage (Create package with Maven)
        stage('Build') {
            steps {
                sh "mvn package"
            }
        }

        // Publish to JFrog (Deploy Maven artifacts to a jFrog repository)
        stage('Publish To Artifactory') {
            steps {
                script {
                    // Ensure that the Artifactory server is configured in Jenkins (as described in Step 2)
                    def server = Artifactory.server 'artifactory-server-id'  // Use the Artifactory server ID configured in Jenkins
                    def buildInfo = Artifactory.newBuildInfo()

                    // Deploy the artifact to Artifactory using Maven
                    server.deployArtifacts buildInfo
                }
            }
        }
        // Docker Image Build & Tagging
        stage('Build & Tag Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker build -t adijaiswal/boardshack:latest ."
                    }
                }
            }
        }

        // Docker Image Scan with Trivy
        stage('Docker Image Scan') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html adijaiswal/boardshack:latest"
            }
        }

        // Push Docker Image to Docker Registry
        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker push adijaiswal/boardshack:latest"
                    }
                }
            }
        }

        // Deploy to Kubernetes
        stage('Deploy To Kubernetes') {
            steps {
                withKubeConfig(credentialsId: 'k8-cred', namespace: 'webapps', serverUrl: 'https://172.31.8.146:6443') {
                    sh "kubectl apply -f deployment-service.yaml"
                }
            }
        }

        // Verify Deployment on Kubernetes
        stage('Verify the Deployment') {
            steps {
                withKubeConfig(credentialsId: 'k8-cred', namespace: 'webapps', serverUrl: 'https://172.31.8.146:6443') {
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
                to: 'jaiswaladi246@gmail.com',
                from: 'jenkins@example.com',
                replyTo: 'jenkins@example.com',
                mimeType: 'text/html',
                attachmentsPattern: 'trivy-image-report.html'
            )
        }
    }
}
