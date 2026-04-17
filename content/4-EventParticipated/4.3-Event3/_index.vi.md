---
title: "AWS Cloud Mastery 2"
date: 2026-04-04T15:59:44+07:00
weight: 3
chapter: false
pre: " <b> 4.3. </b> "
---

**Ngày:** 4 tháng 4, 2026
**Địa điểm:** Hội trường A - Đại học FPT TP.HCM, TP. Hồ Chí Minh
**Vai trò:** Người tham dự (FCJ Cloud Intern - Team NeuraX)

## Mô tả sự kiện

Sự kiện "Cloud Mastery 2" là một buổi gặp gỡ chia sẻ kiến thức chuyên sâu về kỹ thuật điện toán đám mây và phát triển phần mềm, quy tụ các thành viên trong cộng đồng FCJ. Sự kiện tập trung giải quyết các bài toán về triển khai hệ thống, tối ưu hóa hiệu năng bằng các ngôn ngữ lập trình thế hệ mới, và tự động hóa cơ sở hạ tầng thông qua các công cụ IaC (Infrastructure as Code).

## Các hoạt động chính

Sự kiện bao gồm 3 phiên trình bày kỹ thuật chính:

**Phiên 1: Container Orchestration & Kubernetes**
Giới thiệu tổng quan từ Docker, Docker Compose đến sự cần thiết của hệ thống điều phối Kubernetes trong doanh nghiệp. Đi sâu vào cấu trúc K8s (Control Plane, Worker Node, Pod, ConfigMap, Secret) và so sánh sự đánh đổi giữa việc dùng dịch vụ quản lý sẵn Amazon EKS (tiết kiệm công sức, linh hoạt) với tự xây dựng Self-hosted Kubernetes (phức tạp nhưng tiết kiệm chi phí).

**Phiên 2: Lập trình hàm với Elixir & Ứng dụng IoT**  
Trình bày về ngôn ngữ Elixir và máy ảo BEAM với thế mạnh cốt lõi là xử lý đồng thời (concurrency) và khả năng chống lỗi cao (fault tolerance). Trình diễn khả năng khởi tạo hàng ngàn process độc lập nhưng tiêu thụ cực ít CPU, kèm theo demo ứng dụng thực tế trong dự án Tủ quản lý nhà kính thông minh (Smart Greenhouse) dùng MQTT.

**Phiên 3: Infrastructure as Code (IaC) Tools**  
Phân tích 3 công cụ tự động hóa hạ tầng nổi bật: AWS CloudFormation (sử dụng template YAML/JSON), AWS CDK (sử dụng ngôn ngữ lập trình cho developer với các Construct Level) và Terraform (đa nền tảng đám mây, quản lý qua HCL). Session đi kèm demo triển khai host web S3 tĩnh bằng Terraform kết hợp công cụ giả lập LocalStack ngay trên máy trạm.

## Kết quả

- Hiểu sâu về kiến trúc Kubernetes: Nhận biết được giới hạn của Docker và thời điểm cần chuyển đổi sang Kubernetes. Hiểu rõ Amazon EKS là giải pháp phù hợp nhất cho các doanh nghiệp không muốn duy trì đội ngũ tự vận hành hạ tầng quá phức tạp.
- Tiềm năng của Elixir trong hệ thống phân tán: Nắm bắt được triết lý "let it crash" và sức mạnh của supervisor process in Elixir, giúp các ứng dụng IoT Backend tự động phục hồi nhanh chóng khi gặp sự cố mà không làm treo toàn bộ server.
- Lựa chọn đúng công cụ IaC: Phân biệt rõ ranh giới sử dụng giữa CloudFormation, CDK (lợi thế code bằng ngôn ngữ lập trình trực quan) và Terraform (mạnh về đa nền tảng và quản lý state).
- Quy trình làm việc an toàn với hạ tầng: Hiểu tầm quan trọng của việc chạy plan/kiểm tra lỗi trước khi apply trong Terraform, không đẩy các file state hay secret nhạy cảm lên GitHub, và biết cách chia nhỏ module để dễ tái sử dụng mã nguồn.
