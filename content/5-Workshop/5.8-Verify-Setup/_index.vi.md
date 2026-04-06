---
title: "Verify Setup"
date: 2026-01-09T16:03:43+07:00
weight: 30
chapter: false
pre: " <b> 5.8. </b> "
---

## Giai đoạn 6: Triển Khai ECS Fargate Container

Trong giai đoạn này, bạn sẽ triển khai backend FastAPI tự lưu trữ trên Amazon ECS Fargate trong VPC, với Application Load Balancer và Auto Scaling.

#### Tổng quan Kiến trúc

Tầng container chạy song song với backend serverless Amplify, cung cấp thêm khả năng tính toán cho AI inference tùy chỉnh hoặc các tác vụ xử lý nặng vượt giới hạn 15 phút của Lambda.

```
Internet → ALB (Public Subnet) → ECS Fargate (Private Subnet) → VPC Endpoints → AWS Services
```

#### Bước 1: Tạo Dockerfile

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

#### Bước 2: Tạo ECR Repository & Push Image

```bash
aws ecr create-repository --repository-name nutritrack-api --region ap-southeast-2
docker build -t nutritrack-api .
# Tag và push lên ECR...
```

#### Bước 3: Tạo VPC và Tài nguyên ECS

- **VPC** trong `ap-southeast-2` với 2 AZ
- **Public subnets** cho ALB
- **Private subnets** cho Fargate tasks
- **VPC Endpoints** truy cập dịch vụ AWS an toàn
- **Auto Scaling** dựa trên CPU/memory

#### Xác nhận

```bash
curl http://YOUR-ALB-DNS.ap-southeast-2.elb.amazonaws.com/health
# Kỳ vọng: {"status": "healthy"}
```

> 🎯 **Checkpoint:** Container FastAPI đang chạy trên ECS Fargate, truy cập được qua ALB, Auto Scaling đã cấu hình.
