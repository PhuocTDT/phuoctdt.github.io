---
title: "Bedrock"
date: 2026-01-09T16:03:43+07:00
weight: 5
chapter: false
pre: " <b> 5.5.1. </b> "
---

# 5.5.1 Bedrock — Model Access, IAM, Cấu trúc Invocation

Amazon Bedrock là dịch vụ foundation-model duy nhất mà NutriTrack dùng. Trang này bao phủ phần chuẩn bị (model access, region, IAM) và cấu trúc request chính xác mà Qwen3-VL mong đợi. Bỏ qua bước xin model access thì mọi lần test Lambda sau đó sẽ fail với `AccessDeniedException`.

## Bước 1 — Bật model access

> CẢNH BÁO: Các foundation model của Bedrock **mặc định bị tắt**, kể cả khi account đã bật Bedrock. Phải xin access rõ ràng cho từng model, từng region. Model Qwen thường được duyệt ngay, nhưng account mới có thể mất vài phút đến vài giờ.

Thao tác:

1. Mở AWS console → **Amazon Bedrock** → chuyển region sang **Asia Pacific (Sydney) `ap-southeast-2`**. Không thương lượng — Lambda hardcode region này, và chuyển model giữa các region là một quy trình duyệt khác.
2. Sidebar trái → **Model access** → **Modify model access**.
3. Tick **Qwen3-VL 235B A22B** (model id `qwen.qwen3-vl-235b-a22b`).
4. **Next** → **Submit**.
5. Chờ tới khi cột trạng thái hiện **Access granted**.

![Bedrock model access screen](/images/workshop/bedrock-model-access.png)

Verify bằng CLI:

```bash
aws bedrock list-foundation-models \
  --region ap-southeast-2 \
  --query "modelSummaries[?modelId=='qwen.qwen3-vl-235b-a22b']"
```

Nếu mảng trả về không rỗng và `modelLifecycle.status == 'ACTIVE'`, bạn đã có thể invoke.

## Bước 2 — Vì sao chọn Qwen3-VL thay vì Claude

Ba lý do workload này chọn Qwen3-VL:

1. **Đa phương thức trong một lần gọi**. Model nhận `type: 'image_url'` và `type: 'text'` trong cùng mảng `messages[]`. `analyzeFoodImage` gửi JPEG base64 + prompt tiếng Việt ngắn — một round trip.
2. **Tiếng Việt native**. System prompt của Ollie dày đặc giọng Gen-Z Việt (`ê`, `nhé`, `nha`, `nè`). Qwen xử lý thẳng, không cần dịch qua.
3. **Chi phí**. Với tỷ lệ NutriTrack (80% text, 20% vision), Qwen3-VL rẻ hơn đáng kể so với Claude 3.5 Sonnet trên mỗi 1M token. Giá chính xác thay đổi — tra trang pricing Bedrock cho số hiện tại ở `ap-southeast-2`.

Kiểm tra thứ tự độ lớn: chi phí mỗi user mỗi ngày nằm trong khoảng vài cent USD với user tích cực log.

## Bước 3 — IAM policy cho Lambda aiEngine

Policy nằm trong `backend/amplify/backend.ts`. Amplify không tự gắn quyền Bedrock — phải patch role Lambda qua CDK escape hatch:

```typescript
// backend.ts
const aiEngineLambda = backend.aiEngine.resources.lambda;

aiEngineLambda.addToRolePolicy(
  new iam.PolicyStatement({
    effect: iam.Effect.ALLOW,
    actions: ["bedrock:InvokeModel", "bedrock:InvokeModelWithResponseStream"],
    resources: [
      "arn:aws:bedrock:ap-southeast-2::foundation-model/qwen.qwen3-vl-235b-a22b",
    ],
  })
);
```

Điểm quan trọng:

- ARN khoá Lambda vào đúng một model. Sau này thêm model khác thì append ARN mới — không dùng `*`.
- Cả `InvokeModel` và `InvokeModelWithResponseStream` đều được cấp; handler hiện chỉ dùng cái đầu, streaming chỉ là một dòng đổi sau này cho coach response.
- Account ID trong ARN để trống (`::foundation-model`) — đây là chuẩn cho foundation model Bedrock vì chúng do AWS sở hữu.

Cùng file cũng cấp quyền S3 read/delete và Transcribe cho `aiEngine`:

```typescript
s3Bucket.grantRead(aiEngineLambda);
s3Bucket.grantDelete(aiEngineLambda);
aiEngineLambda.addToRolePolicy(
  new iam.PolicyStatement({
    effect: iam.Effect.ALLOW,
    actions: [
      "transcribe:StartTranscriptionJob",
      "transcribe:GetTranscriptionJob",
      "transcribe:DeleteTranscriptionJob",
    ],
    resources: ["*"],
  })
);
```

Và resource policy trên bucket để Transcribe (chạy dưới service principal của nó) đọc được file voice trực tiếp:

