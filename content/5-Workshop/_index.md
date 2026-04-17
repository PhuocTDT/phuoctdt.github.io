---
title: "Workshop"
date: 2026-01-09T16:03:34+07:00
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

# NutriTrack — AWS Full-Stack Deployment Workshop
This guide provides a complete step-by-step process for deploying **NutriTrack**—an automated AI nutrition tracking and image analysis system on AWS. The workshop leverages the **AWS Amplify Gen 2** governance framework to establish a core serverless infrastructure including **Amazon Cognito** (Authentication), **AWS AppSync** & **DynamoDB** (Data), and **Amazon S3** (Storage). The system is extended with a high-performance computing layer using **Amazon ECS Fargate** to handle computer vision tasks and in-depth nutrition analysis via **Amazon Bedrock**, seamlessly connecting to the **React Native** mobile application. The entire solution incorporates an automated **CI/CD** process, optimizing deployment from development to actual operation in the cloud environment.

## Workshop Contents

1. [Overview](5.1-Overview/)
2. [Prerequisites](5.2-Prerequiste/)
3. [Frontend Setup](5.3-Frontend/)
4. [Backend Setup](5.4-Backend/)
5. [ECS Fargate Tier](5.5-ECS-Fargate/)
6. [CI/CD](5.6-CICD/)
7. [5.7 Resource Cleanup](5.7-Cleanup/)

## Cost Estimation

The following table provides an estimated cost for maintaining the NutriTrack system on AWS. Please note that actual costs may vary based on usage levels and specific configurations.

| Services             | Monthly Cost | Cost per Day |
|----------------------|-------------:|-------------:|
| Amazon Route 53      | $0.90        | $0.016       |
| Amplify WAF          | $37.11       | $1.237       |
| CloudFront           | $0.00        | $0.000       |
| AWS Amplify          | $4.65        | $0.110       |
| Fargate ARM64        | $10.23       | $0.341       |
| ALB                  | $28.46       | $0.850       |
| NAT Instance         | $7.63        | $0.254       |
| Amazon Cognito       | $0.00        | $0.000       |
| AWS AppSync          | $3.11        | $0.00004     |
| AWS Lambda           | $0.00        | $0.000       |
| Amazon Transcribe    | $72.57       | $0.120       |
| Amazon Bedrock       | $147.57      | $0.016       |
| Amazon S3            | $1.47        | $0.008       |
| Amazon DynamoDB      | $0.13        | $0.000       |
| CloudWatch           | $0.00        | $0.000       |
| AWS Secrets Manager  | $1.20        | $0.040       |
| **Total**            | **$315.03**  | **$2.892**   |

---
