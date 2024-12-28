# Netflix Clone Deployment and CI/CD Pipeline

## Key Highlights

- **Deployed on Cloud**: Successfully launched and deployed the Netflix Clone application on AWS EC2 using Docker, ensuring scalability and efficient resource utilization.
- **Integrated CI/CD Pipeline**: Implemented a robust Jenkins pipeline with 7+ stages for automation, including SonarQube and Dependency-Check integration.
- **Security Scanning**: Enhanced application security with Trivy and OWASP Dependency-Check scans, ensuring compliance with modern security standards.

---

## Project Overview

This project demonstrates the deployment of a Netflix Clone application on the cloud using Docker containers, coupled with a CI/CD pipeline powered by Jenkins. Security and code quality are prioritized with the inclusion of SonarQube, OWASP Dependency-Check, and Trivy scanning tools.

---

## Instructions

### Phase 1: Initial Setup and Deployment

1. **Launch EC2 Instance**:
   - Provision an EC2 instance (Ubuntu 22.04) on AWS.
   - Connect via SSH.

2. **Clone the Code**:
   ```bash
   git clone https://github.com/abhipraydhoble/netflix/tree/main
   ```

3. **Install Docker and Run the App**:
   ```bash
   sudo apt-get update
   sudo apt-get install docker.io -y
   sudo systemctl start docker
   sudo usermod -aG docker ubuntu
   newgrp docker
   sudo chmod 777 /var/run/docker.sock
   docker build --build-arg TMDB_V3_API_KEY=<your-api-key> -t netflix .
   docker run -d --name netflix -p 8081:80 netflix:latest
   ```

---

### Phase 2: Security

1. **Install and Configure SonarQube**:
   ```bash
   docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
   ```
   Access at `http://<public-IP>:9000` (default credentials: admin/admin).

2. **Install Trivy**:
   ```bash
   sudo apt-get install wget apt-transport-https gnupg lsb-release
   wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
   echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
   sudo apt-get update
   sudo apt-get install trivy
   trivy image <imageid>
   ```

---

### Phase 3: CI/CD Setup

1. **Install Jenkins**:
   ```bash
   sudo apt update
   sudo apt install fontconfig openjdk-17-jre
   sudo wget -O /usr/share/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
   echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
   sudo apt-get update
   sudo apt-get install jenkins
   sudo systemctl start jenkins
   sudo systemctl enable jenkins
   ```
   Access Jenkins at `http://<public-IP>:8080`.

2. **Install Necessary Plugins**:
   - Eclipse Temurin Installer
   - SonarQube Scanner
   - NodeJs Plugin
   - Docker Pipeline

3. **Configure Jenkins Pipeline**:
   ```groovy
   pipeline {
       agent any
       tools {
           jdk 'jdk17'
           nodejs 'node16'
       }
       environment {
           SCANNER_HOME = tool 'sonar-scanner'
       }
       stages {
           stage('Clean Workspace') {
               steps {
                   cleanWs()
               }
           }
           stage('Checkout from Git') {
               steps {
                   git branch: 'main', url: 'https://github.com/abhipraydhoble/netflix.git'
               }
           }
           stage('SonarQube Analysis') {
               steps {
                   withSonarQubeEnv('sonar-server') {
                       sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix -Dsonar.projectKey=Netflix'''
                   }
               }
           }
           stage('Quality Gate') {
               steps {
                   script {
                       waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                   }
               }
           }
           stage('Install Dependencies') {
               steps {
                   sh 'npm install'
               }
           }
           stage('OWASP FS Scan') {
               steps {
                   dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                   dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
               }
           }
           stage('Trivy FS Scan') {
               steps {
                   sh 'trivy fs . > trivyfs.txt'
               }
           }
           stage('Docker Build & Push') {
               steps {
                   script {
                       withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                           sh 'docker build --build-arg TMDB_V3_API_KEY=<your-api-key> -t netflix .'
                           sh 'docker tag netflix abhipraydhoble/netflix:latest'
                           sh 'docker push abhipraydhoble/netflix:latest'
                       }
                   }
               }
           }
           stage('Trivy Image Scan') {
               steps {
                   sh 'trivy image abhipraydhoble/netflix:latest > trivyimage.txt'
               }
           }
           stage('Deploy to Container') {
               steps {
                   sh 'docker run -d --name netflix -p 8081:80 abhipraydhoble/netflix:latest'
               }
           }
       }
   }
   ```