```typescript
s3Bucket.addToResourcePolicy(new iam.PolicyStatement({
  effect: iam.Effect.ALLOW,
  principals: [new iam.ServicePrincipal('transcribe.amazonaws.com')],
  actions: ['s3:GetObject'],
  resources: [`${s3Bucket.bucketArn}/voice/*`],
}));
```

Không có resource policy này, job Transcribe sẽ fail bất đồng bộ với `FailureReason` không rõ — role của Lambda không truyền sang Transcribe service.

## Bước 4 — Cấu trúc invocation cho Qwen3-VL

Qwen3-VL trên Bedrock phơi ra **schema chat-completions kiểu OpenAI**, không phải Messages API của Anthropic. Đừng copy code invoke của Claude — sẽ không chạy.

### Request chỉ text

```typescript
import { BedrockRuntimeClient, InvokeModelCommand } from "@aws-sdk/client-bedrock-runtime";

const client = new BedrockRuntimeClient({ region: "ap-southeast-2" });

const body = JSON.stringify({
  messages: [
    { role: "system", content: "You are Ollie, a Vietnamese nutrition coach." },
    { role: "user", content: "Phở bò có bao nhiêu calo?" },
  ],
  max_tokens: 500,
});

const response = await client.send(new InvokeModelCommand({
  modelId: "qwen.qwen3-vl-235b-a22b",
  contentType: "application/json",
  accept: "application/json",
  body,
}));

const parsed = JSON.parse(new TextDecoder().decode(response.body));
const text = parsed.choices?.[0]?.message?.content ?? "";
```

### Request vision (ảnh + text)

```typescript
const body = JSON.stringify({
  messages: [
    { role: "system", content: GEN_FOOD_SYSTEM_PROMPT },
    {
      role: "user",
      content: [
        {
          type: "image_url",
          image_url: { url: `data:image/jpeg;base64,${base64Image}` },
        },
        {
          type: "text",
          text: "Analyze this food image and return JSON only.",
        },
      ],
    },
  ],
  max_tokens: 1000,
});
```

Helper `callQwen` thật trong `ai-engine/handler.ts` bọc lại đoạn này và có fallback khi parse response:

```typescript
const text = responseBody.choices?.[0]?.message?.content
    || responseBody.output?.message?.content?.[0]?.text
    || responseBody.content?.[0]?.text
    || '';
```

Chuỗi fallback tồn tại vì Bedrock đã có lúc đổi schema giữa các revision Qwen. Luôn parse phòng thủ.

### Tuỳ chọn `ConverseCommand`

AWS có `ConverseCommand` như wrapper model-agnostic:

```typescript
import { ConverseCommand } from "@aws-sdk/client-bedrock-runtime";

await client.send(new ConverseCommand({
  modelId: "qwen.qwen3-vl-235b-a22b",
  messages: [{ role: "user", content: [{ text: "Hello" }] }],
}));
```

Nó chạy cho text-only, nhưng phần image-content lệch với schema native của Qwen. NutriTrack dùng `InvokeModelCommand` trực tiếp để call vision đơn giản hơn.

## Bước 5 — Smoke-test bằng CLI

Trước khi đụng Lambda, chứng minh model phản hồi:

```bash
aws bedrock-runtime invoke-model \
  --model-id qwen.qwen3-vl-235b-a22b \
  --region ap-southeast-2 \
  --content-type application/json \
  --accept application/json \
  --body '{"messages":[{"role":"user","content":"Nói xin chào nhé"}],"max_tokens":50}' \
  --cli-binary-format raw-in-base64-out \
  /tmp/bedrock-out.json

cat /tmp/bedrock-out.json | jq '.choices[0].message.content'
```

Kỳ vọng: một câu chào tiếng Việt ngắn. Nếu gặp `AccessDeniedException`, xem lại bước 1. Nếu gặp `ValidationException: Unknown model`, xem lại region.

## Bước 6 — Retry và throttling

Bedrock áp quota concurrent-request theo account-region. Khi bị throttle, `InvokeModelCommand` ném `ThrottlingException`. aws-sdk-v3 đã có sẵn retry strategy (`standard`, 3 lần retry backoff) xử lý tự động. Với workload burst (ví dụ một push notification kích hàng trăm cuộc gọi `weeklyInsight`), tăng retry:

```typescript
import { StandardRetryStrategy } from "@aws-sdk/util-retry";

const client = new BedrockRuntimeClient({
  region: "ap-southeast-2",
  retryStrategy: new StandardRetryStrategy(async () => 6, { maxRetries: 6 }),
});
```

Cho throughput bền vững, xin tăng quota qua Service Quotas → Amazon Bedrock → **Model invocations per minute for `qwen.qwen3-vl-235b-a22b`**.

## Liên kết

- Lambda thực sự gọi Bedrock: [5.5.2 AIEngine](../5.5.2-AIEngine/)
- Hybrid DB + Bedrock: [5.5.3 ProcessNutrition](../5.5.3-ProcessNutrition/)
- CloudWatch log và alarm: [5.6 Automation Setup](../../5.6-Automation-Setup/)
