---
title: "VPC & Network Setup"
date: 2026-01-09T16:03:43+07:00
weight: 1
pre: " <b> 5.5.1. </b> "
menuTitle: "VPC & Network"
---

This guide helps you set up the AWS networking foundation for the NutriTrack API: a private VPC, 4 subnets across 2 AZs, Internet Gateway, Route Tables, 3 Security Groups, and an S3 Gateway VPC Endpoint.

> **Region:** `ap-southeast-2` (Sydney) | **Estimated Time:** 45–60 minutes

## Why this architecture?

| Decision | Reason |
| :--- | :--- |
| **ECS Private Subnet** | Containers have no public IP → enhanced security, avoiding direct attacks. |
| **ALB Internet-facing** | Single point of entry from the internet, hiding container IPs. |
| **NAT Instance** | Saves **~70%** in costs compared to NAT Gateway (~$10/mo vs $34/mo). |
| **S3 Gateway VPCE** | **Free** access to S3, bypassing the internet and NAT bandwidth costs. |
| **High Availability (HA)** | Deployed across 2 Availability Zones (AZs) to ensure system uptime. |

---

## 1. Create VPC

**VPC (Virtual Private Cloud)** is your private network on AWS. All resources like ECS, ALB, and NAT Instances will reside here.

### 1.1 Initialize VPC
1. Log in to the AWS Console, select Region **`ap-southeast-2`**.
2. Search for **VPC**.
3. Select **Your VPCs** and click **Create VPC**.
4. Configuration:
   - **Resources to create**: `VPC only`
   - **Name tag**: `nutritrack-api-vpc`
   - **IPv4 CIDR**: `10.0.0.0/16`
5. Click **Create VPC**.

### 1.2 Enable DNS for VPC
1. Select the created VPC → **Actions** → **Edit VPC settings**.
2. Check both boxes:
   - ✅ `Enable DNS resolution`
   - ✅ `Enable DNS hostnames`
3. Click **Save**.

---

## 2. Create Subnets

We will create **4 subnets** distributed across 2 AZs (`2a` and `2c`):

| Subnet Name | Availability Zone | CIDR | Subnet Type |
| :--- | :--- | :--- | :--- |
| `nutritrack-api-vpc-public-alb01` | ap-southeast-2a | `10.0.1.0/24` | Public (For ALB & NAT 1) |
| `nutritrack-api-vpc-public-alb02` | ap-southeast-2c | `10.0.2.0/24` | Public (For ALB & NAT 2) |
| `nutritrack-api-vpc-private-ecs01` | ap-southeast-2a | `10.0.3.0/24` | Private (For ECS Tasks) |
| `nutritrack-api-vpc-private-ecs02` | ap-southeast-2c | `10.0.4.0/24` | Private (For ECS Tasks) |

**Note:** After creation, select the 2 Public Subnets and enable **`Enable auto-assign public IPv4 address`** in the Subnet settings.

---

## 3. Internet Gateway & Route Tables

### 3.1 Internet Gateway (IGW)
1. Create an IGW named `nutritrack-api-igw`.
2. **Attach to VPC** `nutritrack-api-vpc`.

### 3.2 Public Route Table
1. Create a Route Table named `nutritrack-api-public-rt`.
2. In the **Routes** tab, add: `0.0.0.0/0` → Target: `Internet Gateway`.
3. In the **Subnet associations** tab, associate both Public Subnets.

---

## 4. Security Groups (SG)

### 4.1 ALB Security Group (`nutritrack-api-vpc-alb-sg`)
- **Inbound Rule**: Allow `HTTP` (Port 80) from `0.0.0.0/0`.

### 4.2 ECS Security Group (`nutritrack-api-vpc-ecs-sg`)
- **Inbound Rule**: Allow `Custom TCP` (Port 8000) from Source: `nutritrack-api-vpc-alb-sg`.

### 4.3 NAT Instance Security Group (`nutritrack-api-vpc-nat-sg`)
- **Inbound Rule**: Allow **All traffic** from Source: `nutritrack-api-vpc-ecs-sg`. Allow `SSH` (Port 22) from your IP.

---

## Next Steps:
- [5.5.2 Supporting Infrastructure](../5.5.2-Infrastructure/)
- [5.5.3 NAT Optimization](../5.5.3-NAT-Instance/)
- [5.5.4 Fargate & ALB Deployment](../5.5.4-Fargate-ALB/)
