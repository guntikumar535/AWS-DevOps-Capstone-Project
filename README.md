Step-by-Step Guide: Deploy a Containerized Web App on AWS using Terraform & ECS

**Step 0: Prerequisites**
Before you start, ensure you have:
AWS Account with permissions for:
VPC, EC2, NAT, ECS, ALB, IAM, ECR, CloudWatch

**Tools Installed:**
Terraform >= 1.5
AWS CLI v2
Docker

#Configured AWS CLI with a profile:
aws configure --profile default

#Basic knowledge of Terraform, Docker, and AWS ECS concepts.

**Step 1: Create Your Project Structure**

Set up a local folder for your project:
aws-ecs-fargate-capstone/
├── python-docker/
│   ├── app.py
│   ├── Dockerfile
│   └── requirements.txt
└── terraform/
    ├── vpc.tf
    ├── security_groups.tf
    ├── ecs.tf
    ├── alb.tf
    ├── variables.tf
    ├── locals.tf
    ├── outputs.tf
    └── terraform.tfvars
python-docker/ → your web app
terraform/ → all infrastructure code

**Step 2: Build Your Web Application**
If using Flask (Python):
app.py

from flask import Flask
app = Flask(__name__)

@app.route("/")
def home():
    return "Hello from ECS Fargate!"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=80)

#requirements.txt
flask

**Dockerfile**
 FROM python:3.10-slim
 WORKDIR /app
 COPY requirements.txt .
 RUN pip install -r requirements.txt
 COPY . .
 EXPOSE 80
 CMD ["python", "app.py"]

**Test locally:**
 cd python-docker
 docker build -t web-app:latest .
 docker run -p 8080:80 web-app:latest

Open **http://localhost:8080** to verify it works.

**Step 3: Push Docker Image to Amazon ECR**

#Create an ECR repository:
 aws ecr create-repository \
   --repository-name web-app \
   --region us-east-1 \
   --profile default

#Authenticate Docker to ECR:
 aws ecr get-login-password --region us-east-1 --profile default \
 | docker login --username AWS --password-stdin <account_id>.dkr.ecr.us-east-1.amazonaws.com


#Build, tag, and push image:
 cd python-docker
 export NEW_TAG="v1"
 docker build -t web-app:latest .
 docker tag web-app:latest <account_id>.dkr.ecr.us-east-1.amazonaws.com/web-app:$NEW_TAG
 docker push <account_id>.dkr.ecr.us-east-1.amazonaws.com/web-app:$NEW_TAG

**Step 4: Write Terraform Code**
1 VPC Setup (vpc.tf)
 Create VPC, 2 public & 2 private subnets
 Configure Internet Gateway & NAT Gateway
 Set up Route Tables
2 Security Groups (security_groups.tf)
 ALB SG → Allow HTTP 80 from internet
 ECS SG → Allow traffic from ALB SG
 Optional SSH SG → Allow port 22 from your IP
3 ECS Setup (ecs.tf)
 Create ECS cluster using Fargate
 Define ECS Task Definition:
   -Image → your ECR image
   -CPU, memory, port 80 mapping
 Create ECS Service with 2 tasks minimum
4 Load Balancer (alb.tf)
 Application Load Balancer in public subnets
 Target group → ECS tasks
 Listeners → HTTP 80 → target group
 Health checks → path /
5 Variables & Locals
 variables.tf → define AWS region, environment, ecr_image_tag
 locals.tf → construct ECR image URL from variables
6 Outputs (outputs.tf)
 output "alb_dns_name" {
   value = aws_lb.web_app.dns_name
 }
 output "ecs_cluster_name" {
   value = aws_ecs_cluster.web_cluster.name
 }
 output "running_task_count" {
   value = aws_ecs_service.web_service.desired_count
 }

**Step 5: Deploy with Terraform**

#Initialize Terraform:
 cd terraform
 terraform init

#Validate plan:
 terraform plan

#Apply:
 terraform apply --auto-approve

Terraform will create all resources and output:
 ALB DNS → visit your app
 ECS cluster name
 Number of running tasks

**Step 6: Update Application**
 Modify app.py
 Build & push a new Docker tag
 Update terraform.tfvars with ecr_image_tag
 Apply Terraform again:
  terraform apply --auto-approve
ECS will create a new task definition and roll out automatically.

**Step 7: Test Application**
 Open the ALB DNS in browser
 Check CloudWatch Logs for ECS task logs
 Confirm high availability by stopping a task → ALB routes traffic to remaining tasks

**Step 8: Clean Up**
 To avoid charges:
 terraform destroy --auto-approve

**Tips for Success**
-Use immutable Docker tags for deterministic deployments.
-Verify subnets & security groups to avoid ALB 5xx errors.
-Keep Terraform modular (vpc.tf, ecs.tf, alb.tf) for clarity.
-Always check CloudWatch logs if ECS service fails.
-Use Terraform outputs to find endpoints and ECS cluster info.
