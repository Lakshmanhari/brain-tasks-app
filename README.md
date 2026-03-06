

# Application Deployment

*(Deploy the given React application to a production ready state)*

# 🚀 Brain Tasks App – DevOps CI/CD Deployment on AWS EKS

## 📖 Project Overview

This project demonstrates a complete production-ready DevOps CI/CD pipeline for deploying a React application using AWS services.
The application is containerized using Docker and deployed into a Kubernetes cluster (Amazon EKS) using AWS CodePipeline and CodeBuild.

---

# 🧭 Architecture

```
GitHub Repository
        ↓
AWS CodePipeline
        ↓
AWS CodeBuild
        ↓
Amazon ECR (Docker Image)
        ↓
Amazon EKS (Kubernetes Cluster)
        ↓
AWS LoadBalancer
        ↓
End Users
```

---

# 🛠 Technologies Used

• React.js (Frontend Application)
• Docker
• Kubernetes
• AWS ECR
• AWS EKS
• AWS CodeBuild
• AWS CodePipeline
• CloudWatch Logs
• Git & GitHub

---

# 📂 Project Structure

```
Brain-Tasks-App/
│
├── src/
├── public/
├── package.json
│
├── Dockerfile
├── nginx.conf
│
├── k8s/
│   ├── deployment.yaml
│   └── service.yaml
│
├── buildspec.yml
└── README.md
```

---

# STEP - 1 (Installing Dependency in Cloud Shell)

```
aws --version
kubectl version --client
eksctl version
```

Note: If eksctl is not installed

```
# for ARM systems, set ARCH to: `arm64`, `armv6` or `armv7`
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH

curl -sLO https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz

# (Optional) Verify checksum
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATFORM | sha256sum --check

tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz

sudo install -m 0755 /tmp/eksctl /usr/local/bin && rm /tmp/eksctl
```

---

# 🚀 STEP 2 — Clone & Prepare Application

Clone repo and add DevOps files
CloudShell or local

```
git clone https://github.com/Vennilavanguvi/Brain-Tasks-App.git
cd Brain-Tasks-App
```

---

# 🐳 STEP 2 — Dockerize Application

The container image is created using Docker because Kubernetes runs applications as containers rather than raw source code. Docker reads the Dockerfile and builds the image layer by layer, packaging the application and its dependencies into a container that can be deployed consistently across environments.

```
nano Dockerfile
```

### Dockerfile

```
FROM nginx:alpine
RUN rm /etc/nginx/conf.d/default.conf
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY dist/ /usr/share/nginx/html/
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

---

# 📦 STEP 3 — Create Container Registry

The image storage is created using Amazon Elastic Container Registry because Kubernetes clusters need a container registry to pull Docker images during deployment.

ECR works similarly to how Git stores code, but instead it securely stores and manages container images so that the Kubernetes cluster can retrieve and run them.

### Create repo

brain-tasks-app

Visibility
✔️ Private

Image tag mutability
✔️ Mutable (default)

Scan settings
Optional
✔️ Enable scan (good practice)

👉 Checks image vulnerabilities
👉 Allows overwriting latest tag

Encryption
Default (AES-256)

👉 AWS manages encryption

Copy repo URI

```
URI_Number.dkr.ecr.ap-south-1.amazonaws.com/brain-tasks-app
```

(to use in the .yaml file in k8)

---

# Login to ECR

```
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com
```

---

# Build & Tag & Push image

```
docker build -t brain-tasks-app .
docker tag brain-tasks-app:latest <ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/brain-tasks-app:latest
docker push <ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/brain-tasks-app:latest
```

---

# ☸️ Kubernetes (Amazon EKS)

## Create Cluster

```
eksctl create cluster \
--name brain-tasks-cluster \
--region ap-south-1 \
--nodegroup-name workers \
--node-type t3.small \
--nodes 2
```

---

# Kubernetes Manifests

Create folder

```
mkdir k8s
```

```
nano k8s/deployment.yaml
```

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: brain-tasks-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: brain-tasks
  template:
    metadata:
      labels:
        app: brain-tasks
    spec:
      containers:
      - name: brain-tasks
        image: URI_Number.dkr.ecr.ap-south-1.amazonaws.com/brain-tasks-app /// Copy URI from ECR repo
        ports:
        - containerPort: 80
```

