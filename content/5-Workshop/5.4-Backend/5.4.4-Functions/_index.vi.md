---
title: "Các hàm Logic (Functions)"
date: 2026-01-09T16:03:43+07:00
weight: 44
chapter: false
pre: " <b> 5.4.4. </b> "
---

## Các hàm chính trong dự án

### 1. `ai-engine` — Bộ não AI
Sử dụng SDK của Amazon Bedrock để gọi mô hình Qwen3-VL.
- **Thư mục**: `amplify/ai-engine/`

### 2. `scan-image` — Phân tích Hình ảnh (Proxy)
Đóng vai trò trung gian gửi yêu cầu tới tầng ECS Fargate.
- **Thư mục**: `amplify/scan-image/`

### 3. `process-nutrition` — Xử lý Dinh dưỡng
Tính toán Macros từ kết quả AI và ghi vào DynamoDB.
- **Th thư mục**: `amplify/functions/process-nutrition/`

### 4. `resize-image` — Tối ưu Hình ảnh
Sử dụng thư viện `sharp` (qua Lambda Layer) để tạo Thumbnail.
- **Thư mục**: `amplify/resize-image/`

### 5. `friend-request` — Hệ thống Kết bạn
Xử lý các Mutation yêu cầu kết bạn và cập nhật bảng Friendship.
- **Thư mục**: `amplify/friend-request/`

## Cách định nghĩa Function trong Amplify Gen 2

Trong Amplify Gen 2, mỗi Lambda function thường bao gồm hai tệp cốt lõi nằm trong thư mục riêng của nó (ví dụ: `amplify/functions/my-func/`):

1.  **`resource.ts`**: Định nghĩa tài nguyên (tên, thời gian timeout, bộ nhớ, quyền truy cập).
2.  **`handler.ts`**: Chứa mã nguồn thực thi chính của hàm.

### Quy trình tạo một hàm mới:
1.  Tạo thư mục: `mkdir -p amplify/[tên-hàm]`
2.  Tạo tệp `resource.ts` để cấu hình function.
3.  Tạo tệp `handler.ts` để viết logic.
4.  Export function vào tệp `amplify/backend.ts`.

---

## Chi tiết mã nguồn các hàm:

Dưới đây là mã nguồn skeleton cho từng hàm. Bạn hãy tạo các tệp tương ứng và dán mã nguồn thực tế của mình vào:

1. [Function ai-engine (Bedrock)](5.4.4.1-AIEngine/)
2. [Function scan-image (ECS Proxy)](5.4.4.2-ScanImage/)
3. [Function process-nutrition (Logic & DB)](5.4.4.3-ProcessNutrition/)
4. [Function friend-request (Social logic)](5.4.4.4-FriendRequest/)
5. [Function resize-image (S3 Trigger & Sharp)](5.4.4.5-ResizeImage/)

---

[Tiếp tục đến 5.5 Tầng Container ECS](../5.5-ECS-Fargate/)
