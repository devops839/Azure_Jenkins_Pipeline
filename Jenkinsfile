pipeline {
    agent any  // This defines the agent where the pipeline will run.
    tools {
        jdk 'jdk17'  // Specifies the JDK to be used for the build (from the Eclipse Temurin plugin).
        maven 'maven3'  // Specifies the Maven installation to be used.
    }
    environment {
        SCANNER_HOME = tool 'sonar_scanner'  // Defines the location of SonarQube Scanner.
        EMAIL_RECIPIENTS = 'pavank839@outlook.com'
        AWS_REGION = 'us-west-2'  // Your AWS region
        ECR_REPO_URI = '481665128974.dkr.ecr.us-west-2.amazonaws.com/app'  // ECR Repository URI
        EKS_CLUSTER_NAME = 'poc-eks'  // EKS Cluster name
        IMAGE_TAG = "${env.BUILD_NUMBER}"  // Tag Docker image with Jenkins build number
    }
    stages {
        // Git Checkout Stage
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'github_cred', url: 'https://github.com/devops839/vote-app-springboot.git'
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
        // SonarQube Analysis Stage
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh '''$SCANNER_HOME/bin/sonar_scanner -Dsonar.projectName=voting-app -Dsonar.projectKey=voting-app -Dsonar.java.binaries=.'''
                }
            }
        }
        // Quality Gate Stage (Wait for the SonarQube analysis result)
        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonarqube_token'
                }
            }
        }
        // Build Stage (Create package with Maven)
        stage('Build') {
            steps {
                sh "mvn package -DskipTests"  // Skip tests if they're already run earlier
            }
        }
        // Publish to JFrog (Deploy Maven artifacts to a JFrog repository)
        stage('Publish to Artifactory') {
            steps {
                script {
                    def server = Artifactory.server 'jfrog'
                    def uploadSpec = """{
                        "files": [
                            {
                                "pattern": "target/*.jar",
                                "target": "example-repo-local/votingapp/${env.BUILD_ID}/"
                            }
                        ]
                    }"""
                    echo "Upload Spec: ${uploadSpec}"
                    def buildInfo = server.upload spec: uploadSpec
                    server.publishBuildInfo buildInfo
                }
            }
        }
        // Docker Image Build & Tagging (Multi-stage Docker build)
        stage('Build & Tag Docker Image') {
            steps {
                script {
                    def dockerTag = "${env.ECR_REPO_URI}:${env.IMAGE_TAG}"
                    sh """
                    docker build -t ${dockerTag} .
                    """
                }
            }
        }
        // **Trivy Docker Image Scan** (Add Trivy scan for the built Docker image)
        stage('Trivy Docker Image Scan') {
            steps {
                script {
                    // Run Trivy to scan the Docker image for HIGH and CRITICAL vulnerabilities
                    def dockerTag = "${env.ECR_REPO_URI}:${env.IMAGE_TAG}"
                    def outputFile = "trivy-image-report-${env.IMAGE_TAG}.txt"  // Output file in table format (text file)
                    sh """
                    trivy image --severity HIGH,CRITICAL --format table -o ${outputFile} ${dockerTag}
                    """
                }
            }
        }
        // Docker Image Build 
/*        stage('Docker Build') {
            steps {
                script {
                    // Build the Docker image
                    sh '''
                    docker build -t pavan539/voting-app-java .
                    docker tag pavan539/voting-app-java pavan539/voting-app-java:${IMAGE_TAG}
                    '''
                }
            }
        }

        // Trivy Docker Image Scan
        stage('Trivy Docker Image Scan') {
            steps {
                script {
                    // Ensure Docker image is tagged with correct name
                    def dockerTag = "pavan539/voting-app-java:${env.IMAGE_TAG}"
                    def outputFile = "trivy-image-report-${env.IMAGE_TAG}.txt"  // Output file in table format (text file)
                    // Scan the image for vulnerabilities
                    sh """
                    trivy image --severity HIGH,CRITICAL --format table -o ${outputFile} ${dockerTag}
                    """
                    // Check if the scan was successful (optional)
                    def trivyScanResults = readFile(outputFile)
                    echo "Trivy Scan Results: ${trivyScanResults}"
                }
            }
        }
        // Docker Push (only if Trivy scan passes)
        stage('Push To DockerHub') {
            steps {
                script {
                    // Ensure the image tag is correctly set
                    def dockerTag = "pavan539/voting-app-java:${env.IMAGE_TAG}"
                    
                    // Push the image to the Docker registry (e.g., Docker Hub or ECR)
                    withDockerRegistry(credentialsId: 'dockerhub_cred', toolName: 'docker') {
                        // Push the tagged image to the registry
                        sh "docker push ${dockerTag}"
                    }
                }
            }
        } */
        // Authenticate & Push Image to ECR
        stage('Authenticate and Push Docker Image to ECR') {
            steps {
                script {
                    sh """
                    aws ecr get-login-password --region ${env.AWS_REGION} | docker login --username AWS --password-stdin ${env.ECR_REPO_URI}
                    """
                    def dockerTag = "${env.ECR_REPO_URI}:${env.IMAGE_TAG}"
                    sh "docker images"
                    sh "docker tag ${dockerTag} ${dockerTag}"
                    sh "docker push ${dockerTag}"
                }
            }
        }
        // Deploy to K8s
        stage('K8S Deploy') {
            steps {
                script {
                    withAWS(credentials: 'aws_cred', region: "${env.AWS_REGION}") {
                        sh "aws eks update-kubeconfig --name ${env.EKS_CLUSTER_NAME} --region ${env.AWS_REGION}"
                        sh """
                        sed 's/\${BUILD_NUMBER}/${env.BUILD_NUMBER}/g' k8s/vote_deploy.yaml > k8s/deployment-new-build.yaml
                        kubectl apply -f k8s/deployment-new-build.yaml
                        """
                    }
                }
            }
        }
    }
}
