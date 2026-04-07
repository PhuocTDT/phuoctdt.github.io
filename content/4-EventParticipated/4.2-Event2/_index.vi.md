---
title: "AWS Cloud Mastery 1"
date: 2026-03-14T15:59:29+07:00
weight: 2
chapter: false
pre: " <b> 4.2. </b> "
---

**Ngày:** 14 tháng 3, 2026
**Địa điểm:** Văn phòng AWS Việt Nam (Tầng 26), TP. Hồ Chí Minh
**Vai trò:** Người tham dự (FCJ Cloud Intern - Team NeuraX)

## Mô tả sự kiện

Sự kiện là buổi chia sẻ chuyên sâu dành cho cộng đồng công nghệ Việt Nam, quy tụ các lập trình viên và những người đam mê điện toán đám mây để tìm hiểu về các ứng dụng thực tế của AI và dịch vụ AWS thông qua các dự án cụ thể được đúc kết từ quá trình thực hành.

## Các hoạt động chính

Sự kiện bao gồm 3 phiên trình bày kỹ thuật chính tập trung vào việc ứng dụng AI, tối ưu hóa Prompt và kiến trúc Serverless:

**Phiên 1: Xây dựng AI Agents & Frameworks**  
Giới thiệu cách xây dựng AI Agent bằng một open SDK framework giúp tự động hóa việc định nghĩa công cụ (tìm kiếm web, chạy code Python) và kết nối với các mô hình như Amazon Bedrock. Phiên trình bày cũng nhấn mạnh các khái niệm cốt lõi của Agent như bộ nhớ (memory), khả năng suy luận đa bước (multi-step reasoning) và gọi tool để giải quyết các luồng công việc phức tạp.

**Phiên 2: Prompt Engineering & Serverless Architecture**  
Đi sâu vào các kỹ thuật Prompt Engineering (Role prompting, Chain of Thought, Tree of Thought) để tối ưu hóa kết quả AI đầu ra và giảm thiểu chi phí token đắt đỏ. Demo thực tế một tiện ích mở rộng (extension) "Prompt Optimizer" được xây dựng hoàn toàn trên kiến trúc AWS Serverless (bao gồm S3, CloudFront, Cognito, API Gateway, Lambda, DynamoDB và Bedrock) với chi phí vận hành gần như bằng không.

**Phiên 3: Ứng dụng AI vào Sản phẩm thực tế**  
Trình bày các dự án ứng dụng AI giải quyết bài toán thực tế. Demo 1 là Tủ thông minh IoT (quản lý thiết bị của câu lạc bộ) sử dụng Raspberry Pi, AWS IoT Core, S3, DynamoDB và Amazon Rekognition để nhận diện khuôn mặt tự động. Demo 2 là ứng dụng Quản lý tài chính cá nhân sử dụng Amazon Textract để quét, phân tích và trích xuất hóa đơn mua hàng một cách chính xác theo ngữ cảnh.

## Kết quả

- Agentic AI là xu hướng: Các AI Agents có thể tự động hóa quy trình làm việc phức tạp thông qua việc liên tục suy luận, đánh giá môi trường và sử dụng các công cụ bên ngoài.
- Tối ưu chi phí bằng Prompt Engineering: Áp dụng các kỹ thuật cấu trúc prompt tốt và ưu tiên sử dụng tiếng Anh giúp giảm thiểu đáng kể chi phí token input/output khi gọi các mô hình AI lớn.
- Kiến trúc Serverless cực kỳ tối ưu: Sự kết hợp giữa AWS Lambda, Cognito, S3 và API Gateway cung cấp một backend với khả năng mở rộng cao, không cần quản lý máy chủ và tiết kiệm chi phí tối đa cho các dự án AI.
- Sức mạnh của Amazon Rekognition: Cung cấp khả năng nhận diện khuôn mặt và vật thể vượt trội, có thể sử dụng Custom Labels để huấn luyện nhận diện riêng cho các dự án phần cứng/IoT với chi phí hợp lý.
- Vượt qua giới hạn của OCR truyền thống: Amazon Textract được hỗ trợ bởi AI giúp hiểu được ngữ cảnh của tài liệu, phân biệt rõ ràng giữa tên sản phẩm và giá tiền, lý tưởng để xử lý các biểu mẫu phức tạp như hóa đơn mà OCR thông thường thường xuyên đọc lỗi.
