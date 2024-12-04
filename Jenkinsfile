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
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'github_cred', url: 'https://github.com/devops839/Azure_Jenkins_Pipeline.git'
            }
        }
        stage('Maven Compile') {
            steps {
                sh "mvn compile"
            }
        }
        stage('Maven Test') {
            steps {
                sh "mvn test"
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=voting-app -Dsonar.projectKey=voting-app -Dsonar.java.binaries=.'''
                }
            }
        }
        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonarqube_token'
                }
            }
        }
        stage('Maven Build') {
            steps {
                sh "mvn package -DskipTests"  // Skip tests if they're already run earlier
            }
        }
        stage('Publish to Jfrog Artifactory') {
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
        stage('Trivy Docker Image Scan') {
            steps {
                script {
                    def dockerTag = "${env.ACR_REPO_URI}:${env.IMAGE_TAG}"
                    def outputFile = "trivy-image-report-${env.IMAGE_TAG}.txt"
                    def trivyResult = sh(script: """
                        trivy image --severity HIGH,CRITICAL --format table -o ${outputFile} ${dockerTag} | tee ${outputFile}
                    """, returnStatus: true)
                    if (trivyResult != 0) {
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }
        stage('Authenticate and Push Docker Image to ACR') {
            steps {
                script {
                    sh """
                    az login --service-principal -u ${AZURE_SP_CRED_USR} -p ${AZURE_SP_CRED_PSW} --tenant ${AZURE_TENANT_ID}
                    """
                    def dockerTag = "${env.ACR_REPO_URI}:${env.IMAGE_TAG}"
                    sh """
                    echo ${ACR_CRED_PSW} | docker login ${env.ACR_REPO_URI} -u ${ACR_CRED_USR} --password-stdin
                    """
                    sh "docker push ${dockerTag}"
                }
            }
        }
        stage('K8S Deploy') {
            steps {
                script {
                    sh """
                    az aks get-credentials --resource-group demo-aks-rg --name ${env.AKS_CLUSTER_NAME}
                    """
                    sh """
                    sed 's#demoacr839.azurecr.io/app:\${IMAGE_TAG}#${ACR_REPO_URI}:${env.IMAGE_TAG}#g' k8s/vote_deploy.yaml > k8s/deployment-new-build.yaml
                    kubectl apply -f k8s/deployment-new-build.yaml
                    """
                }
            }
        }
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
