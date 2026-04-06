---
title: "AIEngine"
date: 2026-01-09T16:03:43+07:00
weight: 5
chapter: false
pre: " <b> 5.5.2. </b> "
---

### 5.5.2 Triển khai Hàm Lambda AI Engine

Thay vì tạo nhiều hàm rác, NutriTrack gom 10 tính năng AI (Phân tích ảnh, Gợi ý công thức, Trợ lý Ollie) vào một Lambda Orchestrator duy nhất mang tên `aiEngine`.

#### Bước 1: Khởi tạo Lambda
Tại thư mục `amplify/functions/aiEngine/resource.ts`:

```typescript
import { defineFunction } from '@aws-amplify/backend';

export const aiEngine = defineFunction({
  entry: './handler.ts',
  timeoutSeconds: 30, // Tăng timeout do Bedrock xử lý hình ảnh khá lâu
  memoryMB: 512,
  environment: {
    MODEL_ID: 'anthropic.claude-3-haiku-20240307-v1:0'
  }
});
```

#### Bước 2: Tích hợp Bedrock Runtime SDK
Tại `handler.ts`:

```typescript
import { BedrockRuntimeClient, InvokeModelCommand } from "@aws-sdk/client-bedrock-runtime";

const client = new BedrockRuntimeClient({ region: "ap-southeast-2" });

export const handler = async (event) => {
  // Extract file base64 từ S3 / AppSync truyền sang
  // Dùng client.send(new InvokeModelCommand({...}))
};
```