---
title: "Function scan-image"
menuTitle: "5.4.4.2. Function scan-image"
weight: 2
---

This function acts as a proxy (intermediary) to send image analysis requests from AppSync to the **ECS Fargate** cluster via an Application Load Balancer.

### 1. Resource Configuration (`resource.ts`)

```typescript
import { defineFunction, secret } from '@aws-amplify/backend';

export const scanImage = defineFunction({
  name: 'scan-image',
  entry: './handler.ts',
  runtime: 22,
  memoryMB: 256,
  timeoutSeconds: 15,
  environment: {
    ECS_BASE_URL: "http://nutritrack-api-vpc-alb-xxxx.ap-southeast-2.elb.amazonaws.com",
    NUTRITRACK_API_KEY: secret('NUTRITRACK_API_KEY'),
  }
});
```

### 2. Handler Logic (`handler.ts`)

```typescript
import type { AppSyncResolverHandler } from 'aws-lambda';

export const handler: AppSyncResolverHandler<any, any> = async (event) => {
  const { image_key } = event.arguments;
  const baseUrl = process.env.ECS_BASE_URL;

  // Forward request to ECS Fargate via ALB
  const response = await fetch(`${baseUrl}/scan`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${process.env.NUTRITRACK_API_KEY}`
    },
    body: JSON.stringify({ image_key })
  });

  const result = await response.json();
  return result;
};
```

---

[Back to functions list](../)
