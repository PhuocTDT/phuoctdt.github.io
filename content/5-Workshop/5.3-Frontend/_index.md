---
title: "Frontend Setup"
date: 2026-01-09T16:03:43+07:00
weight: 30
chapter: false
pre: " <b> 5.3. </b> "
---

## ReactNative & Expo

To facilitate hands-on practice, the Frontend portion has been prepared as a complete skeleton code including UI and state management (store). You will proceed to download the source code and prepare the environment.

## 1. Clone Repository

Open the terminal in your workspace and run the command:

```bash
git clone https://github.com/NeuraX-HQ/neurax-web-app.git
cd neurax-web-app
```

## 2. Install Dependencies (npm install)

Install necessary dependencies for Expo and Authentication:

```bash
cd frontend
npm install
```

## 3. Run Application with Expo

Start the development environment:

```bash
npx expo start
```

At this stage, you will see a QR code. Use your phone with **Expo Go** installed to scan this code. 

{{% notice info %}}
The screen will display a backend carrier error or Config error — this is completely normal as we have not yet deployed the Amplify infrastructure in the next step.
{{% /notice %}}

---

[Continue to 5.4 Backend Setup](../5.4-Backend/)
