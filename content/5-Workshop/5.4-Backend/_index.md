---
title: "AWS Amplify Gen 2 Setup"
menuTitle: "Backend Setup"
date: 2026-01-09T16:03:43+07:00
weight: 40
chapter: false
pre: " <b> 5.4. </b> "
---

We will initialize the data heart of NutriTrack using TypeScript source code.

## 1. Initialize Backend Directory

Create a separate directory to manage the infrastructure:

```bash
cd neurax-web-app
mkdir backend
cd backend
```
## 2. Install Dependencies

```bash
npm create amplify@latest --yes
npm install
```

## 3. Sandbox Deployment (First Time)

Use the Amplify CLI (Gen 2) to initialize the project and deploy your personal Sandbox environment:

```bash
npx ampx pipeline-deploy --branch main --app-id [YOUR_APP_ID]

# Or run the following for local work:
npx ampx sandbox
```
![ampx-sandbox-start.png](/images/ampx-sandbox-start.png)

---

## Resource Layer Details:

Now, we will define the core configuration files located within the `amplify/` directory:

1. [Authentication Layer (Auth)](5.4.1-Auth/)
2. [Data Layer (Data)](5.4.2-Data/)
3. [Storage Layer (Storage)](5.4.3-Storage/)
4. [Logic Functions (Functions)](5.4.4-Functions/)

---

[Continue to 5.5 ECS Container Layer](../5.5-ECS-Fargate/)
