---
title: "AWS re:Invent 2025 Recap - Infrastructure & Security"
date: 2026-01-27T15:59:18+07:00
weight: 3
chapter: false
pre: " <b> 4.1. </b> "
---

**Date:** January 27, 2026
**Location:** AWS Vietnam Office (Floor 26 & 36), Ho Chi Minh City
**Role:** Attendee (FCJ Cloud Intern - Team NeuraX)

## Event Description

This event was a comprehensive recap of the most important infrastructure and security updates from AWS re:Invent 2025. The session brought together cloud architects and developers to explore the latest innovations in silicon, serverless computing, and AI-driven security operations directly from AWS experts.

## Main Activities

The event focused on the following key technical topics with technical details extracted from specialized AWS deep-dive sessions:

**Next-Generation Compute & Silicon**  
Introduction of **AWS Graviton5**, featuring **192 cores per chip** and a **5x larger cache** compared to Graviton4. The new **Amazon EC2 M9g instances** powered by these chips provide up to **25% better price-performance** for general-purpose workloads. For specialized AI tasks, **Trainium3 UltraServers** were unveiled (now available in US East/West), designed to accelerate training for models with trillions of parameters while significantly reducing cost-per-token.

**Serverless Evolution**  
Exploration of **AWS Lambda Managed Instances**, which bridges the gap between serverless simplicity and EC2 flexibility, allowing functions to run on specialized hardware with full VPC control. A major highlight was **AWS Lambda Durable Functions**, which introduces **durable execution** using a checkpoint and replay mechanism. Developers can use the `context.step()` and `context.wait()` primitives to build workflows that **suspend execution for up to one year** without paying for idle compute time. This is ideal for processes awaiting human approvals or long-running AI completions.

**High-Performance Data Infrastructure**  
Deep dive into **Amazon S3 Vectors** (now Generally Available). It supports up to **1 billion vectors per index** and uses the **HNSW (Hierarchical Navigable Small World)** algorithm for sub-100ms query latency. This native S3 capability reduces total cost of ownership by up to **90%** compared to operating specialized vector databases. Additionally, **Amazon S3 Tables** now supports **cross-region replication** and **Intelligent-Tiering**, which automatically moves data between hot (Express One Zone) and cold (Standard) tiers based on access. S3 Tables also includes built-in maintenance features like **file compaction** and **snapshot expiration** to ensure consistent performance.

**AI-Powered Security Operations**  
Preview of the **AWS Security Agent**, which allows teams to scale AppSec expertise using LLMs. It performs **AI-powered design reviews** by analyzing architecture diagrams and **deep code analysis** to identify complex logic flaws beyond the OWASP Top 10. A standout feature is **contextual penetration testing**, where the agent generates and executes exploit scenarios safely within a sandbox to validate vulnerabilities before deployment. **AWS Security Hub** also reached General Availability with near real-time analytics and risk-based prioritization of findings across all AWS accounts.

**Global Infrastructure & Secure Networking**  
Introduction of **AWS AI Factories**, enabling organizations to deploy managed AI infrastructure (specialized hardware and pre-integrated models) in their own data centers to meet data residency requirements. For hybrid connectivity, the **Amazon Route 53 Global Resolver** (preview) was introduced. It uses **secure, anycast-based resolution** for unified DNS management across public and private domains, reducing the need for complex hybrid DNS architectures and manual forwarding rules.

## Outcomes

- **Graviton5 & Trainium3**: Essential for scaling AI workloads efficiently with 192 cores and 25% better performance.
- **Lambda Durable Functions**: Simplify long-running workflows with native state management and up to 1-year suspension.
- **S3 Vectors & Tables**: Provide a robust, cost-effective foundation for massive RAG applications with built-in maintenance and 90% cost savings.
- **AWS Security Agent**: Proactively secures applications via automated pentesting and design reviews throughout the development lifecycle.
- **Route 53 Global Resolver & AI Factories**: Address complex hybrid and sovereign cloud needs with anycast resolution and on-premises managed AI.
- Identified **S3 Tables and GuardDuty** features directly applicable to enhancing NutriTrack's data security and operational visibility.
