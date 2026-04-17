---
menuTitle: "Lớp Xác thực "
date: 2026-01-09T16:03:43+07:00
weight: 41
chapter: false
pre: " <b> 5.4.1. </b> "
---

# Lớp Xác thực 

Sử dụng Amazon Cognito để quản lý danh tính người dùng.

## Cấu hình `auth/resource.ts`

Chúng ta cấu hình đăng nhập bằng Email kết hợp với Google OAuth2.

```typescript
import { defineAuth, secret } from "@aws-amplify/backend";

export const auth = defineAuth({

  loginWith: {
    email: true,
    externalProviders: {
      google: {
        clientId: secret('GOOGLE_CLIENT_ID'),
        clientSecret: secret('GOOGLE_CLIENT_SECRET'),
        scopes: ['email', 'profile', 'openid']
      },
      callbackUrls: [
        'nutritrack://',
        'http://localhost:8081/',
        'https://nutri-track.link/',
        'https://feat-phase3-frontend-integration.d1glc6vvop0xlb.amplifyapp.com/'
      ],
      logoutUrls: [
        'nutritrack://',
        'http://localhost:8081/',
        'https://nutri-track.link/',
        'https://feat-phase3-frontend-integration.d1glc6vvop0xlb.amplifyapp.com/'
      ]
    }
  },

  userAttributes: {
    email: {
      required: true
    },
    preferredUsername: {
      required: false
    }
  },
});

```

---

## Thiết lập Secret cho Google OAuth

Vì chúng ta sử dụng `secret()` trong mã nguồn, bạn cần nạp các giá trị này vào AWS Amplify trước khi triển khai. Chạy các lệnh sau trong terminal tại thư mục backend:

```bash
npx ampx secret set GOOGLE_CLIENT_ID
npx ampx secret set GOOGLE_CLIENT_SECRET
```

Hệ thống sẽ yêu cầu bạn nhập giá trị cho từng biến (lấy từ Google Cloud Console).

![cognito-user-pool.png](/images/cognito-user-pool.png)

---

[Tiếp tục đến 5.4.2 Lớp Dữ liệu (Data)](../5.4.2-Data/)
