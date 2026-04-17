---
title: "Container Layer (ECS Fargate)"
date: 2026-01-09T16:03:43+07:00
weight: 45
chapter: false
pre: " <b> 5.5. </b> "
menuTitle: "ECS Container Layer"
---

The ECS Fargate layer runs a containerized FastAPI service in parallel with the Amplify serverless backend. It handles tasks not suitable for Lambda, such as long-running data processing, custom AI inference, or processes requiring persistent connections.

## System Architecture

![NutriTrack API VPC Architecture](images/only-nutritrack-api-vpc.drawio.svg)

The ECS tasks run in a **Private Subnet** for security, while an **Application Load Balancer (ALB)** resides in the **Public Subnet** to receive internet requests. Tasks access other AWS services through a **NAT Instance** (saving 70% cost compared to a NAT Gateway) or an **S3 Gateway Endpoint** (free).

## Estimated Cost

| Component | Estimated Monthly Cost |
| :--- | :--- |
| 2× `t4g.nano` NAT Instance | ≈$9 |
| 2× Fargate Task (0.5 vCPU / 1 GB) | ≈$17 |
| Application Load Balancer (ALB) | ≈$16 |
| CloudWatch Logs (5 GB) | ≈$2 |
| **Total** | **≈$44** |

> [!TIP]
> Using a NAT Instance instead of a NAT Gateway can save you approximately $32 per month, which is significant for startups or experimental projects.

## Implementation Steps:

1. [5.5.1 VPC & Network Infrastructure](5.5.1-VPC-Network/)
2. [5.5.2 Supporting Infrastructure (S3, Secrets, IAM)](5.5.2-Infrastructure/)
3. [5.5.3 NAT Optimization (NAT Instance)](5.5.3-NAT-Instance/)
4. [5.5.4 Fargate & ALB Deployment](5.5.4-Fargate-ALB/)

---

[Continue to 5.6 CI/CD Deployment](../5.6-CICD/)
