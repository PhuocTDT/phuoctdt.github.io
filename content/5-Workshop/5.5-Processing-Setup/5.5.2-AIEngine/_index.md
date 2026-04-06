---
title: "AIEngine"
date: 2026-01-09T16:03:43+07:00
weight: 5
chapter: false
pre: " <b> 5.5.2. </b> "
---

# 5.5.2 AIEngine — The 10-Action Lambda Orchestrator

`ai-engine` is a single Lambda that handles every AI-powered interaction in NutriTrack. Rather than one Lambda per feature (which would multiply cold starts and IAM policies), we ship one orchestrator with a top-level `action` switch. The AppSync custom query forwards the `action` string and a JSON `payload`; the handler dispatches to the right branch.

## Resource definition

```typescript
// backend/amplify/ai-engine/resource.ts
import { defineFunction } from '@aws-amplify/backend';

export const aiEngine = defineFunction({
  name: 'ai-engine',
  entry: './handler.ts',
  runtime: 22,
  memoryMB: 512,
  timeoutSeconds: 120, // voiceToFood: Transcribe polling + Bedrock can exceed 60s
});
```

Why 120 seconds: `voiceToFood` starts an Amazon Transcribe job on an audio file in `s3://{bucket}/voice/...` and polls it with `GetTranscriptionJob` every 2 seconds, up to 25 iterations (50 seconds). After Transcribe completes, the transcript is sent to Qwen3-VL, which adds another 5-15 seconds. The 60-second default is not enough.

## Handler constants

```typescript
const REGION = "ap-southeast-2";
const bedrockClient = new BedrockRuntimeClient({ region: REGION });
const transcribeClient = new TranscribeClient({ region: REGION });
const s3Client = new S3Client({ region: REGION });

const QWEN_MODEL_ID = process.env.QWEN_MODEL_ID || "qwen.qwen3-vl-235b-a22b";
const STORAGE_BUCKET = process.env.STORAGE_BUCKET_NAME || "";
```

- `REGION` is hard-coded to `ap-southeast-2` because that is where the Bedrock model lives. Do not read this from `AWS_REGION` — the Lambda itself can run anywhere, but Bedrock calls must go to Sydney.
- `QWEN_MODEL_ID` defaults to the correct value but can be overridden via env var for future A/B testing.
- `STORAGE_BUCKET_NAME` is injected from `backend.ts` via a CDK escape hatch — the bucket ARN is not known at Lambda build time:

```typescript
// In backend.ts
const cfnAiEngineFn = aiEngineLambda.node.defaultChild as cdk.aws_lambda.CfnFunction;
cfnAiEngineFn.addPropertyOverride(
  'Environment.Variables.STORAGE_BUCKET_NAME',
  s3Bucket.bucketName
);
```

## The 10 actions

The handler routes on `event.arguments.action`. Every branch parses `event.arguments.payload` (a JSON string), calls one or more AWS services, and returns `JSON.stringify({ success, ... })`.

| Action | Purpose | Services used |
| --- | --- | --- |
| `analyzeFoodImage` | Photo of food → structured nutrition JSON | S3 get, Bedrock (vision) |
| `generateCoachResponse` | Conversational Ollie replies with card delimiters | Bedrock (text) |
| `searchFoodNutrition` | Food name lookup when DB misses | Bedrock (text) |
| `fixFood` | User correction to a logged food item | Bedrock (text) |
| `voiceToFood` | Audio upload → transcript → parsed food JSON | Transcribe, S3, Bedrock |
| `ollieCoachTip` | One-sentence daily nudge based on stats | Bedrock (text) |
| `generateRecipe` | 1–3 recipes from fridge inventory | Bedrock (text) |
| `calculateMacros` | Mifflin-St Jeor TDEE from biometrics | Bedrock (text) |
| `challengeSummary` | Leaderboard blurb | Bedrock (text) |
| `weeklyInsight` | 3-sentence weekly progress summary | Bedrock (text) |

## System prompts — the Ollie persona

Every action ships with a dedicated system prompt, all starting with `You are Ollie`. The five foundational prompts:

