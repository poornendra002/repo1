Complete Assessment Guide: Jenkins + EKS Deployment

STEP 1: Create EC2 Instance
Go to AWS Console → EC2 → us-east-2 (Ohio)
1a. Create Security Group

EC2 → Security Groups → Create security group
Name: jenkins-server-security-group
Inbound rules:

Type: Custom TCP, Port: 8080, Source: 0.0.0.0/0
Type: SSH, Port: 22, Source: 0.0.0.0/0



1b. Create Key Pair

EC2 → Key Pairs → Create key pair
Name: jenkins-ssh-key, Type: RSA, Format: .pem

1c. Launch EC2 Instance

EC2 → Launch Instance
Name: jenkins-server
AMI: Ubuntu (latest LTS)
Instance type: t3.small
Key pair: jenkins-ssh-key
Security group: jenkins-server-security-group
User Data (paste exactly):

bash#!/bin/bash
sudo apt-get update
sudo apt-get install -y amazon-ssm-agent
sudo systemctl enable amazon-ssm-agent
sudo systemctl start amazon-ssm-agent

STEP 2: Configure IAM Role

Go to IAM → Roles → Create Role
Trusted entity: EC2
Policy: AdministratorAccess
Role name: ec2-eks-role
Attach to EC2:

EC2 → Select jenkins-server → Actions → Security → Modify IAM Role
Select ec2-eks-role → Save




STEP 3: Install Required Software
SSH into your instance:
bashssh -i jenkins-ssh-key.pem ubuntu@<public-IP>
Install Java
bashsudo apt-get update
sudo apt install default-jdk -y
Install AWS CLI & IAM Authenticator
bashsudo apt-get install zip -y
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

curl -Lo aws-iam-authenticator https://github.com/kubernetes-sigs/aws-iam-authenticator/releases/download/v0.5.9/aws-iam-authenticator_0.5.9_linux_amd64
chmod +x aws-iam-authenticator
sudo mv aws-iam-authenticator /usr/local/bin/
Install Docker
bashsudo apt install apt-transport-https ca-certificates curl software-properties-common -y
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io -y

# Start Docker
sudo systemctl start docker
sudo systemctl enable docker
sudo systemctl status docker
Install Jenkins
bashcurl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins -y

# Start Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins
sudo systemctl status jenkins
Unlock Jenkins
bashsudo cat /var/lib/jenkins/secrets/initialAdminPassword
Then open http://<public-IP>:8080, paste the password, install suggested plugins, and create admin user:

Username: jenkins-admin-user
Password: jenkins_springboot4684$


STEP 4: Docker Volume & Network
bash# Create Docker volume
sudo docker volume create jenkins_data

# Run Jenkins container with volume
sudo docker run -d \
  --name jenkins-container \
  -v jenkins_data:/var/jenkins_home \
  -p 8080:8080 \
  jenkins/jenkins:lts

# Create Docker network
sudo docker network create jenkins_network

# Attach Jenkins container to network
sudo docker network connect jenkins_network jenkins-container

STEP 5: Install Kubernetes Tools
Install kubectl
bashcurl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
kubectl --help
Install eksctl
bashARCH=amd64
PLATFORM=$(uname -s)_$ARCH
curl -sLO "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz
sudo mv /tmp/eksctl /usr/local/bin
eksctl --help

STEP 6: Create ECR Repository

Go to ECR → Create repository
Visibility: Private
Name: springboot-jenkins-ecr-repo
Click Create — note the Repository URI


STEP 7: Configure Jenkins for Docker
bash# Add Jenkins to Docker group
sudo usermod -a -G docker jenkins
sudo service jenkins restart
sudo systemctl daemon-reload
sudo service docker stop
sudo service docker start

# Switch to Jenkins user and configure AWS
sudo su -s /bin/bash jenkins
whoami   # should show: jenkins

aws configure
# Enter: Access Key, Secret Key, Region: us-east-2, Output: json
# (Credentials are in /home/labuser/.access/config)

STEP 8: Create EKS Cluster & Node Group
bash# Still as jenkins user
# Create cluster (takes 15-20 mins)
eksctl create cluster \
  --name jenkins-cluster \
  --region us-east-2 \
  --without-nodegroup

# Save kubeconfig
cat /var/lib/jenkins/.kube/config
# Copy this output → save as kubeconfig.txt on your local VM

# Create node group
eksctl create nodegroup \
  --cluster jenkins-cluster \
  --region us-east-2 \
  --name jenkins-nodegroup \
  --node-type t3.small \
  --nodes 2

STEP 9: Set Up GitHub Repository
On your VM-lab terminal:
bashgit config --global user.name "your-github-username"
git config --global user.email "your-email"

Create a private GitHub repo named springboot-jenkins-eks
Upload project files to the repo
In eks-deploy-k8s.yaml, replace <<REPOSITORY URI>> with your ECR repo URI


STEP 10: Configure Maven & Jenkins Plugins

Manage Jenkins → Tools → Add Maven → Name: Maven3 → Save
Manage Jenkins → Plugins → Available → Search and install:

Kubernetes CLI
Amazon ECR
Docker
Docker Pipeline
Install without restart




STEP 11: Configure Jenkins Credentials
Go to Manage Jenkins → Credentials → Global → Add Credentials
CredentialKindDetailsGitHubUsername + PasswordUsername: your GitHub username, Password: PAT token, ID: github-credentialsKubernetesSecret FileUpload kubeconfig.txt, ID: K8S

STEP 12: Create & Run Jenkins Pipeline

New Item → Name: springboot-jenkins-eks → Pipeline
Description: Automation of Spring Boot microservices builds using Jenkins pipeline and deploy it on AWS EKS Cluster.
Pipeline script: paste contents of jenkins_script.txt
Update ECR URI and GitHub repo in the script
Save → Build Now


STEP 13: Access Deployed Application

Go to EC2 → Load Balancers
Find the Classic Load Balancer created by Kubernetes
Copy the DNS name and open in browser → your Spring Boot app should be live 🎉
