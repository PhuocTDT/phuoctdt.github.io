---
title: "Hạ tầng Mạng (VPC & Network)"
date: 2026-01-09T16:03:43+07:00
weight: 1
pre: " <b> 5.5.1. </b> "
menuTitle: "VPC & Network"
---

Hướng dẫn này giúp bạn thiết lập nền tảng mạng AWS cho NutriTrack API: VPC riêng, 4 subnets trên 2 AZ, Internet Gateway, Route Tables, 3 Security Groups, và S3 Gateway VPC Endpoint.

> **Region:** `ap-southeast-2` (Sydney) | **Thời gian ước tính:** 45–60 phút

## Tại sao chọn kiến trúc này?

| Quyết định | Lý do |
| :--- | :--- |
| **ECS Private Subnet** | Container không có IP public → tăng tính bảo mật, tránh tấn công trực tiếp. |
| **ALB Internet-facing** | Điểm duy nhất nhận yêu cầu từ internet, che giấu IP của các container. |
| **NAT Instance** | Tiết kiệm **~70%** chi phí so với NAT Gateway (~$10/tháng so với $34/tháng). |
| **S3 Gateway VPCE** | Truy cập S3 **miễn phí**, không đi qua internet và không tốn băng thông NAT. |
| **High Availability (HA)** | Triển khai trên 2 Availability Zone (AZ) để đảm bảo hệ thống luôn hoạt động. |

---

## 1. Tạo VPC

**VPC (Virtual Private Cloud)** là mạng riêng ảo của bạn trên AWS. Mọi tài nguyên như ECS, ALB, NAT Instance đều sẽ nằm trong VPC này.

### 1.1 Khởi tạo VPC
1. Đăng nhập AWS Console, chọn Region **`ap-southeast-2`**.
2. Tìm kiếm dịch vụ **VPC**.
3. Chọn **Your VPCs** và nhấn **Create VPC**.
4. Cấu hình:
   - **Resources to create**: `VPC only`
   - **Name tag**: `nutritrack-api-vpc`
   - **IPv4 CIDR**: `10.0.0.0/16`
5. Nhấn **Create VPC**.

### 1.2 Bật DNS cho VPC
1. Chọn VPC vừa tạo → **Actions** → **Edit VPC settings**.
2. Bật cả 2 checkbox:
   - ✅ `Enable DNS resolution`
   - ✅ `Enable DNS hostnames`
3. Nhấn **Save**.

---

## 2. Tạo Subnets

Chúng ta sẽ tạo **4 subnets** chia đều trên 2 AZ (`2a` và `2c`):

| Subnet Name | Availability Zone | CIDR | Loại Subnet |
| :--- | :--- | :--- | :--- |
| `nutritrack-api-vpc-public-alb01` | ap-southeast-2a | `10.0.1.0/24` | Public (Cho ALB & NAT 1) |
| `nutritrack-api-vpc-public-alb02` | ap-southeast-2c | `10.0.2.0/24` | Public (Cho ALB & NAT 2) |
| `nutritrack-api-vpc-private-ecs01` | ap-southeast-2a | `10.0.3.0/24` | Private (Cho ECS Tasks) |
| `nutritrack-api-vpc-private-ecs02` | ap-southeast-2c | `10.0.4.0/24` | Private (Cho ECS Tasks) |

**Lưu ý:** Sau khi tạo xong, hãy chọn 2 Public Subnet và bật tính năng **`Enable auto-assign public IPv4 address`** trong phần Subnet settings.

---

## 3. Internet Gateway & Route Tables

### 3.1 Internet Gateway (IGW)
1. Tạo IGW với tên `nutritrack-api-igw`.
2. Sau khi tạo, hãy **Attach to VPC** vào VPC `nutritrack-api-vpc`.

### 3.2 Public Route Table
1. Tạo Route Table tên `nutritrack-api-public-rt`.
2. Trong tab **Routes**, thêm route: `0.0.0.0/0` → Target là `Internet Gateway`.
3. Trong tab **Subnet associations**, gắn (associate) cả 2 Public Subnets vào đây.

### 3.3 Private Route Tables
Tạo 2 Route Table riêng biệt cho 2 AZ để sau này cấu hình NAT Instance HA:
- `nutritrack-api-private-rt-01` (Gắn với Private Subnet 01)
- `nutritrack-api-private-rt-02` (Gắn với Private Subnet 02)

---

## 4. Security Groups (SG)

Bạn cần tạo 3 Security Group theo thứ tự logic sau:

### 4.1 ALB Security Group (`nutritrack-api-vpc-alb-sg`)
- **Inbound Rule**: Cho phép `HTTP` (Port 80) từ `0.0.0.0/0`.
- **Lưu ý**: Sau khi triển khai xong Lambda, bạn nên đổi Source về SG của Lambda để bảo mật hơn.

### 4.2 ECS Security Group (`nutritrack-api-vpc-ecs-sg`)
- **Inbound Rule**: Chỉ cho phép `Custom TCP` (Port 8000) từ Source là `nutritrack-api-vpc-alb-sg`.
- **Outbound Rule**: Cho phép gửi traffic đến NAT SG và S3 Prefix List.

### 4.3 NAT Instance Security Group (`nutritrack-api-vpc-nat-sg`)
- **Inbound Rule**: Cho phép **All traffic** từ Source là `nutritrack-api-vpc-ecs-sg`. Cho phép `SSH` (Port 22) từ IP máy cá nhân của bạn.
- **Outbound Rule**: Cho phép **All traffic** ra `0.0.0.0/0`.

---

## 5. S3 Gateway VPC Endpoint (Miễn phí)

Giúp ECS Task truy cập S3 mà không cần đi qua Internet hay NAT.

1. Vào **Endpoints** → **Create endpoint**.
2. Chọn Service: `com.amazonaws.ap-southeast-2.s3` (loại **Gateway**).
3. Chọn cả 2 Private Route Tables để AWS tự động thêm route.
4. Nhấn **Create endpoint**.

---

## Các bước tiếp theo:

Sau khi hạ tầng mạng đã sẵn sàng, chúng ta sẽ chuẩn bị các tài nguyên lưu trữ và định danh:
- [5.5.2 Hạ tầng hỗ trợ (S3, Secrets, IAM)](../5.5.2-Infrastructure/)
- [5.5.3 Tối ưu hóa NAT (NAT Instance)](../5.5.3-NAT-Instance/)
- [5.5.4 Triển khai Fargate & ALB](../5.5.4-Fargate-ALB/)
