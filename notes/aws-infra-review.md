# LabLumen — AWS Infrastructure Review & Connections Audit

This document lists all AWS resources configured in the LabLumen Terraform codebase (`lablumen-terraform`) and validates all connection paths, protocols, and security rules between them.

---

## 1. Resource Catalog by Component

### A. Network & VPC (`modules/vpc`)
*   **VPC (`lablumen-vpc`)**: `10.0.0.0/16` CIDR block spanning two Availability Zones (`us-east-1a`, `us-east-1b`).
*   **Public Subnets**: `10.0.101.0/24` (AZ-a), `10.0.102.0/24` (AZ-b). Contains the Internet Gateway (IGW) and a single NAT Gateway (placed in AZ-a).
*   **Private Subnets**: `10.0.1.0/24` (AZ-a), `10.0.2.0/24` (AZ-b). Houses EKS worker nodes, Karpenter-provisioned nodes, and VPC-connected Lambda ENIs.
*   **Database Subnets**: `10.0.201.0/24` (AZ-a), `10.0.202.0/24` (AZ-b). Isolated tier for RDS PostgreSQL with no routes to IGW or NAT GW.
*   **VPC Endpoints (PrivateLink)**: Interface endpoints placed in private subnets with a shared security group allowing port `443` inbound from the VPC CIDR:
    *   `ssm` (SSM Parameter Store)
    *   `secretsmanager` (Secrets Manager)
    *   `bedrock-runtime` (Bedrock AI Model execution)
    *   `textract` (OCR Document analysis)
    *   `ecr.api` & `ecr.dkr` (ECR image registry)
    *   `logs` (CloudWatch logs ingestion)
    *   `sqs` (Queue operations)
    *   **S3 Gateway Endpoint**: Routing table association for private subnets. Avoids NAT GW data transfer fees for PDF uploads/downloads.

### B. Compute (`modules/eks` & Lambda)
*   **EKS Control Plane (`lablumen-eks`)**: Running Kubernetes version `1.31` with modern EKS API Auth. CloudWatch audit logs are enabled.
*   **EKS Managed Node Group (`default`)**: Placed in private subnets. Utilizes `c7i-flex.large` instances.
*   **Karpenter Node Provisioner**: Dynamically scales `t3.medium` / `t3.large` on-demand or spot nodes in private subnets.
*   **AI Processing Lambda (`lablumen-ai-service`)**: Python 3.12 SAM function running inside private subnets (VPC-connected ENI).

### C. Data & Storage (`modules/rds` & `modules/s3`)
*   **RDS PostgreSQL (`lablumen-pg`)**: Running version `16.4` on `db.t4g.micro` with pgvector extension enabled. Located in isolated DB subnets. Master password is managed by Secrets Manager.
*   **Reports S3 Bucket (`reports-bucket`)**: Private, versioned, blocked public access, and encrypted with the platform KMS key. EventBridge notifications are enabled.
*   **SAM Artifacts S3 Bucket (`sam-artifacts-bucket`)**: Private bucket used for SAM zip deployment artifacts.

### D. Security & Secrets Management (`modules/secretsmanager`, `ssm`, `cognito`, `iam`)
*   **KMS Key (`alias/lablumen-platform`)**: Customer Managed Key (CMK) with annual rotation enabled. Encrypts Secrets Manager secrets, SSM parameter strings, ECR image layers, and S3 reports.
*   **Secrets Manager Secrets**:
    *   `lablumen/app/database-url`: Database DSN (host, username, password) used by services and the AI Lambda.
    *   `lablumen/app/grafana-admin`: Grafana admin username and password.
*   **SSM Parameter Store**: Stores 14 non-sensitive parameters under `/lablumen/config/*` (bucket names, queue URLs, Cognito pool IDs, Bedrock models, CORS origins).
*   **Cognito User Pool**: Manages user authentication. Includes `PATIENT`, `LAB_STAFF`, and `LAB_ADMIN` user groups, and an SPA web client without a client secret.
*   **IAM OIDC Providers**:
    *   **GitHub Actions OIDC Provider**: Authenticates CI/CD pipelines.
    *   **EKS OIDC Provider**: Authenticates EKS pods via IAM Roles for Service Accounts (IRSA).

### E. Integrations (`modules/sqs`, `modules/ses`)
*   **SQS Queue (`lablumen-notifications`)**: Standard queue for buffering events from the booking system.
*   **SES Domain Identity**: Configured with Route53 records for Easy DKIM.

---

## 2. Connections & Protocols Audit

Here we detail and double-check every connection direction, protocol, and port configuration.

