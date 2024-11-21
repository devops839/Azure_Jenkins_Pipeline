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
        ECR_REPO_URI = '123456789012.dkr.ecr.us-east-1.amazonaws.com/boardshack'  // ECR Repository URI
        EKS_CLUSTER_NAME = 'your-eks-cluster'  // EKS Cluster name
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
                withSonarQubeEnv('sonar') {
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
        stage('Publish To Artifactory') {
            steps {
                script {
                    // Ensure that the Artifactory server is configured in Jenkins (as described in Step 2)
                    def server = Artifactory.server 'jfrog'  // Use the Artifactory server ID configured in Jenkins
                    def buildInfo = Artifactory.newBuildInfo()

                    // Deploy the artifact to Artifactory using Maven
                    server.deployArtifacts buildInfo
                }
            }
        }
        // Docker Image Build & Tagging (Multi-stage Docker build)
        stage('Build & Tag Docker Image') {
            steps {
                script {
                    // Build the Docker image using your multi-stage Dockerfile
                    def dockerTag = "${env.ECR_REPO_URI}:${env.IMAGE_TAG}"
                    // Build the Docker image
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
                    trivy image --exit-code 1 --severity HIGH,CRITICAL --format table -o ${outputFile} ${dockerTag}
                    """
                }
            }
        }
        // Authenticate to AWS ECR
        stage('Authenticate to AWS ECR') {
            steps {
                script {
                    // Log in to ECR using AWS CLI (ensure AWS CLI is configured on Jenkins)
                    sh """
                    aws ecr get-login-password --region ${env.AWS_REGION} | docker login --username AWS --password-stdin ${env.ECR_REPO_URI}
                    """
                }
            }
        }
        // Push Docker Image to AWS ECR
        stage('Push Docker Image to ECR') {
            steps {
                script {
                    def dockerTag = "${env.ECR_REPO_URI}:${env.IMAGE_TAG}"
                    // Tag the image for ECR
                    sh "docker tag ${env.IMAGE_TAG} ${dockerTag}"
                    // Push the image to ECR
                    sh "docker push ${dockerTag}"
                }
            }
        }
        // Deploy to EKS
        stage('Deploy to EKS') {
            steps {
                script {
                    // Update kubeconfig for EKS
                    sh """
                    aws eks update-kubeconfig --name ${env.EKS_CLUSTER_NAME} --region ${env.AWS_REGION}
                    """
                    // Replace the image tag dynamically in the deployment YAML and apply it
                    sh """
                    sed -i 's|<your-ecr-repo-uri>/voting-app:latest|${env.ECR_REPO_URI}:${env.IMAGE_TAG}|g' k8s/deployment.yaml
                    kubectl apply -f k8s/deployment.yaml -n webapps
                    kubectl rollout status deployment/voting-app-deployment -n webapps
                    """
                }
            }
        }
        // Verify Deployment on Kubernetes
        stage('Verify the Deployment') {
            steps {
                script {
                    // Check the Kubernetes resources
                    sh "kubectl get pods -n webapps"
                    sh "kubectl get svc -n webapps"
                }
            }
        }
    }
    post {
        always {
            script {
                def subject = ""
                def body = ""
                // Check the build result and customize the email content
                if (currentBuild.currentResult == 'SUCCESS') {
                    subject = "SUCCESS: Jenkins Build ${env.JOB_NAME} #${env.BUILD_NUMBER}"
                    body = """
                    <p>Build Result: SUCCESS</p>
                    <p>Job: ${env.JOB_NAME}</p>
                    <p>Build URL: <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                    <p>Commit: ${env.GIT_COMMIT}</p>
                    """
                } else if (currentBuild.currentResult == 'FAILURE') {
                    subject = "FAILURE: Jenkins Build ${env.JOB_NAME} #${env.BUILD_NUMBER}"
                    body = """
                    <p>Build Result: FAILURE</p>
                    <p>Job: ${env.JOB_NAME}</p>
                    <p>Build URL: <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                    <p>Commit: ${env.GIT_COMMIT}</p>
                    """
                }
                // Send the email
                emailext (
                    to: "${env.EMAIL_RECIPIENTS}",
                    subject: subject,
                    body: body
                )
            }
        }
    }
}
