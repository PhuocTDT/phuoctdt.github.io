---
title: "Nhật ký công việc Tuần 9"
date: 2026-03-16T09:00:00+07:00
weight: 2
chapter: false
pre: " <b> 1.9. </b> "
---

### Mục tiêu Tuần 9:

- **NutriTrack Giai đoạn 3**: Tích hợp Generative AI với Amazon Bedrock (Qwen3-VL)
- **Xử lý âm thanh**: Triển khai Voice-to-Food bằng AWS Transcribe
- **AI Hướng sự kiện**: Xây dựng đường ống phương tiện qua S3 và Lambda
- **Phòng thủ mạng**: Bảo vệ backend với AWS WAF & Shield

---

### Các nhiệm vụ thực hiện trong tuần:

| Ngày | Nhiệm vụ | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu |
| --- | --------- | ------------ | --------------- | -------------- |
| 2 | Triển khai Bedrock `ai-engine` với Qwen3-VL (vùng Sydney) | 16/03/2026 | 16/03/2026 | <https://aws.amazon.com/bedrock/> |
| 3 | Thiết lập AWS Transcribe để chuyển giọng nói thành văn bản | 17/03/2026 | 17/03/2026 | <https://docs.aws.amazon.com/transcribe/> |
| 4 | Kích hoạt AI hướng sự kiện: Lambda `resize-image` trên S3 event | 18/03/2026 | 18/03/2026 | <https://docs.aws.amazon.com/lambda/> |
| 5 | Triển khai quy tắc AWS WAF và bảo vệ DDoS bằng Shield | 19/03/2026 | 19/03/2026 | <https://docs.aws.amazon.com/waf/> |
| 6 | Hoàn thiện Dự án: Xác minh luồng AI-Bảo mật NutriTrack từ đầu đến cuối | 20/03/2026 | 20/03/2026 | <https://000008.awsstudygroup.com/> |

---

### Kết quả Tuần 9:

- Tích hợp thành công **Amazon Bedrock**, cung cấp trí tuệ AI đa phương thức.
- Cho phép **Ghi nhật ký giọng nói mượt mà** bằng AWS Transcribe.
- Tự động hóa **Tối ưu hóa hình ảnh** qua các bộ kích hoạt S3 và Lambda.
- Gia cố ứng dụng với **AWS WAF & Shield** trước các tấn công web và DDoS.