1. **`GEN_FOOD_SYSTEM_PROMPT`** — used by `analyzeFoodImage` and `searchFoodNutrition`. Forces a rigid JSON schema (`food_id`, `name_vi`, `name_en`, `macros`, `micronutrients`, `serving`, `ingredients`, `verified`, `source`) and enforces the caloric-consistency rule `Protein*4 + Carbs*4 + Fat*9 ≈ calories`.
2. **`FIX_FOOD_SYSTEM_PROMPT`** — used by `fixFood`. Same output schema but `"source": "AI Fixed"`. Recalculates all macros when weights change.
3. **`VOICE_SYSTEM_PROMPT`** — used by `voiceToFood` after Transcribe. Returns `action: "log" | "clarify"` so the client knows whether to prompt the user or log immediately.
4. **`OLLIE_COACH_SYSTEM_PROMPT`** — used by `ollieCoachTip`. Max 2 sentences, 1-2 emojis, references actual streak/calorie stats.
5. **`AI_COACH_SYSTEM_PROMPT`** — used by `generateCoachResponse`. The conversational one. Emits `===FOOD_CARD_START===` / `===FOOD_CARD_END===` delimiters so the frontend can render rich cards.

Plus three smaller specialist prompts: `RECIPE_SYSTEM_PROMPT`, `MACRO_CALCULATOR_SYSTEM_PROMPT`, `CHALLENGE_SYSTEM_PROMPT`, `WEEKLY_INSIGHT_SYSTEM_PROMPT`.

All are in `handler.ts` at the top of the file — intentionally inlined to keep deployment artifacts self-contained.

## The Bedrock call helper

All ten actions eventually hit this helper:

```typescript
async function callQwen(messages: any[], maxTokens = 1000): Promise<string> {
  const body = JSON.stringify({ messages, max_tokens: maxTokens });
  const command = new InvokeModelCommand({
    modelId: QWEN_MODEL_ID,
    contentType: "application/json",
    accept: "application/json",
    body,
  });
  const response = await bedrockClient.send(command);
  const responseBody = JSON.parse(new TextDecoder().decode(response.body));
  const text = responseBody.choices?.[0]?.message?.content
    || responseBody.output?.message?.content?.[0]?.text
    || responseBody.content?.[0]?.text
    || '';
  return text;
}
```

The fallback chain for reading the text exists because the Bedrock Qwen response schema has shifted between model revisions. Always parse defensively.

## Deep dive: `analyzeFoodImage`

This is the headline feature — point the camera at phở, get back structured nutrition. Full handler branch:

```typescript
if (action === 'analyzeFoodImage') {
  const { s3Key } = data;
  if (!STORAGE_BUCKET) throw new Error('STORAGE_BUCKET_NAME not configured');
  if (!s3Key || s3Key.includes('..')) throw new Error('Invalid s3Key');

  // Read image from S3, convert to base64
  const s3Obj = await s3Client.send(new GetObjectCommand({
    Bucket: STORAGE_BUCKET,
    Key: s3Key,
  }));
  const chunks: Uint8Array[] = [];
  for await (const chunk of s3Obj.Body as any) chunks.push(chunk);
  const imageBuffer = Buffer.concat(chunks);
  const base64 = imageBuffer.toString('base64');
  const contentType = s3Obj.ContentType || 'image/jpeg';

  const text = await callQwen([
    { role: "system", content: GEN_FOOD_SYSTEM_PROMPT },
    {
      role: "user",
      content: [
        { type: "image_url", image_url: { url: `data:${contentType};base64,${base64}` } },
        { type: "text", text: "Analyze this food image and estimate its nutritional profile. Use Vietnamese for name_vi and all ingredient name_vi fields. Return ONLY the JSON object." },
      ],
    },
  ]);

  return JSON.stringify({ success: true, text });
}
```

Security notes:

- `s3Key.includes('..')` blocks path traversal attempts.
- The Lambda has IAM read on the whole bucket (granted in `backend.ts` via `s3Bucket.grantRead(aiEngineLambda)`), so the caller cannot escalate by passing an arbitrary key — they still need AppSync auth.
- The image is base64-encoded and sent inline in the Bedrock payload, avoiding a second round trip through AppSync.

