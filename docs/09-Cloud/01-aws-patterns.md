# AWS Production Patterns

> **Category:** Cloud Infrastructure
> **Version:** 1.0.0
> **Level:** Staff Engineer / Platform Engineer

---

## Table of Contents

1. [IAM — Identity and Access Management](#1-iam--identity-and-access-management)
2. [VPC and Networking](#2-vpc-and-networking)
3. [Compute — EC2, ECS, Lambda](#3-compute--ec2-ecs-lambda)
4. [Storage — S3, EBS, EFS](#4-storage--s3-ebs-efs)
5. [Database — RDS, Aurora, DynamoDB](#5-database--rds-aurora-dynamodb)
6. [Secrets and Configuration](#6-secrets-and-configuration)
7. [Observability — CloudWatch, X-Ray](#7-observability--cloudwatch-x-ray)
8. [Security Services](#8-security-services)
9. [Cost Optimization](#9-cost-optimization)
10. [Infrastructure as Code (Terraform)](#10-infrastructure-as-code-terraform)

---

## 1. IAM — Identity and Access Management

### Least Privilege Principles

```json
// NEVER use AdministratorAccess for applications
// NEVER use long-lived access keys in code
// ALWAYS use IAM roles (attached to EC2/ECS/Lambda), not users with keys

// Bad: wildcard permissions
{
  "Effect": "Allow",
  "Action": "s3:*",
  "Resource": "*"
}

// Good: minimal required permissions on specific resources
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::my-app-uploads-prod/*"
    },
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::my-app-uploads-prod",
      "Condition": {
        "StringLike": {
          "s3:prefix": ["uploads/${aws:userid}/*"]
        }
      }
    }
  ]
}
```

### OIDC for GitHub Actions (No Static Keys)

```hcl
# terraform/github-oidc.tf
# Allow GitHub Actions to assume IAM role without storing AWS credentials

resource "aws_iam_openid_connect_provider" "github" {
  url = "https://token.actions.githubusercontent.com"
  
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]
}

resource "aws_iam_role" "github_actions_deploy" {
  name = "github-actions-deploy-prod"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Federated = aws_iam_openid_connect_provider.github.arn
      }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
        }
        StringLike = {
          # Only allow from specific repo and branch
          "token.actions.githubusercontent.com:sub" = "repo:Gu1lh4s/myapp:ref:refs/heads/main"
        }
      }
    }]
  })
}

resource "aws_iam_role_policy" "github_actions_ecr" {
  role = aws_iam_role.github_actions_deploy.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["ecr:GetAuthorizationToken"]
        Resource = "*"
      },
      {
        Effect = "Allow"
        Action = [
          "ecr:BatchGetImage",
          "ecr:BatchCheckLayerAvailability",
          "ecr:CompleteLayerUpload",
          "ecr:GetDownloadUrlForLayer",
          "ecr:InitiateLayerUpload",
          "ecr:PutImage",
          "ecr:UploadLayerPart",
        ]
        Resource = "arn:aws:ecr:us-east-1:${var.aws_account_id}:repository/myapp-api"
      }
    ]
  })
}
```

```yaml
# .github/workflows/deploy.yml — uses OIDC, no stored secrets
- name: Configure AWS credentials via OIDC
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789:role/github-actions-deploy-prod
    aws-region: us-east-1
    # No access-key-id or secret-access-key — OIDC handles auth
```

### IAM Permission Boundaries

```json
// Permission boundary: maximum permissions an IAM entity can have
// Useful when delegating IAM creation to teams (they can't escalate beyond boundary)
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:*",
        "dynamodb:*",
        "sqs:*",
        "sns:*",
        "cloudwatch:*",
        "logs:*"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Deny",
      "Action": [
        "iam:*",
        "organizations:*",
        "account:*"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## 2. VPC and Networking

### Production VPC Architecture

```
REGION: us-east-1

VPC: 10.0.0.0/16

  Availability Zone A (us-east-1a)     Availability Zone B (us-east-1b)
  ┌─────────────────────────────┐       ┌─────────────────────────────┐
  │  Public Subnet              │       │  Public Subnet              │
  │  10.0.1.0/24                │       │  10.0.2.0/24                │
  │  [NAT Gateway]              │       │  [NAT Gateway]              │
  │  [ALB nodes]                │       │  [ALB nodes]                │
  ├─────────────────────────────┤       ├─────────────────────────────┤
  │  Private Subnet (App)       │       │  Private Subnet (App)       │
  │  10.0.11.0/24               │       │  10.0.12.0/24               │
  │  [ECS tasks / EC2 / Lambda] │       │  [ECS tasks / EC2 / Lambda] │
  ├─────────────────────────────┤       ├─────────────────────────────┤
  │  Private Subnet (Data)      │       │  Private Subnet (Data)      │
  │  10.0.21.0/24               │       │  10.0.22.0/24               │
  │  [RDS Primary]              │       │  [RDS Replica]              │
  │  [ElastiCache]              │       │  [ElastiCache replica]      │
  └─────────────────────────────┘       └─────────────────────────────┘

Internet Gateway ← routes to Public Subnets only
NAT Gateway ← Private subnets route outbound through NAT (in Public Subnet)
```

```hcl
# terraform/vpc.tf
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = { Name = "prod-vpc" }
}

resource "aws_subnet" "public" {
  for_each = {
    "us-east-1a" = "10.0.1.0/24"
    "us-east-1b" = "10.0.2.0/24"
  }
  
  vpc_id                  = aws_vpc.main.id
  cidr_block              = each.value
  availability_zone       = each.key
  map_public_ip_on_launch = true  # Only for public subnets
  
  tags = { Name = "public-${each.key}" }
}

resource "aws_subnet" "private_app" {
  for_each = {
    "us-east-1a" = "10.0.11.0/24"
    "us-east-1b" = "10.0.12.0/24"
  }
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value
  availability_zone = each.key
  # No map_public_ip_on_launch for private subnets
  
  tags = { Name = "private-app-${each.key}" }
}

# Security Groups (stateful firewall)
resource "aws_security_group" "api" {
  name        = "api-sg"
  vpc_id      = aws_vpc.main.id
  description = "API service security group"
  
  ingress {
    from_port       = 3000
    to_port         = 3000
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]  # Only from ALB
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]  # Allow all outbound (to reach RDS, Redis, external APIs)
  }
}

resource "aws_security_group" "rds" {
  name        = "rds-sg"
  vpc_id      = aws_vpc.main.id
  description = "RDS security group — only accessible from API"
  
  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.api.id]  # Only from API
  }
  
  # No egress rule = no outbound (DB doesn't initiate connections)
}
```

---

## 3. Compute — EC2, ECS, Lambda

### ECS Fargate Task Definition

```json
{
  "family": "api-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::123456789:role/ecs-execution-role",
  "taskRoleArn": "arn:aws:iam::123456789:role/ecs-task-role",
  
  "containerDefinitions": [{
    "name": "api",
    "image": "123456789.dkr.ecr.us-east-1.amazonaws.com/api:1.2.3",
    
    "portMappings": [{
      "containerPort": 3000,
      "protocol": "tcp"
    }],
    
    "environment": [
      { "name": "NODE_ENV", "value": "production" },
      { "name": "PORT", "value": "3000" }
    ],
    
    "secrets": [
      {
        "name": "DATABASE_URL",
        "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789:secret:prod/api/database_url"
      },
      {
        "name": "JWT_SECRET",
        "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789:secret:prod/api/jwt_secret"
      }
    ],
    
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/api",
        "awslogs-region": "us-east-1",
        "awslogs-stream-prefix": "api"
      }
    },
    
    "healthCheck": {
      "command": ["CMD-SHELL", "curl -f http://localhost:3000/health || exit 1"],
      "interval": 30,
      "timeout": 5,
      "retries": 3,
      "startPeriod": 10
    },
    
    "readonlyRootFilesystem": true,
    
    "linuxParameters": {
      "tmpfs": [{
        "containerPath": "/tmp",
        "size": 64
      }]
    }
  }]
}
```

### Lambda Best Practices

```typescript
// Initialization outside handler (runs once per cold start, reused across invocations)
import { SecretsManagerClient, GetSecretValueCommand } from '@aws-sdk/client-secrets-manager'
import { Pool } from 'pg'

let pool: Pool | null = null  // Reuse across invocations

async function getPool(): Promise<Pool> {
  if (pool) return pool  // Connection reuse (warm invocation)
  
  const secrets = await getSecrets()
  pool = new Pool({
    connectionString: secrets.DATABASE_URL,
    max: 2,  // Keep low for Lambda (many concurrent executions = many connections)
    idleTimeoutMillis: 120_000,
  })
  return pool
}

// Handler: keep thin, delegate to business logic
export const handler = async (event: APIGatewayProxyEvent): Promise<APIGatewayProxyResult> => {
  const db = await getPool()
  
  try {
    const bookings = await db.query(
      'SELECT * FROM bookings WHERE tenant_id = $1 LIMIT 20',
      [event.pathParameters?.tenantId]
    )
    
    return {
      statusCode: 200,
      headers: {
        'Content-Type': 'application/json',
        'Cache-Control': 'no-store',
      },
      body: JSON.stringify({ data: bookings.rows }),
    }
  } catch (err) {
    console.error(JSON.stringify({ error: err, traceId: event.requestContext?.requestId }))
    return {
      statusCode: 500,
      body: JSON.stringify({ error: 'Internal error' }),
    }
  }
}

// Lambda performance configuration
// - Memory: set higher (more memory = more CPU; test optimal for your function)
// - Timeout: set conservatively (don't use 15 min for an API endpoint)
// - Provisioned Concurrency: for latency-sensitive endpoints (eliminates cold starts)
// - ARM64 (Graviton2): 20% cheaper, same or better performance for most workloads
```

---

## 4. Storage — S3, EBS, EFS

### S3 Secure Configuration

```hcl
resource "aws_s3_bucket" "uploads" {
  bucket = "myapp-uploads-prod"
}

# Block all public access (critical!)
resource "aws_s3_bucket_public_access_block" "uploads" {
  bucket = aws_s3_bucket.uploads.id
  
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Encryption at rest (AES-256 via KMS)
resource "aws_s3_bucket_server_side_encryption_configuration" "uploads" {
  bucket = aws_s3_bucket.uploads.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.s3.arn
    }
    bucket_key_enabled = true  # Reduce KMS API calls (cost optimization)
  }
}

# Versioning (protect against accidental deletion)
resource "aws_s3_bucket_versioning" "uploads" {
  bucket = aws_s3_bucket.uploads.id
  versioning_configuration { status = "Enabled" }
}

# Lifecycle: move old versions to Glacier after 90 days
resource "aws_s3_bucket_lifecycle_configuration" "uploads" {
  bucket = aws_s3_bucket.uploads.id
  
  rule {
    id     = "archive-old-versions"
    status = "Enabled"
    
    noncurrent_version_transition {
      noncurrent_days = 90
      storage_class   = "GLACIER"
    }
    
    noncurrent_version_expiration {
      noncurrent_days = 365  # Delete after 1 year
    }
  }
}
```

### Pre-Signed URLs (Secure File Upload Pattern)

```typescript
import { S3Client, PutObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3'
import { getSignedUrl } from '@aws-sdk/s3-request-presigner'

const s3 = new S3Client({ region: 'us-east-1' })

// Generate upload URL (client uploads directly to S3 — bypass API for large files)
async function generateUploadUrl(userId: string, filename: string, contentType: string) {
  const key = `uploads/${userId}/${randomUUID()}-${sanitize(filename)}`
  
  // Validate content type (prevent uploading executables)
  const ALLOWED_TYPES = ['image/jpeg', 'image/png', 'image/webp', 'application/pdf']
  if (!ALLOWED_TYPES.includes(contentType)) {
    throw new ValidationError('File type not allowed')
  }
  
  const command = new PutObjectCommand({
    Bucket: process.env.UPLOADS_BUCKET,
    Key: key,
    ContentType: contentType,
    // Enforce file size at S3 level via conditions in presigned URL
    ContentLengthRange: [1, 10 * 1024 * 1024],  // 1 byte to 10MB
    Metadata: { 'uploaded-by': userId },
  })
  
  const url = await getSignedUrl(s3, command, { expiresIn: 300 })  // 5 minutes
  
  return { uploadUrl: url, key }
}

// Generate download URL (authenticated access only)
async function generateDownloadUrl(key: string, userId: string) {
  // First: verify user has permission to this file
  await verifyOwnership(key, userId)
  
  const command = new GetObjectCommand({
    Bucket: process.env.UPLOADS_BUCKET,
    Key: key,
    ResponseContentDisposition: 'attachment',  // Force download
  })
  
  return getSignedUrl(s3, command, { expiresIn: 900 })  // 15 minutes
}
```

---

## 5. Database — RDS, Aurora, DynamoDB

### RDS PostgreSQL Production Config

```hcl
resource "aws_db_instance" "postgres" {
  identifier        = "prod-postgres"
  engine            = "postgres"
  engine_version    = "16.2"
  instance_class    = "db.r6g.large"  # ARM64 (Graviton) — 20% cheaper
  
  # Storage
  allocated_storage     = 100
  max_allocated_storage = 1000   # Auto-scaling up to 1TB
  storage_type          = "gp3"  # gp3 better cost/performance than gp2
  storage_encrypted     = true
  kms_key_id            = aws_kms_key.rds.arn
  
  # High availability
  multi_az = true  # Standby replica in another AZ; auto-failover
  
  # Credentials (from Secrets Manager rotation)
  manage_master_user_password = true  # AWS manages rotation automatically
  
  # Network
  db_subnet_group_name   = aws_db_subnet_group.private.name
  vpc_security_group_ids = [aws_security_group.rds.id]
  publicly_accessible    = false  # NEVER expose DB to internet
  
  # Backup and recovery
  backup_retention_period = 30     # 30 days PITR
  backup_window           = "03:00-04:00"  # UTC — low traffic period
  maintenance_window      = "Mon:04:00-Mon:05:00"
  deletion_protection     = true   # Prevents accidental deletion
  
  # Monitoring
  monitoring_interval             = 60  # Enhanced monitoring
  monitoring_role_arn             = aws_iam_role.rds_monitoring.arn
  performance_insights_enabled    = true
  performance_insights_retention_period = 7  # 7 days free tier
  
  # Logging
  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]
  
  # Parameter group
  parameter_group_name = aws_db_parameter_group.postgres16.name
}

resource "aws_db_parameter_group" "postgres16" {
  name   = "prod-postgres16"
  family = "postgres16"
  
  parameter {
    name  = "shared_buffers"
    value = "{DBInstanceClassMemory/4}"  # 25% of instance RAM
  }
  
  parameter {
    name  = "log_min_duration_statement"
    value = "1000"  # Log queries > 1 second
  }
  
  parameter {
    name  = "log_lock_waits"
    value = "on"
  }
  
  parameter {
    name  = "idle_in_transaction_session_timeout"
    value = "30000"  # Kill idle-in-transaction after 30s
  }
}
```

### Aurora Serverless v2 (for variable workloads)

```hcl
resource "aws_rds_cluster" "aurora" {
  cluster_identifier = "prod-aurora"
  engine             = "aurora-postgresql"
  engine_version     = "16.1"
  
  # Serverless v2 scaling
  serverlessv2_scaling_configuration {
    min_capacity = 0.5   # 0.5 ACU = ~0.9 GB RAM
    max_capacity = 16    # 16 ACU = ~32 GB RAM
    # Scales in steps based on load
  }
  
  # Global Aurora: replicate to another region (DR)
  # enable_global_write_forwarding = true
  
  database_name   = "appdb"
  master_username = "appuser"
  
  backup_retention_period = 30
  preferred_backup_window = "03:00-04:00"
  
  storage_encrypted   = true
  kms_key_id          = aws_kms_key.aurora.arn
  deletion_protection = true
  
  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name   = aws_db_subnet_group.private.name
}
```

---

## 6. Secrets and Configuration

```hcl
# AWS Secrets Manager — automatic rotation
resource "aws_secretsmanager_secret" "api_secrets" {
  name                    = "prod/api/secrets"
  description             = "Production API secrets"
  kms_key_id              = aws_kms_key.secrets.arn
  recovery_window_in_days = 7  # 7 days before permanent deletion
}

resource "aws_secretsmanager_secret_version" "api_secrets" {
  secret_id = aws_secretsmanager_secret.api_secrets.id
  
  secret_string = jsonencode({
    DATABASE_URL  = "postgresql://..."
    JWT_SECRET    = random_password.jwt_secret.result
    SMTP_PASSWORD = var.smtp_password
  })
}

# Auto-rotation for RDS credentials
resource "aws_secretsmanager_secret_rotation" "db_password" {
  secret_id           = aws_secretsmanager_secret.db_password.id
  rotation_lambda_arn = aws_lambda_function.rotate_secret.arn
  
  rotation_rules {
    automatically_after_days = 30
  }
}
```

```typescript
// Read secret in application (cache to avoid API rate limits)
import { SecretsManagerClient, GetSecretValueCommand } from '@aws-sdk/client-secrets-manager'

const client = new SecretsManagerClient({ region: 'us-east-1' })
let cachedSecret: Record<string, string> | null = null
let cacheExpiry = 0

async function getSecrets(): Promise<Record<string, string>> {
  if (cachedSecret && Date.now() < cacheExpiry) return cachedSecret
  
  const command = new GetSecretValueCommand({ SecretId: 'prod/api/secrets' })
  const response = await client.send(command)
  
  cachedSecret = JSON.parse(response.SecretString!)
  cacheExpiry = Date.now() + 5 * 60 * 1000  // Cache 5 minutes
  
  return cachedSecret
}
```

---

## 7. Observability — CloudWatch, X-Ray

```typescript
// Structured logging to CloudWatch Logs Insights
// Use JSON — CloudWatch Insights queries work on structured fields

logger.info({
  service: 'api',
  traceId: context.awsRequestId,
  userId: req.user?.id,
  route: req.path,
  duration: Date.now() - startTime,
  statusCode: res.statusCode,
}, 'Request completed')

// CloudWatch Logs Insights query
// Find slow requests:
/*
fields @timestamp, route, duration, userId
| filter service = "api" and duration > 1000
| sort duration desc
| limit 20
*/

// CloudWatch Alarms
resource "aws_cloudwatch_metric_alarm" "api_error_rate" {
  alarm_name          = "prod-api-high-error-rate"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "5XXError"
  namespace           = "AWS/ApplicationELB"
  period              = 60
  statistic           = "Sum"
  threshold           = 10  # > 10 errors in a minute
  
  dimensions = {
    LoadBalancer = aws_alb.main.arn_suffix
  }
  
  alarm_actions = [aws_sns_topic.alerts.arn]
  ok_actions    = [aws_sns_topic.alerts.arn]
}
```

---

## 8. Security Services

```hcl
# AWS WAF on ALB/CloudFront
resource "aws_wafv2_web_acl" "main" {
  name  = "prod-waf"
  scope = "REGIONAL"  # Use CLOUDFRONT for CloudFront distributions
  
  default_action { allow {} }
  
  # AWS Managed Rules (free — covers OWASP Top 10)
  rule {
    name     = "AWSManagedRulesCommonRuleSet"
    priority = 1
    
    override_action { none {} }
    
    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"
      }
    }
    
    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "CommonRules"
      sampled_requests_enabled   = true
    }
  }
  
  # Rate limiting: 100 req/5min per IP
  rule {
    name     = "RateLimit"
    priority = 0  # Checked first
    
    action { block {} }
    
    statement {
      rate_based_statement {
        limit              = 100
        aggregate_key_type = "IP"
      }
    }
    
    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "RateLimit"
      sampled_requests_enabled   = true
    }
  }
}

