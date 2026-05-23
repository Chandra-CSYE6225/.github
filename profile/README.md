# Cloud-Native Web Application Platform

> A production-grade, cloud-native web application built on AWS using Infrastructure as Code, immutable AMI deployments, and a fully automated CI/CD pipeline.

## Overview

This platform is a multi-tier web application designed around modern cloud-native principles. The application layer is a FastAPI REST service that handles user management, a product catalog, and image storage. The infrastructure is fully codified in Terraform and runs across two AWS accounts — a DEV account where AMIs are built and a DEMO account where the production fleet runs. Application images are baked by Packer, deployed via Auto Scaling Group instance refresh, and shipped through a GitHub Actions pipeline that runs an integration test suite on every pull request.

The architecture emphasizes immutability, least-privilege identity, encryption at rest and in transit, and end-to-end observability. Asynchronous workflows (such as email verification) are handled by Lambda functions triggered by SNS, with idempotency enforced through DynamoDB. Application logs and custom StatsD metrics flow into CloudWatch, giving a single pane of glass for debugging and performance monitoring.

---

## Architecture Diagram

![Architecture Diagram](./architecture.png)



---

## Repositories in This Organization

| Repo | Language | Purpose |
|------|----------|---------|
| **[webapp](https://github.com/Chandra-CSYE6225/webapp)** | Python | FastAPI REST application — user management, product catalog, image storage |
| **[serverless](https://github.com/Chandra-CSYE6225/serverless)** | Python | AWS Lambda function for email verification (SNS-triggered, SendGrid delivery) |
| **[tf-aws-infra](https://github.com/Chandra-CSYE6225/tf-aws-infra)** | HCL | Terraform IaC — VPC, ALB, ASG, RDS, S3, IAM, Route 53, ACM, KMS, SNS, DynamoDB |

---

## Component Reference

Each component below corresponds to a node in the architecture diagram. Components are grouped by layer and include their full responsibility, the technology choice, and the rationale for that choice.

### Networking Layer

#### Route 53 (DNS)
- **Responsibility:** Resolves the application's public hostname (e.g., `demo.chandra-csye6225.me`) to the Application Load Balancer's DNS name via an **A-record alias**.
- **Tech choice:** AWS Route 53 hosted zone, managed by Terraform.
- **Why:** Alias records resolve directly to the ALB without an extra CNAME lookup, and Route 53 health checks integrate natively with the load balancer. The hosted zone is managed in the same Terraform configuration as the rest of the stack, so DNS, certificate, and load balancer are deployed atomically.

#### ACM (AWS Certificate Manager)
- **Responsibility:** Issues and renews the public TLS certificate attached to the ALB's HTTPS listener (port 443).
- **Tech choice:** AWS Certificate Manager with DNS-validated public certificates.
- **Why:** Free certificates with automatic renewal, native ALB integration, and no manual key rotation. DNS validation is automated end-to-end because Route 53 lives in the same AWS account.

#### Internet Gateway
- **Responsibility:** Provides the VPC's route to and from the public internet. Attached to public subnet route tables.
- **Tech choice:** AWS Internet Gateway (one per VPC).
- **Why:** Required for the public-facing ALB and for the NAT Gateway's outbound traffic. Stateless, horizontally scaled by AWS, no operational burden.

#### NAT Gateway
- **Responsibility:** Provides outbound-only internet access for resources in private subnets — Lambda calling SendGrid, EC2 instances pulling OS updates, the CloudWatch Agent uploading data.
- **Tech choice:** AWS-managed NAT Gateway (one per AZ for HA in production; single instance for the demo environment to control cost).
- **Why:** Private subnets remain unreachable from the internet, which is the key security property. AWS-managed NAT avoids running and patching NAT instances.

#### Application Load Balancer
- **Responsibility:** Terminates TLS, distributes HTTP traffic across EC2 instances in the Auto Scaling Group, runs health checks against `/healthz`, and removes failing instances from rotation.
- **Tech choice:** AWS Application Load Balancer (Layer 7), HTTPS listener on 443 to target group on port 8000.
- **Why:** Layer 7 routing supports the app's REST paths, health checks are critical for the instance-refresh deployment strategy, and the ALB integrates directly with the ASG target group attachment for automatic registration.

### Compute Layer

#### EC2 + Auto Scaling Group
- **Responsibility:** Runs the FastAPI application (via Uvicorn + systemd `webapp.service`) on Amazon Linux 2023. The ASG maintains a desired count of instances across private subnets, replaces unhealthy instances automatically, and performs **instance refresh** when a new AMI is published.
- **Tech choice:** EC2 `t2.micro` instances, custom AMI baked by Packer, Launch Template references the AMI ID and IAM instance profile.
- **Why:** Immutable infrastructure — every code change produces a new AMI, and deployments roll the fleet rather than mutating live servers. This guarantees the running fleet matches what was tested in CI.

#### Custom AMI (Packer)
- **Responsibility:** Packer bakes a fully-configured Amazon Linux 2023 AMI containing the FastAPI app, Python runtime, CloudWatch Agent, StatsD configuration, and the systemd unit. The AMI is built in the **DEV account**, then shared with the **DEMO account** for deployment.
- **Tech choice:** HashiCorp Packer with `amazon-ebs` builder, provisioner runs `scripts/setup.sh`.
- **Why:** Builds an immutable image once, so every new instance launches with identical software in seconds — no runtime provisioning, no configuration drift between instances.

#### Lambda (Email Verification)
- **Responsibility:** Triggered by SNS messages on user registration. Fetches the SendGrid API key from Secrets Manager, checks DynamoDB for an existing send record (idempotency), sends the verification email, then writes the send record back to DynamoDB.
- **Tech choice:** Python Lambda function, VPC-attached (so it can reach private resources and route through the NAT for SendGrid's API).
- **Why:** Email sending is bursty and asynchronous — Lambda scales to zero, costs nothing when idle, and decouples the web tier from email delivery latency. The user gets a fast `201 Created` response while the email flows in the background.

### Data Layer

#### RDS PostgreSQL
- **Responsibility:** Primary relational store for users, products, and product image metadata. Schema is managed via Alembic migrations run during deployment.
- **Tech choice:** Amazon RDS for PostgreSQL 15, deployed in private subnets, KMS-encrypted at rest, accessible only from the EC2 security group on port 5432.
- **Why:** Managed backups, automated minor-version patching, and KMS encryption are non-negotiable for any persistent data. PostgreSQL matches the SQLAlchemy ORM models used in the FastAPI app.

#### S3 (Product Images)
- **Responsibility:** Stores user-uploaded product images at key path `users/{user_id}/products/{product_id}/{filename}`. Versioned, KMS-encrypted, and gated by a lifecycle policy that transitions objects from Standard to Standard-IA after 30 days.
- **Tech choice:** AWS S3 with KMS-CMK server-side encryption, private bucket (no public access), accessed via IAM instance profile (no static credentials).
- **Why:** Object storage scales independently of the compute tier, lifecycle policies cut storage cost automatically, and IAM-based access removes the need to bake or rotate credentials.

#### DynamoDB (Email Dedup)
- **Responsibility:** Records every verification email sent, keyed by the recipient address. The Lambda function checks this table before sending — if a record exists, the message is dropped, preventing duplicate emails when SNS retries.
- **Tech choice:** DynamoDB on-demand capacity, single partition key.
- **Why:** Single-digit-millisecond reads at any scale, and on-demand pricing means it costs effectively nothing for the workload volume. Idempotency at the data layer is more reliable than at the message layer.

### Messaging Layer

#### SNS (user-registration topic)
- **Responsibility:** Receives publish events from the webapp when a new user is created and fans out to subscribed Lambda functions.
- **Tech choice:** AWS SNS standard topic, Lambda function as the only subscriber.
- **Why:** SNS decouples the web tier from the email tier. If the Lambda is throttled or temporarily failing, SNS retries with backoff; if a second consumer (analytics, audit log) is needed later, it subscribes without changing the publisher.

### Security & Identity Layer

#### IAM (Roles & Instance Profiles)
- **Responsibility:** Defines least-privilege roles for every compute resource. The EC2 instance profile grants access to S3, Secrets Manager, SNS, and CloudWatch. The Lambda execution role grants access to DynamoDB, Secrets Manager, and CloudWatch Logs.
- **Tech choice:** IAM roles with policy documents authored in Terraform, attached via instance profile (EC2) and `aws_iam_role_policy_attachment` (Lambda).
- **Why:** Eliminates static credentials entirely — the application code never sees an AWS access key. Permissions are version-controlled with the rest of the infrastructure.

#### KMS (AWS Key Management Service)
- **Responsibility:** Provides customer-managed encryption keys for S3 objects, RDS storage, EBS volumes, and Secrets Manager secrets. Key policies grant `Encrypt`/`Decrypt` only to the IAM roles that need them.
- **Tech choice:** Customer-managed KMS keys (one per data domain) with rotation enabled.
- **Why:** Customer-managed keys give per-resource auditability via CloudTrail and the ability to revoke access by editing the key policy — capabilities the default AWS-managed keys do not provide.

#### Secrets Manager
- **Responsibility:** Stores the RDS master credentials and the SendGrid API key. EC2 instances fetch DB creds at application startup via `boto3`; Lambda fetches the SendGrid key on each cold start.
- **Tech choice:** AWS Secrets Manager with automatic rotation enabled for the database secret.
- **Why:** Secrets never enter Terraform state in plaintext, never end up in environment variable files on disk, and rotation can be automated without redeploying the application.

### Observability Layer

#### CloudWatch (Logs + Metrics)
- **Responsibility:** Centralizes application logs (shipped from `/var/log/webapp/csye6225.log` by the CloudWatch Agent) and custom metrics (`webapp.api.{endpoint}.count`, `webapp.api.{endpoint}.duration`, `webapp.db.{operation}.duration`, `webapp.s3.{operation}.duration`) emitted via StatsD on `localhost:8125`.
- **Tech choice:** CloudWatch Unified Agent with StatsD plugin, custom namespace `CSYE6225`, 10-second collection / 300-second aggregation intervals.
- **Why:** Logs and metrics in the same console enable a single pane of glass for debugging. StatsD inside the app means the metrics call is non-blocking (UDP fire-and-forget) and adds no measurable request latency.

### External Services

#### SendGrid
- **Responsibility:** Delivers verification emails to end users via authenticated transactional SMTP API. Domain authentication (SPF, DKIM) is configured against the project's custom domain.
- **Tech choice:** Twilio SendGrid free tier.
- **Why:** Free tier handles the demo workload, authenticated domain prevents emails from being flagged as spam, and the HTTPS API is far more reliable than self-hosted SMTP.

### CI/CD & IaC Layer

#### GitHub Actions
- **Responsibility:** Two workflows. **CI** (`ci.yml`) runs the 102-test integration suite against a real PostgreSQL service container on every PR. **CD** (`packer-build.yml`) builds and ships a new AMI on every merge to `main`.
- **Tech choice:** GitHub Actions with OIDC federated authentication to AWS (no long-lived access keys stored as secrets).
- **Why:** OIDC federation is the modern security pattern — every workflow run gets a short-lived credential scoped to a specific IAM role. Native to the platform where the code already lives.

#### Terraform (tf-aws-infra)
- **Responsibility:** Provisions and manages every AWS resource in the diagram: VPC, subnets, route tables, IGW, NAT, ALB, target group, listener, ASG, Launch Template, RDS, S3, KMS keys, IAM roles, SNS topic, DynamoDB table, Lambda function, Route 53 records, ACM certificate.
- **Tech choice:** Terraform with remote state in S3 + DynamoDB lock table.
- **Why:** Single source of truth for the infrastructure, reproducible across DEV/DEMO accounts, and changes go through PR review the same way application changes do.

---

## End-to-End Request Flows

### User Registration Flow
1. End user `POST /v1/user` resolves via **Route 53**, hits **ALB** (TLS terminated using **ACM** cert), then routes to a healthy **EC2** instance.
2. EC2 fetches DB credentials from **Secrets Manager** (cached at startup), writes the new user to **RDS PostgreSQL** with `is_verified = false`.
3. EC2 publishes a message to the **SNS** `user-registration` topic.
4. **Lambda** is invoked, checks **DynamoDB** for an existing send record; if absent, fetches the SendGrid API key from **Secrets Manager** and sends the email via **SendGrid**, then records the send in DynamoDB.
5. User receives the email, clicks the verification link, hits `GET /v1/user/verify`, the token is validated (1-minute TTL), and `is_verified` is flipped to `true` in RDS.

### Image Upload Flow
1. Authenticated, verified user sends `POST /v1/product/{id}/image` with multipart upload to **ALB**, which routes to **EC2**.
2. EC2 streams the file to **S3** using its IAM instance profile; S3 encrypts at rest with **KMS**.
3. EC2 writes the image metadata (S3 key, filename) to **RDS**.
4. CloudWatch Agent ships the `webapp.s3.upload.duration` metric and structured log line to **CloudWatch**.

### Deployment Flow
1. Developer opens a PR. **GitHub Actions CI** runs `pytest` against a PostgreSQL service container.
2. PR is merged. The **CD workflow** validates the Packer template, builds a new AMI in the **DEV** account, and shares it with **DEMO**.
3. Workflow updates the **DEMO Launch Template** to reference the new AMI ID.
4. ASG **Instance Refresh** is triggered — instances are replaced one batch at a time, with ALB health checks gating each batch.
5. Zero-downtime rollout completes in 15-20 minutes; traffic is served from the new AMI.

---

## Design Principles

- **Immutable infrastructure** — every change produces a new AMI; servers are never mutated in place.
- **Least-privilege IAM** — every role has a hand-written policy granting only the resources and actions it actually uses.
- **No static credentials** — IAM instance profiles for EC2, IAM execution roles for Lambda, OIDC federation for GitHub Actions, Secrets Manager for everything else.
- **Encryption everywhere** — KMS-CMK for S3, RDS, EBS, and Secrets Manager. TLS for all in-transit traffic.
- **Observable by default** — every endpoint logs, every endpoint emits a duration metric, every database and S3 call is timed.
- **Idempotent side effects** — DynamoDB dedup table guarantees one email per registration even if SNS retries.
