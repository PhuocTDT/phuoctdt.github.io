---
title: "Storage Layer (Storage)"
date: 2026-01-09T16:03:43+07:00
weight: 43
chapter: false
pre: " <b> 5.4.3. </b> "
---

# 5.4.3 Storage Layer (Storage) — Amazon S3

Managing images and media assets.

## `storage/resource.ts`

```typescript
import { defineStorage } from "@aws-amplify/backend";

export const storage = defineStorage({
  name: "nutritrack_media",
  access: (allow) => ({
    "profile-pictures/{entity_id}/*": [
      allow.guest.to(["read"]),
      allow.entity("identity").to(["read", "write", "delete"])
    ],
    "meal-photos/*": [
      allow.authenticated.to(["read", "write"])
    ],
    "voice-notes/*": [
      allow.authenticated.to(["read", "write"])
    ],
  }),
});
```
![s3-bucket-console.png](/images/s3-bucket-console.png)
![s3-prefixes.png](/images/s3-prefixes.png)

---

[Continue to 5.4.4 Logic Functions (Functions)](../5.4.4-Functions/)
