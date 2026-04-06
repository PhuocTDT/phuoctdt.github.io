---
title: "IAM Policies"
date: 2026-01-09T16:03:43+07:00
weight: 5
chapter: false
pre: " <b> 5.11.2. </b> "
---

# 5.11.2 IAM Policies

This appendix documents every IAM statement attached to the workshop's Lambda execution roles and the S3 bucket resource policy. All JSON below is extracted from `backend/amplify/backend.ts` — the single CDK entry point where Amplify Gen 2 layers additional permissions on top of the auto-generated roles.

Amplify Gen 2 gives each Lambda a managed execution role by default (with `AWSLambdaBasicExecutionRole` for CloudWatch logs). Everything else — Bedrock, Transcribe, DynamoDB, S3 — is added explicitly via the CDK escape hatch:

```typescript
backend.<fn>.resources.lambda.addToRolePolicy(new iam.PolicyStatement({ ... }))
```

If a statement is not in `backend.ts`, the Lambda does not have it. There is no hidden configuration.

## Source reference

- File: `backend/amplify/backend.ts`
- CDK imports: `aws-cdk-lib/aws-iam`, `aws-cdk-lib/aws-s3`, `aws-cdk-lib/aws-s3-notifications`

## aiEngine role

The `aiEngine` Lambda handles all Bedrock calls, voice transcription, and image reads from S3. It is the most privileged of the four workshop Lambdas.

### Bedrock: invoke the Qwen3-VL model

```json
{
  "Effect": "Allow",
  "Action": [
    "bedrock:InvokeModel",
    "bedrock:InvokeModelWithResponseStream"
  ],
  "Resource": [
    "arn:aws:bedrock:ap-southeast-2::foundation-model/qwen.qwen3-vl-235b-a22b"
  ]
}
```

Notes:

- The resource ARN is pinned to a single model in `ap-southeast-2`. If you add a second model (for example a text-only Claude fallback), append its ARN to the `Resource` array rather than using `*`.
- `InvokeModelWithResponseStream` is included because the handler keeps the streaming door open even though the current code calls `InvokeModel` synchronously via `callQwen()`.

### Transcribe: async voice-to-text jobs

```json
{
  "Effect": "Allow",
  "Action": [
    "transcribe:StartTranscriptionJob",
    "transcribe:GetTranscriptionJob",
    "transcribe:DeleteTranscriptionJob"
  ],
  "Resource": ["*"]
}
```

Notes:

- Transcribe does not support resource-level ARNs for `StartTranscriptionJob` in every region, which is why `Resource` is `*`. This is the widest grant in the stack — see the callout at the bottom of this page.
- `DeleteTranscriptionJob` is present so the Lambda can clean up finished jobs; the handler currently leaves this commented out for debugging, but the permission stays so we don't redeploy when re-enabling.

### S3: read images and voice files, delete on cleanup

These grants come from the CDK helpers rather than raw policy statements:

```typescript
s3Bucket.grantRead(aiEngineLambda);
s3Bucket.grantDelete(aiEngineLambda);
```

Which expands to `s3:GetObject`, `s3:GetObject*`, `s3:List*`, `s3:DeleteObject*` on the bucket ARN and `bucket/*`. The Lambda reads from `incoming/` (photos) and `voice/` (audio) and deletes when cleanup is enabled.

### Environment variable injection

The S3 bucket name is passed in via a CFN property override, not a hard-coded value:

```typescript
const cfnAiEngineFn = aiEngineLambda.node.defaultChild as cdk.aws_lambda.CfnFunction;
cfnAiEngineFn.addPropertyOverride(
  'Environment.Variables.STORAGE_BUCKET_NAME',
  s3Bucket.bucketName
);
```

The handler reads this as `process.env.STORAGE_BUCKET_NAME` and fails fast if it is empty.

## processNutrition role

The `processNutrition` Lambda answers nutrition lookups by scanning the `Food-*` DynamoDB table with a fuzzy-match algorithm, and only calls Bedrock when the DB lookup misses.

