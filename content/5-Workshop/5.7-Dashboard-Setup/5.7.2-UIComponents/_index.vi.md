---
title: "UIComponents"
date: 2026-01-09T16:03:43+07:00
weight: 5
chapter: false
pre: " <b> 5.7.2. </b> "
---

### 5.7.2 Tích hợp Giao diện UI/UX

Giao diện của NutriTrack được định hướng theo dạng Gamification và tập trung vào tương tác AI.

#### Luồng hoạt động chính
1. **Camera Scanner**: Tận dụng `expo-camera` để chụp mâm cơm người dùng. Ảnh không ném thẳng qua GraphQL mà gửi luồng Base64 sang S3 bucket nhánh `/incoming`.
2. **Popup Kết quả AI**: Lắng nghe AppSync Subscriptions thời gian thực từ Bedrock chạy nền để render bảng calo không cần reload.
3. **Voice Logger**: Dùng `expo-av` để ghi âm bữa ăn, nén dưới dạng `.wav` hoặc gửi dạng stream lên AWS Transcribe.
4. **Quản lý biến trạng thái (Zustand)**: Lưu token JWT, AuthSession, và bộ lưu đệm Cache để tiết kiệm hàng trăm lệnh GET query vào DynamoDB.

Sau khi luồng Emulator hoạt động trơn tru thì ta mới tiến hành Build Production cho mobile.