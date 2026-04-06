---
title: "Prerequisites"
date: 2026-01-09T16:03:43+07:00
weight: 90
chapter: false
pre: " <b> 5.2. </b> "
---

# 5.2 Prerequisites

This page is a checklist. Do not skip it. One missing item — especially Bedrock model access — will stall the entire workshop for 24 hours while AWS approves your request.

## AWS Account Requirements

### 1. Active AWS account with admin access

You need an AWS account where you can:

- Create IAM users and roles.
- Create DynamoDB tables, Lambda functions, AppSync APIs, S3 buckets, Cognito user pools, ECS clusters.
- Request Bedrock model access.

For the workshop, use either the root account or an IAM user with the `AdministratorAccess` managed policy. Do **not** use a read-only or developer sandbox account — several steps will fail on IAM permission boundaries.

### 2. Bedrock model access — REQUEST THIS FIRST

> **IMPORTANT — request in advance.** Amazon Bedrock foundation models are gated. Access for `qwen.qwen3-vl-235b-a22b` in `ap-southeast-2` is usually granted in minutes but can take up to 24 hours. Submit the request **before** starting section 5.5, ideally the day before the workshop.

Steps:

1. Open the AWS Console and switch the region to **Asia Pacific (Sydney) — ap-southeast-2**.
2. Go to **Amazon Bedrock → Model access**.
3. Click **Modify model access**.
4. Enable **Qwen 3 VL 235B A22B** (`qwen.qwen3-vl-235b-a22b`).
5. Submit and wait for the status to become **Access granted**.

![Bedrock model access granted for Qwen3-VL](//images/workshop/workshop/bedrock-model-access.png)

The IAM policy attached to the `ai-engine` Lambda in `backend/amplify/backend.ts` only grants `bedrock:InvokeModel` on this exact model ARN:

```text
arn:aws:bedrock:ap-southeast-2::foundation-model/qwen.qwen3-vl-235b-a22b
```

If you switch regions or models, you must also update `backend.ts` and `ai-engine/handler.ts`.

### 3. AWS Budgets alert

Before running any commands, create a budget so Bedrock usage does not surprise you:

- **AWS Console → Billing → Budgets → Create budget**.
- Cost budget, monthly, **$25 USD** threshold with alerts at 50%, 80%, and 100%.

## Local Tooling

Install the following on your workstation. Versions listed are the minimum tested against the codebase.

| Tool                                       | Minimum version     | Used for                                        |
| ------------------------------------------ | ------------------- | ----------------------------------------------- |
| Node.js                                    | **20 LTS** or newer | Running `ampx` CLI, Lambda builds, Expo         |
| npm                                        | **10+**             | Package install (bundled with Node 20+)         |
| AWS CLI                                    | **v2**              | Credential setup, CloudFormation and S3 ops     |
| Git                                        | **2.40+**           | Cloning the template, pushing branches for CI   |
| Docker Desktop                             | **latest stable**   | Building the FastAPI image for ECS              |
| Expo Go (phone) or Android Studio / Xcode  | latest              | Running the mobile client                       |

### Version checks

```bash
node --version
npm --version
```

```bash
aws --version
docker --version
```

```bash
git --version
```

All five should print without errors. Node must report **v20.x or higher**.

### AWS CLI profile

Configure a profile that the Amplify CLI and the `aws` commands in cleanup will use:

```bash
aws configure --profile nutritrack
aws sts get-caller-identity --profile nutritrack
```

Export `AWS_PROFILE=nutritrack` in the shell you run `npx ampx sandbox` from, or set it in your shell profile.

## Google Cloud Console — OAuth Client

Cognito federates to Google for social sign-in. You need a Google OAuth 2.0 Web client.

1. Open the [Google Cloud Console](https://console.cloud.google.com/) and create (or reuse) a project.
2. Go to **APIs & Services → OAuth consent screen**. Configure an **External** consent screen with your email as the test user.
3. Go to **APIs & Services → Credentials → Create credentials → OAuth client ID**.
4. Application type: **Web application**.
5. Authorized redirect URIs: you will add the Cognito Hosted UI callback URL after section 5.3 creates the user pool. For now, leave it blank — you will return.
6. Copy the **Client ID** and **Client secret**.

![Google OAuth Web client credentials](//images/workshop/workshop/google-oauth-client.png)

Store them as Amplify sandbox secrets later (from `backend/`):

```bash
cd backend
npx ampx sandbox secret set GOOGLE_CLIENT_ID
npx ampx sandbox secret set GOOGLE_CLIENT_SECRET
```

The secret names `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET` are referenced by `backend/amplify/auth/resource.ts` — do not rename them.

## Knowledge Prerequisites

You do not need to be an expert in any of these, but you should be able to read and modify the code without looking up basic syntax.

- **TypeScript** — the entire backend and frontend is TypeScript. You should understand types, generics at a basic level, and `async/await`.
- **AWS Console navigation** — you should know how to switch regions, find a service, and read CloudWatch logs.
- **GraphQL (helpful)** — Amplify Data generates a GraphQL schema. You do not need to write resolvers, but reading a query helps.
- **React Native (optional)** — the Expo app runs as-is. Familiarity helps if you want to customize screens.

## Cost Warning

Bedrock is metered per input and output token. Running the full workshop, including a few dozen AI calls for smoke testing, will cost roughly **$1 to $5 USD** in Bedrock charges. The ECS Fargate tier adds fractional-hour charges while the cluster is up. Everything else (DynamoDB on-demand, AppSync, Lambda, S3) stays well inside the free tier for a single day of light use.

If you stop at 5.10 Cleanup the same day, expect a total AWS bill of under **$10 USD**. If you leave resources running for a month, expect **$50 to $150 USD** depending on how much you exercise the AI features.

## Ready?

When every box above is checked — Bedrock access granted, CLIs installed, Google OAuth client created, budget alert armed — continue to [5.3 Foundation Setup](../5.3-Foundation-Setup/).
