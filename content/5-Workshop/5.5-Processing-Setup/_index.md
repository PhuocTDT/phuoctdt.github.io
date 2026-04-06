---
title: "Processing Setup"
date: 2026-01-09T16:03:43+07:00
weight: 60
chapter: false
pre: " <b> 5.5. </b> "
---

# 5.5 Compute & AI — Lambdas and Bedrock

This section covers the compute layer of NutriTrack: four Lambda functions plus Amazon Bedrock. Together they turn a photo of phở into a structured nutrition log, answer Vietnamese-language coaching questions, and keep the image bucket tidy.

## The four Lambdas

All four are defined under `backend/amplify/*` and registered in `defineBackend` at `backend.ts`. Every one runs **Node.js 22 on ARM64** — no exceptions.

| Function | Entry | Memory | Timeout | Resource group | Trigger |
| --- | --- | --- | --- | --- | --- |
| `ai-engine` | `ai-engine/handler.ts` | 512 MB | 120 s | (default) | AppSync query `aiEngine` |
| `process-nutrition` | `process-nutrition/handler.ts` | 512 MB | 30 s | `data` | AppSync query `processNutrition` |
| `friend-request` | `friend-request/handler.ts` | (default) | (default) | (default) | AppSync mutation `friendRequest` |
| `resize-image` | `resize-image/handler.ts` | 512 MB | (default) | `storage` | S3 `ObjectCreated` on `incoming/` |

The `resourceGroupName` matters: it tells Amplify which CloudFormation stack the Lambda belongs to. `storage` = grouped with the S3 bucket (no circular deps), `data` = grouped with the DynamoDB tables. `friend-request` and `ai-engine` sit in the root stack.

The 120-second timeout on `ai-engine` is required because `voiceToFood` starts an Amazon Transcribe job and polls it — Transcribe can take 30-60 seconds end-to-end before Bedrock is even called.

## Architecture

```mermaid
flowchart LR
  subgraph Client
    App[Expo app]
  end

  subgraph API
    AppSync[AWS AppSync]
  end

  subgraph Compute
    AiEngine[ai-engine<br/>Node 22 / 512 MB / 120s]
    ProcNut[process-nutrition<br/>Node 22 / 512 MB / 30s]
    FriendReq[friend-request<br/>Node 22]
    ResizeImg[resize-image<br/>Node 22 / 512 MB]
  end

  subgraph AI
    Bedrock[Bedrock<br/>qwen.qwen3-vl-235b-a22b<br/>ap-southeast-2]
    Transcribe[Amazon Transcribe<br/>vi-VN]
  end

  subgraph Storage
    S3[(S3 bucket)]
    DDB[(DynamoDB<br/>Food, user, Friendship)]
  end

  App --> AppSync
  AppSync --> AiEngine
  AppSync --> ProcNut
  AppSync --> FriendReq
  S3 -->|ObjectCreated incoming/*| ResizeImg

  AiEngine --> Bedrock
  AiEngine --> Transcribe
  AiEngine --> S3
  ProcNut --> DDB
  ProcNut --> Bedrock
  FriendReq --> DDB
  ResizeImg --> S3
```

## Ground-truth Bedrock model

Every Bedrock call in NutriTrack targets exactly one model:

- **Model ID**: `qwen.qwen3-vl-235b-a22b`
- **Region**: `ap-southeast-2` (Sydney)
- **Invocation API**: `InvokeModelCommand` from `@aws-sdk/client-bedrock-runtime`
- **Schema**: OpenAI-compatible chat-completions (`messages[]` with `role` and `content`, supporting `type: 'text'` and `type: 'image_url'`)

This model is multimodal (vision + text), handles Vietnamese natively, and costs a fraction of Claude 3.5 Sonnet for this workload. The choice is discussed in 5.5.1.

> Any older docs or stale comments referencing `anthropic.claude-3-haiku-...` are wrong — production runs Qwen3-VL.

## The AI coach persona: Ollie

Every system prompt in `ai-engine/handler.ts` begins with `You are Ollie...`. Ollie is the NutriTrack brand persona: a Gen-Z Vietnamese nutrition coach, casual tone, responds in Vietnamese by default. Do not rename to "Bảo" or anything else — the frontend, analytics, and prompt library all depend on "Ollie."

## Sub-pages

- [5.5.1 Bedrock — model access, pricing, IAM, invocation shape](5.5.1-Bedrock/)
- [5.5.2 AIEngine — the 10-action Lambda orchestrator](5.5.2-AIEngine/)
- [5.5.3 ProcessNutrition — hybrid DB + Bedrock fuzzy lookup](5.5.3-ProcessNutrition/)
- [5.5.4 ResizeImage — S3-triggered sharp resizer](5.5.4-ResizeImage/)

Next: [5.6 Automation Setup](../5.6-Automation-Setup/).
