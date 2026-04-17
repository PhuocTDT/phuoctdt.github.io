---
title: "Workshop"
date: 2026-01-09T16:03:34+07:00
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

# NutriTrack — Workshop Triển Khai Full-Stack Trên AWS

## Tổng quan
Hướng dẫn này cung cấp quy trình từng bước hoàn chỉnh để triển khai **NutriTrack** — hệ thống theo dõi dinh dưỡng và phân tích hình ảnh AI tự động trên AWS. Workshop tận dụng khung quản trị **AWS Amplify Gen 2** để thiết lập hạ tầng serverless cốt lõi bao gồm **Amazon Cognito** (Xác thực), **AWS AppSync** & **DynamoDB** (Dữ liệu), và **Amazon S3** (Lưu trữ). Hệ thống được mở rộng với tầng tính toán hiệu năng cao bằng **Amazon ECS Fargate** để xử lý các tác vụ thị giác máy tính và phân tích dinh dưỡng sâu qua **Amazon Bedrock**, kết nối linh hoạt với ứng dụng mobile **React Native**. Toàn bộ giải pháp được tích hợp quy trình **CI/CD** tự động, giúp tối ưu hóa việc triển khai từ giai đoạn phát triển đến vận hành thực tế trên môi trường cloud.

## Nội dung thực hành

1. [Tổng quan](5.1-Overview/)
2. [Điều kiện tiên quyết](5.2-Prerequiste/)
3. [Thiết lập Frontend](5.3-Frontend/)
4. [Thiết lập Backend](5.4-Backend/)
5. [Tầng Container ECS Fargate](5.5-ECS-Fargate/)
6. [CI/CD](5.6-CICD/)
7. [Dọn dẹp tài nguyên](5.7-Cleanup/)

## Ước tính chi phí

Dưới đây là bảng ước tính chi phí cho việc duy trì hệ thống NutriTrack trên AWS. Lưu ý rằng chi phí thực tế có thể thay đổi tùy thuộc vào mức độ sử dụng và cấu hình cụ thể.

| Dịch vụ              | Chi phí tháng | Chi phí/Ngày |
|----------------------|-------------:|-------------:|
| Amazon Route 53      | $0.90        | $0.016       |
| Amplify WAF          | $37.11       | $1.237       |
| CloudFront           | $0.00        | $0.000       |
| AWS Amplify          | $4.65        | $0.110       |
| Fargate ARM64        | $10.23       | $0.341       |
| ALB                  | $28.46       | $0.850       |
| NAT Instance         | $7.63        | $0.254       |
| Amazon Cognito       | $0.00        | $0.000       |
| AWS AppSync          | $3.11        | $0.00004     |
| AWS Lambda           | $0.00        | $0.000       |
| Amazon Transcribe    | $72.57       | $0.120       |
| Amazon Bedrock       | $147.57      | $0.016       |
| Amazon S3            | $1.47        | $0.008       |
| Amazon DynamoDB      | $0.13        | $0.000       |
| CloudWatch           | $0.00        | $0.000       |
| AWS Secrets Manager  | $1.20        | $0.040       |
| **Tổng cộng**        | **$315.03**  | **$2.892**   |

---
