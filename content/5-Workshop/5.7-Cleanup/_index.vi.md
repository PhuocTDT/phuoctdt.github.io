---
title: "Dọn dẹp"
date: 2026-01-09T16:03:43+07:00
weight: 100
chapter: false
pre: " <b> 5.7. </b> "
---

Sau khi hoàn thành workshop, việc dọn dẹp tài nguyên là bước tối quan trọng để tránh các chi phí phát sinh ngoài ý muốn từ AWS. Một số tài nguyên như NAT Instances và ALB sẽ tính phí theo giờ ngay cả khi không có lưu lượng truy cập.

> [!IMPORTANT]
> Tổng chi phí duy trì các tài nguyên trong workshop này khoảng **$44/tháng**. Hãy thực hiện các bước dưới đây ngay khi bạn không còn nhu cầu thử nghiệm.

## Thứ tự dọn dẹp tài nguyên

Để tránh lỗi "Resource in use", bạn nên thực hiện việc xóa theo thứ tự từ lớp ứng dụng ra đến lớp hạ tầng mạng.

### 1. Lớp Ứng dụng & Compute (ECS & ALB)

1. **ECS Service**: Vào ECS Cluster -> chọn Service `nutritrack-api-service` -> nhấn **Delete**. Đợi cho các Task dừng hẳn.
2. **ECS Cluster**: Sau khi Service đã xóa xong, bạn có thể xóa Cluster.
3. **Application Load Balancer (ALB)**: Vào phần EC2 -> Load Balancers -> Chọn ALB của workshop -> **Actions** -> **Delete**.
4. **Target Group**: Xóa Target Group tương ứng của ALB.

### 2. Lớp Hạ tầng Mạng (VPC & NAT)

1. **NAT Instances**: Vào EC2 Console -> Instances -> Chọn các NAT Instance -> **Terminate instance**.
2. **VPC**: Vào VPC Console -> Your VPCs -> Chọn `nutritrack-api-vpc` -> **Actions** -> **Delete VPC**. 
    - *Lưu ý: AWS sẽ tự động xóa Subnets, Internet Gateways, và Route Tables đi kèm.*

### 3. Lớp Backend & Lưu trữ (Amplify & S3)

1. **AWS Amplify**: Vào Amplify Console -> Chọn ứng dụng NutriTrack -> **Actions** -> **Delete app**. Việc này sẽ xóa toàn bộ Resources đi kèm (Cognito, DynamoDB, AppSync).
2. **S3 Bucket**: Vào S3 Console -> Tìm bucket cache (`nutritrack-cache-xxxx`). Bạn phải **Empty** (Làm trống) bucket trước khi có thể **Delete** (Xóa).
3. **Secrets Manager**: Vào Secrets Manager -> Chọn secret chứa API Keys -> **Delete secret**. (Lưu ý AWS mặc định sẽ giữ secret 7-30 ngày trước khi xóa hẳn).

### 4. IAM & CloudWatch

1. **IAM Roles**: Xóa các Role đã tạo thủ công như `ecsTaskRole`, `ecsTaskExecutionRole` nếu bạn không còn dùng cho dự án khác.
2. **CloudWatch Logs**: Xóa các Log Groups (`/ecs/nutritrack-api`) để làm sạch giao diện quản lý.

---

Chúc mừng bạn đã hoàn thành trọn vẹn workshop triển khai hệ thống NutriTrack trên AWS! Hy vọng những kiến thức về Hybrid Architecture (Serverless + Container) và tối ưu hóa chi phí mạng sẽ giúp ích cho các dự án thực tế của bạn.

[Quay lại trang chủ](../../)
