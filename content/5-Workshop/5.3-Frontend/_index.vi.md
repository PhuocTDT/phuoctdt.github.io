---
title: "Thiết lập Frontend"
date: 2026-01-09T16:03:43+07:00
weight: 30
chapter: false
pre: " <b> 5.3. </b> "
---

## ReactNative & Expo

Để thuận tiện cho việc thực hành, phần Frontend đã được chuẩn bị sẵn dưới dạng skeleton code hoàn chỉnh về giao diện và store. Bạn sẽ tiến hành tải mã nguồn và chuẩn bị môi trường chạy.

```text
frontend/
  app/                     # Expo Router — every file is a route
    _layout.tsx            # Root: LanguageProvider, GestureHandlerRootView, auth guard
    (tabs)/
      _layout.tsx          # Tab bar with 6 tabs + center "+" button
      home.tsx             # Dashboard: daily macros, streak, Ollie pet
      kitchen.tsx          # Fridge inventory + AI recipe suggestions
      battle.tsx           # Friend leaderboard + challenges
      ai-coach.tsx         # Chat with Ollie
      progress.tsx         # Weekly/monthly nutrition charts
      add.tsx              # Food logging: photo, voice, manual
    welcome.tsx            # Onboarding / landing
    login.tsx              # Email+password + Google OAuth
    signup.tsx             # Registration form
    verify-otp.tsx         # Email OTP verification
  src/
    store/                 # Zustand stores (authStore, userStore, mealStore, ...)
    services/              # Business logic (authService, aiService, audioService, ...)
    lib/amplify.ts         # Amplify.configure() — import as side-effect in _layout
    i18n/                  # LanguageProvider + translations (vi/en)
    security/              # Biometric auth, screen capture prevention, input validation
    constants/             # colors.ts, typography.ts
  assets/                  # Images, fonts
  MANHINH/                 # Pet evolution videos 1.mp4–5.mp4
  amplify_outputs.json     # Auto-generated per environment — do NOT edit manually
  package.json
  .npmrc                   # legacy-peer-deps=true
```

## 1. Tải mã nguồn (Clone Repository)

Mở terminal tại thư mục làm việc của bạn và chạy lệnh:

```bash
git clone https://github.com/NeuraX-HQ/neurax-web-app.git
cd neurax-web-app
```

## 2. Cài đặt thư viện (npm install)

Cài đặt các phụ thuộc cần thiết cho Expo và xác thực:

```bash
cd frontend
npm install
```

## 3. Chạy ứng dụng với Expo

Khởi động môi trường phát triển:

```bash
npx expo start
```

Tại đây, bạn sẽ thấy một mã QR. Sử dụng điện thoại đã cài **Expo Go** để quét mã này. 

{{% notice info %}}
Màn hình sẽ hiển thị thông báo lỗi kết nối Backend hoặc lỗi Config — điều này hoàn toàn bình thường vì chúng ta chưa triển khai hạ tầng Amplify ở bước tiếp theo.
{{% /notice %}}

---

[Tiếp tục đến 5.4 Thiết lập Backend](../5.4-Backend/)
