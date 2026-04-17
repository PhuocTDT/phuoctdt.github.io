---
title: "Tổng quan"
date: 2026-01-09T16:03:43+07:00
weight: 10
chapter: false
pre: " <b> 5.1. </b> "
---

## Giới thiệu 
Workshop này được thiết kế để dẫn dắt bạn qua quy trình xây dựng và triển khai **NutriTrack** — một nền tảng theo dõi dinh dưỡng hiện đại, hội tụ sức mạnh của **AWS Amplify Gen 2**, **Amazon Bedrock** và **Amazon ECS Fargate**. Với nội dung bám sát 100% vào codebase thực tế đang vận hành, workshop không chỉ giúp bạn làm chủ hạ tầng serverless mà còn khai phá cách tích hợp các tác vụ AI hiệu năng cao vào ứng dụng thực tiễn một cách tối ưu.

## Kiến trúc Tổng thể

![Kiến trúc NutriTrack](/images/architect.jpg)

## Tóm tắt Hạ tầng Backend

Đây là các thành phần chính mà bạn sẽ triển khai trong suốt Workshop:

### 1. Cơ sở dữ liệu (Database - DynamoDB)
| Tên Table | Chức năng chính | Ghi chú |
| :--- | :--- | :--- |
| **`Food`** | Danh mục thực phẩm & Dinh dưỡng | Chứa thông tin Macro/Micro của thực phẩm. |
| **`user`** | Hồ sơ người dùng | Lưu trữ chiều cao, cân nặng, mục tiêu calo. |
| **`FoodLog`** | Nhật ký ăn uống | Lịch sử bữa ăn và chỉ số dinh dưỡng đã nạp. |
| **`FridgeItem`** | Quản lý tủ lạnh | Theo dõi thực phẩm tồn kho. |
| **`Friendship`** | Hệ thống mạng xã hội | Quản lý lời mời kết bạn. |
| **`UserPublicStats`** | Chỉ số công khai | Lưu trữ Streak, Pet Level để bạn bè xem. |

### 2. Các hàm xử lý (Lambda Functions)
1. **`ai-engine`**: "Bộ não" gọi Bedrock (AI) và điều phối Transcribe.
2. **`scan-image`**: Kết nối với ECS Fargate để phân tích hình ảnh.
3. **`process-nutrition`**: Tính toán dinh dưỡng và xử lý fallback AI.
4. **`friend-request`**: Xử lý logic kết bạn.
5. **`resize-image`**: Tự động tối ưu ảnh khi tải lên S3.

### 3. Khả năng của AI (AI Capabilities)
* **Voice Logging**: Chuyển giọng nói tiếng Việt thành văn bản để bóc tách món ăn.
* **Photo Analysis**: Nhận diện thành phần và ước lượng khối lượng qua ảnh.
* **AI Coach (Ollie)**: Tư vấn dinh dưỡng cá nhân hóa.

---

[Tiếp tục đến 5.2 Điều kiện tiên quyết](../5.2-Prerequiste/)
