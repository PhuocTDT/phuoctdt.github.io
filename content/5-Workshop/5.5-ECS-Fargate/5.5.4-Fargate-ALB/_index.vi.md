---
title: "Triển khai Fargate & ALB"
date: 2026-01-09T16:03:43+07:00
weight: 4
pre: " <b> 5.5.4. </b> "
menuTitle: "Fargate & ALB"
---

Đây là bước cuối cùng để hoàn thiện hệ thống: Build Docker image, đẩy lên registry, và thiết lập cụm ECS Fargate cùng Application Load Balancer (ALB) để phục vụ các yêu cầu từ internet.

## 1. Build & Push Docker Image

Chúng ta cần có một Docker Image chứa mã nguồn FastAPI (Python) được tối ưu cho kiến trúc ARM64 của AWS Graviton.

### Clone mã nguồn (Ví dụ)
```bash
git clone https://github.com/justHman/NUTRI_TRACK_API
cd NUTRI_TRACK_API
```

### Build và Push lên Docker Hub
```bash
# Đăng nhập Docker Hub
docker login

# Build multi-platform (ARM64) và push trực tiếp
docker buildx build \
  --platform linux/arm64 \
  --tag <username>/nutritrack-api:latest \
  --push .
```

---

## 2. Khởi tạo ECS Cluster

1. Vào **ECS Console** → **Clusters** → **Create cluster**.
2. **Cluster name**: `nutritrack-api-cluster`.
3. **Infrastructure**: Chọn `AWS Fargate (serverless)`.
4. Nhấn **Create**.

---

## 3. Định nghĩa Task (Task Definition)

Task Definition chỉ định image nào sẽ chạy, tài nguyên CPU/RAM và các biến môi trường cần thiết.

1. Vào **Task Definitions** → **Create new task definition**.
2. **OS/Architecture**: `Linux/ARM64` (Để tiết kiệm chi phí với Graviton).
3. **CPU**: `1 vCPU`, **Memory**: `2 GB`.
4. **Task Execution Role**: `ecsTaskExecutionRole`.
5. **Container Details**:
   - **Name**: `nutritrack-api-container`
   - **Image**: `<username>/nutritrack-api:latest`
   - **Port mapping**: `8000` (TCP).
6. **Environment variables**: Inject các API Keys từ Secrets Manager sử dụng cú pháp `ValueFrom`.

---

## 4. Application Load Balancer (ALB)

ALB đóng vai trò tiếp nhận traffic từ người dùng và phân phối đến các container của ECS.

1. **Target Group**: Tạo Target Group tên `nutritrack-api-tg`, port 8000, type **IP**. Health check path: `/health`.
2. **Load Balancer**: Tạo ALB loại **Internet-facing**, chọn các Public Subnet đã tạo ở bước 5.5.1.
3. **Security Group**: Gắn `nutritrack-api-vpc-alb-sg`.
4. **Listener**: Chuyển hướng traffic từ port 80 sang Target Group vừa tạo.

---

## 5. Cấu hình Bảo mật với AWS WAF

Hãy chèn thêm một lớp bảo mật (WAF) để ngăn chặn các cuộc tấn công brute-force và đảm bảo chỉ Lambda `scan-image` mới có quyền gọi đến ALB.

- **Rate Limit**: Giới hạn mỗi IP tối đa 100 request trong 5 phút.
- **Custom Header**: Chỉ chấp nhận request có header `Authorization: Bearer <token>`.

---

## 6. ECS Service

Cuối cùng, hãy tạo Service để duy trì số lượng task luôn chạy.

1. Vào Cluster → tab **Services** → **Create**.
2. **Capacity Provider**: `FARGATE_SPOT` (Để tối ưu chi phí tối đa).
3. **Deployment Configuration**: Chọn Task Definition vừa tạo.
4. **Networking**: Chọn 2 Private Subnets và Security Group của ECS.
5. **Load Balancing**: Chọn ALB và Target Group đã thiết lập.
6. Nhấn **Create**.

---

## Kết quả đạt được:

Hệ thống của bạn hiện đã hoàn thiện:
- Một Mobile App kết nối với Amplify Gen 2.
- Các tác vụ AI nặng được xử lý bởi FastAPI trên ECS Fargate.
- Toàn bộ được bảo mật trong một VPC riêng với NAT tối ưu chi phí.

---

[Tiếp tục đến 6.0 Tổng kết & Tài liệu tham khảo](../../6-Conclusion/)
