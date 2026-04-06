---
title: "Xưởng thực hành"
date: 2026-01-09T16:03:34+07:00
weight: 5
chapter: true
pre: " <b> 5. </b> "
---

# NutriTrack — Workshop Triển Khai Full-Stack Trên AWS

#### Tổng quan

Workshop này là hướng dẫn từng bước để triển khai **NutriTrack**, một nền tảng theo dõi dinh dưỡng serverless mức production trên AWS. Bạn sẽ xây dựng ứng dụng mobile React Native (Expo SDK 54) kết nối tới **AWS Amplify Gen 2**, **Amazon Bedrock** (Qwen3-VL 235B đa phương thức), **AWS AppSync** GraphQL, **Amazon DynamoDB**, và một dịch vụ **FastAPI** đóng gói container chạy trên **ECS Fargate**. Hệ thống hoàn chỉnh nhận ảnh món ăn, chuyển giọng nói tiếng Việt thành văn bản, gọi AI coach tên **Ollie**, và lưu nhật ký dinh dưỡng với real-time GraphQL subscriptions.

Toàn bộ nội dung workshop phản ánh đúng codebase đang chạy thật — không có kiến trúc giả, không có ví dụ đồ chơi. Mọi IAM statement, mọi `runtime: 22` của Lambda, mọi GSI DynamoDB, và mọi S3 prefix đều trùng với môi trường `main` hiện tại.

#### Bạn sẽ xây dựng những gì

- **Frontend**: Ứng dụng Expo Router (React Native 0.81, React 19) với xác thực sinh trắc, camera/voice capture, Zustand stores, và màn hình pet evolution dùng `@react-three/fiber`.
- **Backend**: Amplify Gen 2 (`defineBackend`) cấp phát Cognito + Google OAuth, 8 model DynamoDB thông qua AppSync, một S3 bucket với 4 access prefix, và 4 Lambda (`aiEngine`, `processNutrition`, `friendRequest`, `resizeImage`).
- **Lớp AI**: Bedrock Qwen3-VL (`qwen.qwen3-vl-235b-a22b`) ở `ap-southeast-2`, được gọi qua một Lambda multi-action duy nhất điều phối 10 AI action (phân tích ảnh, voice-to-food, phản hồi coach, sinh công thức, insight hàng tuần…).
- **Tầng container**: FastAPI trên ECS Fargate đứng sau ALB, deploy từ ECR, chạy trong một VPC riêng.
- **Vận hành**: Amplify CI/CD qua ba môi trường (sandbox, `feat/phase3`, `main`), log CloudWatch, và playbook dọn dẹp.

#### Nội dung

1. [Tổng Quan](5.1-Overview/)
2. [Điều Kiện Tiên Quyết](5.2-Prerequiste/)
3. [Thiết Lập Nền Tảng — Amplify, Cognito, S3](5.3-Foundation-Setup/)
4. [Tầng Dữ Liệu — AppSync & DynamoDB](5.4-Monitoring-Setup/)
5. [Compute & AI — Bedrock, Lambda](5.5-Processing-Setup/)
6. [API & Xã Hội — Friends, Realtime Subscriptions](5.6-Automation-Setup/)
7. [Frontend — Expo, UI, Giọng Nói & Camera](5.7-Dashboard-Setup/)
8. [Triển Khai ECS — VPC, ECR, Fargate, ALB](5.8-Verify-Setup/)
9. [CI/CD — Amplify Đa Môi Trường](5.9-Use-CDK/)
10. [Dọn Dẹp](5.10-Cleanup/)
11. [Phụ Lục — Ngân Sách, IAM, Xử Lý Lỗi, Prompt](5.11-Appendices/)

#### Workshop dành cho ai

Các kỹ sư đã quen với TypeScript, có kinh nghiệm AWS ở mức cơ bản, và muốn thấy cấu trúc end-to-end của một dự án Amplify Gen 2 thực tế. Bạn **không** cần kinh nghiệm trước về Bedrock, Amplify Gen 2, hay React Native — mỗi bước đều kèm command, đường dẫn file, và IAM policy cụ thể để sao chép.
