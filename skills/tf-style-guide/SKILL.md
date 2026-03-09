---
name: tf-style-guide
category: Infrastructure
tags: terraform, infrastructure-as-code, cloud, devops, automation, aws, azure, gcp
description: >-
  Terraform style conventions, formatting standards, and best practices for writing clean HCL.
  Use when users ask about Terraform coding standards, file organization, naming conventions,
  formatting rules, variable/output structure, or when writing new Terraform configurations.
  Also use when reviewing Terraform for style compliance or when users want to establish
  team-wide Terraform conventions. Covers file layout, naming, block ordering, security
  practices, version pinning, and validation tooling.
---

# Terraform Style Guide

Comprehensive style conventions for writing clean, consistent, and maintainable Terraform configurations.

## File Organization

Organize Terraform code into purpose-specific files:

| File           | Contents                                                           |
| -------------- | ------------------------------------------------------------------ |
| `terraform.tf` | `terraform` block with `required_version` and `required_providers` |
| `providers.tf` | Provider configurations and aliases                                |
| `main.tf`      | Primary resources (or split by domain: `network.tf`, `compute.tf`) |
| `variables.tf` | All input variable declarations                                    |
| `outputs.tf`   | All output declarations                                            |
| `locals.tf`    | Local value computations                                           |
| `data.tf`      | Data sources (or co-locate with consuming resources)               |

For modules with many resources, split `main.tf` by logical domain rather than creating one massive file.

## Block Ordering Within Files

Follow a consistent ordering within each file:

```hcl
# 1. Meta-arguments first
resource "aws_instance" "web" {
  count = var.instance_count

  # 2. Required arguments
  ami           = var.ami_id
  instance_type = var.instance_type
  subnet_id     = var.subnet_id

  # 3. Optional arguments
  monitoring = true

  # 4. Nested blocks
  root_block_device {
    volume_size = 50
    encrypted   = true
  }

  # 5. Tags last
  tags = local.common_tags
}
```

## Formatting Standards

- **Indentation**: 2 spaces, no tabs
- **Aligned equals**: Align `=` signs within a block for readability
- **Blank lines**: One blank line between top-level blocks, no trailing blanks
- **Line length**: Keep under 120 characters; break long expressions
- **Quotes**: Double quotes for strings, no quotes for keywords/booleans
- **Enforce**: Run `terraform fmt` before every commit

```hcl
# Good — aligned equals
resource "aws_s3_bucket" "logs" {
  bucket        = "${var.prefix}-logs"
  force_destroy = false

  tags = local.common_tags
}

# Bad — misaligned
resource "aws_s3_bucket" "logs" {
  bucket = "${var.prefix}-logs"
  force_destroy = false
  tags = local.common_tags
}
```

## Naming Conventions

- **Resources/data sources**: `snake_case`, descriptive nouns — `aws_s3_bucket.application_logs`
- **Variables/outputs/locals**: `snake_case`, no resource type in name
- **Modules**: `kebab-case` directory names
- **Files**: `snake_case` with `.tf` extension

```hcl
# Good — descriptive, no type in name
variable "instance_count" {
  type = number
}

resource "aws_security_group" "application" {}

# Bad — type in name, vague
variable "sg_count_number" {
  type = number
}

resource "aws_security_group" "aws_sg" {}
```

## Variables and Outputs

Every variable MUST have `type` and `description`. Add `validation` blocks for constrained inputs.

```hcl
variable "environment" {
  type        = string
  description = "Deployment environment (dev, staging, prod)"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "tags" {
  type        = map(string)
  description = "Resource tags applied to all resources"
  default     = {}
}
```

Every output MUST have `description`. Mark sensitive outputs.

```hcl
output "database_endpoint" {
  description = "RDS instance endpoint for application connections"
  value       = aws_db_instance.main.endpoint
}

output "database_password" {
  description = "Generated database password"
  value       = random_password.db.result
  sensitive   = true
}
```

## Dynamic Resources

Prefer `for_each` over `count` — it produces stable resource addresses and handles additions/removals without index shifts.

```hcl
# Good — for_each with map
resource "aws_iam_user" "team" {
  for_each = toset(var.team_members)
  name     = each.value
}

# Avoid — count causes index shifting on removal
resource "aws_iam_user" "team" {
  count = length(var.team_members)
  name  = var.team_members[count.index]
}
```

Use `count` only for conditional creation:

```hcl
resource "aws_cloudwatch_log_group" "this" {
  count = var.enable_logging ? 1 : 0
  name  = "/app/${var.name}"
}
```

