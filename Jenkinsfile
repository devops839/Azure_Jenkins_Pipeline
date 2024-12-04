pipeline {
    agent any  // This defines the agent where the pipeline will run.
    tools {
        jdk 'jdk17'  // Specifies the JDK to be used for the build (from the Eclipse Temurin plugin).
        maven 'maven3'  // Specifies the Maven installation to be used.
    }
    environment {
        SCANNER_HOME = tool 'sonar_scanner'  // Defines the location of SonarQube Scanner.
        EMAIL_RECIPIENTS = 'pavank839@outlook.com'
        AZURE_REGION = 'eastus'  // Your Azure region
        ACR_REPO_URI = 'demoacr839.azurecr.io/app'  // Azure Container Registry URI
        AKS_CLUSTER_NAME = 'demo-aks'  // Azure Kubernetes Service (AKS) Cluster name
        IMAGE_TAG = "${env.BUILD_NUMBER}"  // Tag Docker image with Jenkins build number
    }
    stages {
        // Git Checkout Stage
        stage('Git Checkout') {
            when {
                branch 'azure_jenkins'  // Run only when the current branch is 'azure_jenkins'
            }
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
        // Publish to Artifactory (Deploy Maven artifacts to a JFrog repository)
        stage('Publish to Artifactory') {
            steps {
                script {
                    def server = Artifactory.server 'jfrog'
                    def uploadSpec = """{
                        "files": [
                            {
                                "pattern": "target/*.jar",
                                "target": "libs-release-local//votingapp/${env.BUILD_ID}/"
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
                    def dockerTag = "${env.ACR_REPO_URI}:${env.IMAGE_TAG}"
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
                    def dockerTag = "${env.ACR_REPO_URI}:${env.IMAGE_TAG}"
                    def outputFile = "trivy-image-report-${env.IMAGE_TAG}.txt"  // Output file in table format (text file)
                    sh """
                    trivy image --severity HIGH,CRITICAL --format table -o ${outputFile} ${dockerTag}
                    """
                }
            }
        }
        // Authenticate & Push Image to Azure Container Registry (ACR)
        stage('Authenticate and Push Docker Image to ACR') {
            steps {
                script {
                    // Login to Azure using a service principal
                    sh """
                    az acr login --name demoacr839
                    """
                    def dockerTag = "${env.ACR_REPO_URI}:${env.IMAGE_TAG}"
                    sh "docker tag ${dockerTag} ${dockerTag}"
                    sh "docker push ${dockerTag}"
                }
            }
        }
        // Deploy to AKS
        stage('K8S Deploy') {
            steps {
                script {
                    // Set up kubectl to use Azure's AKS
                    sh """
                    az aks get-credentials --resource-group demo-aks-rg --name ${env.AKS_CLUSTER_NAME}
                    """
                    sh """
                    sed 's/\${BUILD_NUMBER}/${env.BUILD_NUMBER}/g' k8s/vote_deploy.yaml > k8s/deployment-new-build.yaml
                    kubectl apply -f k8s/deployment-new-build.yaml
                    """
                }
            }
        }
    }
}
