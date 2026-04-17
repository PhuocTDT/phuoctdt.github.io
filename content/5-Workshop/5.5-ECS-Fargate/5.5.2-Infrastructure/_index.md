---
title: "Supporting Infrastructure"
date: 2026-01-09T16:03:43+07:00
weight: 2
pre: " <b> 5.5.2. </b> "
menuTitle: "Infrastructure"
---

This section guides you through setting up three critical infrastructure components for the ECS Fargate cluster: **S3 Bucket** for caching, **Secrets Manager** for secure API key management, and **IAM Roles** for execution permissions.

> **Prerequisite:** Completed [5.5.1 Network Infrastructure (VPC & Network)](../5.5.1-VPC-Network/).

---

## 1. S3 Bucket (Cache Layer)

We need an S3 Bucket to cache results from external APIs (USDA, OpenFoodFacts, Avocavo). This reduces latency and saves API call costs.

### 1.1 Create S3 Bucket
1. Go to AWS Console → **S3** → **Create bucket**.
2. **Bucket name**: `nutritrack-cache-xxxx` (Must be globally unique).
3. **Region**: `ap-southeast-2`.
4. **Block all public access**: ✅ On (Default).
5. Click **Create bucket**.

---

## 2. Secrets Manager

Secrets Manager stores API keys securely and automatically injects them into the container as environment variables, avoiding plaintext exposure in source code.

### 2.1 Create Secret
1. Go to **Secrets Manager** → **Store a new secret**.
2. Select **Secret type**: `Other type of secret`.
3. Add the following **Key/value** pairs:
   - `USDA_API_KEY`: <Your Key>
   - `AVOCAVO_API_KEY`: <Your Key>
   - `NUTRITRACK_API_KEY`: <Shared JWT secret between Lambda and ECS>
4. Name the Secret: `nutritrack/prod/api-keys`.
5. After saving, copy the **ARN** of this Secret.

---

## 3. IAM Roles for ECS

ECS uses two distinct roles for different purposes:

| Role Name | Who uses it | Purpose |
| :--- | :--- | :--- |
| **`ecsTaskExecutionRole`** | AWS ECS Agent | Pull Docker image, send logs to CloudWatch, read Secrets Manager. |
| **`ecsTaskRole`** | App inside container | Invoke Amazon Bedrock, read/write to S3 Cache. |

### 3.1 Configure `ecsTaskExecutionRole`
1. Find or create a role named `ecsTaskExecutionRole`.
2. Ensure it has the managed policy: `AmazonECSTaskExecutionRolePolicy`.
3. Add an **Inline Policy** (JSON) to allow reading the Secret ARN copied above.

### 3.2 Create `ecsTaskRole`
1. Create a new role with Trusted Entity: `Elastic Container Service Task`.
2. Attach an **Inline Policy** allowing Bedrock and S3 Cache access.

---

## Next Steps:
- [5.5.3 NAT Optimization (NAT Instance)](../5.5.3-NAT-Instance/)
- [5.5.4 Fargate & ALB Deployment](../5.5.4-Fargate-ALB/)
