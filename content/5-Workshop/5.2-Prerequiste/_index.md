---
title: "Prerequisites"
date: 2026-01-09T16:03:43+07:00
weight: 20
chapter: false
pre: " <b> 5.2. </b> "
---

## Access and Information Requirements
Before you begin, ensure you have the following tools and access permissions:

## 1. AWS Account & Sydney Region
The region used throughout this workshop is `ap-southeast-2` (Sydney). 
Four mandatory steps in Sydney:
1. **Model Access**: Access the Amazon Bedrock console -> Model access -> Select **Edit** -> Enable the **Qwen3-VL-235B-A22B** model.

![bedrock-model-access.png](/images/bedrock-model-access.png)

2. **IAM User**: Create an account with `AdministratorAccess` permissions.
3. **Secret Manager**: Store `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET` retrieved from your Google Cloud Console.

![google-oauth-client.png](/images/google-oauth-client.png)

## 2. Local Machine Tools
- **Node.js**: Version 22.x (LTS) - extremely important because our Lambda functions run on Node 22.
- **Docker Desktop**: Necessary for building images for ECS Fargate.
- **AWS CLI**: Run the `aws configure` command with Admin credentials.
- **npm**: Version 11 or higher.

## 3. Expo Go (Mobile)
Download the **Expo Go** application from the App Store or Play Store to test the Frontend source code on a real phone as quickly as possible.

---

[Continue to 5.3 Frontend Setup](../5.3-Frontend/)
