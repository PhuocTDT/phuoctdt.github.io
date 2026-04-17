---
title: "Fargate & ALB Deployment"
date: 2026-01-09T16:03:43+07:00
weight: 4
pre: " <b> 5.5.4. </b> "
menuTitle: "Fargate & ALB"
---

This is the final step to complete the system: Build the Docker image, push it to a registry, and set up the ECS Fargate cluster and Application Load Balancer (ALB) to handle internet traffic.

## 1. Build & Push Docker Image

We need a Docker Image containing the FastAPI (Python) source code, optimized for the AWS Graviton (ARM64) architecture.

### Build and Push to Docker Hub
```bash
# Log in to Docker Hub
docker login

# Build multi-platform (ARM64) and push
docker buildx build \
  --platform linux/arm64 \
  --tag <username>/nutritrack-api:latest \
  --push .
```

---

## 2. Initialize ECS Cluster

1. Go to **ECS Console** → **Clusters** → **Create cluster**.
2. **Cluster name**: `nutritrack-api-cluster`.
3. **Infrastructure**: Select `AWS Fargate (serverless)`.
4. Click **Create**.

---

## 3. Task Definition

The Task Definition specifies which image to run, CPU/RAM resources, and required environment variables.

1. Go to **Task Definitions** → **Create new task definition**.
2. **OS/Architecture**: `Linux/ARM64` (To save costs with Graviton).
3. **CPU**: `1 vCPU`, **Memory**: `2 GB`.
4. **Task Execution Role**: `ecsTaskExecutionRole`.
5. **Container Details**:
   - **Name**: `nutritrack-api-container`
   - **Image**: `<username>/nutritrack-api:latest`
   - **Port mapping**: `8000` (TCP).

---

## 4. Application Load Balancer (ALB)

ALB receives user traffic and distributes it to the ECS containers.

1. **Target Group**: Create `nutritrack-api-tg`, port 8000, type **IP**. Health check path: `/health`.
2. **Load Balancer**: Create an **Internet-facing** ALB, select the Public Subnets created in 5.5.1.
3. **Security Group**: Attach `nutritrack-api-vpc-alb-sg`.
4. **Listener**: Route port 80 traffic to the Target Group.

---

## 5. Security with AWS WAF

Add a security layer (WAF) to prevent brute-force attacks and ensure only the `scan-image` Lambda can call the ALB.

- **Rate Limit**: Max 100 requests per IP per 5 minutes.
- **Custom Header**: Accept only requests with an `Authorization: Bearer <token>` header.

---

## 6. ECS Service

Finally, create a Service to maintain the running tasks.

1. Go to Cluster → **Services** tab → **Create**.
2. **Capacity Provider**: `FARGATE_SPOT` (For maximum cost optimization).
3. **Deployment Configuration**: Select your Task Definition.
4. **Networking**: Select 2 Private Subnets and the ECS Security Group.
5. **Load Balancing**: Select the ALB and Target Group.
6. Click **Create**.

---

## Summary of Achievement:

Your system is now complete:
- A Mobile App connected to Amplify Gen 2.
- Heavy AI tasks processed by FastAPI on ECS Fargate.
- Fully secured within a private VPC with cost-optimized NAT.

---

[Continue to 6.0 Conclusion & Next Steps](../../6-Conclusion/)
