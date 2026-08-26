# CI/CD Pipeline with GitHub, Jenkins, SonarQube, Docker and Nginx

This project demonstrates a CI/CD pipeline for deploying a static website to AWS EC2. The infrastructure is split across separate EC2 instances so that each service has a clear responsibility.

## Architecture

```text
Developer -> GitHub -- webhook --> Jenkins EC2
                                  |
                                  v
                         SonarQube EC2 (analysis)
                                  |
                         quality gate passes
                                  v
                    Deployment EC2 (Docker + Nginx)
                                  |
                                  v
                         Website on port 80
```

## EC2 Instances

| Instance | Responsibility | Main port |
| --- | --- | --- |
| Jenkins | Receives the GitHub webhook and runs the pipeline | `8080` |
| SonarQube | Analyses source code and reports the quality gate | `9000` |
| Deployment | Runs the Docker image with Nginx | `80` |

Use Ubuntu instances and replace these placeholders with your own addresses:

```text
JENKINS_IP=<jenkins-ec2-ip>
SONARQUBE_IP=<sonarqube-ec2-ip>
DEPLOYMENT_IP=<deployment-ec2-ip>
```

## Prerequisites

- Three Ubuntu AWS EC2 instances with SSH access
- A GitHub repository containing this project and a `Jenkinsfile`
- Jenkins installed directly on the Jenkins instance
- SonarQube installed on its separate EC2 instance
- Docker installed only on the deployment instance
- AWS security groups configured for the ports listed below

## 1. Connect to EC2

Run this from your local machine for each instance:

```bash
chmod 400 your-key.pem
ssh -i "your-key.pem" ubuntu@<ec2-ip>
```

Update each instance:

```bash
sudo apt update && sudo apt upgrade -y
```

## 2. Install Jenkins Directly on the Jenkins EC2 Instance

Jenkins is installed as a native system service, not inside Docker. Install Java and the official Jenkins package:

```bash
sudo apt update
sudo apt install -y fontconfig openjdk-17-jre
java -version
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
    https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
```

Use the official repository and start Jenkins:

```bash
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install -y jenkins
sudo systemctl enable --now jenkins
sudo systemctl status jenkins
```

Open `http://<jenkins-ec2-ip>:8080` and get the initial password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Install these plugins: **Pipeline**, **Git**, **GitHub Integration**, **Docker Pipeline**, **SonarQube Scanner**, and **Credentials Binding**. Create a Pipeline job that loads the `Jenkinsfile` from GitHub, then enable **GitHub hook trigger for GITScm polling**.

## 4. Connect GitHub to Jenkins with a Webhook

In GitHub, open **Repository Settings > Webhooks > Add webhook**:

```text
Payload URL: http://<jenkins-ec2-ip>:8080/github-webhook/
Content type: application/json
Events: Just the push event
Active: enabled
```

Every push now sends a webhook to Jenkins and starts the pipeline.

## 5. Install SonarQube on Its Separate EC2 Instance

```bash
docker run -d \
  --name sonarqube \
  -p 9000:9000 \
  --restart unless-stopped \
  sonarqube:lts-community
```

Open `http://<sonarqube-ec2-ip>:9000` and sign in with `admin` / `admin`. Create a project such as `luminary-site`, generate a project token, and save it in Jenkins as a **Secret text** credential with ID `sonarqube-token`.

In **Manage Jenkins > System > SonarQube servers**, add:

```text
Name: sonarqube
Server URL: http://<sonarqube-ec2-ip>:9000
```

Install the SonarScanner CLI on the Jenkins instance or configure it under **Manage Jenkins > Tools**. Jenkins will connect to the SonarQube server over its private IP.

## 6. Install Docker on the Deployment EC2 Instance

Docker is installed only at the last stage, on the server that hosts the website. Run these commands on the deployment instance:

```bash
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker "$USER"
newgrp docker
docker --version
```

## 7. Prepare the Deployment EC2 Instance

The repository `Dockerfile` copies the website into the official Nginx image:

```bash
git clone https://github.com/<github-user>/<repository>.git
cd <repository>
docker build -t luminary-site:latest .
docker rm -f luminary-site 2>/dev/null || true
docker run -d --name luminary-site --restart unless-stopped -p 80:80 luminary-site:latest
```


Configure SSH from Jenkins to the deployment instance using an SSH key credential or by adding the Jenkins public key with `ssh-copy-id`. The Jenkins instance needs `sonar-scanner`, `ssh`, and `scp`; it does not need Docker.

## 8. AWS Security Group Rules

- `22/tcp`: SSH, restricted to your IP or trusted security group
- `80/tcp`: HTTP, open to website visitors
- `8080/tcp`: Jenkins, restricted to administrators and GitHub webhook traffic
- `9000/tcp`: SonarQube, restricted to Jenkins and administrators
- `50000/tcp`: Jenkins agents only, if inbound agents are used

Prefer private IPs and security-group-to-security-group rules for Jenkins-to-SonarQube and Jenkins-to-deployment traffic.

## Useful Commands

```bash
docker ps
docker images
sudo systemctl status jenkins
sudo journalctl -u jenkins -f
docker logs -f sonarqube
docker logs -f luminary-site
docker restart luminary-site
sudo ss -tulpn
curl -I http://<deployment-ec2-ip>
```

## Complete Workflow

1. The developer pushes code to GitHub.
2. GitHub sends a webhook to Jenkins.
3. Jenkins checks out the repository on its native Jenkins installation.
4. Jenkins sends the source code to SonarQube on its separate EC2 instance.
5. The pipeline stops if the SonarQube quality gate fails.
6. Jenkins connects to the deployment EC2 instance over SSH.
7. The deployment server builds and runs the Docker image with Nginx on port `80`.
8. Visitors open the deployment IP or domain name to see the website.
