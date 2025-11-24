# AWS ECS Fargate Capstone – Containerized Web Application

This project demonstrates deploying a **containerized web application** (Python Flask or Node.js) on **AWS ECS Fargate** using **Terraform** for Infrastructure as Code (IaC). The deployment includes a custom VPC, ECS cluster, task definitions, security groups, and an Application Load Balancer (ALB) for high availability.

---

## Scenario

Your organization is developing a microservice-based web application. You have been asked to deploy a simple containerized web service on AWS using Terraform. The setup must be **scalable, repeatable, and secure**.
**#Architecture Overview**
                ┌─────────────────────────────┐
                │       Internet Users        │
                └─────────────┬─────────────┘
                              │ HTTP/HTTPS (80/443)
                              ▼
                ┌─────────────────────────────┐
                │   Application Load Balancer │
                │      (ALB – Public)        │
                └─────────────┬─────────────┘
                              │
                   ┌──────────┴──────────┐
                   │                     │
           ┌───────────────┐     ┌───────────────┐
           │  Target Group │     │  Target Group │
           │   (Private)   │     │   (Private)   │
           └──────┬────────┘     └──────┬────────┘
                  │                     │
           ┌───────────────┐     ┌───────────────┐
           │ ECS Fargate   │     │ ECS Fargate   │
           │ Task (Flask) │     │ Task (Flask) │
           └───────────────┘     └───────────────┘
                  │                     │
                  └─────────────┬───────┘
                                │
                        ┌───────────────┐
                        │  VPC & Subnets │
                        │ - 2 Public     │
                        │ - 2 Private    │
                        └───────────────┘
                                │
                       ┌─────────────────┐
                       │ Internet & NAT  │
                       │ Gateways        │
                       └─────────────────┘
**#Repository Structure**
aws-ecs-fargate-capstone/
│
├── python-docker/                   # Containerized web application
│   ├── app.py                       # Flask or Node.js application
│   ├── Dockerfile                   # Docker image definition
│   └── requirements.txt             # Python dependencies
│
├── terraform/                       # Terraform configuration files
│   ├── vpc.tf                       # VPC, subnets, gateways
│   ├── security_groups.tf           # ALB & ECS security groups
│   ├── ecs.tf                       # ECS cluster, task definitions, services
│   ├── alb.tf                       # Application Load Balancer and target groups
│   ├── variables.tf                 # Terraform input variables
│   ├── locals.tf                     # Derived values and locals
│   ├── outputs.tf                   # Terraform outputs
│   └── terraform.tfvars             # Environment-specific variable overrides
│
├── README.md                         # Project documentation & instructions
└── .gitignore                        # Optional: ignores files like .terraform/, __pycache__/

## Requirements Implemented

### 1. AWS Infrastructure Setup
Using Terraform:

- Create a VPC with **2 public and 2 private subnets** across multiple availability zones
- Configure an **Internet Gateway**, **NAT Gateway**, and appropriate route tables
- Set up **Security Groups**:
  - HTTP (port 80) access for ALB
  - SSH (port 22) if needed for management
  - Internal ECS communication restricted to VPC

### 2. ECS Setup
- Create an **ECS Cluster** using **Fargate**
- Define **Task Definitions**:
  - Container image from **Amazon ECR** or **Docker Hub**
  - Port mapping **80:80**
  - CPU and memory resource limits
- Deploy an **ECS Service** with **at least 2 tasks** for high availability

### 3. Load Balancing
- Deploy an **Application Load Balancer (ALB)** to route traffic to ECS tasks
- Configure **target groups and listeners** for proper routing and health checks

### 4. Automation and Outputs
- Terraform outputs include:
  - `alb_dns_name` – public endpoint of the app
  - `ecs_cluster_name` – ECS cluster name
  - `running_task_count` – number of tasks currently running
- Terraform variables allow environment selection (e.g., `dev` or `prod`)

---

## Prerequisites

- AWS account with permissions for: **ECR, ECS, IAM, VPC, EC2, ELB, CloudWatch Logs**
- Tools installed:
  - Terraform >= 1.5
  - AWS CLI v2
  - Docker
- AWS credentials configured locally (e.g., profile `default`):
```bash
aws sts get-caller-identity --profile default
Repo Layout
python-docker/app.py: Flask app (or Node.js)

python-docker/Dockerfile: Builds container image on port 80

variables.tf / locals.tf / ecs.tf / alb.tf / vpc.tf: Terraform infrastructure

terraform.tfvars: Environment-specific overrides

One-time Setup
Create or verify ECR repository:

bash
Copy code
aws ecr describe-repositories \
  --repository-names web-app \
  --region us-east-1 \
  --profile default || \
aws ecr create-repository \
  --repository-name web-app \
  --region us-east-1 \
  --profile default
Authenticate Docker to ECR:

bash
Copy code
aws ecr get-login-password --region us-east-1 --profile default \
| docker login --username AWS --password-stdin <account_id>.dkr.ecr.us-east-1.amazonaws.com
Build and Push Container Image
bash
Copy code
cd python-docker
docker build -t web-app:latest -f Dockerfile .

export NEW_TAG="v$(date +%Y%m%d-%H%M%S)"

docker tag web-app:latest <account_id>.dkr.ecr.us-east-1.amazonaws.com/web-app:$NEW_TAG
docker push <account_id>.dkr.ecr.us-east-1.amazonaws.com/web-app:$NEW_TAG
Configure Terraform Variables
Edit terraform.tfvars:

hcl
Copy code
aws_region      = "us-east-1"
aws_profile     = "default"
environment     = "dev"
ecr_image_tag   = "<your NEW_TAG from above>"
Note: ecr_image_url is derived automatically in locals.tf.

Deploy Infrastructure
bash
Copy code
terraform init
terraform plan
terraform apply --auto-approve
Outputs:

alb_dns_name – public endpoint

ecs_cluster_name – ECS cluster

running_task_count – number of active tasks

Open the app in your browser:

cpp
Copy code
http://<alb_dns_name>
Updating Application
Edit python-docker/app.py

Rebuild and push a new image tag:

bash
Copy code
export NEW_TAG="v$(date +%Y%m%d-%H%M%S)"
docker build -t web-app:latest -f Dockerfile .
docker tag web-app:latest <account_id>.dkr.ecr.us-east-1.amazonaws.com/web-app:$NEW_TAG
docker push <account_id>.dkr.ecr.us-east-1.amazonaws.com/web-app:$NEW_TAG
Update terraform.tfvars with the new ecr_image_tag and run:

bash
Copy code
terraform apply --auto-approve
Clean Up
Destroy all resources to avoid charges:

bash
Copy code
terraform destroy --auto-approve
Notes
ECS Cluster: web-app-<env>

ECS Service: web-app-service

Image URL constructed via locals.tf from account_id, aws_region, ecr_repo_name, and ecr_image_tag

yaml
Copy code

---  THANK YOU ----


