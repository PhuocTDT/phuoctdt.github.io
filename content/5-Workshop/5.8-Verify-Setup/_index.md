---
title: "Verify Setup"
date: 2026-01-09T16:03:43+07:00
weight: 30
chapter: false
pre: " <b> 5.8. </b> "
---

## Phase 6: ECS Fargate Container Deployment

In this phase, you will deploy the FastAPI self-hosted backend on Amazon ECS Fargate within a VPC, with Application Load Balancer and Auto Scaling.

#### Architecture Overview

The containerized layer runs alongside the serverless Amplify backend, providing additional compute capacity for custom AI inference or heavy processing tasks that exceed Lambda's 15-minute timeout.

```
Internet → ALB (Public Subnet) → ECS Fargate (Private Subnet) → VPC Endpoints → AWS Services
```

#### Step 1: Create Dockerfile

```dockerfile
# backend/Dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

#### Step 2: Create ECR Repository

```bash
aws ecr create-repository --repository-name nutritrack-api --region ap-southeast-2

# Build and push Docker image
aws ecr get-login-password --region ap-southeast-2 | docker login --username AWS --password-stdin YOUR_ACCOUNT.dkr.ecr.ap-southeast-2.amazonaws.com
docker build -t nutritrack-api .
docker tag nutritrack-api:latest YOUR_ACCOUNT.dkr.ecr.ap-southeast-2.amazonaws.com/nutritrack-api:latest
docker push YOUR_ACCOUNT.dkr.ecr.ap-southeast-2.amazonaws.com/nutritrack-api:latest
```

#### Step 3: Create VPC and ECS Resources

The architecture uses:
- **VPC** in `ap-southeast-2` with 2 AZs
- **Public subnets** for ALB
- **Private subnets** for Fargate tasks
- **VPC Endpoints** for secure AWS service access (S3, Bedrock, ECR)
- **Auto Scaling** based on CPU/memory utilization

#### Step 4: Configure ECS Task Definition

```json
{
  "family": "nutritrack-api",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "containerDefinitions": [{
    "name": "nutritrack-api",
    "image": "YOUR_ACCOUNT.dkr.ecr.ap-southeast-2.amazonaws.com/nutritrack-api:latest",
    "portMappings": [{ "containerPort": 8000, "protocol": "tcp" }],
    "environment": [
      { "name": "AWS_REGION", "value": "ap-southeast-2" }
    ]
  }]
}
```

#### Step 5: Set Up ALB and Auto Scaling

Configure:
- ALB health check: `GET /health` → HTTP 200
- Target tracking scaling: CPU utilization at 70%
- Minimum: 1 task, Maximum: 4 tasks

#### Verification

```bash
curl http://YOUR-ALB-DNS.ap-southeast-2.elb.amazonaws.com/health
# Expected: {"status": "healthy"}
```

> 🎯 **Checkpoint:** FastAPI container is running on ECS Fargate, accessible via ALB, and Auto Scaling is configured.
