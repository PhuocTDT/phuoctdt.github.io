---
title: "Cleanup"
date: 2026-01-09T16:03:43+07:00
weight: 100
chapter: false
pre: " <b> 5.7. </b> "
---

After completing the workshop, cleaning up your resources is a critical step to avoid unexpected AWS charges. Some resources, such as NAT Instances and ALBs, incur hourly costs even when not processing any traffic.

> [!IMPORTANT]
> The estimated cost of maintaining the resources in this workshop is approximately **$44/month**. Please follow the steps below as soon as you are finished testing.

## Resource Deletion Order

To avoid "Resource in use" errors, you should perform the deletion in order from the application layer out to the network infrastructure layer.

### 1. Application & Compute Layer (ECS & ALB)

1. **ECS Service**: Go to ECS Cluster -> select the service `nutritrack-api-service` -> click **Delete**. Wait for all Tasks to stop completely.
2. **ECS Cluster**: Once the Service has been deleted, you can delete the cluster itself.
3. **Application Load Balancer (ALB)**: Go to EC2 -> Load Balancers -> Select the workshop ALB -> **Actions** -> **Delete**.
4. **Target Group**: Delete the corresponding Target Group for the ALB.

### 2. Networking Layer (VPC & NAT)

1. **NAT Instances**: Go to the EC2 Console -> Instances -> Select your NAT Instances -> **Terminate instance**.
2. **VPC**: Go to the VPC Console -> Your VPCs -> Select `nutritrack-api-vpc` -> **Actions** -> **Delete VPC**.
    - *Note: AWS will automatically delete Subnets, Internet Gateways, and Route Tables associated with the VPC.*

### 3. Backend & Storage Layer (Amplify & S3)

1. **AWS Amplify**: Go to the Amplify Console -> Select the NutriTrack app -> **Actions** -> **Delete app**. This will remove all associated resources (Cognito, DynamoDB, AppSync).
2. **S3 Bucket**: Go to the S3 Console -> Find the cache bucket (`nutritrack-cache-xxxx`). You must **Empty** the bucket before you can **Delete** it.
3. **Secrets Manager**: Go to Secrets Manager -> Select the secret containing your API Keys -> **Delete secret**. (Note: AWS defaults to a 7-30 day waiting period before permanent deletion).

### 4. IAM & CloudWatch

1. **IAM Roles**: Delete manually created roles such as `ecsTaskRole` and `ecsTaskExecutionRole` if they are no longer needed for other projects.
2. **CloudWatch Logs**: Delete the Log Groups (`/ecs/nutritrack-api`) to keep your management interface clean.

---

Congratulations on successfully completing the NutriTrack deployment workshop on AWS! We hope the knowledge of Hybrid Architecture (Serverless + Container) and networking cost optimization will be valuable for your future real-world projects.

[Back to Homepage](../../)
