---
title: "Nhật ký công việc Tuần 8"
date: 2026-03-09T09:00:00+07:00
weight: 8
chapter: false
pre: " <b> 1.8. </b> "
---

### Mục tiêu Tuần 8:

- **NutriTrack Giai đoạn 2**: Xây dựng Tầng API NutriTrack với AWS AppSync (GraphQL)
- **Thiết kế Dữ liệu**: Xây dựng Tầng Dữ liệu với 8 mô hình DynamoDB
- **Tuân thủ**: Kiểm toán mọi hành động API bằng AWS CloudTrail
- **Cấu hình**: Thực thi các quy tắc bảo mật tài nguyên với AWS Config

---

### Các nhiệm vụ thực hiện trong tuần:

| Ngày | Nhiệm vụ | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu |
| --- | --------- | ------------ | --------------- | -------------- |
| 2 | Thiết kế Schema GraphQL với Phân quyền dựa trên chủ sở hữu | 09/03/2026 | 09/03/2026 | <https://docs.aws.amazon.com/appsync/> |
| 3 | Triển khai 8 mô hình cấp độ production với tối ưu hóa GSI | 10/03/2026 | 10/03/2026 | <https://docs.aws.amazon.com/dynamodb/> |
| 4 | Phát triển bộ xử lý `friend-request` (Lambda mutation) | 11/03/2026 | 11/03/2026 | <https://docs.aws.amazon.com/lambda/> |
| 5 | Thiết lập CloudTrail để giám sát API và các quy tắc AWS Config | 12/03/2026 | 12/03/2026 | <https://docs.aws.amazon.com/config/> |
| 6 | Thử nghiệm Tích hợp: Xác minh các hoạt động GraphQL | 13/03/2026 | 13/03/2026 | <https://docs.amplify.aws/gen2/build-a-backend/data/> |

---

### Kết quả Tuần 8:

- Xây dựng thành công **Tầng API NutriTrack**, cung cấp một backend bảo mật và thời gian thực.
- Thiết kế một **Tầng Dữ liệu có khả năng mở rộng** với 8 bảng DynamoDB được tối ưu hóa.
- Đạt được **Tính Minh bạch Vận hành** với CloudTrail để ghi nhật ký lệnh gọi API.
- Thiết lập **Quản trị Tuân thủ** bằng AWS Config để đảm bảo bảo mật tài nguyên.
