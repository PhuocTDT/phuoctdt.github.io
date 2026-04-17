---
title: "Điều kiện tiên quyết"
date: 2026-01-09T16:03:43+07:00
weight: 20
chapter: false
pre: " <b> 5.2. </b> "
---

## Các yêu cầu về truy cập và thông tin
Trước khi bắt đầu, hãy đảm bảo bạn có đầy đủ các công cụ và quyền truy cập sau đây:

## 1. Tài khoản AWS & Region Sydney
Region được sử dụng xuyên suốt workshop là `ap-southeast-2` (Sydney). 
Bốn bước bắt buộc tại Sydney:
1. **Model Access**: Truy cập Amazon Bedrock console -> Model access -> Chọn **Edit** -> Kích hoạt mô hình **Qwen3-VL-235B-A22B**

![bedrock-model-access.png](/images/bedrock-model-access.png)

2. **IAM User**: Tạo tài khoản có quyền `AdministratorAccess`.
3. **Secret Manager**: Lưu trữ `GOOGLE_CLIENT_ID` và `GOOGLE_CLIENT_SECRET` nạp từ Google Cloud Console.

![google-oauth-client.png](/images/google-oauth-client.png)

## 2. Công cụ máy tính Local
- **Node.js**: Phiên bản 22.x (LTS) - cực kỳ quan trọng vì Lambda của chúng ta chạy Node 22.
- **Docker Desktop**: Cần thiết để build Image cho ECS Fargate.
- **AWS CLI**: Đã chạy lệnh `aws configure` với quyền Admin.
- **npm**: Phiên bản 11 trở lên.

## 3. Expo Go (Mobile)
Tải ứng dụng **Expo Go** từ App Store hoặc Play Store để có thể chạy thử mã nguồn Frontend trên điện thoại thật một cách nhanh nhất.

---

[Tiếp tục đến 5.3 Thiết lập Frontend](../5.3-Frontend/)