# AWS GuardDuty — threat detection
resource "aws_guardduty_detector" "main" {
  enable = true
  
  datasources {
    s3_logs { enable = true }
    kubernetes { audit_logs { enable = true } }
    malware_protection {
      scan_ec2_instance_with_findings { ebs_volumes { enable = true } }
    }
  }
}

# AWS Config — compliance monitoring
resource "aws_config_rule" "s3_bucket_public_read_prohibited" {
  name = "s3-bucket-public-read-prohibited"
  source {
    owner             = "AWS"
    source_identifier = "S3_BUCKET_PUBLIC_READ_PROHIBITED"
  }
}
```

---

## 9. Cost Optimization

```
EC2/ECS/EKS cost reduction:
  → Savings Plans: commit to compute usage (up to 66% savings vs On-Demand)
  → Reserved Instances: 1-3 year commitment (up to 72% savings)
  → Spot Instances: for batch/worker workloads (70-90% savings, but interruptible)
  → ARM64 (Graviton3): same or better performance, 20% cheaper

RDS:
  → Reserved instances: 1 year = 40% savings, 3 year = 60% savings
  → Aurora Serverless v2: pay per ACU-hour (great for variable load)
  → Stop non-production DBs at night (save 65% on dev/staging)

S3:
  → Intelligent-Tiering: auto-move objects between tiers based on access patterns
  → S3 Glacier for backups older than 90 days (10% of S3 Standard cost)
  → Request cost: reduce by caching S3 responses at CloudFront

