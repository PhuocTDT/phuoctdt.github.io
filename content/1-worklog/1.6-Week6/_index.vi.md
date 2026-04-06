---
title: "Nhật ký công việc Tuần 6"
date: 2026-02-09T09:00:00+07:00
weight: 5
chapter: false
pre: " <b> 1.6. </b> "
---

### Mục tiêu Tuần 6:

- Làm chủ Quản lý Định danh và Truy cập (IAM)
- Tìm hiểu sâu về Chính sách IAM (JSON, Condition keys)
- Hiểu về IAM Roles cho EC2 và Quan hệ tin cậy (Trust Relationships)
- Cấu hình IAM Identity Center (SSO) và Xác thực đa yếu tố (MFA)
- Tìm hiểu và triển khai Resource-based Policies

---

### Các nhiệm vụ thực hiện trong tuần:

| Ngày | Nhiệm vụ | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu |
| --- | --------- | ------------ | --------------- | -------------- |
| 2 | Nền tảng IAM: Người dùng, Nhóm và Nguyên tắc quyền tối thiểu | 09/02/2026 | 09/02/2026 | <https://docs.aws.amazon.com/IAM/latest/UserGuide/> |
| 3 | Chính sách IAM nâng cao: Viết các chính sách JSON với các khóa điều kiện và quyền trên từng tài nguyên | 10/02/2026 | 10/02/2026 | <https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html> |
| 4 | IAM Roles và Instance Profiles: Ủy thác quyền một cách an toàn cho các thực thể EC2 | 11/02/2026 | 11/02/2026 | <https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2.html> |
| 5 | Định danh liên kết: Cấu hình IAM Identity Center (SSO) và Bắt buộc sử dụng MFA | 12/02/2026 | 12/02/2026 | <https://docs.aws.amazon.com/singlesignon/> |
| 6 | Resource-based Policies: Nghiên cứu sâu về S3 Bucket Policies và KMS Key Policies | 13/02/2026 | 13/02/2026 | <https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_identity-vs-resource.html> |

---

### Kết quả Tuần 6:

- Làm chủ **Mô hình bảo mật IAM**, đảm bảo chỉ những người dùng và dịch vụ được phép mới có thể truy cập tài nguyên đám mây.
- Hiểu sâu về **Chính sách IAM nâng cao**, cho phép kiểm soát chính xác hạ tầng đám mây.
- Triển khai thành công **IAM Roles**, loại bỏ nguy cơ lộ thông tin đăng nhập trực tiếp trên máy chủ.
- Thiết lập môi trường đăng nhập an toàn bằng **SSO và MFA**, giảm thiểu rủi ro truy cập trái phép.
- Triển khai tự tin các **Resource-based Policies**, cung cấp thêm một lớp bảo mật trực tiếp trên dữ liệu và khóa.
