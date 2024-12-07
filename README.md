# DevSecOps Pipeline for Voting App

This repository contains the **DevSecOps pipeline implementation** for the **Voting App**. The pipeline integrates various stages to ensure code quality, security, and continuous integration/continuous deployment (CI/CD) to **Azure Kubernetes Service (AKS)**. The pipeline is built using **Jenkins**.

## 🛠 Pipeline Overview

The pipeline automates the process of building, testing, scanning, and deploying the Voting App. It incorporates essential steps like static code analysis, security scanning, and deployment to Azure Kubernetes Service (AKS). Below are the stages in the pipeline:

## 🚀 Pipeline Stages

### 1. **Git Checkout**
   - **Description**: In this stage, the pipeline checks out the latest version of the application source code from the GitHub repository. This ensures that the pipeline uses the most up-to-date code for the build and deployment process.
   - **Goal**: Ensure that the pipeline works with the latest version of the application code.

### 2. **Maven Compile**
   - **Description**: The code is compiled using Maven to resolve dependencies and prepare it for testing. This step verifies that the project can be successfully compiled and that all dependencies are available.
   - **Goal**: Compile the source code and resolve dependencies to make sure the project is in a buildable state.

### 3. **Maven Test**
   - **Description**: This stage runs unit tests on the application code. The tests ensure that the core functionality of the application is working correctly before proceeding further in the pipeline.
   - **Goal**: Run automated tests to check that the application logic is functioning as expected and to catch any regressions early.

### 4. **SonarQube Analysis**
   - **Description**: The code undergoes static code analysis through SonarQube to identify potential code quality issues, bugs, and vulnerabilities. This stage ensures that the code adheres to quality standards before being deployed.
   - **Goal**: Analyze the code for potential quality issues and security vulnerabilities, providing insights to improve the overall code quality.

### 5. **Quality Gate**
   - **Description**: After the SonarQube analysis, the pipeline waits for the **SonarQube Quality Gate** to pass. The Quality Gate evaluates whether the code meets the predefined quality criteria (such as no critical issues or vulnerabilities) to proceed further.
   - **Goal**: Ensure the code quality is high and meets defined standards. If the code doesn't meet the criteria, the pipeline is paused or failed.

### 6. **Maven Build**
   - **Description**: This stage builds the final executable artifact (such as a JAR file) for the Voting App using Maven. The build will skip tests if they have already been executed in the previous stage.
   - **Goal**: Generate the application artifact (e.g., JAR file) for distribution or deployment.

### 7. **Publish to JFrog Artifactory**
   - **Description**: The build artifact (JAR file) is uploaded to **JFrog Artifactory**, a repository manager that stores build artifacts for reuse in future builds or deployments. This stage ensures that the artifact is stored securely and is available for future reference.
   - **Goal**: Store the application build in a repository for versioning and easier distribution.

### 8. **Docker Build & Scan**
   - **Description**: A Docker image is built from the application, containerizing the Voting App for deployment. The image is then scanned for security vulnerabilities using **Trivy**, a container security scanner. This step ensures that the Docker image is secure and free from critical vulnerabilities before being pushed to a container registry.
   - **Goal**: Containerize the application and ensure the Docker image is secure by scanning it for vulnerabilities.

### 9. **Authenticate and Push to ACR**
   - **Description**: The pipeline authenticates to **Azure** using a service principal and then pushes the Docker image to **Azure Container Registry (ACR)**. This allows the image to be stored in a private registry, ready for deployment to Azure Kubernetes Service (AKS).
   - **Goal**: Push the Docker image to ACR, making it available for deployment on Azure.

### 10. **Deploy to AKS**
   - **Description**: The Docker image is deployed to **Azure Kubernetes Service (AKS)** using Kubernetes configuration files. The pipeline uses `kubectl` to apply the Kubernetes deployment files, ensuring that the application is running on a cloud-based Kubernetes cluster.
   - **Goal**: Deploy the containerized application to AKS, enabling cloud-based, scalable deployment.

### 11. **Verify Deployment**
   - **Description**: The pipeline checks the deployment status in AKS by verifying the state of the pods running the application. If the pod is not in the `Running` state, the deployment is considered to have failed, and an error is raised.
   - **Goal**: Ensure the application was successfully deployed and is running as expected in AKS.

## ⚙️ Setup and Configuration

### 1. **Prerequisites**

Before setting up the pipeline, ensure the following tools and services are configured:

- **Jenkins**: Ensure Jenkins is set up with necessary plugins like Docker, Maven, Kubernetes, and SonarQube.
- **Azure Subscription**: Set up an Azure subscription and configure service principal credentials for AKS and ACR access.
- **Azure Kubernetes Service (AKS)**: Set up an AKS cluster and have access credentials.
- **Azure Container Registry (ACR)**: Set up an ACR instance to store Docker images.
- **SonarQube**: Install and configure SonarQube for static code analysis.
- **JFrog Artifactory**: Set up JFrog Artifactory for managing build artifacts.
- **Trivy**: Install **Trivy** for Docker image vulnerability scanning.

### 2. **Clone the Repository**

```bash
git clone https://github.com/yourusername/voting-app-java.git
cd voting-app-java
