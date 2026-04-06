---
title: "ReactNative"
date: 2026-01-09T16:03:43+07:00
weight: 5
chapter: false
pre: " <b> 5.7.1. </b> "
---

### 5.7.1 Cài đặt React Native (Expo)

Frontend của NutriTrack được thiết kế hoàn toàn bằng React Native chạy qua Expo nhằm biên dịch nhanh cho hai nền tảng iOS/Android.

#### Bước 1: Tạo project Expo
```bash
npx create-expo-app nutritrack-mobile -t expo-template-blank-typescript
cd nutritrack-mobile
```

#### Bước 2: Cài đặt thư viện Amplify
```bash
npm install aws-amplify @aws-amplify/react-native
```

#### Bước 3: Đấu nối Backend
Khởi tạo cấu hình kết nối AWS trong file `App.tsx` bằng cách đọc tệp `amplify_outputs.json` do amplify tự động sinh ra sau khi deploy.

```typescript
import { Amplify } from 'aws-amplify';
import outputs from './amplify_outputs.json';

Amplify.configure(outputs);

export default function App() {
  return <SafeAreaView>...</SafeAreaView>;
}
```