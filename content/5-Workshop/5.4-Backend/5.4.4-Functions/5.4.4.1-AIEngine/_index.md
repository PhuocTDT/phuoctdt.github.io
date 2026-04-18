---
title: "Function ai-engine (Bedrock)"
menuTitle: "5.4.4.1. Function ai-engine"
weight: 1
---

This function acts as the "brain" of the application, responsible for interacting with **Amazon Bedrock** to analyze natural language and images.

### 1. Install Libraries (npm install)

This function requires the Bedrock Runtime SDK and the Transcribe SDK (if processing voice):

```bash
cd amplify/ai-engine
npm install @aws-sdk/client-bedrock-runtime @aws-sdk/client-transcribe
```

### 2. Resource Configuration (`resource.ts`)

```typescript
import { defineFunction } from '@aws-amplify/backend';

export const aiEngine = defineFunction({
  name: 'ai-engine',
  entry: './handler.ts',
  runtime: 22,
  memoryMB: 512,
  timeoutSeconds: 30,
});
```

### 3. Handler Logic (`handler.ts`)

```typescript
import { defineFunction } from '@aws-amplify/backend';

export const aiEngine = defineFunction({
  name: 'ai-engine',
  entry: './handler.ts',
  runtime: 22,
  memoryMB: 512,
  timeoutSeconds: 120, // voiceToFood: Transcribe polling + Bedrock can exceed 60s
  resourceGroupName: 'data',
});

```

---

[Back to functions list](../)
