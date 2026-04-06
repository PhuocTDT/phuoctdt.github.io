---
title: "Workshop"
date: 2026-01-09T16:03:34+07:00
weight: 5
chapter: true
pre: " <b> 5. </b> "
---

# NutriTrack — Full-Stack AWS Deployment Workshop

#### Overview

This workshop is a step-by-step guide to deploying **NutriTrack**, a production-grade serverless nutrition-tracking platform on AWS. You will build a React Native (Expo SDK 54) mobile app backed by **AWS Amplify Gen 2**, **Amazon Bedrock** (Qwen3-VL 235B multimodal), **AWS AppSync** GraphQL, **Amazon DynamoDB**, and a containerized **FastAPI** service on **ECS Fargate**. The finished system ingests food photos, transcribes Vietnamese voice input, calls an AI coach named **Ollie**, and persists nutrition logs with real-time GraphQL subscriptions.

Everything in this workshop mirrors the actual deployed codebase — no placeholder architecture, no toy examples. Every IAM statement, every Lambda `runtime: 22`, every DynamoDB GSI, and every S3 prefix matches what runs in the `main` environment today.

#### What you will build

- **Frontend**: Expo Router app (React Native 0.81, React 19) with biometric auth, camera/voice capture, Zustand stores, and a `@react-three/fiber` pet evolution screen.
- **Backend**: Amplify Gen 2 (`defineBackend`) provisioning Cognito + Google OAuth, 8 DynamoDB models via AppSync, an S3 bucket with 4 access prefixes, and 4 Lambda functions (`aiEngine`, `processNutrition`, `friendRequest`, `resizeImage`).
- **AI layer**: Bedrock Qwen3-VL (`qwen.qwen3-vl-235b-a22b`) in `ap-southeast-2`, invoked through a single multi-action Lambda that routes 10 AI actions (photo analysis, voice-to-food, coach replies, recipe generation, weekly insights…).
- **Container tier**: FastAPI on ECS Fargate behind an ALB, deployed from ECR, networked through a purpose-built VPC.
- **Ops**: Amplify CI/CD across three environments (sandbox, `feat/phase3`, `main`), CloudWatch logs, and a cleanup playbook.

#### Contents

1. [Overview](5.1-Overview/)
2. [Prerequisites](5.2-Prerequiste/)
3. [Foundation Setup — Amplify, Cognito, S3](5.3-Foundation-Setup/)
4. [Data Layer — AppSync & DynamoDB](5.4-Monitoring-Setup/)
5. [Compute & AI — Bedrock, Lambdas](5.5-Processing-Setup/)
6. [API & Social — Friends, Realtime Subscriptions](5.6-Automation-Setup/)
7. [Frontend — Expo, UI, Voice & Camera](5.7-Dashboard-Setup/)
8. [ECS Deployment — VPC, ECR, Fargate, ALB](5.8-Verify-Setup/)
9. [CI/CD — Amplify Multi-Environment](5.9-Use-CDK/)
10. [Cleanup](5.10-Cleanup/)
11. [Appendices — Budget, IAM, Troubleshooting, Prompts](5.11-Appendices/)

#### Who this is for

Engineers who are comfortable with TypeScript, have used AWS at least casually, and want to see how a real Amplify Gen 2 project is structured end-to-end. You do **not** need prior Bedrock, Amplify Gen 2, or React Native experience — every step includes the exact commands, file paths, and IAM policies to copy.
