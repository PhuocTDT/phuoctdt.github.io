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
import { BedrockRuntimeClient, InvokeModelCommand } from "@aws-sdk/client-bedrock-runtime";

const client = new BedrockRuntimeClient({ region: "us-east-1" });

export const handler = async (event: any) => {
  const { prompt } = event.arguments;

  const input = {
    modelId: "anthropic.claude-3-haiku-20240307-v1:0",
    contentType: "application/json",
    accept: "application/json",
    body: JSON.stringify({
      anthropic_version: "bedrock-2023-05-31",
      max_tokens: 1000,
      messages: [
        {
          role: "user",
          content: [{ type: "text", text: prompt }],
        },
      ],
    }),
  };

  try {
    const command = new InvokeModelCommand(input);
    const response = await client.send(command);
    const result = JSON.parse(new TextDecoder().decode(response.body));
    
    return result.content[0].text;
  } catch (error) {
    console.error(error);
    return "Error calling AI Engine";
  }
};
```

---

[Back to functions list](../)
