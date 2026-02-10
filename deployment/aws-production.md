# AWS Production Deployment

Production deployment on AWS using Terraform and ECS.

## Prerequisites

- AWS account with admin access
- Terraform 1.6+
- GitHub repository with Actions enabled
- Domain name managed in Route 53 (or external DNS)

## Cost Tiers

| Tier | Monthly Cost | Setup |
|------|-------------|-------|
| **Starter** | $75–150 | Single t3.medium, RDS db.t3.micro, ElastiCache t3.micro |
| **Growth** | $300–600 | ECS Fargate (2 tasks), RDS db.r6g.large, ElastiCache r6g.large |
| **Production** | $950–1750 | ECS Fargate (4+ tasks), Aurora Multi-AZ, ElastiCache cluster |

## 1. Bootstrap Terraform

```bash
cd infrastructure/terraform

# Initialize with S3 backend
terraform init \
  -backend-config="bucket=discordination-tfstate" \
  -backend-config="key=prod/terraform.tfstate" \
  -backend-config="region=us-east-1"
```

## 2. Configure Variables

Create `terraform.tfvars`:

```hcl
environment     = "prod"
domain          = "discordination.com"
aws_region      = "us-east-1"
vpc_cidr        = "10.0.0.0/16"

# Database
db_instance_class   = "db.r6g.large"
db_multi_az         = true
db_backup_retention = 7

# Cache
redis_node_type = "cache.r6g.large"

# ECS
api_cpu         = 512
api_memory      = 1024
api_desired     = 2
gateway_cpu     = 1024
gateway_memory  = 2048
gateway_desired = 2

# LiveKit (optional)
livekit_instance_type = "c5.xlarge"
livekit_count         = 2
```

## 3. Deploy Infrastructure

```bash
# Review the plan
terraform plan -out=tfplan

# Apply
terraform apply tfplan
```

This provisions:
- VPC with public/private subnets across 2 AZs
- RDS Aurora PostgreSQL cluster
- ElastiCache Redis cluster
- ECS Fargate cluster with services
- Application Load Balancer with TLS
- S3 bucket for file storage
- CloudFront distribution
- Route 53 DNS records
- Security groups and IAM roles

## 4. Configure GitHub Actions

Set the following GitHub Secrets:

| Secret | Description |
|--------|-------------|
| `AWS_ROLE_ARN` | IAM role ARN for OIDC federation |
| `AWS_REGION` | AWS region (e.g., `us-east-1`) |
| `ECR_REGISTRY` | ECR registry URL |

OIDC federation means no stored AWS keys — GitHub Actions assumes the role via JWT.

## 5. Deploy Services

Push to main to trigger the CI/CD pipeline:

```bash
git push origin main
```

The pipeline:
1. Runs tests
2. Builds Docker images
3. Pushes to GHCR
4. Updates ECS task definitions
5. Deploys with rolling update (zero downtime)

## 6. Database Migrations

Migrations run automatically as an ECS task before the API service deploys. To run manually:

```bash
aws ecs run-task \
  --cluster discordination-prod \
  --task-definition discordination-migrate \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-xxx],securityGroups=[sg-xxx]}"
```

## Monitoring

### CloudWatch Dashboards

Terraform creates a CloudWatch dashboard with:
- ECS CPU/memory utilization
- ALB request count and latency (p50, p95, p99)
- RDS connections and IOPS
- Redis memory and cache hit ratio

### Alerts

Default alerts (SNS → email):
- ECS task unhealthy (health check failures)
- RDS CPU > 80% for 5 minutes
- Redis memory > 80%
- ALB 5xx error rate > 1%
- Disk space < 20%

### Logs

All services log to CloudWatch Logs in JSON format:

```bash
# View API logs
aws logs tail /ecs/discordination-api --follow

# View Gateway logs
aws logs tail /ecs/discordination-gateway --follow
```

## Scaling

### Horizontal (ECS Auto Scaling)

```hcl
# In terraform.tfvars
api_min_count = 2
api_max_count = 10
api_scale_cpu_target = 70    # Scale up at 70% CPU

gateway_min_count = 2
gateway_max_count = 8
gateway_scale_connections = 5000  # Scale at 5k connections per task
```

### Vertical

Update task CPU/memory in `terraform.tfvars` and apply:

```bash
terraform apply -var="api_cpu=1024" -var="api_memory=2048"
```

### Database

```bash
# Modify RDS instance class (causes brief failover if Multi-AZ)
terraform apply -var="db_instance_class=db.r6g.xlarge"

# Add read replicas
terraform apply -var="db_read_replicas=2"
```