## Security Practices

- **Never hardcode credentials** — use variables, SSM, Secrets Manager, or environment
- **Encrypt by default** — enable encryption on S3, RDS, EBS, EFS, SQS, SNS
- **Mark sensitive values** — `sensitive = true` on variables and outputs containing secrets
- **Least privilege IAM** — scope policies to specific resources and actions
- **No wildcard actions** — avoid `"Action": "*"` in IAM policies

```hcl
# Good — scoped IAM policy
data "aws_iam_policy_document" "lambda" {
  statement {
    actions   = ["s3:GetObject", "s3:PutObject"]
    resources = ["${aws_s3_bucket.data.arn}/*"]
  }
}

# Bad — overly permissive
data "aws_iam_policy_document" "lambda" {
  statement {
    actions   = ["s3:*"]
    resources = ["*"]
  }
}
```

### Secrets Management

Use [tf-module-simple-secrets](https://github.com/RingierIMU/tf-module-simple-secrets) for all SSM secrets. Never create raw `aws_ssm_parameter` resources — the module handles KMS encryption and consistent path structure.

```hcl
# Define secrets as a map, use for_each
module "secrets" {
  source   = "git@github.com:RingierIMU/tf-module-simple-secrets"
  for_each = tomap({
    webhook_secret    = var.webhook_secret
    slack_bot_token   = var.slack_bot_token
    anthropic_api_key = var.anthropic_api_key
  })

  key_id     = data.terraform_remote_state.env.outputs.kms_params_arn
  ssm_prefix = "${terraform.workspace}/${var.app_name}"
  tags       = local.tags
  key        = each.key
  value      = each.value
}
```

SSM path convention: `/{workspace}/{app_name}/{key}` (e.g., `/staging/my-app/webhook_secret`).

Reference secret ARNs via module outputs — never hardcode SSM ARNs:

```hcl
# Good — reference module output
secrets = {
  WEBHOOK_SECRET = module.secrets["webhook_secret"].arn
}

# Bad — hardcoded ARN
secrets = {
  WEBHOOK_SECRET = "arn:aws:ssm:eu-west-1:123456:parameter/staging/my-app/webhook_secret"
}
```

## ECS Services

Use the [terraform-aws-modules/ecs/aws//modules/service](https://github.com/terraform-aws-modules/terraform-aws-ecs) submodule for ECS services and task definitions. Never create raw `aws_ecs_service` or `aws_ecs_task_definition` resources — the module handles task definitions, service configuration, and autoscaling consistently.

```hcl
module "ecs_service" {
  source  = "terraform-aws-modules/ecs/aws//modules/service"
  version = "~> 5.0"

  name        = "${var.app_name}-api"
  cluster_arn = var.ecs_cluster_arn

  cpu    = 256
  memory = 512

  container_definitions = {
    api = {
      cpu       = 256
      memory    = 512
      essential = true
      image     = "${var.ecr_repo_url}:${var.image_tag}"

      port_mappings = [
        {
          name          = "api"
          containerPort = var.container_port
          hostPort      = var.container_port
          protocol      = "tcp"
        }
      ]
    }
  }

  subnet_ids = var.private_subnet_ids

  load_balancer = {
    service = {
      target_group_arn = aws_lb_target_group.api.arn
      container_name   = "api"
      container_port   = var.container_port
    }
  }

  tags = local.common_tags
}
```

## Version Pinning

- **Terraform version**: Pin with `required_version` using `~>` operator
- **Providers**: Pin in `required_providers` with `~>` for patch flexibility
- **Modules**: Pin with `version` constraint or git tag
- **Lock file**: Commit `.terraform.lock.hcl` to version control

```hcl
terraform {
  required_version = "~> 1.7"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
}
```

## Validation Tools

Run these before committing:

| Tool                 | Command                    | Purpose                                       |
| -------------------- | -------------------------- | --------------------------------------------- |
| `terraform fmt`      | `terraform fmt -recursive` | Auto-format all `.tf` files                   |
| `terraform validate` | `terraform validate`       | Check syntax and internal consistency         |
| `tflint`             | `tflint --recursive`       | Lint for errors, deprecations, best practices |
| `tfsec` / `trivy`    | `trivy config .`           | Security scanning for misconfigurations       |

## Common Tags Pattern

Define tags once in locals, spread to all resources:

```hcl
locals {
  common_tags = {
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "terraform"
    Owner       = var.team
  }
}
```
