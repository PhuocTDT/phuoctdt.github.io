---
title: "Function resize-image"
menuTitle: "5.4.4.5. Function resize-image"
weight: 5
---

This function is automatically triggered every time an image is uploaded to S3 to resize it and create a thumbnail.

### 1. Install Libraries (npm install)

This function requires the Sharp imaging library and the S3 Client SDK:

```bash
cd amplify/resize-image
npm install sharp @aws-sdk/client-s3
```

### 2. Prepare Lambda Layer (Sharp)

Since the `sharp` library needs native binaries compatible with the AWS Lambda Linux environment, we should package it as a separate Layer by uploading a `.zip` file to the AWS Console.

![Create Lambda Layer](/images/layer.PNG)

Once created, AWS will provide an **ARN**. You will use this ARN in the `resource.ts` file below.

### 3. Resource Configuration (`resource.ts`)

```typescript
import { defineFunction } from '@aws-amplify/backend';

export const resizeImage = defineFunction({
  name: 'resize-image',
  entry: './handler.ts',
  runtime: 22,
  memoryMB: 512,
  layers: {
    "sharp": `arn:aws:lambda:ap-southeast-2:${process.env.ACCOUNT_ID}:layer:sharp-node-layer:2`,
  },
  resourceGroupName: 'storage',
});
```

### 4. Handler Logic (`handler.ts`)

```typescript
import { S3Handler } from 'aws-lambda';
import { S3Client, HeadObjectCommand, PutObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3';
import sharp from 'sharp';

const s3Client = new S3Client({});

const MAX_DIMENSION = 1280; // px
const JPEG_QUALITY = 85;

export const handler: S3Handler = async (event) => {
  for (const record of event.Records) {
    const bucket = record.s3.bucket.name;
    const key = decodeURIComponent(record.s3.object.key.replace(/\+/g, ' '));

    // Path format: incoming/{entity_id}/{filename}
    // Logic for resizing and saving to media/ path...
    console.log(`Processing image: ${key}`);
  }
};
```

---

[Back to functions list](../)
