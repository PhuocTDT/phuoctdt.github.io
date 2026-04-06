---
title: "Appendices"
date: 2026-01-09T16:03:43+07:00
weight: 5
chapter: false
pre: " <b> 5.11. </b> "
---

# Appendices

This section collects reference material that accompanies the hands-on workshop sections. It is intentionally kept separate from the step-by-step instructions: the walkthrough tells you *what* to click, while the appendices tell you *why* a given value was chosen, *where* it lives in source, and *how* to recover when something goes wrong.

Every appendix is traceable to a file in `backend/amplify/`. If you find a discrepancy between a page here and the code, the code is the source of truth — open an issue and we will reconcile.

## Contents

- [5.11.1 Budget Breakdown](5.11.1-Budget-Breakdown/) — Detailed monthly cost estimate for a 1,000-user deployment, broken down by AWS service, with free-tier offsets and scaling notes. *(Content maintained separately — pending CSV export from the pricing worksheet.)*
- [5.11.2 IAM Policies](5.11.2-IAM-Policies/) — Full policy JSON for every Lambda execution role and the S3 bucket resource policy, extracted verbatim from `backend.ts`.
- [5.11.3 Troubleshooting](5.11.3-Troubleshooting/) — A categorised runbook: common errors, root causes, and the exact command or file change that fixes them.
- [5.11.4 Prompt Templates](5.11.4-Prompt-Templates/) — The actual system prompts shipped in `ai-engine/handler.ts`, one per aiEngine action, with input variables and output schemas.

## How to read these pages

- Code blocks are always language-tagged; copy them directly.
- File paths are relative to the repository root (for example, `backend/amplify/backend.ts`).
- Where a policy or prompt contains a region, model ID, or table prefix, the value matches the workshop defaults — change only if you know why.
- "Ollie" is the name of the AI coach persona. Do not rename it in prompts; downstream UI copy and tests depend on the string literal.

## Invariants used across all appendices

| Invariant | Value |
|---|---|
| Lambda runtime | Node.js 22 on ARM64 |
| Bedrock model | `qwen.qwen3-vl-235b-a22b` |
| Bedrock region | `ap-southeast-2` |
| Lambda functions | `aiEngine`, `processNutrition`, `friendRequest`, `resizeImage` |
| Coach persona | Ollie |
| S3 prefixes | `incoming/`, `voice/`, `media/` |

If any of the above drifts in your fork, update the corresponding appendix before shipping.
