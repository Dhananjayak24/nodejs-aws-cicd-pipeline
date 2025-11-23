# Node.js CI/CD Pipeline using GitHub Actions, Docker, AWS ECR & EC2

This repository contains a fully automated **CI/CD pipeline** that builds, pushes, and deploys a Node.js application to an AWS EC2 instance using **Docker**, **AWS ECR**, and **GitHub Actions**.

This project simulates a real-world DevOps scenario where frequent updates to the application should deploy automatically to production without manual intervention.

---

## ** Project Overview**

You are a DevOps Engineer at **Brightcode Solutions**. Developers push updates to the `main` branch, but manual deployments are slow and inconsistent.

Your goal is to automate:

| Step | Purpose | Tool |
|------|---------|------|
| Containerize App | Package Node.js app | Docker |
| Store Image | Push to registry | AWS ECR |
| Deploy | Pull & run on server | EC2 + Docker |
| Automate Process | Trigger on push | GitHub Actions |

This pipeline ensures consistent deployments, faster iteration, and eliminates manual server management.

---

## ** Technologies Used**

- **Node.js (LTS)**
- **Docker & Docker Compose**
- **AWS Elastic Container Registry (ECR)**
- **AWS EC2 (Ubuntu)**
- **IAM Users & Roles**
- **GitHub Actions (CI/CD)**
- **SSH Deployment**

---

## ** Repository Setup**

### **Step 1: Clone or Initialize Repo**

```sh
git init
git remote add origin git@github.com:Dhananjayak24/nodejs-aws-cicd-pipeline.git
git branch -M main
```

### **Step 2: Create Dockerfile**
Create Dockerfile in root:(where the other src files exist)
```
FROM node:lts-alpine

WORKDIR /app

COPY ./package*.json ./
RUN npm install

COPY . .

EXPOSE 80

CMD ["npm", "start"]
```

### **Step3: Create docker-compose.yml**
```
version: "3.8"

services:
  app:
    image: nodejs-sample-image
    build: .
    ports:
      - "80:80"
```

### **Step 4: Create CI/CD Workflow**
Create folder:
```
.github/workflows/cicd.yml
```

Add:

```
name: CI CD Pipeline

on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  docker-build-push:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ secrets.AWS_REGION }}

    - name: Log in to Amazon ECR
      uses: aws-actions/amazon-ecr-login@v2

    - name: Build Docker image
      run: |
        docker build -t ${{ secrets.ECR_REPO }}:latest .

    - name: Push Docker image to ECR
      run: |
        docker push ${{ secrets.ECR_REPO }}:latest

  deploy:
    runs-on: ubuntu-latest
    needs: docker-build-push

    steps:
      - name: Deploy via SSH to EC2
        uses: appleboy/ssh-action@v1.2.0
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ubuntu
          key: ${{ secrets.SSH_PRIVATE_KEY_EC2 }}
          script: |
            docker compose pull
            docker compose up -d --force-recreate
```

### **Step 5: Set Up AWS Resources**
Create ECR Repository

AWS Console → ECR → Create Repository

Example:
```
coursera/nodejs-docker
```

Copy the repo URI — used later as ECR_REPO.

IAM Setup
Create IAM User for GitHub Actions

Go to IAM → Users → Create User

Name: github-nodejs-user

Attach policy:
AmazonEC2ContainerRegistryFullAccess

Create IAM Role for EC2

IAM → Roles → Create Role

Use case: EC2

Policy: AmazonEC2ContainerRegistryFullAccess

Name: EC2-NODEJS-Access

### **Step 6: Launch EC2 Instance**
settings:

| Setting  | Value                               |
| -------- | ----------------------------------- |
| AMI      | Ubuntu                              |
| IAM Role | `EC2-NODEJS-Access`                 |
| Network  | Allow HTTP traffic                  |
| Key Pair | Create new: `Nodejs-access-key.pem` |

Launch instance.

### **Step 7: SSH into EC2**
If you're using WSL, move .pem into Linux home dir:
```
cp Nodejs-access-key.pem ~/
chmod 400 ~/Nodejs-access-key.pem
```
Then connect:

```
ssh -i ~/Nodejs-access-key.pem ubuntu@<EC2_PUBLIC_IP>
```

### **Step 8: Install Docker & Compose**
Copy install script into EC2:

```
scp -i ~/Nodejs-access-key.pem your_script.sh ubuntu@<IP>:~/
```
Run it:

```
chmod +x your_script.sh
./your_script.sh
```

Success message expected:

```
Setup of Docker, Docker Compose & Docker Credential Helper completed successfully!
```
### **Step 9: Create Docker Compose on EC2**
```
nano docker-compose.yml
```
example:

```
version: "3.8"

services:
  app:
    image: <ECR-URI>:latest
    ports:
      - "80:80"
```

Save and exit.

### **Step 10: Add GitHub Secrets**
| Name                    | Value                   |
| ----------------------- | ----------------------- |
| `AWS_ACCESS_KEY_ID`     | From IAM user           |
| `AWS_SECRET_ACCESS_KEY` | From IAM user           |
| `AWS_REGION`            | ex: `ap-southeast-2`    |
| `ECR_REPO`              | ECR URI                 |
| `SSH_PRIVATE_KEY_EC2`   | Contents of `.pem` file |
| `EC2_HOST`              | Public IP               |

### **Step 11: Push Code**

```
git add .
git commit -m "Initial CI/CD setup"
git push origin main
```
GitHub Actions should trigger automatically.

## **Testing Deployment**

Open browser → enter EC2 public IP:

```
http://<EC2_PUBLIC_IP>
http://3.107.195.7/swagger-ui/index.html

```
You should see the running app.

# **Common Errors & Fixes**

| Issue                                     | Cause                    | Fix                                                |
| ----------------------------------------- | ------------------------ | -------------------------------------------------- |
| `docker buildx build requires 1 argument` | Space between `$` & `{{` | `docker build -t ${{ secrets.ECR_REPO }}:latest .` |
| `unknown instruction: CMD["npm",`         | No space after CMD       | `CMD ["npm", "start"]`                             |
| `docker-compose: command not found`       | Compose v2 installed     | `docker compose`                                   |

## **What You Learned**

- Automating CI/CD end-to-end on AWS

- Using Docker to create portable application builds

- Securely pulling images into EC2 using IAM roles

- Triggering deployments from git commits

- Debugging real DevOps pipeline issues

# **Acknowledgment**

Inspired by a Coursera guided project.
Rewritten for clarity, maintainability, and real-world use.
