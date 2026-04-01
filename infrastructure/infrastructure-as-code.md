# Infrastructure as Code

## Context & Problem

Manually provisioned infrastructure is unreproducible, undocumented, and dangerous. "Someone set up the Kafka cluster last year, but nobody remembers the configuration" is how production incidents start.

Infrastructure as Code (IaC) declares infrastructure in version-controlled files. The same code that defines the application also defines where and how it runs. Changes are reviewed, tested, and applied through the same PR process as application code.

## Design Decisions

### Tool Selection

| Tool | Scope | Language | State | Best For |
|---|---|---|---|---|
| **Terraform** | Cloud resources (AWS, GCP, Azure) | HCL | Remote state file | Multi-cloud, cloud-agnostic |
| **Pulumi** | Cloud resources | Python, TypeScript, Go | Remote state | Teams that prefer real languages |
| **Docker Compose** | Local development | YAML | Stateless | Local infra (already covered) |
| **Ansible** | Configuration management | YAML | Stateless | VM configuration, on-prem |

For this repository's scope — systems that run locally and deploy to cloud — **Terraform** for cloud resources and **Docker Compose** for local development.

### Project Structure

```
infrastructure/
├── terraform/
│   ├── environments/
│   │   ├── staging/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   └── terraform.tfvars
│   │   └── production/
│   │       ├── main.tf
│   │       ├── variables.tf
│   │       └── terraform.tfvars
│   ├── modules/
│   │   ├── database/           # RDS/PostgreSQL module
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   └── outputs.tf
│   │   ├── kafka/              # MSK/Confluent module
│   │   ├── redis/              # ElastiCache module
│   │   ├── networking/         # VPC, subnets, security groups
│   │   └── application/        # ECS/EKS service definition
│   └── shared/
│       └── backend.tf          # Remote state configuration
```

### Module Pattern

Reusable Terraform modules mirror the application's bounded contexts:

```hcl
# modules/database/main.tf

resource "aws_db_instance" "main" {
  identifier     = "${var.project}-${var.environment}-db"
  engine         = "postgres"
  engine_version = "16"
  instance_class = var.instance_class

  db_name  = var.db_name
  username = var.db_username
  password = var.db_password

  # TimescaleDB via shared_preload_libraries
  parameter_group_name = aws_db_parameter_group.timescale.name

  # Security
  vpc_security_group_ids = [var.db_security_group_id]
  db_subnet_group_name   = var.db_subnet_group_name
  storage_encrypted      = true

  # Backups
  backup_retention_period = 30
  backup_window          = "03:00-04:00"

  # Monitoring
  monitoring_interval          = 60
  performance_insights_enabled = true

  tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

resource "aws_db_parameter_group" "timescale" {
  family = "postgres16"
  name   = "${var.project}-${var.environment}-timescale"

  parameter {
    name  = "shared_preload_libraries"
    value = "timescaledb"
  }

  parameter {
    name  = "max_connections"
    value = "200"
  }
}
```

### Environment Configuration

```hcl
# environments/staging/main.tf

module "database" {
  source = "../../modules/database"

  project       = "platform"
  environment   = "staging"
  instance_class = "db.t3.medium"
  db_name       = "app"
  db_username   = "app"
  db_password   = var.db_password  # from secrets
}

module "kafka" {
  source = "../../modules/kafka"

  project      = "platform"
  environment  = "staging"
  broker_count = 3
  instance_type = "kafka.t3.small"
}

module "redis" {
  source = "../../modules/redis"

  project       = "platform"
  environment   = "staging"
  node_type     = "cache.t3.micro"
  num_cache_nodes = 1
}
```

### Local-to-Cloud Parity

The local Docker Compose and cloud Terraform should produce equivalent environments. This mapping ensures local development behavior matches production:

| Local (Docker Compose) | Cloud (Terraform) |
|---|---|
| `timescale/timescaledb:latest-pg16` | AWS RDS PostgreSQL 16 + TimescaleDB |
| `confluentinc/cp-kafka:7.6.0` | AWS MSK or Confluent Cloud |
| `redis:7-alpine` | AWS ElastiCache Redis 7 |
| `openfga/openfga:latest` | ECS/EKS container |
| `minio/minio:latest` | AWS S3 |
| `elasticsearch:8.13.0` | AWS OpenSearch |

### State Management

```hcl
# shared/backend.tf

terraform {
  backend "s3" {
    bucket         = "platform-terraform-state"
    key            = "state/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

- Remote state in S3 (or equivalent) — never local
- DynamoDB lock table prevents concurrent modifications
- State is encrypted at rest

### CI/CD for Infrastructure

```yaml
# .github/workflows/terraform.yml

name: Infrastructure

on:
  pull_request:
    paths: ["infrastructure/terraform/**"]
  push:
    branches: [main]
    paths: ["infrastructure/terraform/**"]

jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init
        working-directory: infrastructure/terraform/environments/staging
      - run: terraform plan -out=plan.tfplan
        working-directory: infrastructure/terraform/environments/staging
      # Post plan as PR comment for review

  apply:
    if: github.ref == 'refs/heads/main'
    needs: [plan]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init && terraform apply -auto-approve
        working-directory: infrastructure/terraform/environments/staging
```

## Failure Modes

| Failure | Cause | Mitigation |
|---|---|---|
| State corruption | Concurrent applies, interrupted apply | Remote state with locking (DynamoDB), never manual edits |
| Drift | Manual changes in cloud console | `terraform plan` in CI detects drift, enforce no manual changes |
| Destroy instead of update | Wrong resource lifecycle | `prevent_destroy` on critical resources, plan review |
| Secret in state file | Passwords stored in Terraform state | Encrypt state, use dynamic secrets (Vault), limit state access |
| Module version mismatch | Different module versions across environments | Pin module versions, version constraints |

## Related Documents

- [Docker Compose Patterns](docker-compose-patterns.md) — local equivalent of cloud infrastructure
- [Secret Management](secret-management.md) — handling credentials in IaC
- [CI/CD Patterns](ci-cd-patterns.md) — automating infrastructure deployment
