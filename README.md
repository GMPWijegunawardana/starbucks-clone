![Starbucks Clone Deployment](https://github.com/user-attachments/assets/6b654f47-9537-4b88-9584-41c760fc49ac)

# Deploy Starbucks Clone Application AWS using DevSecOps Approach
https://app.eraser.io/workspace/59NJfCay26dUMl5YAlFl?origin=share

# **Install AWS CLI**
```
sudo apt install unzip -y
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```
 
# **Install Jenkins on Ubuntu:**

```
#!/bin/bash
sudo apt update -y
wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | sudo tee /etc/apt/keyrings/adoptium.asc
echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | sudo tee /etc/apt/sources.list.d/adoptium.list
sudo apt update -y
sudo apt install temurin-17-jdk -y
/usr/bin/java --version
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update -y
sudo apt-get install jenkins -y
sudo systemctl start jenkins
sudo systemctl status jenkins
```


# **Install Docker on Ubuntu:**
```
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
sudo usermod -aG docker ubuntu
sudo chmod 777 /var/run/docker.sock
newgrp docker
sudo systemctl status docker
```

# **Install Trivy on Ubuntu:**

Reference Doc: https://aquasecurity.github.io/trivy/v0.55/getting-started/installation/
```
sudo apt-get install wget apt-transport-https gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
```


# **Install Docker Scout:**
```
docker login       `Give Dockerhub credentials here`
```
```
curl -sSfL https://raw.githubusercontent.com/docker/scout-cli/main/install.sh | sh -s -- -b /usr/local/bin
```
# Deployment Stages:
<img width="966" alt="Screenshot 2024-09-15 at 7 20 49 AM" src="https://github.com/user-attachments/assets/ddb5e618-79ab-49b3-8f13-b5114824eec3">

![Image](https://github.com/user-attachments/assets/9617ba22-8b76-468d-a8e2-e60fd9b64cb3)

# Jenkins Complete pipeline
```
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

        stage("Clean Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Git Checkout") {
            steps {
                git branch: 'main', url: 'https://github.com/yeshwanthlm/starbucks.git'
            }
        }

        stage("SonarQube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh """
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=starbucks \
                        -Dsonar.projectKey=starbucks
                    """
                }
            }
        }

        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage("Install NPM Dependencies") {
            steps {
                sh "npm install"
            }
        }

        stage("OWASP Dependency-Check Scan") {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage("Trivy File Scan") {
            steps {
                sh "trivy fs . --skip-files 'node_modules|dependency-check-report.xml' > trivy.txt || true"
            }
        }

        stage("Build Docker Image") {
            steps {
                sh "docker build -t starbucks ."
            }
        }

        stage("Tag & Push to DockerHub") {
            steps {
                withCredentials([usernamePassword(credentialsId: 'Docker', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker tag starbucks $DOCKER_USER/starbucks:latest
                        docker push $DOCKER_USER/starbucks:latest
                    '''
                }
            }
        }

        stage("Docker Scout Scan") {
            steps {
                withCredentials([usernamePassword(credentialsId: 'Docker', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker-scout quickview $DOCKER_USER/starbucks:latest || true
                        docker-scout cves $DOCKER_USER/starbucks:latest || true
                        docker-scout recommendations $DOCKER_USER/starbucks:latest || true
                    '''
                }
            }
        }

        stage("Deploy to Container") {
            steps {
                withCredentials([usernamePassword(credentialsId: 'Docker', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        docker rm -f starbucks || true
                        docker run -d --name starbucks -p 3000:3000 $DOCKER_USER/starbucks:latest
                    '''
                }
            }
        }
    }

    post {
        always {
            emailext attachLog: true,
                subject: "'${currentBuild.result}' - Starbucks Pipeline",
                body: """
                    <html>
                        <body>
                            <h2>Pipeline Result: ${currentBuild.result}</h2>
                            <p><b>Project:</b> ${env.JOB_NAME}</p>
                            <p><b>Build Number:</b> ${env.BUILD_NUMBER}</p>
                            <p><b>URL:</b> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                        </body>
                    </html>
                """,
                to: 'manishawijegunawardana@gmail.com',
                mimeType: 'text/html',
                attachmentsPattern: 'trivy.txt, **/dependency-check-report.xml'
        }
    }
}


```
# ☕ Starbucks CI/CD Pipeline with Security Scans

This project demonstrates a full CI/CD pipeline using **Jenkins, Docker, Trivy, and OWASP Dependency-Check**.

## 🔹 Pipeline Overview
- Jenkins pipeline (Groovy-based)
- SonarQube analysis
- OWASP Dependency-Check scan
- Trivy image scanning
- Docker image build & push to DockerHub
- Deployment on container

## 🔹 Reports
- [Trivy Vulnerability Report (TXT)](reports/trivy-report.txt)
- [Trivy Vulnerability Report (JSON)](reports/trivy-report.json)
- [OWASP Dependency-Check Report (HTML)](reports/dependency-check-report.html)

## 🔹 Pipeline Screenshot
![Jenkins Pipeline](screenshots/jenkins-pipeline.png)

## 🔹 Security Report Screenshot
![Trivy Report](screenshots/trivy.png)
![Dependency-Check Report](screenshots/depcheck.png)

---
✅ DockerHub: [manishapasandul/starbucks](https://hub.docker.com/r/manishapasandul/starbucks)  
✅ GitHub Repo: (add your repo link here)
