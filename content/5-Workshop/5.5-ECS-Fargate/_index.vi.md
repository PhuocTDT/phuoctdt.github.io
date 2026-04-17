---
title: "Tầng Container (ECS Fargate)"
date: 2026-01-09T16:03:43+07:00
weight: 45
chapter: false
pre: " <b> 5.5. </b> "
menuTitle: "Tầng Container ECS"
---

Tầng ECS Fargate chạy một FastAPI service được container hóa song song với backend Amplify serverless. Nó xử lý các tác vụ không phù hợp với Lambda như: xử lý dữ liệu kéo dài, suy luận AI tùy chỉnh, hoặc các tiến trình cần duy trì kết nối liên tục.

## Kiến trúc Hệ thống

![Kiến trúc NutriTrack API VPC](images/only-nutritrack-api-vpc.drawio.svg)

Các ECS task chạy trong **Private Subnet** để đảm bảo an ninh; **Application Load Balancer (ALB)** nằm trong **Public Subnet** để tiếp nhận yêu cầu từ internet. Task truy cập vào các dịch vụ AWS khác thông qua **NAT Instance** (giúp tiết kiệm 70% chi phí so với NAT Gateway) hoặc **S3 Gateway Endpoint** (miễn phí).

## Ước tính Chi phí

| Thành phần | Chi phí ước tính/tháng |
| :--- | :--- |
| 2× NAT Instance `t4g.nano` | ≈$9 |
| 2× Fargate Task (0.5 vCPU / 1 GB) | ≈$17 |
| Application Load Balancer (ALB) | ≈$16 |
| CloudWatch Logs (5 GB) | ≈$2 |
| **Tổng cộng** | **≈$44** |

> [!TIP]
> Sử dụng NAT Instance thay vì NAT Gateway có thể tiết kiệm cho bạn khoảng $32 mỗi tháng, một con số đáng kể cho các dự án khởi nghiệp hoặc thử nghiệm.

## Các bước triển khai:

1. [5.5.1 Hạ tầng Mạng (VPC & Network)](5.5.1-VPC-Network/)
2. [5.5.2 Hạ tầng hỗ trợ (S3, Secrets, IAM)](5.5.2-Infrastructure/)
3. [5.5.3 Tối ưu hóa NAT (NAT Instance)](5.5.3-NAT-Instance/)
4. [5.5.4 Triển khai Fargate & ALB](5.5.4-Fargate-ALB/)

---

[Tiếp tục đến 6.0 Kết luận & Tiếp theo](../6-Conclusion/)
