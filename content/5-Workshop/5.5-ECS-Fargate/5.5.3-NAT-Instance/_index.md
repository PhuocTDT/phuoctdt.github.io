---
title: "NAT Optimization"
date: 2026-01-09T16:03:43+07:00
weight: 3
pre: " <b> 5.5.3. </b> "
menuTitle: "NAT Instance"
---

In an AWS environment, resources in a Private Subnet (like ECS Tasks) typically need a NAT Gateway to access the internet. However, NAT Gateway has a high fixed cost (~$32/mo). In this workshop, we use a **NAT Instance** to save up to 70% in costs.

> **Prerequisite:** Completed [VPC & Network](../5.5.1-VPC-Network/) and [Infrastructure](../5.5.2-Infrastructure/).

## Why NAT Instance?

| Metric | NAT Gateway | NAT Instance (t4g.nano) |
| :--- | :--- | :--- |
| **Cost** | ≈$32–34/mo | **≈$4.33/mo** |
| **Throughput** | 100 Gbps | 5 Gbps (Enough for workshop) |
| **Management** | Managed by AWS | Managed by you (Self-setup) |

---

## 1. Initialize NAT Instance

We use a `t4g.nano` instance (ARM Graviton) for optimal cost and performance.

1. Go to **EC2 Console** → **Launch Instances**.
2. **AMI**: Select **Amazon Linux 2023** (64-bit Arm).
3. **Instance Type**: `t4g.nano`.
4. **Network Settings**:
   - VPC: `nutritrack-api-vpc`
   - Subnet: `nutritrack-api-vpc-public-alb01` (Public Subnet)
   - Auto-assign Public IP: **Enable**
   - Security Group: `nutritrack-api-vpc-nat-sg`
5. **IAM instance profile**: Select `nutritrack-api-vpc-nat-instance-role`.
6. Click **Launch instance**.

> [!IMPORTANT]
> Once the instance is running, you **must** disable **Source/Destination Check**:
> Select Instance → **Actions** → **Networking** → **Change source/destination check** → Select **Stop**.

---

## 2. NAT Configuration (Auto-script)

Once you've connected to the NAT Instance via SSH or SSM, run the following script to enable packet forwarding and Masquerading:

```bash
#!/bin/bash
# 1. Enable IP Forwarding
sudo sysctl -w net.ipv4.ip_forward=1

# 2. Install iptables-services
sudo dnf install iptables-services -y
sudo systemctl enable iptables
sudo systemctl start iptables

# 3. Configure NAT Masquerade
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
sudo iptables-save | sudo tee /etc/sysconfig/iptables
```

---

## 3. High Availability (ASG)

To ensure system continuity if a NAT Instance fails, you should use an **Auto Scaling Group (ASG)**.

The ASG will automatically detect instance failure and launch a replacement. You can use **User Data** in the Launch Template to automate the configuration and update the Route Table to point to the new Instance ID.

---

## Next Steps:
- [5.5.4 Fargate & ALB Deployment](../5.5.4-Fargate-ALB/)
