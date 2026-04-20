🚀 PHASE 0 — AWS SETUP (CONSOLE)

👉 Region MUST be:

us-east-2 (Ohio)
🔹 1. Create Security Group

Go to EC2 → Security Groups → Create

Name:
jenkins-server-security-group
Inbound rules:
Type	Port	Source
SSH	22	Anywhere (0.0.0.0/0)
Custom TCP	8080	Anywhere (0.0.0.0/0)

👉 Save

🔹 2. Create Key Pair

Go to EC2 → Key Pairs → Create

Name:
jenkins-ssh-key
Type: RSA
Format: .pem

👉 Download and keep safe

🔹 3. Create IAM Role

Go to IAM → Roles → Create Role

Trusted entity: EC2
Permissions:
👉 Attach AdministratorAccess
Name:
ec2-eks-role

👉 Create role

🔹 4. Launch EC2 Instance

Go to EC2 → Launch Instance

Basic config:
Field	Value
Name	jenkins-server
AMI	Ubuntu (latest)
Instance Type	t3.small
Key Pair	jenkins-ssh-key
Security Group	jenkins-server-security-group
🔹 5. Add User Data (IMPORTANT)

Paste this:

#!/bin/bash
sudo apt-get update
sudo apt-get install -y amazon-ssm-agent
sudo systemctl enable amazon-ssm-agent
sudo systemctl start amazon-ssm-agent
🔹 6. Attach IAM Role

After instance is created:

Go to EC2 → Instance → Actions → Security → Modify IAM Role
Select:
ec2-eks-role
🎯 PHASE 1 — CONNECT TO EC2
ssh -i jenkins-ssh-key.pem ubuntu@<PUBLIC-IP>
🔹 Verify IAM Role (VERY IMPORTANT)
aws sts get-caller-identity

👉 Must show:

assumed-role/ec2-eks-role
🎯 PHASE 2 — INSTALL BASIC TOOLS
Update
sudo apt update -y
Java
sudo apt install default-jdk -y
🎯 PHASE 3 — INSTALL DOCKER
sudo apt install apt-transport-https ca-certificates curl software-properties-common -y
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io -y

Start:

sudo systemctl start docker
sudo systemctl enable docker
🎯 PHASE 4 — INSTALL JENKINS
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install jenkins -y

Start:

sudo systemctl start jenkins
sudo systemctl enable jenkins
🔹 Access Jenkins
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

Open:

http://<EC2-IP>:8080
Install suggested plugins
Create user:
jenkins-admin-user
jenkins_springboot4684$
🎯 PHASE 5 — DOCKER ACCESS FOR JENKINS
sudo usermod -a -G docker jenkins
sudo service jenkins restart
🎯 PHASE 6 — INSTALL K8S TOOLS
kubectl
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.27.0/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
eksctl
curl -LO https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz
tar -xzf eksctl_Linux_amd64.tar.gz
sudo mv eksctl /usr/local/bin


🎯 PHASE 7 — CREATE ECR

👉 Go to AWS Console → ECR → Create Repository

Details:
Name: springboot-jenkins-ecr-repo
Type: Private
Region: us-east-2

👉 Click Create

🔹 Copy ECR URI

Example:

123456789012.dkr.ecr.us-east-2.amazonaws.com/springboot-jenkins-ecr-repo

👉 Save this — you need it in Jenkins

🎯 PHASE 8 — CREATE EKS CLUSTER

👉 Go to EC2 terminal (ubuntu user)

🔹 Create Cluster (NO NODEGROUP)
eksctl create cluster \
--name jenkins-cluster \
--region us-east-2 \
--without-nodegroup

⏳ Wait ~15–20 mins

🔹 Create Nodegroup
eksctl create nodegroup \
--cluster jenkins-cluster \
--region us-east-2 \
--name jenkins-nodegroup \
--node-type t3.small \
--nodes 2
🎯 PHASE 9 — CONNECT TO CLUSTER
aws eks update-kubeconfig \
--region us-east-2 \
--name jenkins-cluster
🔹 Verify
kubectl get nodes

👉 You should see 2 nodes

🎯 PHASE 10 — SAVE KUBECONFIG (IMPORTANT)
cat /var/lib/jenkins/.kube/config

👉 Copy full output
👉 Save as:

kubeconfig.txt
🎯 PHASE 11 — SWITCH TO JENKINS USER
sudo su - jenkins
🔹 Configure AWS (VERY IMPORTANT)
aws configure

👉 Use credentials from:

cat /home/labuser/.access

Enter:

Access Key
Secret Key
Region: us-east-2
🎯 PHASE 12 — JENKINS CONFIG

👉 Open Jenkins UI

🔹 Install Plugins

Go to:

Manage Jenkins → Plugins

Install:

Docker
Docker Pipeline
Kubernetes CLI
Amazon ECR
🔹 Configure Maven
Manage Jenkins → Tools → Add Maven

Name:

Maven3
🔹 Add Credentials
GitHub
ID: github-credentials
Kubernetes
Kind: Secret File
Upload: kubeconfig.txt
ID:
K8S
🎯 PHASE 13 — CREATE PIPELINE

👉 New Item → Pipeline

Name:

springboot-jenkins-eks
🔹 Add Jenkinsfile

Paste your pipeline script

🔹 Replace values
<<REPOSITORY URI>> → your ECR URI
<<GITHUB REPO URL>> → your repo
🎯 PHASE 14 — RUN PIPELINE

👉 Click:

Build Now
🔹 Expected stages
Checkout ✅
Build jar ✅
Build docker image ✅
Push to ECR ✅
Deploy to EKS ✅
🎯 PHASE 15 — VERIFY DEPLOYMENT
kubectl get pods
kubectl get svc
🔹 Get LoadBalancer
kubectl get svc

👉 Copy:

EXTERNAL-IP
🔹 Open in browser

👉 You should see your app running
