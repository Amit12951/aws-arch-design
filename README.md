# Cloud Infrastructure – Innovate Inc - (home task).


## Overview

Innovate Inc. is building a web application that includes:

- A REST API backend built with Python (Flask)
- A Single Page Application (SPA) frontend built with React
- A PostgreSQL database

The app will initially serve a few hundred users per day but is expected to scale to millions. It handles sensitive user data, so strong security is a must. The team also plans to adopt CI/CD practices from day one.

The high-level architecture diagram is located in the project root directory: aws-arch-design/eksArch.png

---

## AWS Account Structure

We use four AWS accounts under an AWS Organization:

- Management – for billing, account-level policies (SCPs), and centralized IAM
- Dev – for development environments and feature testing
- Staging – mirrors production for full validation before release
- Prod – production workloads for live users

This structure keeps environments isolated, reduces risk, and simplifies permissions and cost tracking.

---

## CI/CD

- **GitHub Actions** is used for building, testing, and deploying both frontend and backend code
- **Argo CD** is deployed inside the EKS cluster in the **prod** account and handles continuous delivery using GitOps

---

## VPC and Networking

Each environment (dev, staging, prod) has its own dedicated VPC with:

- Public and private subnets across three Availability Zones
- NAT Gateway (one per environment)
- Internet Gateway in public subnets
- Private subnets for application workloads and databases
- VPC endpoints for services like S3, ECR, and Secrets Manager

Security is enforced using security groups. CloudFront and AWS WAF are used to filter and protect incoming traffic.

---

## Compute – Amazon EKS

Each environment includes a separate EKS cluster running in private subnets.

- Ingress is handled by an AWS ALB Ingress Controller
- Karpenter is used to manage auto-scaling of nodes
- Two node groups:
  - One for system workloads (e.g., logging, monitoring)
  - One for application workloads, using Spot instances for cost savings
  ## We separate system and application workloads to avoid conflicts. System pods run on stable nodes, while app pods use scalable Spot nodes to save costs.

---

## Containerization and Deployment

### Backend

- Built with GitHub Actions and pushed to ECR
- Deployed to EKS through Argo CD using GitOps
 # Code pushed → GitHub Actions builds & pushes image to ECR → updates Git tag in K8s manifest → Argo CD detects change & deploys to EKS.

### Frontend (React SPA)

- Frontend code built automaticly with GitHub Actions
- After building the code it will be uploaded to an S3 bucket in the prod account
- GitHub Actions → Build React SPA → Upload to S3 → Serve via CloudFront (with cache invalidation - ensure users see the latest version instantly)

Secrets are stored in AWS Secrets Manager and accessed in pods.

---

## Database – PostgreSQL

Aurora PostgreSQL Serverless v2 is used in all environments -  ideal for expected growth with minimal ops overhead.

### Why Aurora Serverless v2

- Automatically scales based on demand
- Cost-efficient and fully managed
- Multi-AZ high availability with automatic failover
- Strong security (KMS encryption, TLS, Secrets Manager)

### Backups and DR

- Daily automated backups with 7-day retention
- Point-in-time recovery enabled
- Cross-region snapshot copy for disaster recovery
- Weekly manual snapshots for compliance

* Optional: Recovery Targets
If needed, we can define RTO ( Recovery Time Objective - 15 minutes) and RPO (Recovery Point Objective - 5 minutes) as soft targets, achievable via PITR, snapshots, and Aurora’s native multi-AZ failover. These can be and tested periodically.

---

## High-Level Diagram

includes:

- CloudFront + WAF for external traffic
- ALB and Ingress to EKS
- Backend services running in pods
- Aurora PostgreSQL in private subnets
- SPA served via S3 and CloudFront
- CI/CD pipelines using GitHub Actions and Argo CD
- Secrets in Secrets Manager and logging with CloudWatch