Data Transfer:
  → Biggest hidden cost: data transfer OUT of AWS ($0.09/GB)
  → Use CloudFront: S3 → CloudFront is free; CloudFront → user is cheaper
  → VPC endpoints: S3/DynamoDB within VPC (avoid NAT Gateway charges)

Cost monitoring:
  → AWS Cost Explorer: trends, anomalies
  → AWS Budgets: alert when spend exceeds threshold
  → AWS Compute Optimizer: right-sizing recommendations
```

```hcl
# Cost budget alert
resource "aws_budgets_budget" "monthly" {
  name         = "monthly-production-budget"
  budget_type  = "COST"
  limit_amount = "2000"
  limit_unit   = "USD"
  time_unit    = "MONTHLY"
  
  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80  # Alert at 80% of budget
    threshold_type             = "PERCENTAGE"
    notification_type          = "FORECASTED"  # Alert when forecasted to exceed
    subscriber_email_addresses = ["platform@company.com"]
  }
}
```

---

## 10. Infrastructure as Code (Terraform)

### Module Structure

```
terraform/
├── environments/
│   ├── prod/
│   │   ├── main.tf       # Module calls
│   │   ├── variables.tf  # Environment-specific values
│   │   └── terraform.tfvars
│   └── staging/
│       ├── main.tf
│       └── terraform.tfvars
│
└── modules/
    ├── vpc/              # Reusable VPC module
    ├── rds/              # Reusable RDS module
    ├── ecs-service/      # Reusable ECS Fargate service
    └── alb/              # Reusable ALB module