### DynamoDB: read the Food table

```json
{
  "Effect": "Allow",
  "Action": [
    "dynamodb:Scan",
    "dynamodb:Query",
    "dynamodb:GetItem",
    "dynamodb:BatchGetItem",
    "dynamodb:DescribeTable"
  ],
  "Resource": ["arn:aws:dynamodb:*:*:table/Food-*"]
}
```

The `Food-*` wildcard matches the Amplify-generated name (e.g. `Food-abc123xyz-NONE`). No writes are allowed — the seed data is immutable at runtime.

### DynamoDB: list tables (legacy discovery path)

```json
{
  "Effect": "Allow",
  "Action": ["dynamodb:ListTables"],
  "Resource": ["*"]
}
```

This is the risky one. The Lambda's `discoverTableName()` helper falls back to `ListTables()` when `FOOD_TABLE_NAME` is unset, then picks the first name starting with `Food-`. In a shared AWS account with multiple sandboxes, that picks the *wrong* table.

**Recommended**: inject `FOOD_TABLE_NAME` explicitly (mirroring the pattern used for `friendRequest`) and remove `ListTables` from the policy before production:

```typescript
const cfnProcessNutritionFn = backend.processNutrition.resources.cfnResources.cfnFunction;
cfnProcessNutritionFn.addPropertyOverride(
  'Environment.Variables.FOOD_TABLE_NAME',
  backend.data.resources.tables['Food'].tableName
);
```

### Bedrock: fallback path

The same Bedrock statement as `aiEngine` is attached here as well, because the nutrition lookup falls through to a Qwen3-VL call when no DB row matches. Resource ARN is identical.

## friendRequest role

The `friendRequest` Lambda implements the friend graph: send / accept / decline / remove / block. It only touches DynamoDB.

### DynamoDB: user + Friendship tables

```json
{
  "Effect": "Allow",
  "Action": [
    "dynamodb:GetItem",
    "dynamodb:PutItem",
    "dynamodb:UpdateItem",
    "dynamodb:DeleteItem",
    "dynamodb:Query",
    "dynamodb:Scan",
    "dynamodb:BatchGetItem",
    "dynamodb:BatchWriteItem",
    "dynamodb:DescribeTable",
    "dynamodb:TransactWriteItems"
  ],
  "Resource": [
    "arn:aws:dynamodb:*:*:table/user-*",
    "arn:aws:dynamodb:*:*:table/user-*/index/*",
    "arn:aws:dynamodb:*:*:table/Friendship-*",
    "arn:aws:dynamodb:*:*:table/Friendship-*/index/*"
  ]
}
```

Notes:

- `TransactWriteItems` is required because accepting a friend request writes two rows (A→B and B→A) in a single transaction.
- The `/index/*` suffix allows `Query` on the `friend_id` GSI.
- No Bedrock. No S3. If a future feature needs to send a Bedrock-generated "friend suggestion", add a new statement — do not relax this one.

### Environment variable injection (the right way)

```typescript
const cfnFriendRequestFn = friendRequestLambda.node.defaultChild as cdk.aws_lambda.CfnFunction;
cfnFriendRequestFn.addPropertyOverride(
  'Environment.Variables.USER_TABLE_NAME',
  backend.data.resources.tables['user'].tableName
);
cfnFriendRequestFn.addPropertyOverride(
  'Environment.Variables.FRIENDSHIP_TABLE_NAME',
  backend.data.resources.tables['Friendship'].tableName
);
```

CDK resolves the correct table name at synth time per environment, so sandbox and branch deploys never collide. This is the pattern `processNutrition` should adopt.

## resizeImage role

The `resizeImage` Lambda is wired as an S3 event trigger on the `incoming/` prefix. It reads the uploaded image, resizes it with sharp, writes to `media/`, and deletes the original.

Permissions are granted via CDK helpers, not raw statements:

```typescript
s3Bucket.grantReadWrite(resizeLambda);
s3Bucket.grantDelete(resizeLambda);
```

