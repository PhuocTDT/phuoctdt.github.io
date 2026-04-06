---
title: "Overview"
date: 2026-01-09T16:03:43+07:00
weight: 100
chapter: false
pre: " <b> 5.1. </b> "
---

# 5.1 Overview

NutriTrack is a production-grade, AI-powered nutrition tracking platform built on AWS Amplify Gen 2. This workshop walks you through deploying the exact backend and mobile client that ships in `TEMPLATE/neurax-web-app/`, end to end, in a single working day.

## What You Will Build

By the end of this workshop you will have a running stack that contains:

- **8 DynamoDB models** managed by AppSync (`Food`, `user`, `FoodLog`, `FridgeItem`, `Challenge`, `ChallengeParticipant`, `Friendship`, `UserPublicStats`) defined in `backend/amplify/data/resource.ts`.
- **4 Lambda functions** on **Node.js 22 / ARM64**:
  - `ai-engine` — multi-action AI handler, 512 MB, 120 s timeout.
  - `process-nutrition` — hybrid DynamoDB + AI nutrition lookup.
  - `friend-request` — friend system mutations.
  - `resize-image` — S3 event trigger on the `incoming/` prefix.
- **10 AI actions** served by the `aiEngine` Lambda: `analyzeFoodImage`, `generateCoachResponse`, `searchFoodNutrition`, `fixFood`, `voiceToFood`, `ollieCoachTip`, `generateRecipe`, `calculateMacros`, `challengeSummary`, `weeklyInsight`.
- **Amazon Bedrock** foundation model `qwen.qwen3-vl-235b-a22b` in **ap-southeast-2** (Sydney), invoked by the AI coach persona **Ollie**.
- **Amazon S3** storage bucket with `incoming/`, `voice/`, and `media/` prefixes, wired to `resize-image` via an S3 event notification and a 1-day lifecycle rule on `incoming/`.
- **Amazon Cognito** user pool with email + OTP signup and Google federated identity.
- **Amazon Transcribe** for voice-to-food logging, invoked from `ai-engine` with a resource-policy grant on `voice/*`.
- **ECS Fargate** container tier running a FastAPI service (`backend/main.py`) behind an Application Load Balancer, deployed from `infrastructure/` (Terraform) or `ECS/` (Docker + CI/CD).
- **Expo mobile app** (SDK 54, React Native 0.81, React 19, Expo Router 6, Zustand 5, `@react-three/fiber`) in `frontend/`.

## Architecture at a Glance

```mermaid
flowchart LR
  subgraph Client
    A[Expo App<br/>React Native 0.81]
  end

  subgraph AuthAPI[Auth and API]
    B[Cognito User Pool<br/>Email OTP + Google OAuth]
    C[AppSync GraphQL]
  end

  subgraph Data
    D[(DynamoDB<br/>8 models)]
  end

  subgraph Compute[Lambdas - Node.js 22 ARM64]
    E[ai-engine]
    F[process-nutrition]
    G[friend-request]
    H[resize-image]
  end

  subgraph AI[AI and Media]
    I[Bedrock<br/>qwen3-vl-235b-a22b<br/>ap-southeast-2]
    J[Transcribe]
    K[(S3 bucket<br/>incoming/ voice/ media/)]
  end

  subgraph Container[Container Tier]
    L[ALB] --> M[ECS Fargate<br/>FastAPI]
  end

  A -->|signInWithRedirect| B
  A -->|GraphQL over HTTPS| C
  C --> D
  C --> E
  C --> F
  C --> G
  E --> I
  E --> J
  E --> K
  F --> D
  G --> D
  K -->|ObjectCreated: incoming/| H
  A -.-> L
```

## Learning Outcomes

After completing this workshop you will be able to:

1. Bootstrap an Amplify Gen 2 backend from scratch and evolve it through three environments (sandbox, `feat/phase3`, `main`).
2. Model a real multi-tenant domain in Amplify Data with owner-scoped authorization and GSIs.
3. Wire Node.js 22 Lambdas into AppSync as custom queries and mutations, and attach IAM policies with the CDK escape hatch.
4. Call Amazon Bedrock multimodal foundation models (Qwen3-VL) from Lambda, including image and voice inputs.
5. Configure S3 event notifications, resource policies for Transcribe, and lifecycle rules directly in `backend.ts`.
6. Ship the Expo client against the auto-generated `amplify_outputs.json` and run it on a device via Expo Go.
7. Decommission everything cleanly so your AWS bill returns to zero.

## Estimated Cost

Running this workshop end to end for one day in a single region typically lands in the **$5 to $15 USD** range. The dominant line items are Bedrock token usage and ECS Fargate hours. Left running for a full month on light dev traffic, expect **$50 to $150 USD**, again dominated by Bedrock. See the cost breakdown in `../5.11-Appendices/` and enable AWS Budgets before you begin.

## Duration and Difficulty

- **Duration**: ~1 full working day (6 to 8 hours) if you follow every step in order without detours.
- **Difficulty**: **Intermediate**. You should be comfortable with TypeScript, the AWS Console, and a terminal. React Native experience is helpful but not required — the frontend runs as-is.

## Workshop Sections

1. [5.2 Prerequisites](../5.2-Prerequiste/) — accounts, tooling, Bedrock access request.
2. [5.3 Foundation Setup](../5.3-Foundation-Setup/) — repo layout, Amplify sandbox, Cognito.
3. [5.4 Monitoring Setup](../5.4-Monitoring-Setup/) — AppSync and DynamoDB models.
4. [5.5 Processing Setup](../5.5-Processing-Setup/) — Bedrock + `ai-engine` Lambda.
5. [5.6 Automation Setup](../5.6-Automation-Setup/) — S3, resize-image trigger, Transcribe.
6. [5.7 Dashboard Setup](../5.7-Dashboard-Setup/) — Expo app configuration.
7. [5.8 Verify Setup](../5.8-Verify-Setup/) — end-to-end smoke tests.
8. [5.9 CI/CD — Amplify Multi-Environment](../5.9-Use-CDK/) — `amplify.yml`, sandbox → `feat/phase3` → `main`.
9. [5.10 Cleanup](../5.10-Cleanup/) — destructive-safe teardown.
10. [5.11 Appendices](../5.11-Appendices/) — cost breakdown, troubleshooting, references.

## Source of Truth

Every claim in this workshop is grounded in the real implementation under `TEMPLATE/neurax-web-app/`. When the documentation and the code disagree, the code wins. Key files to keep open in a second tab:

- `TEMPLATE/neurax-web-app/backend/amplify/backend.ts`
- `TEMPLATE/neurax-web-app/backend/amplify/data/resource.ts`
- `TEMPLATE/neurax-web-app/backend/amplify/ai-engine/handler.ts`
- `TEMPLATE/neurax-web-app/amplify.yml`
- `TEMPLATE/neurax-web-app/CLAUDE.md`
