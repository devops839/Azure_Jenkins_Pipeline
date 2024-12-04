pipeline {
    agent any  // This defines the agent where the pipeline will run.

    tools {
        jdk 'jdk17'  // Specifies the JDK to be used for the build (from the Eclipse Temurin plugin).
        maven 'maven3'  // Specifies the Maven installation to be used.
    }

    environment {
        SCANNER_HOME = tool 'sonarqube_scanner'  // Defines the location of SonarQube Scanner.
        EMAIL_RECIPIENTS = 'pavankalluri.14533@gmail.com'
        AZURE_REGION = 'eastus'  // Your Azure region
        ACR_REPO_URI = 'demoacr839.azurecr.io/app'  // Azure Container Registry URI
        AKS_CLUSTER_NAME = 'demo-aks'  // Azure Kubernetes Service (AKS) Cluster name
        IMAGE_TAG = "${env.BUILD_NUMBER}"  // Tag Docker image with Jenkins build number
        AZURE_SP_CRED = credentials('az_sp_id_password')  // Service Principal ID and Secret (from Jenkins credentials)
        AZURE_TENANT_ID = credentials('az_sp_tenant') // Azure Tenant ID (from Jenkins credentials)
        ACR_CRED = credentials('acr_cred')  // Azure Container Registry credentials
    }

    stages {
        // Git Checkout Stage
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'github_cred', url: 'https://github.com/devops839/Azure_Jenkins_Pipeline.git'
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
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=voting-app -Dsonar.projectKey=voting-app -Dsonar.java.binaries=.'''
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
                                "target": "libs-release-local/votingapp/${env.BUILD_ID}/"
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

        // Trivy Docker Image Scan (Add Trivy scan for the built Docker image)
        stage('Trivy Docker Image Scan') {
            steps {
                script {
                    def dockerTag = "${env.ACR_REPO_URI}:${env.IMAGE_TAG}"
                    def outputFile = "trivy-image-report-${env.IMAGE_TAG}.txt"  // Output file in table format (text file)

                    // Run Trivy to scan the Docker image for HIGH and CRITICAL vulnerabilities
                    def trivyResult = sh(script: """
                        trivy image --severity HIGH,CRITICAL --format table -o ${outputFile} ${dockerTag} | tee ${outputFile}
                    """, returnStatus: true)

                    // If Trivy scan fails, mark the build as unstable
                    if (trivyResult != 0) {
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }

        // Authenticate & Push Image to Azure Container Registry (ACR)
        stage('Authenticate and Push Docker Image to ACR') {
            steps {
                script {
                    // Login to Azure using the service principal credentials stored in Jenkins
                    sh """
                    az login --service-principal -u ${AZURE_SP_CRED_USR} -p ${AZURE_SP_CRED_PSW} --tenant ${AZURE_TENANT_ID}
                    """
                    def dockerTag = "${env.ACR_REPO_URI}:${env.IMAGE_TAG}"
                    // Login to ACR using the credentials stored in Jenkins
                    sh """
                    echo ${ACR_CRED_PSW} | docker login ${env.ACR_REPO_URI} -u ${ACR_CRED_USR} --password-stdin
                    """
                    // Push the Docker image to Azure Container Registry (ACR)
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

                    // Modify Kubernetes YAML with the correct image tag and deploy
                    sh """
                    sed 's#demoacr839.azurecr.io/app:\${IMAGE_TAG}#${ACR_REPO_URI}:${env.IMAGE_TAG}#g' k8s/vote_deploy.yaml > k8s/deployment-new-build.yaml
                    kubectl apply -f k8s/deployment-new-build.yaml
                    """
                }
            }
        }

        // Deployment Verification Stage
        stage('Verify Deployment') {
            steps {
                script {
                    def podStatus = sh(script: "kubectl get pods -l app=voting-app -o jsonpath='{.items[0].status.phase}'", returnStdout: true).trim()
                    if (podStatus != 'Running') {
                        error "Deployment failed. Pod status is: ${podStatus}"
                    } else {
                        echo "Deployment successful. Pod status: ${podStatus}"
                    }
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
                    to: 'pavankalluri.14533@gmail.com',
                    from: 'jenkins@example.com',
                    replyTo: 'jenkins@example.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-image-report-${env.IMAGE_TAG}.txt'
                )
            }
        }
    }
}
