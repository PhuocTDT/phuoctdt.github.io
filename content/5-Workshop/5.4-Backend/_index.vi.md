---
title: "Thiết lập AWS Amplify Gen 2"
menuTitle: "Thiết lập Backend"
date: 2026-01-09T16:03:43+07:00
weight: 40
chapter: false
pre: " <b> 5.4. </b> "
---

Chúng ta sẽ khởi tạo trái tim dữ liệu của NutriTrack bằng mã nguồn TypeScript.

## 1. Khởi tạo Thư mục Backend

Tạo một thư mục riêng biệt để quản lý hạ tầng:

```bash
cd neurax-web-app
mkdir backend
cd backend
```
## 2. Cài đặt Phụ lục

```bash
npm create amplify@latest --yes
npm install
```

## 3. Triển khai Sandbox (Lần đầu)

Sử dụng Amplify CLI (Gen 2) để khởi tạo dự án và triển khai môi trường Sandbox cá nhân:

```bash
npx ampx pipeline-deploy --branch main --app-id [YOUR_APP_ID]

# Hoặc chạy lệnh sau để làm việc local:
npx ampx sandbox
```
![ampx-sandbox-start.png](/images/ampx-sandbox-start.png)

---

## Chi tiết các lớp tài nguyên:

Bây giờ, chúng ta sẽ lần lượt định nghĩa các tệp cấu hình cốt lõi nằm trong thư mục `amplify/`:

1. [Lớp Xác thực (Auth)](5.4.1-Auth/)
2. [Lớp Dữ liệu (Data)](5.4.2-Data/)
3. [Lớp Lưu trữ (Storage)](5.4.3-Storage/)
4. [Các hàm Logic (Functions)](5.4.4-Functions/)

---

[Tiếp tục đến 5.5 Tầng Container ECS](../5.5-ECS-Fargate/)
