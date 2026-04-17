---
menuTitle: "Authentication"
date: 2026-01-09T16:03:43+07:00
weight: 41
chapter: false
pre: " <b> 5.4.1. </b> "
---

# Authentication Layer 

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

## Setting Secrets for Google OAuth

Since we're using `secret()` in the source code, you need to load these values ​​into AWS Amplify before deployment. Run the following commands in the terminal at the backend directory:

```bash
npx ampx secret set GOOGLE_CLIENT_ID
npx ampx secret set GOOGLE_CLIENT_SECRET
```

The system will prompt you to enter values ​​for each variable (taken from the Google Cloud Console).

![cognito-user-pool.png](/images/cognito-user-pool.png)
---

[Continue to 5.4.2 Data Layer (Data)](../5.4.2-Data/)
