---
title: "Logic Functions (Functions)"
date: 2026-01-09T16:03:43+07:00
weight: 44
chapter: false
pre: " <b> 5.4.4. </b> "
---

Lambda functions orchestrate business logic and AI integrations. In Amplify Gen 2, each function is defined in its own directory within `amplify/functions/`.

## Core Project Functions

### 1. `ai-engine` — AI Core
Invokes Amazon Bedrock (Qwen3-VL) for multimodal analysis.
- **Directory**: `amplify/ai-engine/`

### 2. `scan-image` — Image Analysis (Proxy)
Acts as a proxy to send requests to the ECS Fargate tier.
- **Directory**: `amplify/scan-image/`

### 3. `process-nutrition` — Nutrition Logic
Calculates macronutrients using DynamoDB data or AI fallbacks.
- **Directory**: `amplify/process-nutrition/`

### 4. `resize-image` — Image Optimization
Uses the `sharp` library (via Lambda Layer) to generate thumbnails.
- **Directory**: `amplify/resize-image/`

### 5. `friend-request` — Social Layer
Handles friendship mutations and updates the Friendship table.
- **Directory**: `amplify/friend-request/`

---

## Detailed Implementation:

Create the following files and paste the provided skeleton code for each function:

1. [Function ai-engine (Bedrock)](5.4.4.1-AIEngine/)
2. [Function scan-image (ECS Proxy)](5.4.4.2-ScanImage/)
3. [Function process-nutrition (Logic & DB)](5.4.4.3-ProcessNutrition/)
4. [Function friend-request (Social logic)](5.4.4.4-FriendRequest/)
5. [Function resize-image (S3 Trigger & Sharp)](5.4.4.5-ResizeImage/)

---

[Continue to 5.5 ECS Fargate Tier](../5.5-ECS-Fargate/)