Deployment.yaml

• 2 replicas
• Pulls image from ECR
• Container Port: 80

---

# 🌐 service.yaml

```
nano k8s/service.yaml
```

```
apiVersion: v1
kind: Service
metadata:
  name: brain-tasks-service
spec:
  type: LoadBalancer
  selector:
    app: brain-tasks
  ports:
    - protocol: TCP
      port: 3000
      targetPort: 80
```

Service YAML

• Type: LoadBalancer
• Service Port: 3000
• Target Port: 80

---

# 8️⃣ Deploy to Kubernetes

```
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
```

Check

```
kubectl get nodes
kubectl get pods
kubectl get svc
```

Look for

EXTERNAL-IP

---

# nginx.conf

```
nano nginx.conf
```

```
server {
    listen 80;
    location / {
        root /usr/share/nginx/html;
        index index.html;
        try_files $uri /index.html;
    }
}
```

---

# Access Application

You can access your app exactly like this

```
http://<LoadBalancer-DNS>:3000
```

For your deployment specifically

```
http://a3e4ad7a7208642a6ad198c9386ded1f-337684612.ap-south-1.elb.amazonaws.com:3000
```

---

# Your NodePort details

From your service

```
3000:30362/TCP
```

👉 This means

Service port → 3000
NodePort → 30362

---

# How to access via NodePort

You must use

```
http://<EC2-Node-Public-IP>:30362
```

---

# Push Project to GitHub

```
git remote -v
git remote remove origin
git remote add origin https://github.com/Lakshmanhari/brain-tasks-app.git

git init
git add .
git commit -m "final project setup"
git branch -M main
git remote add origin <your-repo-url>
git push -u origin main
```

---

# Create buildspec.yml

```
nano buildspec.yml
```

```
version: 0.2
env:
  variables:
    ECR_REPO: "brain-tasks-app"
    IMAGE_TAG: "latest"
    ACCOUNT_ID: "xxxxxxxxxxx"
    AWS_REGION: "ap-south-1"
phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
  build:
    commands:
      - echo Building and Pushing Docker image...
      - docker build -t $ECR_REPO:$IMAGE_TAG .
      - docker tag $ECR_REPO:$IMAGE_TAG $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:$IMAGE_TAG
      - docker push $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:$IMAGE_TAG
  post_build:
    commands:
      - echo "Forcing credential refresh..."
      - aws eks update-kubeconfig --region $AWS_REGION --name $CLUSTER_NAME
      - echo "Deploying with validation disabled..."
      - kubectl apply -f k8s/deployment.yaml -f k8s/service.yaml --validate=false
      - echo "Updating image..."
      - kubectl set image deployment/brain-tasks-deployment brain-tasks=$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:$IMAGE_TAG
      - echo "Deployment Complete!"
artifacts:
  files:
    - 'k8s/deployment.yaml'
    - 'k8s/service.yaml'
```

---

# CodeBuild

Create Project

Project Name
brain-tasks-build

Source Provider
GitHub

Repository
Repository in my GitHub account

Repository URL

```
https://github.com/Lakshmanhari/brain-tasks-app
```

Branch
main

Explanation
CodeBuild fetches the source code from the GitHub repository whenever a pipeline execution is triggered.

GitHub Authentication
GitHub authentication is configured using a Personal Access Token (PAT).

Service Role Permissions
A service role is required for CodeBuild to interact with AWS services such as ECR, EKS, and CloudWatch.

Selected Service Role

```
arn:aws:iam::Account_id:role/CodeBuild-BrainTasks-service-Role
```

---

# 🔁 CI/CD Pipeline

Go to

AWS Console → Developer Tools → CodePipeline → Create Pipeline

Select

Build custom pipeline

---

# Step 2 — Pipeline Settings

Pipeline Name
brain-tasks-pipeline

Execution Mode
Queued

Service Role
New service role