### Connection 1: User / Browser → EKS Ingress
*   **Path**: Internet → Route53 (DNS resolution) → Application Load Balancer (ALB) → EKS pods (`frontend` / API gateways).
*   **Protocol & Ports**: HTTPS (`443`) terminated at ALB via ACM wildcard certificate, routing traffic to pods over HTTP.
*   **Status**: **Verified**. ALB Controller (`lbc` IRSA) dynamically creates/manages target groups pointing to EKS pod IPs.

### Connection 2: Pods / Lambda → RDS PostgreSQL
*   **Path**: EKS Private Subnets (Pods) / Lambda ENIs → RDS DB Subnets.
*   **Protocol & Ports**: TCP on port `5432`.
*   **Security Groups**:
    *   `rds` security group allows ingress on `5432` from the VPC CIDR (`10.0.0.0/16`).
    *   `ai_lambda` security group allows egress on `5432` to the VPC.
*   **Status**: **Verified**. Isolated database subnets do not allow any public ingress, but the SG correctly permits internal VPC routing.

### Connection 3: Pods → Redis (In-Cluster)
*   **Path**: EKS Private Subnets (`appointment-service` → `redis`).
*   **Protocol & Ports**: TCP on port `6379`.
*   **Status**: **Verified**. Redis is deployed in-cluster within the private EKS node group.

### Connection 4: External Secrets Operator (ESO) → Secrets Manager & SSM
*   **Path**: EKS Private Subnets (ESO Pod) → VPC Interface Endpoints → Secrets Manager / SSM.
*   **Protocol & Ports**: HTTPS (`443`) over PrivateLink.
*   **IAM Permission**: `secretsmanager:GetSecretValue` on `lablumen/app/*`, `ssm:GetParameter*` on `/lablumen/config/*`, and `kms:Decrypt` on the platform CMK.
*   **Status**: **Verified**. ESO syncs Secrets Manager secrets and SSM parameters directly into Kubernetes `Secret` and `ConfigMap` resources inside namespaces.

### Connection 5: Report Service → S3 & Bedrock
*   **Path**: `report-service` pod → S3 Gateway / Bedrock Runtime Interface Endpoint.
*   **Protocol & Ports**: HTTPS (`443`).
*   **IAM Permission**: `s3:GetObject` / `s3:PutObject` on `reports-bucket/*` and `bedrock:InvokeModel` on `*`.
*   **Status**: **Verified**. Configured via `report_service` IRSA role.

### Connection 6: Notification Service → SQS & SES
*   **Path**: `notification-service` pod → SQS Interface Endpoint / SES.
*   **Protocol & Ports**: HTTPS (`443`).
*   **IAM Permission**: `sqs:SendMessage`, `sqs:ReceiveMessage`, `sqs:DeleteMessage` on queue, and `ses:SendEmail` on identity.
*   **Status**: **Verified**. Configured via `notification_service` IRSA role.

### Connection 7: Async AI Pipeline
*   **Path 1**: User/Staff uploads PDF via `report-service` to S3 `reports-bucket`.
*   **Path 2**: S3 publishes `Object Created` event to EventBridge.
*   **Path 3**: EventBridge triggers `AiProcessingFunction` Lambda.
*   **Path 4 (Lambda execution)**:
    1.  Lambda fetches PostgreSQL secret from Secrets Manager (via VPC endpoint) and decrypts using the platform KMS key.
    2.  Lambda fetches PDF from S3 (via Gateway endpoint).
    3.  Lambda calls Textract for OCR (via VPC endpoint).
    4.  Lambda assumes the cross-account Bedrock role (`var.bedrock_cross_account_role_arn`) to invoke Claude/Nova models.
    5.  Lambda writes summaries and embeddings (pgvector) to RDS PostgreSQL (port `5432`).
*   **Status**: **Verified**. Execution role has all necessary KMS decrypt, S3 read, Textract read, STS AssumeRole, and Secrets Manager read permissions.

---

## 3. Architecture Layout Diagram Summary

The generated diagram `extras/notes/aws_infra_architecture.png` illustrates:
1.  **Ingress flow**: Users → Route53 → ACM → ALB → EKS.
2.  **VPC boundary**: Spanning two AZs with public, private, and database tiers.
3.  **Active resources** placed inside AZ-a (NAT Gateway, EKS worker nodes running service pods, EKS-attached Lambda, and RDS PostgreSQL).
4.  **VPC Interface Endpoints** showing PrivateLink traffic staying within the VPC boundary to reach Secrets Manager, SSM Parameter Store, SQS, Bedrock, Textract, S3 (via Gateway), and CloudWatch.
5.  **External integrations**: Cognito, SES (DKIM Route53 records), and OIDC-based GitHub Actions deployments.
