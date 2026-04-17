---
menuTitle: "Lớp Lưu trữ (Storage)"
date: 2026-01-09T16:03:43+07:00
weight: 43
chapter: false
pre: " <b> 5.4.3. </b> "
---

# Lớp Lưu trữ — Amazon S3

Quản lý hình ảnh và các tệp phương tiện.

## Cấu hình `storage/resource.ts`

Chúng ta phân chia S3 bucket thành các khu vực truy cập khác nhau dựa trên tiền tố (Prefix).

```typescript
import { defineStorage } from '@aws-amplify/backend';

export const storage = defineStorage({
  name: 'nutritrack_media_bucket',
  access: (allow) => ({
    // Khu vực hạ cánh (Landing Zone) - User upload trực tiếp vào đây
    'incoming/{entity_id}/*': [
      allow.authenticated.to(['read', 'write', 'delete'])
    ],
    // Voice recordings - tạm lưu để Transcribe xử lý (ephemeral, Lambda xóa sau khi xong)
    'voice/*': [
      allow.authenticated.to(['read', 'write', 'delete'])
    ],

    'avatar/{entity_id}/*': [
      allow.entity('identity').to(['read', 'write', 'delete']),
      allow.authenticated.to(['read']),
    ],
    // Khu vực lưu trữ vĩnh viễn (Trusted Zone) - Lambda sẽ lưu kết quả tại đây
    'media/{entity_id}/*': [
      allow.authenticated.to(['read', 'delete'])
    ]
  })
});
```
![s3-bucket-console.png](/images/s3-bucket-console.png)
![s3-prefixes.png](/images/s3-prefixes.png)

---

[Tiếp tục đến 5.4.4 Các hàm Logic (Functions)](../5.4.4-Functions/)