AWS automatically creates a service role that allows CodePipeline to interact with other AWS services.

Role created

```
AWSCodePipelineServiceRole-ap-south-1-brain-tasks-pipeline
```

---

# Step 3 — Source Stage

In this stage the pipeline connects to GitHub to fetch the application code.

Source Provider
GitHub (via GitHub App)

Repository
Lakshmanhari/brain-tasks-app

Branch
main

Artifact Format
CodePipeline default

Whenever new code is pushed to the GitHub repository, the pipeline automatically starts.

---

# Step 4 — Build Stage

In the build stage the pipeline triggers AWS CodeBuild.

Build Provider
AWS CodeBuild

Project Name
brain-tasks-build

---

# Step 5 — Deploy Stage

In the deploy stage the application is deployed to the Kubernetes cluster.

Deployment happens using Kubernetes manifests

```
k8s/deployment.yaml
k8s/service.yaml
```

These files create

• Kubernetes Deployment
• Kubernetes Service
• LoadBalancer to expose the application

The deployment runs on the Amazon EKS cluster.

Webkook Event

Check the webhook check box to start your pipeline on push and pull request events

Create pipeline

---

# Enable EKS API Authentication

```
aws eks update-cluster-config \
    --name brain-tasks-cluster \
    --access-config authenticationMode=API_AND_CONFIG_MAP \
    --region ap-south-1
```

What this does

This enables the new EKS access system.

There are two ways EKS gives access

1. Old method → aws-auth ConfigMap
2. New method → Access Entry API

API_AND_CONFIG_MAP enables both methods.

So now you can register IAM roles using CLI.

---

# Create Access Entry

```
aws eks create-access-entry \
    --cluster-name brain-tasks-cluster \
    --principal-arn arn:aws:iam::216989110867:role/CodeBuild-BrainTasks-service-Role \
    --region ap-south-1 \
    --type STANDARD
```

What this does

This registers the IAM role inside EKS.

---

# Give the Role Permission

```
aws eks associate-access-policy \
    --cluster-name brain-tasks-cluster \
    --principal-arn arn:aws:iam::216989110867:role/CodeBuild-BrainTasks-service-Role \
    --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
    --access-scope type=cluster \
    --region ap-south-1
```

What this does

This gives admin permission to the role inside Kubernetes.

---

# Check Registered Roles

```
aws eks list-access-entries \
    --cluster-name brain-tasks-cluster \
    --region ap-south-1
```

This shows all IAM roles that can access the cluster.

---

# CI/CD Flow Summary

### 1️⃣ Source Stage

• Connected to GitHub repository

### 2️⃣ Build Stage (CodeBuild)

• Logs into ECR
• Builds Docker image
• Pushes image to ECR
• Updates kubeconfig
• Deploys to EKS using kubectl

### 3️⃣ Deploy Stage

• Deploys to EKS cluster
• Updates running image

---

# 🧾 buildspec.yml Summary

• pre_build → ECR login
• build → Docker build & push
• post_build → kubectl deployment

---

# Final Output
kubectl get nodes
kubectl get pods
kubectl get svc -o wide
```

Look for

Check

You can access your app exactly like this

```
http://<LoadBalancer-DNS>:3000
```

For your deployment specifically

```
http://a3e4ad7a7208642a6ad198c9386ded1f-337684612.ap-south-1.elb.amazonaws.com:3000
```

---

# Your NodePort details

From your service

```
3000:30362/TCP
```

👉 This means

Service port → 3000
NodePort → 30362

---

# How to access via NodePort

You must use

```
http://<EC2-Node-Public-IP>:30362
```

and check the app is accesible

---

🧱 Final Flow Architecture Diagram

Developer (Git Push)
        │
        ▼
     GitHub
        │
        ▼
   CodePipeline
        │
        ▼
    CodeBuild
        │
        ▼
     Docker Build
        │
        ▼
   Amazon ECR
        │
        ▼
   Amazon EKS
        │
        ▼
 Kubernetes Pods
        │
        ▼
 AWS LoadBalancer
        │
        ▼
     End Users
