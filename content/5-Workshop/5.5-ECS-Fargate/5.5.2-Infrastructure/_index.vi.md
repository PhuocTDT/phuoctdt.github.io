---
title: "Hạ tầng hỗ trợ"
date: 2026-01-09T16:03:43+07:00
weight: 2
pre: " <b> 5.5.2. </b> "
menuTitle: "Infrastructure"
---

Phần này hướng dẫn thiết lập ba thành phần hạ tầng quan trọng hỗ trợ cho cụm ECS Fargate: **S3 Bucket** để lưu trữ cache, **Secrets Manager** để quản lý API keys bảo mật, và **IAM Roles** để cấp quyền thực thi.

> **Điều kiện tiên quyết:** Đã hoàn thành [5.5.1 Hạ tầng Mạng (VPC & Network)](../5.5.1-VPC-Network/).

---

## 1. S3 Bucket (Cache Layer)

Chúng ta cần một S3 Bucket để cache kết quả từ các API bên ngoài (USDA, OpenFoodFacts, Avocavo). Việc này giúp giảm độ trễ và tiết kiệm chi phí gọi API.

### 1.1 Tạo S3 Bucket
1. Vào AWS Console → **S3** → **Create bucket**.
2. **Bucket name**: `nutritrack-cache-xxxx` (tên phải duy nhất toàn cầu, bạn nên thêm ngày tháng hoặc tên cá nhân).
3. **Region**: `ap-southeast-2`.
4. **Block all public access**: ✅ Bật (Mặc định).
5. Nhấn **Create bucket**.

---

## 2. Secrets Manager

Secrets Manager giúp lưu trữ API keys một cách an toàn và tự động inject vào container dưới dạng biến môi trường, tránh việc lộ plaintext trong mã nguồn.

### 2.1 Tạo Secret
1. Vào **Secrets Manager** → **Store a new secret**.
2. Chọn **Secret type**: `Other type of secret`.
3. Thêm các cặp **Key/value** sau:
   - `USDA_API_KEY`: <Key của bạn>
   - `AVOCAVO_API_KEY`: <Key của bạn>
   - `NUTRITRACK_API_KEY`: <Secret dùng chung cho JWT giữa Lambda và ECS>
4. Đặt tên Secret: `nutritrack/prod/api-keys`.
5. Sau khi lưu, hãy copy mã **ARN** của Secret này để dùng cho bước tiếp theo.

---

## 3. IAM Roles cho ECS

ECS sử dụng hai Role riêng biệt cho hai mục đích khác nhau:

| Role Name | Đối tượng sử dụng | Mục đích |
| :--- | :--- | :--- |
| **`ecsTaskExecutionRole`** | AWS ECS Agent | Pull Docker image, gửi log về CloudWatch, đọc Secrets Manager. |
| **`ecsTaskRole`** | Ứng dụng bên trong container | Gọi Amazon Bedrock, đọc/ghi dữ liệu vào S3 Cache. |

### 3.1 Cấu hình `ecsTaskExecutionRole`
1. Tìm hoặc tạo Role tên `ecsTaskExecutionRole`.
2. Đảm bảo Role đã có managed policy: `AmazonECSTaskExecutionRolePolicy`.
3. Thêm **Inline Policy** (JSON) để cho phép đọc Secret ARN đã copy ở trên:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["secretsmanager:GetSecretValue"],
      "Resource": ["<YOUR_SECRET_ARN>"]
    }
  ]
}
```

### 3.2 Tạo `ecsTaskRole`
1. Tạo Role mới với Trusted Entity là `Elastic Container Service Task`.
2. Gắn **Inline Policy** cho phép truy cập Bedrock và S3 Bucket cache:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["bedrock:InvokeModel", "bedrock:ListFoundationModels"],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::nutritrack-cache-xxxx",
        "arn:aws:s3:::nutritrack-cache-xxxx/*"
      ]
    }
  ]
}
```

---

## Các bước tiếp theo:

Hạ tầng nền tảng đã sẵn sàng. Bây giờ chúng ta sẽ tối ưu hóa chi phí đường truyền internet:
- [5.5.3 Tối ưu hóa NAT (NAT Instance)](../5.5.3-NAT-Instance/)
- [5.5.4 Triển khai Fargate & ALB](../5.5.4-Fargate-ALB/)
