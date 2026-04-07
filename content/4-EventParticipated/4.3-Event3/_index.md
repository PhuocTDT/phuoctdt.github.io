---
title: "AWS Cloud Mastery 2"
date: 2026-04-04T15:59:44+07:00
weight: 1
chapter: false
pre: " <b> 4.3. </b> "
---

**Date:** April 4, 2026
**Location:** Hall A - FPT University HCMC, Ho Chi Minh City
**Role:** Attendee (FCJ Cloud Intern - Team NeuraX)

## Event Description

The "Cloud Mastery 2" event was an in-depth knowledge-sharing meetup on cloud computing engineering and software development, bringing together members of the FCJ community. The event focused on solving system deployment challenges, optimizing performance using next-generation programming languages, and automating infrastructure through IaC (Infrastructure as Code) tools.

## Main Activities

The event included 3 main technical presentation sessions:

**Session 1: Container Orchestration & Kubernetes**
An overview from Docker and Docker Compose to the necessity of Kubernetes orchestration systems in enterprises. Deep dive into K8s structure (Control Plane, Worker Node, Pod, ConfigMap, Secret) and comparing the trade-offs between using a managed service like Amazon EKS (saves effort, flexible) versus building a Self-hosted Kubernetes (complex but cost-saving).

**Session 2: Functional Programming with Elixir & IoT Applications**
Presentation on the Elixir language and the BEAM virtual machine, with core strengths in concurrency and fault tolerance. Demonstrated the ability to initialize thousands of independent processes while consuming very little CPU, alongside a practical application demo for a Smart Greenhouse management project using MQTT.

**Session 3: Infrastructure as Code (IaC) Tools**
Analysis of 3 prominent infrastructure automation tools: AWS CloudFormation (using YAML/JSON templates), AWS CDK (using programming languages for developers with Construct Levels), and Terraform (multi-cloud platform, managed via HCL). The session included a demo of deploying a static S3 web host using Terraform combined with the LocalStack emulation tool directly on the workstation.

## Outcomes

- Deep understanding of Kubernetes architecture: Recognized the limitations of Docker and when to transition to Kubernetes. Understood that Amazon EKS is the most suitable solution for enterprises that do not want to maintain a highly complex infrastructure operations team.
- The potential of Elixir in distributed systems: Grasped the "let it crash" philosophy and the power of the supervisor process in Elixir, helping IoT Backend applications recover rapidly from failures without freezing the entire server.
- Choosing the right IaC tool: Clearly distinguished the use cases between CloudFormation, CDK (advantage of coding with intuitive programming languages), and Terraform (strong in multi-platform and state management).
- Safe infrastructure workflow: Understood the importance of running plan/checking for errors before applying in Terraform, not pushing sensitive state files or secrets to GitHub, and knowing how to modularize for easy source code reuse.