## Deep dive: `voiceToFood` + Transcribe

This is the longest-running branch. Flow:

1. User records audio in the Expo app and uploads it to `s3://{bucket}/voice/{user}/{uuid}.m4a` using an Amplify Storage presigned URL.
2. Client calls the `aiEngine` query with `action: 'voiceToFood'` and the `s3Key`.
3. Lambda starts an Amazon Transcribe job:

```typescript
const jobName = `nutritrack-voice-${randomUUID()}`;
const ext = s3Key.split('.').pop()?.toLowerCase() || 'm4a';
const mediaFormat = ext === 'webm' ? 'webm'
  : ext === 'mp3'  ? 'mp3'
  : ext === 'wav'  ? 'wav'
  : ext === 'flac' ? 'flac'
  : 'mp4'; // m4a, mp4 → 'mp4'

await transcribeClient.send(new StartTranscriptionJobCommand({
  TranscriptionJobName: jobName,
  LanguageCode: 'vi-VN',
  MediaFormat: mediaFormat as any,
  Media: { MediaFileUri: `s3://${STORAGE_BUCKET}/${s3Key}` },
}));
```

4. `waitForTranscription(jobName)` polls every 2 seconds, up to 25 iterations. When `COMPLETED`, it fetches the transcript JSON from the URL Transcribe writes to a managed S3 location.
5. The transcript string is fed to Qwen with `VOICE_SYSTEM_PROMPT`, which returns a food-log JSON.

Critical detail: the language code is **hard-coded to `vi-VN`**. An earlier version used `IdentifyLanguage: true`, which caused empty transcripts on WebM inputs from web clients. Do not use automatic language detection here.

## Wiring to AppSync

The schema entry that makes the Lambda callable:

```typescript
// data/resource.ts
aiEngine: a
  .query()
  .arguments({
    action: a.string().required(),
    payload: a.string(),
  })
  .returns(a.string())
  .handler(a.handler.function(aiEngine))
  .authorization((allow) => [allow.authenticated()]),
```

And the frontend call:

```typescript
const res = await client.queries.aiEngine({
  action: 'analyzeFoodImage',
  payload: JSON.stringify({ s3Key: 'incoming/user-abc/photo-1.jpg' }),
});
const parsed = JSON.parse(res.data ?? '{}');
if (parsed.success) {
  const foodData = JSON.parse(parsed.text); // Qwen's JSON string
  // ... render nutrition card
}
```

Double-stringification is intentional: AppSync returns `String`, Lambda returns `JSON.stringify({ success, text })`, and `text` is itself a JSON string produced by Qwen. The frontend parses twice. This keeps the GraphQL schema simple and avoids declaring every possible Qwen output shape in the schema.

## Error handling

```typescript
} catch (error: any) {
  debug('Bedrock Lambda Error:', error.message);
  return JSON.stringify({ success: false, error: error.message });
}
```

Errors always return a successful AppSync response with `success: false` — never a GraphQL error. This makes the frontend's error handling trivial and avoids retry storms from the Amplify client's automatic retry on GraphQL errors.

Common failure modes:

- `STORAGE_BUCKET_NAME not configured` — missing env var. Rerun `npx ampx sandbox`.
- `AccessDeniedException` on Bedrock — model access not granted in Sydney. See 5.5.1 step 1.
- `ThrottlingException` — concurrent request quota. Handled by sdk retry strategy.
- `Transcription timed out` — more than 50 seconds of polling. Usually a bad audio format.

## Observing

CloudWatch Logs group: `/aws/lambda/amplify-*-ai-engine-*`. Enable the `DEBUG=true` env var for verbose logging — the handler's `debug()` helper is a no-op in production.

![CloudWatch ai-engine logs](/images/workshop/cloudwatch-ai-logs.png)

## Cross-links

- Bedrock prereqs: [5.5.1 Bedrock](../5.5.1-Bedrock/)
- The other Lambdas: [5.5.3 ProcessNutrition](../5.5.3-ProcessNutrition/), [5.5.4 ResizeImage](../5.5.4-ResizeImage/)
- Alerting on errors: [5.6 Automation Setup](../../5.6-Automation-Setup/)
