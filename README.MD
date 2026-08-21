# AWS EC2 Deployment with Docker, Jenkins, and SonarQube

This project is a static website deployed on an AWS EC2 instance using Docker, Jenkins, and SonarQube.

## Architecture

Developer -> GitHub -> Jenkins -> Docker Build -> SonarQube Analysis -> EC2 Deployment

## Prerequisites

- AWS EC2 Ubuntu instance
- SSH access
- Docker installed on the EC2 instance
- Jenkins running in Docker
- SonarQube running in Docker

## 1. Connect to EC2

```bash
ssh -i your-key.pem ubuntu@your-ec2-public-ip
```

## 2. Install Docker

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo $VERSION_CODENAME) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER
newgrp docker
```

## 3. Run Jenkins in Docker

```bash
docker volume create jenkins_home

docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  --restart unless-stopped \
  jenkins/jenkins:lts
```

Open:

```text
http://<your-ec2-public-ip>:8080
```

Get the admin password:

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Install required Jenkins plugins:
- Git
- GitHub
- Docker Pipeline
- Docker Commons
- SonarQube Scanner

## 4. Run SonarQube in Docker

```bash
docker run -d \
  --name sonarqube \
  -p 9000:9000 \
  -p 9092:9092 \
  sonarqube:lts-community
```

Open:

```text
http://<your-ec2-public-ip>:9000
```

Login:
- Username: admin
- Password: admin

Generate a project token in SonarQube and add it to Jenkins as a secret.

## 5. Create Dockerfile

```dockerfile
FROM nginx:alpine
COPY . /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Build image:

```bash
docker build -t my-static-site:latest .
```

Run container:

```bash
docker run -d --name my-static-site -p 80:80 my-static-site:latest
```

## 6. Jenkins Pipeline Example

Create a `Jenkinsfile`:

```groovy
pipeline {
    agent any

    environment {
        SONAR_TOKEN = credentials('sonarqube-token')
        IMAGE_NAME = 'my-static-site'
    }

    stages {
        stage('Checkout') {
            steps {
                git 'https://github.com/your-username/your-repo.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t ${IMAGE_NAME}:latest .' 
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh '''
                        sonar-scanner \
                        -Dsonar.projectKey=my-static-site \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=http://<your-ec2-public-ip>:9000 \
                        -Dsonar.login=${SONAR_TOKEN}
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                sh 'docker rm -f my-static-site || true'
                sh 'docker run -d --name my-static-site -p 80:80 ${IMAGE_NAME}:latest'
            }
        }
    }
}
```

## 7. Useful Commands

```bash
docker ps
docker ps -a
docker logs -f jenkins
docker logs -f sonarqube
docker stop my-static-site
docker rm my-static-site
docker images
```

## 8. Security Group Rules

Allow these ports in AWS EC2 Security Group:
- 22 (SSH)
- 80 (HTTP)
- 443 (HTTPS)
- 8080 (Jenkins)
- 9000 (SonarQube)

## 9. Final Workflow

1. Push code to GitHub
2. Jenkins pulls the code
3. Jenkins builds Docker image
4. SonarQube scans the code
5. Jenkins deploys the app on EC2

## 10. Quick Start

```bash
ssh -i your-key.pem ubuntu@your-ec2-public-ip

sudo apt update && sudo apt install -y docker.io
sudo usermod -aG docker $USER
newgrp docker

docker volume create jenkins_home
docker run -d --name jenkins -p 8080:8080 -p 50000:50000 -v jenkins_home:/var/jenkins_home jenkins/jenkins:lts

docker run -d --name sonarqube -p 9000:9000 sonarqube:lts-community

docker build -t my-static-site:latest .
docker run -d --name my-static-site -p 80:80 my-static-site:latest
```

This is a basic CI/CD setup for deploying a teir-1 project on AWS EC2 using Docker, Jenkins, and SonarQube.
