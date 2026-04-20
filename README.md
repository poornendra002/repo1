# Jenkins + EKS Deployment on AWS

> Automation of Spring Boot microservices builds using Jenkins pipeline and deploy it on AWS EKS Cluster.

---

## Table of Contents

1. [Create EC2 Instance](#1-create-ec2-instance)
2. [Configure IAM Role](#2-configure-iam-role)
3. [Install Required Software](#3-install-required-software)
4. [Configure Jenkins with Docker Volumes and Networks](#4-configure-jenkins-with-docker-volumes-and-networks)
5. [Install Kubernetes Tools](#5-install-kubernetes-tools)
6. [Create an ECR Repository](#6-create-an-ecr-repository)
7. [Configure Jenkins for Docker](#7-configure-jenkins-for-docker)
8. [Create EKS Cluster and Node Group](#8-create-eks-cluster-and-node-group)
9. [Set Up GitHub Repository](#9-set-up-github-repository)
10. [Configure Maven Build Tool and Install Jenkins Plugins](#10-configure-maven-build-tool-and-install-jenkins-plugins)
11. [Configure Jenkins Credentials](#11-configure-jenkins-credentials)
12. [Create and Execute the Jenkins Pipeline](#12-create-and-execute-the-jenkins-pipeline)
13. [Access the Deployed Application](#13-access-the-deployed-application)

---

## 1. Create EC2 Instance

> **Region:** `us-east-2` (Ohio) — all resources must be created here.

### 1a. Create Security Group

1. Go to **EC2 → Security Groups → Create security group**
2. Name: `jenkins-server-security-group`
3. Add the following **Inbound Rules:**

| Type       | Protocol | Port | Source    |
|------------|----------|------|-----------|
| Custom TCP | TCP      | 8080 | 0.0.0.0/0 |
| SSH        | TCP      | 22   | 0.0.0.0/0 |

### 1b. Create Key Pair

1. Go to **EC2 → Key Pairs → Create key pair**
2. Name: `jenkins-ssh-key`
3. Type: `RSA`, Format: `.pem`

### 1c. Launch EC2 Instance

1. Go to **EC2 → Launch Instance**
2. Configure with the following settings:

| Setting         | Value                        |
|-----------------|------------------------------|
| Name            | `jenkins-server`             |
| AMI             | Ubuntu (latest LTS)          |
| Instance type   | `t3.small`                   |
| Key pair        | `jenkins-ssh-key`            |
| Security group  | `jenkins-server-security-group` |

3. Under **Advanced Details → User Data**, paste the following:

```bash
#!/bin/bash
sudo apt-get update
sudo apt-get install -y amazon-ssm-agent
sudo systemctl enable amazon-ssm-agent
sudo systemctl start amazon-ssm-agent
```

---

## 2. Configure IAM Role

1. Go to **IAM → Roles → Create Role**
2. Trusted entity type: **AWS Service → EC2**
3. Attach policy: `AdministratorAccess`
4. Role name: `ec2-eks-role`

### Attach Role to EC2 Instance

1. Go to **EC2 → Instances → Select `jenkins-server`**
2. Click **Actions → Security → Modify IAM Role**
3. Select `ec2-eks-role` → **Update IAM Role**

---

## 3. Install Required Software

SSH into the instance:

```bash
ssh -i jenkins-ssh-key.pem ubuntu@<public-IP>
```

### Install Java

```bash
sudo apt-get update
sudo apt install default-jdk -y
```

### Install AWS CLI & IAM Authenticator

```bash
sudo apt-get install zip -y
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

curl -Lo aws-iam-authenticator https://github.com/kubernetes-sigs/aws-iam-authenticator/releases/download/v0.5.9/aws-iam-authenticator_0.5.9_linux_amd64
chmod +x aws-iam-authenticator
sudo mv aws-iam-authenticator /usr/local/bin/
```

### Install Docker

```bash
sudo apt install apt-transport-https ca-certificates curl software-properties-common -y
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io -y
```

#### Start & Enable Docker

```bash
sudo systemctl start docker
sudo systemctl enable docker
sudo systemctl status docker
```

### Install Jenkins

```bash
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins -y
```

#### Start & Enable Jenkins

```bash
sudo systemctl start jenkins
sudo systemctl enable jenkins
sudo systemctl status jenkins
```

### Unlock Jenkins

1. Get the initial admin password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

2. Open `http://<public-IP>:8080` in your browser
3. Paste the password and click **Install suggested plugins**
4. Create the first admin user:

| Field    | Value                    |
|----------|--------------------------|
| Username | `jenkins-admin-user`     |
| Password | `jenkins_springboot4684$` |

---

## 4. Configure Jenkins with Docker Volumes and Networks

Run the following on the EC2 instance (use `sudo` where required):

### Create Docker Volume

```bash
sudo docker volume create jenkins_data
```

### Run Jenkins in a Docker Container

```bash
sudo docker run -d \
  --name jenkins-container \
  -v jenkins_data:/var/jenkins_home \
  -p 8080:8080 \
  jenkins/jenkins:lts
```

### Create Docker Network

```bash
sudo docker network create jenkins_network
```

### Attach Jenkins Container to the Network

```bash
sudo docker network connect jenkins_network jenkins-container
```

---

## 5. Install Kubernetes Tools

### Install kubectl

```bash
curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
kubectl --help
```

### Install eksctl

```bash
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH
curl -sLO "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz
sudo mv /tmp/eksctl /usr/local/bin
eksctl --help
```

---

## 6. Create an ECR Repository

1. Go to **Amazon ECR → Create repository**
2. Visibility: **Private**
3. Repository name: `springboot-jenkins-ecr-repo`
4. Click **Create repository**

> **Note:** Copy the **Repository URI** — it will be needed later in `eks-deploy-k8s.yaml` and the Jenkins pipeline script.

---

## 7. Configure Jenkins for Docker

```bash
sudo usermod -a -G docker jenkins
sudo service jenkins restart
sudo systemctl daemon-reload
sudo service docker stop
sudo service docker start
```

### Switch to Jenkins User and Configure AWS

```bash
sudo su -s /bin/bash jenkins
whoami  # should output: jenkins

aws configure
# Enter: Access Key ID, Secret Access Key, Region: us-east-2, Output format: json
```

> **Note:** AWS credentials are available at `/home/labuser/.access/config`.

---

## 8. Create EKS Cluster and Node Group

> Run these commands as the **Jenkins user**.

### Create EKS Cluster (without node group)

```bash
eksctl create cluster \
  --name jenkins-cluster \
  --region us-east-2 \
  --without-nodegroup
```

> **Note:** Cluster creation takes approximately 15–20 minutes. Kubeconfig is saved to `/var/lib/jenkins/.kube/config`.

### Save Kubeconfig

```bash
cat /var/lib/jenkins/.kube/config
```

Save this output to a file named `kubeconfig.txt` on your local machine — it will be needed for Jenkins credentials.

### Create Node Group

```bash
eksctl create nodegroup \
  --cluster jenkins-cluster \
  --region us-east-2 \
  --name jenkins-nodegroup \
  --node-type t3.small \
  --nodes 2
```

---

## 9. Set Up GitHub Repository

Open a terminal on your **VM-lab** and configure git:

```bash
git config --global user.name "your-github-username"
git config --global user.email "your-email@example.com"
```

1. Create a **private** GitHub repository named `springboot-jenkins-eks`
2. Upload all project files to the repository
3. In `eks-deploy-k8s.yaml`, replace `<<REPOSITORY URI>>` with your ECR repository URI

---

## 10. Configure Maven Build Tool and Install Jenkins Plugins

### Add Maven

1. Go to **Manage Jenkins → Tools**
2. Under **Maven installations**, click **Add Maven**
3. Name: `Maven3`
4. Click **Save**

### Install Plugins

Go to **Manage Jenkins → Plugins → Available plugins** and install the following (without restart):

- `Kubernetes CLI`
- `Amazon ECR`
- `Docker`
- `Docker Pipeline`

---

## 11. Configure Jenkins Credentials

Go to **Manage Jenkins → Credentials → Global → Add Credentials**

### GitHub Credentials

| Field    | Value               |
|----------|---------------------|
| Kind     | Username with password |
| Username | Your GitHub username |
| Password | Your GitHub PAT (Personal Access Token) |
| ID       | `github-credentials` |

### Kubernetes Credentials

| Field | Value          |
|-------|----------------|
| Kind  | Secret file    |
| File  | Upload `kubeconfig.txt` |
| ID    | `K8S`          |

---

## 12. Create and Execute the Jenkins Pipeline

1. Click **New Item**
2. Name: `springboot-jenkins-eks`
3. Type: **Pipeline** → Click **OK**
4. Description:
   ```
   Automation of Spring Boot microservices builds using Jenkins pipeline and deploy it on AWS EKS Cluster.
   ```
5. Under **Pipeline**, paste the contents of `jenkins_script.txt`
6. Update the ECR repository URI and GitHub repo details in the script
7. Click **Save**
8. Click **Build Now**

A successful run will dockerize the Spring Boot application and deploy it to the EKS cluster.

---

## 13. Access the Deployed Application

1. Go to **EC2 → Load Balancers**
2. Locate the **Classic Load Balancer** created by the Kubernetes service
3. Copy the **DNS name** and open it in your browser

Your Spring Boot application should now be live and accessible.

---

## Assessment Checklist

| # | Task | Marks |
|---|------|-------|
| 1 | EC2 instance `jenkins-server` created | 5 |
| 2 | IAM role `ec2-eks-role` created and attached | 5 |
| 3 | Security group created and attached with correct inbound rules | 5 |
| 4 | Jenkins installed and accessible on port 8080 | 5 |
| 5 | Plugins installed: Docker, Docker Pipeline, Kubernetes CLI | 10 |
| 6 | ECR repository created with Dockerized Spring Boot image | 10 |
| 7 | Docker volume `jenkins_data` created | 10 |
| 8 | Docker network `jenkins_network` created | 10 |
| 9 | Jenkins pipeline executed successfully | 10 |
| 10 | EKS cluster `jenkins-cluster` created | 10 |
| 11 | Node group `jenkins-nodegroup` created | 10 |
| 12 | Kubernetes deployment successful | 10 |
| | **Total** | **100** |
access token : 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
