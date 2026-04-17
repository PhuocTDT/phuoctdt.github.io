---
menuTitle: "CI/CD"
date: 2026-01-09T16:03:43+07:00
weight: 90
chapter: false
pre: " <b> 5.6. </b> "
---

# CI/CD — Automated Deployment

In a production-grade AI project, CI/CD automation ensures that your Cloud infrastructure remains perfectly synchronized with your source code. We will use **AWS Amplify Console** connected directly to your GitHub repository.

## 1. Branch Strategy
Connect GitHub branches to specific AWS environments:
- **`main`**: For the **Production** environment.
- **`staging`**: (Optional) For QA and final testing.
- **`sandbox`**: For personal development (typically using `npx ampx sandbox`).

## 2. Build Configuration (`amplify.yml`)

Below is an advanced `amplify.yml` configuration optimized to build both the Backend (CDK) and Frontend (Expo) in a single pass.

> [!IMPORTANT]
> **Lambda Dependencies**: You must `cd` into each function directory to install their specific dependencies before Amplify packages them for deployment.

```yaml
version: 1
backend:
  phases:
    build:
     commands:
        - cd backend
        - npm install --legacy-peer-deps --include=dev

        - cd amplify/ai-engine
        - npm install --include=dev
        - cd ../..

        - cd amplify/process-nutrition
        - npm install --include=dev
        - cd ../..

        - cd amplify/friend-request
        - npm install --include=dev
        - cd ../..

        - cd amplify/resize-image
        - rm -rf node_modules package-lock.json
        - npm install --platform=linux --arch=x64 sharp
        - npm install --legacy-peer-deps
        - cd ../..

        - npx ampx pipeline-deploy --branch $AWS_BRANCH --app-id $AWS_APP_ID --outputs-out-dir ../frontend
        - cd ..
frontend:
  phases:
    preBuild:
      commands:
        - cd frontend && npm install --legacy-peer-deps && cd ..
    build:
      commands:
        - cd frontend && npm run build
  artifacts:
    baseDirectory: frontend/dist
    files:
      - "**/*"
  cache:
    paths:
      - frontend/node_modules/**/*
      - frontend/.expo/**/*
  customRules:
    - source: </^[^.]+$|\.(?!(css|gif|ico|jpg|js|png|txt|svg|woff|woff2|ttf|map|json|webp)$)([^.]+$)/>
      target: /index.html
      status: '200'
  customHeaders:
    - pattern: "**/*.html"
      headers:
        - key: Cache-Control
          value: no-cache
    - pattern: "**/*.js"
      headers:
        - key: Cache-Control
          value: public, max-age=31536000, immutable
    - pattern: "**/*.css"
      headers:
        - key: Cache-Control
          value: public, max-age=31536000, immutable
    - pattern: "**/*.(png|jpg|jpeg|svg|webp)"
      headers:
        - key: Cache-Control
          value: public, max-age=31536000, immutable
    - pattern: "**/*.(woff|woff2|ttf)"
      headers:
        - key: Cache-Control
          value: public, max-age=31536000, immutable
```

## 3. Optimization & Technical Notes

### Resolving Dependency Conflicts
Since the project uses **Expo 54** and **React 19**, you may encounter `peer dependency conflict` errors. Always use the `--legacy-peer-deps` flag in your install commands to ensure the build succeeds.

---

## 3. Editing Build Settings in the Console

After connecting your project to GitHub, you must update the `amplify.yml` content in the AWS Console:

1. Go to the [AWS Amplify Console](https://console.aws.amazon.com/amplify).
2. Select your application.
3. In the left-hand menu, go to **App settings** -> **Build settings**.
4. Click the **Edit** button in the **Build specification** section.
5. Paste the YAML content from Section 2 into the editor and click **Save**.

## 4. Syncing Local Development (`amplify_outputs.json`)

The `amplify_outputs.json` file acts as the bridge that tells your Frontend application where to find the Backend resources (API, Auth, Storage). While Amplify creates this automatically during the CI/CD pipeline, you need to manually fetch it for local development.

### Synchronization Command:
Open your terminal in the project root and run:

```bash
npx ampx generate outputs --branch <branch-name> --app-id <your-app-id>
```

> [!TIP]
> You can find your **App ID** on the project's Overview page within the Amplify Console.

> [!CAUTION]
> Never push the `amplify_outputs.json` file to GitHub to prevent exposing sensitive endpoints and API Keys.

---

[Continue to 5.7 Resource Cleanup](../5.7-Cleanup/)