Which expands to:

- `s3:GetObject*`, `s3:GetBucket*`, `s3:List*` on the bucket
- `s3:PutObject`, `s3:PutObjectLegalHold`, `s3:PutObjectRetention`, `s3:PutObjectTagging`, `s3:PutObjectVersionTagging`, `s3:Abort*` for writes
- `s3:DeleteObject*` for cleanup

The Lambda has **no** Bedrock, **no** DynamoDB, **no** Transcribe. If your build adds another service, the least-privilege rule says: create a new statement, do not expand one of the existing ones.

## S3 bucket resource policy

Transcribe runs asynchronously under its own service role, not the Lambda's. That role has no S3 access by default, so we attach a bucket policy granting the Transcribe service principal read on the `voice/*` prefix:

```json
{
  "Effect": "Allow",
  "Principal": {
    "Service": "transcribe.amazonaws.com"
  },
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::<bucket-name>/voice/*"
}
```

In CDK:

```typescript
s3Bucket.addToResourcePolicy(new iam.PolicyStatement({
  effect: iam.Effect.ALLOW,
  principals: [new iam.ServicePrincipal('transcribe.amazonaws.com')],
  actions: ['s3:GetObject'],
  resources: [`${s3Bucket.bucketArn}/voice/*`],
}));
```

Without this statement, `StartTranscriptionJob` fails with `AccessDenied` even though the Lambda role has `transcribe:*` — because Transcribe, not the Lambda, is the caller that reads S3.

## S3 lifecycle rule (not IAM, but relevant)

The `incoming/` prefix auto-expires after 1 day so the landing zone never accumulates unprocessed uploads:

```typescript
cfnBucket.lifecycleConfiguration = {
  rules: [{
    id: 'CleanupIncomingLandingZone',
    status: 'Enabled',
    prefix: 'incoming/',
    expirationInDays: 1
  }]
};
```

This is mentioned here because reviewers often ask "why isn't `incoming/` growing?" — this rule is the answer, not an IAM policy.

## Managed Cognito service role

Amplify Gen 2 creates the Cognito User Pool, its client, the identity pool, and their associated IAM roles automatically from `auth/resource.ts`. **Do not edit those roles by hand.** If you need to add a post-confirmation Lambda trigger, wire it through the Amplify API (`defineAuth({ triggers: { ... } })`) — the scaffolding will update the role for you on the next deploy. Hand-editing drifts the CloudFormation state and the next `ampx sandbox` run will revert or fail.

## Principle of least privilege — pre-production checklist

Before shipping beyond the workshop sandbox, tighten every `Resource: "*"` in the stack:

1. **Transcribe**: scope by job name prefix. Use a resource ARN of the form `arn:aws:transcribe:ap-southeast-2:<account>:transcription-job/nutritrack-voice-*` for `GetTranscriptionJob` and `DeleteTranscriptionJob`. `StartTranscriptionJob` may still require `*` depending on region — verify against the current AWS IAM reference.
2. **DynamoDB ListTables**: remove entirely from `processNutrition` after injecting `FOOD_TABLE_NAME` via env var (see the block above).
3. **S3 bucket policy**: confirm `<bucket-name>` resolves to the single workshop bucket. If you fork the stack into multiple buckets, each needs its own statement.
4. **Bedrock**: keep the single-model ARN. If you add another model, append — never switch to `arn:aws:bedrock:*::foundation-model/*`.
5. **Cognito**: no manual edits.

Every one of those `*` is a finding a reviewer will flag. Fix them before, not after.

## Related files

- `backend/amplify/backend.ts` — the file this appendix documents
- `backend/amplify/process-nutrition/handler.ts` — `discoverTableName()` pattern
- `backend/amplify/friend-request/handler.ts` — env-var pattern (the good one)
- `backend/amplify/ai-engine/handler.ts` — Bedrock + Transcribe + S3 consumer
- `backend/amplify/resize-image/handler.ts` — S3 event consumer