```

```hcl
# terraform/environments/prod/main.tf

terraform {
  required_version = ">= 1.6"
  
  backend "s3" {
    bucket         = "myapp-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"  # Prevent concurrent applies
  }
}

module "vpc" {
  source = "../../modules/vpc"
  
  environment = "prod"
  cidr        = "10.0.0.0/16"
  azs         = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

module "api" {
  source = "../../modules/ecs-service"
  
  name          = "api"
  image         = "${var.ecr_registry}/api:${var.image_tag}"
  cpu           = 512
  memory        = 1024
  desired_count = 3
  
  vpc_id          = module.vpc.vpc_id
  private_subnets = module.vpc.private_app_subnet_ids
  alb_target_group_arn = module.alb.api_target_group_arn
  
  environment_variables = {
    NODE_ENV = "production"
    PORT     = "3000"
  }
  
  secrets = {
    DATABASE_URL = module.secrets.database_url_arn
    JWT_SECRET   = module.secrets.jwt_secret_arn
  }
}

# Tagging strategy (critical for cost allocation and compliance)
locals {
  common_tags = {
    Environment = "production"
    Project     = "myapp"
    ManagedBy   = "terraform"
    CostCenter  = "engineering"
    Owner       = "platform-team"
  }
}
```

### Terraform Best Practices

```hcl
# 1. Lock provider versions
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"  # Allow 5.x, not 6.x
    }
  }
}

# 2. Use data sources for existing resources (don't import)
data "aws_vpc" "existing" {
  tags = { Name = "prod-vpc" }
}

# 3. Sensitive variables never hardcoded
variable "db_password" {
  type      = string
  sensitive = true  # Won't appear in plan output
}

# 4. Validate variables
variable "environment" {
  type = string
  validation {
    condition     = contains(["prod", "staging", "dev"], var.environment)
    error_message = "Environment must be prod, staging, or dev"
  }
}

# 5. Output important values for other modules/teams
output "api_url" {
  value       = "https://${aws_alb.main.dns_name}"
  description = "API load balancer URL"
}
```
