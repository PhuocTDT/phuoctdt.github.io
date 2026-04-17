---
title: "Authentication (Auth)"
date: 2026-01-09T16:03:43+07:00
weight: 41
chapter: false
pre: " <b> 5.4.1. </b> "
---

# 5.4.1 Authentication Layer (Auth)

Using Amazon Cognito for identity management.

## `auth/resource.ts`

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

![cognito-user-pool.png](/images/cognito-user-pool.png)
---

[Continue to 5.4.2 Data Layer (Data)](../5.4.2-Data/)
