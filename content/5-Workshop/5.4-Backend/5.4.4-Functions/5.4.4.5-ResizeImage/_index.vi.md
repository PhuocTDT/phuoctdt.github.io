---
title: "Function resize-image"
menuTitle: "5.4.4.5. Function resize-image"
weight: 5
---

Hàm này được kích hoạt tự động mỗi khi có hình ảnh được tải lên S3 để thu nhỏ kích thước (Resize) và tạo hình ảnh thu nhỏ (Thumbnail).

### 1. Cài đặt thư viện (npm install)

Hàm này yêu cầu thư viện xử lý hình ảnh `sharp` và SDK của S3:

```bash
cd amplify/resize-image
npm install sharp @aws-sdk/client-s3
```

### 2. Chuẩn bị Lambda Layer (Sharp)

Vì thư viện `sharp` cần các file thực thi (binaries) tương thích với môi trường Linux của AWS Lambda, chúng ta nên đóng gói nó thành một Layer riêng bằng cách tải file `.zip` lên AWS Console.

![Tạo Lambda Layer](/images/layer.PNG)

Sau khi tạo xong, AWS sẽ trả về một mã **ARN**. Bạn sẽ dùng mã này để điền vào tệp `resource.ts` dưới đây.

### 3. Cấu hình Tài nguyên (`resource.ts`)

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

### 4. Mã nguồn xử lý (`handler.ts`)

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
    const parts = key.split('/');
    if (parts.length < 3 || parts[0] !== 'incoming') {
      console.warn(`Skip — unexpected path format: ${key}`);
      continue;
    }

    // entity_id is everything between 'incoming/' and the filename
    const entityId = parts.slice(1, parts.length - 1).join('/');
    const originalFilename = parts[parts.length - 1];
    const baseName = originalFilename.replace(/\.[^.]+$/, '');
    const destKey = `media/${entityId}/${baseName}.jpg`;

    console.log(`Processing: ${key} → ${destKey}`);

    try {
      // 1. Validate content type
      const head = await s3Client.send(new HeadObjectCommand({ Bucket: bucket, Key: key }));
      const contentType = head.ContentType || '';
      if (!['image/jpeg', 'image/png', 'image/webp', 'image/heic', 'image/heif'].includes(contentType)) {
        console.error(`Unsupported content type: ${contentType} for ${key}`);
        continue;
      }

      // 2. Read original image
      const s3Obj = await s3Client.send(new GetObjectCommand({ Bucket: bucket, Key: key }));
      const chunks: Uint8Array[] = [];
      for await (const chunk of s3Obj.Body as any) chunks.push(chunk);
      const originalBuffer = Buffer.concat(chunks);

      // 3. Resize — scale down to MAX_DIMENSION on the longest side, keep aspect ratio
      const resizedBuffer = await sharp(originalBuffer)
        .rotate()
        .resize(MAX_DIMENSION, MAX_DIMENSION, { fit: 'inside', withoutEnlargement: true })
        .jpeg({ quality: JPEG_QUALITY, progressive: true })
        .toBuffer();
        
      const originalKB = Math.round(originalBuffer.byteLength / 1024);
      const resizedKB = Math.round(resizedBuffer.byteLength / 1024);
      console.log(`Resized: ${originalKB}KB → ${resizedKB}KB`);

      // 4. Save compressed version to media/ (for food-detail display)
      await s3Client.send(new PutObjectCommand({
        Bucket: bucket,
        Key: destKey,
        Body: resizedBuffer,
        ContentType: 'image/jpeg',
      }));

      // NOTE: Do NOT delete from incoming/ — aiEngine reads from that path and
      // food-detail also uses the incoming/ key for its presigned URL.
      // The S3 lifecycle rule (expirationInDays: 1 on incoming/) handles cleanup.

      console.log(`Done: compressed image saved to ${destKey}`);
    } catch (err) {
      console.error(`Error processing ${key}:`, err);
    }
  }
};
```

---

[Quay lại danh sách các hàm](../)
