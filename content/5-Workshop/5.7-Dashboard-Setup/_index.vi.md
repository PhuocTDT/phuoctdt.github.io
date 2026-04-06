---
title: "Dashboard Setup"
date: 2026-01-09T16:03:43+07:00
weight: 40
chapter: false
pre: " <b> 5.7. </b> "
---

## Giai đoạn 5: Frontend React Native

Trong giai đoạn này, bạn sẽ xây dựng ứng dụng di động React Native (Expo) — giao diện người dùng của NutriTrack với hỗ trợ song ngữ, phân tích đồ ăn AI, và gamification.

#### Bước 1: Khởi tạo Expo Project

```bash
cd NutriTrack
npx create-expo-app frontend --template blank-typescript
cd frontend
npm install --legacy-peer-deps
```

#### Bước 2: Cấu hình Amplify Client

Tạo `frontend/src/lib/amplify.ts` và import trong root layout.

#### Bước 3: Kiến trúc App (Expo Router)

```
frontend/app/
├── _layout.tsx           # Root: Auth guard + LanguageProvider
├── (tabs)/
│   ├── home.tsx          # Dashboard: macros hàng ngày, streak, thú cưng
│   ├── kitchen.tsx       # Quản lý tủ lạnh + gợi ý công thức AI
│   ├── battle.tsx        # Thử thách & bảng xếp hạng
│   ├── ai-coach.tsx      # Chat với AI coach "Ollie"
│   └── progress.tsx      # Biểu đồ dinh dưỡng tuần/tháng
├── add.tsx               # Ghi nhật ký: ảnh, giọng nói, thủ công, mã vạch
├── welcome.tsx           # Màn hình giới thiệu
├── login.tsx             # Email/mật khẩu + Google OAuth
└── verify-otp.tsx        # Xác nhận OTP email
```

#### Bước 4: Quản lý State (Zustand)

Tạo stores trong `frontend/src/store/`: `authStore`, `userStore`, `foodStore`, `mealStore`, `fridgeStore`, `settingsStore`.

#### Bước 5: Tích hợp Dịch vụ AI

Tạo `frontend/src/services/aiService.ts` với 10 actions thông qua AppSync client.

#### Bước 6: Hỗ trợ Song ngữ (i18n)

Vietnamese làm ngôn ngữ mặc định, toggle English qua `LanguageProvider`.

#### Bước 7: Design System

- Primary: Dark Navy `#1B2838`
- Accent: Green `#2ECC71`
- Font: Inter, Roboto

#### Xác nhận

```bash
cd frontend && npm start
```

1. Xác nhận luồng auth: Welcome → Login → Home
2. Test phân tích ảnh đồ ăn: Camera → Chụp → Kiểm tra AI response
3. Xác nhận toggle song ngữ hoạt động trên tất cả màn hình

> 🎯 **Checkpoint:** App di động chạy, xác thực qua Cognito, và phân tích ảnh đồ ăn thành công qua AI engine.
