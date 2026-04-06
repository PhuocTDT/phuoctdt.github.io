---
title: "Processing Setup"
date: 2026-01-09T16:03:43+07:00
weight: 60
chapter: false
pre: " <b> 5.5. </b> "
---

# 5.5 Compute & AI — Lambda và Bedrock

Phần này nói về tầng compute của NutriTrack: bốn Lambda function cộng Amazon Bedrock. Kết hợp lại, chúng biến tấm ảnh tô phở thành một bản ghi nutrition có cấu trúc, trả lời câu hỏi coaching tiếng Việt, và giữ bucket ảnh gọn gàng.

## Bốn Lambda

Cả bốn nằm dưới `backend/amplify/*` và được đăng ký trong `defineBackend` ở `backend.ts`. Tất cả chạy **Node.js 22 trên ARM64** — không ngoại lệ.

| Function | Entry | Memory | Timeout | Resource group | Trigger |
| --- | --- | --- | --- | --- | --- |
| `ai-engine` | `ai-engine/handler.ts` | 512 MB | 120 s | (mặc định) | AppSync query `aiEngine` |
| `process-nutrition` | `process-nutrition/handler.ts` | 512 MB | 30 s | `data` | AppSync query `processNutrition` |
| `friend-request` | `friend-request/handler.ts` | (mặc định) | (mặc định) | (mặc định) | AppSync mutation `friendRequest` |
| `resize-image` | `resize-image/handler.ts` | 512 MB | (mặc định) | `storage` | S3 `ObjectCreated` trên `incoming/` |

`resourceGroupName` rất quan trọng: nó cho Amplify biết Lambda thuộc stack CloudFormation nào. `storage` = cùng stack với S3 bucket (tránh circular dep), `data` = cùng stack với bảng DynamoDB. `friend-request` và `ai-engine` nằm ở stack gốc.

Timeout 120 giây của `ai-engine` là bắt buộc vì `voiceToFood` khởi động một job Amazon Transcribe và poll — Transcribe có thể mất 30-60 giây end-to-end trước khi Bedrock được gọi.

## Kiến trúc

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

## Model Bedrock chuẩn

Mọi lời gọi Bedrock trong NutriTrack nhắm đúng một model:

- **Model ID**: `qwen.qwen3-vl-235b-a22b`
- **Region**: `ap-southeast-2` (Sydney)
- **API gọi**: `InvokeModelCommand` từ `@aws-sdk/client-bedrock-runtime`
- **Schema**: chat-completions kiểu OpenAI (`messages[]` với `role` và `content`, hỗ trợ `type: 'text'` và `type: 'image_url'`)

Model này đa phương thức (vision + text), xử lý tiếng Việt tốt từ gốc, và chi phí chỉ bằng một phần nhỏ Claude 3.5 Sonnet cho workload này. Lý do lựa chọn được bàn ở 5.5.1.

> Bất kỳ doc cũ nào hay comment cũ còn đề cập `anthropic.claude-3-haiku-...` đều sai — production đang chạy Qwen3-VL.

## AI coach persona: Ollie

Mỗi system prompt trong `ai-engine/handler.ts` đều bắt đầu bằng `You are Ollie...`. Ollie là persona thương hiệu của NutriTrack: coach dinh dưỡng Gen-Z Việt Nam, giọng điệu casual, mặc định trả lời bằng tiếng Việt. Không đổi tên thành "Bảo" hay tên khác — frontend, analytics, prompt library đều phụ thuộc vào "Ollie".

## Trang con

- [5.5.1 Bedrock — model access, pricing, IAM, cấu trúc invoke](5.5.1-Bedrock/)
- [5.5.2 AIEngine — Lambda orchestrator 10 action](5.5.2-AIEngine/)
- [5.5.3 ProcessNutrition — hybrid DB + Bedrock fuzzy lookup](5.5.3-ProcessNutrition/)
- [5.5.4 ResizeImage — resize ảnh bằng sharp, trigger từ S3](5.5.4-ResizeImage/)

Kế tiếp: [5.6 Automation Setup](../5.6-Automation-Setup/).
