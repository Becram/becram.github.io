---
title: "Terraform, Terragrunt & Atlantis: A Production-Grade IaC Workflow"
date: 2025-09-21
categories:
  - devops
tags:
  - terraform
  - terragrunt
  - atlantis
  - iac
  - aws
---

Infrastructure as Code at scale comes with a familiar set of problems: duplicated module calls across environments, state file collisions, no PR-based review for infra changes, and sprawling root modules that nobody dares touch. This post walks through how I structure Terraform modules, layer Terragrunt on top to eliminate repetition, and wire Atlantis into GitHub pull requests so that infrastructure changes go through the same review workflow as application code.

---

## The Problem with Vanilla Terraform at Scale

A single Terraform workspace works fine for a small project. As soon as you have multiple environments (dev, staging, prod) and multiple AWS accounts, you hit the same walls:

- **DRY violation** — you copy `backend.tf`, `provider.tf`, and module calls into every environment directory and then drift slowly apart
- **State isolation** — one mistake in a shared workspace can destroy prod
- **No guardrails** — anyone with credentials can run `terraform apply` locally
- **No audit trail** — you cannot tell from git who applied what and when

Terragrunt solves the first two. Atlantis solves the last two.

---

## Part 1: Terraform Module Structure

A clean module structure separates *reusable modules* from *environment-specific root modules*.

```
infrastructure/
├── modules/                   # reusable building blocks
│   ├── vpc/
│   ├── eks/
│   ├── rds/
│   └── ecs-service/
└── live/                      # environment root modules (consumed by Terragrunt)
    ├── _envcommon/            # shared variable definitions
    ├── dev/
    │   ├── account.hcl
    │   └── us-east-1/
    │       ├── region.hcl
    │       ├── vpc/
    │       ├── eks/
    │       └── rds/
    ├── staging/
    └── prod/
```

### Writing a reusable module

Every module under `modules/` follows the same layout:

```
modules/ecs-service/
├── main.tf
├── variables.tf
├── outputs.tf
└── versions.tf
```

**`versions.tf`** — pin providers so modules do not break on consumer upgrades:

```hcl
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

**`variables.tf`** — explicit types and descriptions, no defaults for values that differ per environment:

```hcl
variable "cluster_name" {
  type        = string
  description = "Name of the ECS cluster to deploy the service into"
}

variable "service_name" {
  type        = string
  description = "ECS service name, used as a prefix for all resources"
}

variable "container_image" {
  type        = string
  description = "Full ECR image URI including tag"
}

variable "desired_count" {
  type        = number
  description = "Desired number of running tasks"
  default     = 2
}

variable "cpu" {
  type        = number
  description = "Task CPU units (256, 512, 1024, 2048, 4096)"
  default     = 256
}

variable "memory" {
  type        = number
  description = "Task memory in MiB"
  default     = 512
}

variable "environment_variables" {
  type        = map(string)
  description = "Non-secret environment variables injected into the container"
  default     = {}
}

variable "secrets" {
  type        = map(string)
  description = "Map of env var name to SSM Parameter Store ARN"
  default     = {}
}

variable "target_group_arn" {
  type        = string
  description = "ALB target group ARN for load balancer attachment"
}

variable "subnets" {
  type        = list(string)
  description = "Subnet IDs the ECS tasks run in"
}

variable "security_groups" {
  type        = list(string)
  description = "Security group IDs attached to the ECS tasks"
}

variable "tags" {
  type        = map(string)
  description = "Tags applied to all resources"
  default     = {}
}
```

**`main.tf`** — the actual resources, composed from inputs only:

```hcl
resource "aws_ecs_task_definition" "this" {
  family                   = var.service_name
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = var.cpu
  memory                   = var.memory
  execution_role_arn       = aws_iam_role.execution.arn
  task_role_arn            = aws_iam_role.task.arn

  container_definitions = jsonencode([{
    name  = var.service_name
    image = var.container_image

    environment = [
      for k, v in var.environment_variables : { name = k, value = v }
    ]

    secrets = [
      for k, v in var.secrets : { name = k, valueFrom = v }
    ]

    portMappings = [{
      containerPort = 8080
      protocol      = "tcp"
    }]

    logConfiguration = {
      logDriver = "awslogs"
      options = {
        "awslogs-group"         = "/ecs/${var.service_name}"
        "awslogs-region"        = data.aws_region.current.name
        "awslogs-stream-prefix" = "ecs"
      }
    }
  }])

  tags = var.tags
}

resource "aws_ecs_service" "this" {
  name            = var.service_name
  cluster         = var.cluster_name
  task_definition = aws_ecs_task_definition.this.arn
  desired_count   = var.desired_count
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = var.subnets
    security_groups  = var.security_groups
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = var.target_group_arn
    container_name   = var.service_name
    container_port   = 8080
  }

  lifecycle {
    ignore_changes = [desired_count]
  }

  tags = var.tags
}

resource "aws_cloudwatch_log_group" "this" {
  name              = "/ecs/${var.service_name}"
  retention_in_days = 30
  tags              = var.tags
}

data "aws_region" "current" {}
```

**`outputs.tf`** — expose what downstream modules need:

```hcl
output "service_name" {
  value = aws_ecs_service.this.name
}

output "task_definition_arn" {
  value = aws_ecs_task_definition.this.arn
}

output "log_group_name" {
  value = aws_cloudwatch_log_group.this.name
}
```

### Module versioning with git tags

Pin modules by git tag, not by path, so environments can upgrade independently:

```hcl
module "ecs_service" {
  source = "git::https://github.com/your-org/terraform-modules.git//modules/ecs-service?ref=v1.4.2"
  # ...
}
```

---

## Part 2: Terragrunt — DRY Environments

Terragrunt wraps Terraform and solves three things: backend configuration inheritance, provider inheritance, and dependency graphs between stacks.

### Root `terragrunt.hcl`

Place this at the top of `live/`:

```hcl
# live/terragrunt.hcl

locals {
  account_vars = read_terragrunt_config(find_in_parent_folders("account.hcl"))
  region_vars  = read_terragrunt_config(find_in_parent_folders("region.hcl"))

  account_id   = local.account_vars.locals.account_id
  account_name = local.account_vars.locals.account_name
  aws_region   = local.region_vars.locals.aws_region
}

# Generate backend config for every child module automatically
generate "backend" {
  path      = "backend.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<EOF
terraform {
  backend "s3" {
    bucket         = "my-org-terraform-state-${local.account_id}"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "${local.aws_region}"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
EOF
}

# Generate provider block for every child module
generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<EOF
provider "aws" {
  region = "${local.aws_region}"

  assume_role {
    role_arn = "arn:aws:iam::${local.account_id}:role/TerraformDeployRole"
  }

  default_tags {
    tags = {
      ManagedBy   = "terraform"
      Environment = "${local.account_name}"
    }
  }
}
EOF
}
```

### Account and region files

```hcl
# live/prod/account.hcl
locals {
  account_id   = "123456789012"
  account_name = "prod"
}
```

```hcl
# live/prod/us-east-1/region.hcl
locals {
  aws_region = "us-east-1"
}
```

### A leaf module's `terragrunt.hcl`

```hcl
# live/prod/us-east-1/ecs-service/api/terragrunt.hcl

include "root" {
  path = find_in_parent_folders()
}

include "envcommon" {
  path   = "${dirname(find_in_parent_folders())}/_envcommon/ecs-service.hcl"
  expose = true
}

# Declare dependency on ECS cluster
dependency "cluster" {
  config_path = "../cluster"
  mock_outputs = {
    cluster_name = "mock-cluster"
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

dependency "vpc" {
  config_path = "../../vpc"
  mock_outputs = {
    private_subnet_ids    = ["subnet-00000000"]
    app_security_group_id = "sg-00000000"
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

dependency "alb" {
  config_path = "../alb"
  mock_outputs = {
    api_target_group_arn = "arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/mock/abc"
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

terraform {
  source = "git::https://github.com/your-org/terraform-modules.git//modules/ecs-service?ref=v1.4.2"
}

inputs = merge(
  include.envcommon.locals.common_inputs,
  {
    cluster_name     = dependency.cluster.outputs.cluster_name
    service_name     = "api"
    container_image  = "123456789012.dkr.ecr.us-east-1.amazonaws.com/api:latest"
    desired_count    = 3
    cpu              = 512
    memory           = 1024
    target_group_arn = dependency.alb.outputs.api_target_group_arn
    subnets          = dependency.vpc.outputs.private_subnet_ids
    security_groups  = [dependency.vpc.outputs.app_security_group_id]

    environment_variables = {
      APP_ENV    = "production"
      LOG_LEVEL  = "info"
    }

    secrets = {
      DATABASE_URL = "arn:aws:ssm:us-east-1:123456789012:parameter/prod/api/database_url"
      API_KEY      = "arn:aws:ssm:us-east-1:123456789012:parameter/prod/api/api_key"
    }
  }
)
```

### Shared envcommon

```hcl
# live/_envcommon/ecs-service.hcl
locals {
  account_vars = read_terragrunt_config(find_in_parent_folders("account.hcl"))

  common_inputs = {
    tags = {
      Team       = "platform"
      CostCenter = "engineering"
    }
  }
}
```

### Running across all environments

```bash
# Plan everything under prod
terragrunt run-all plan --terragrunt-working-dir live/prod

# Apply a single stack
terragrunt apply --terragrunt-working-dir live/prod/us-east-1/ecs-service/api

# Destroy in dependency order (reverse)
terragrunt run-all destroy --terragrunt-working-dir live/dev
```

Terragrunt builds a directed acyclic graph from `dependency` blocks and applies stacks in the correct order automatically.

---

## Part 3: Atlantis — PR-Driven Infrastructure

Atlantis is a self-hosted Go service that listens to GitHub webhooks and runs `terraform plan` / `terraform apply` in response to pull request comments. Infrastructure changes go through the same review process as code.

### How it works

```
Developer opens PR
        │
        ▼
GitHub webhook → Atlantis
        │
        ▼
atlantis plan   ← posted as PR comment
        │
        ▼
Reviewer approves PR
        │
        ▼
atlantis apply  ← comment on PR
        │
        ▼
Terraform apply runs, output posted back to PR
        │
        ▼
PR merged
```

### Deploying Atlantis on ECS Fargate

```hcl
# modules/atlantis/main.tf (simplified)

resource "aws_ecs_task_definition" "atlantis" {
  family                   = "atlantis"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = 1024
  memory                   = 2048
  execution_role_arn       = aws_iam_role.execution.arn
  task_role_arn            = aws_iam_role.atlantis_task.arn

  container_definitions = jsonencode([{
    name  = "atlantis"
    image = "ghcr.io/runatlantis/atlantis:v0.28.0"

    environment = [
      { name = "ATLANTIS_REPO_ALLOWLIST",    value = "github.com/your-org/*" },
      { name = "ATLANTIS_GH_USER",           value = "atlantis-bot" },
      { name = "ATLANTIS_REPO_CONFIG_FILE",  value = "/atlantis.yaml" },
      { name = "ATLANTIS_TERRAFORM_VERSION", value = "1.7.0" },
      { name = "ATLANTIS_PORT",              value = "4141" },
    ]

    secrets = [
      { name = "ATLANTIS_GH_TOKEN",          valueFrom = aws_ssm_parameter.gh_token.arn },
      { name = "ATLANTIS_GH_WEBHOOK_SECRET", valueFrom = aws_ssm_parameter.webhook_secret.arn },
      { name = "ATLANTIS_ATLANTIS_URL",      valueFrom = aws_ssm_parameter.atlantis_url.arn },
    ]

    portMappings = [{ containerPort = 4141 }]

    mountPoints = [{
      sourceVolume  = "atlantis-data"
      containerPath = "/atlantis"
    }]
  }])

  volume {
    name = "atlantis-data"
    efs_volume_configuration {
      file_system_id = aws_efs_file_system.atlantis.id
      root_directory = "/"
    }
  }
}

# Atlantis needs broad IAM permissions to manage infrastructure
resource "aws_iam_role_policy" "atlantis_task" {
  name = "atlantis-task-policy"
  role = aws_iam_role.atlantis_task.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["sts:AssumeRole"]
      Resource = [
        "arn:aws:iam::*:role/TerraformDeployRole"
      ]
    }]
  })
}
```

Each AWS account has a `TerraformDeployRole` that trusts the Atlantis task role to assume it — no static credentials needed.

### `atlantis.yaml` — repo-level config

This file lives at the root of your `live/` repo and tells Atlantis how to map directories to projects:

```yaml
version: 3

automerge: false
delete_source_branch_on_merge: false

repos:
  - id: github.com/your-org/infrastructure
    apply_requirements:
      - approved
      - mergeable
    workflow: terragrunt
    allowed_overrides:
      - workflow
      - apply_requirements
    allow_custom_workflows: true

workflows:
  terragrunt:
    plan:
      steps:
        - env:
            name: TERRAGRUNT_TFPATH
            command: "which terraform"
        - run: terragrunt plan -no-color -out=$PLANFILE
    apply:
      steps:
        - run: terragrunt apply -no-color $PLANFILE

projects:
  - name: prod-us-east-1-vpc
    dir: live/prod/us-east-1/vpc
    workspace: default
    workflow: terragrunt
    autoplan:
      when_modified:
        - "*.hcl"
        - "*.tf"
      enabled: true

  - name: prod-us-east-1-eks
    dir: live/prod/us-east-1/eks
    workspace: default
    workflow: terragrunt
    autoplan:
      when_modified:
        - "*.hcl"
        - "*.tf"
      enabled: true

  - name: prod-us-east-1-api
    dir: live/prod/us-east-1/ecs-service/api
    workspace: default
    workflow: terragrunt
    autoplan:
      when_modified:
        - "*.hcl"
        - "*.tf"
      enabled: true
```

With `autoplan` enabled, Atlantis runs `terragrunt plan` automatically when a PR modifies files in that directory. The plan output is posted as a PR comment.

### The PR workflow in practice

```
# Open a PR that changes the ECS service desired_count from 2 to 3

atlantis plan -p prod-us-east-1-api   ← auto or manual trigger

# Atlantis replies:
# Plan: 0 to add, 1 to change, 0 to destroy.
# ~ aws_ecs_service.this
#     desired_count: 2 -> 3

# After approval:
atlantis apply -p prod-us-east-1-api

# Atlantis replies:
# Apply complete! Resources: 0 added, 1 changed, 0 destroyed.
```

All plan and apply output is stored as PR comments — a full audit trail with who triggered it and when.

### Locking

Atlantis locks a project directory while a plan is in progress and until apply completes or the lock is discarded. This prevents two engineers from applying conflicting changes simultaneously:

```
atlantis unlock             # discard lock for all projects in the PR
atlantis unlock -p prod-us-east-1-api   # discard lock for one project
```

---

## Part 4: State Management Best Practices

### Bootstrap the S3 backend before anything else

```hcl
# bootstrap/main.tf — apply this once manually before Terragrunt takes over

resource "aws_s3_bucket" "state" {
  bucket = "my-org-terraform-state-${var.account_id}"
}

resource "aws_s3_bucket_versioning" "state" {
  bucket = aws_s3_bucket.state.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "state" {
  bucket = aws_s3_bucket.state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "state" {
  bucket                  = aws_s3_bucket.state.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_dynamodb_table" "locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

### State isolation strategy

| Level | State file path | What it contains |
|-------|----------------|------------------|
| Network | `prod/us-east-1/vpc/terraform.tfstate` | VPC, subnets, route tables |
| Cluster | `prod/us-east-1/eks/terraform.tfstate` | EKS cluster, node groups |
| Service | `prod/us-east-1/ecs-service/api/terraform.tfstate` | ECS service, task def, IAM |
| Data | `prod/us-east-1/rds/terraform.tfstate` | RDS instance, parameter groups |

Separate state files mean a botched apply in one service cannot corrupt another.

---

## Part 5: Security Considerations

### Never use long-lived credentials

Atlantis assumes an IAM role per account using `sts:AssumeRole`. No AWS access keys live in environment variables or SSM — the task role is the only identity.

```hcl
# In each account, create a role Atlantis can assume
resource "aws_iam_role" "terraform_deploy" {
  name = "TerraformDeployRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = {
        AWS = "arn:aws:iam::ATLANTIS_ACCOUNT_ID:role/atlantis-task"
      }
      Action    = "sts:AssumeRole"
    }]
  })
}
```

### Enforce `apply_requirements` in Atlantis

```yaml
apply_requirements:
  - approved       # at least one approval required
  - mergeable      # no merge conflicts, all status checks green
```

This ensures no one can apply infrastructure changes without a code review.

### Webhook secret validation

Atlantis validates the HMAC signature on every incoming GitHub webhook. Store the secret in SSM Parameter Store and inject it as an ECS secret — never in plain text.

---

## Putting It All Together

The complete workflow for shipping an infrastructure change:

1. **Branch** off `main`, edit the relevant `terragrunt.hcl` under `live/`
2. **Open a PR** — Atlantis detects the changed directory via `autoplan` and runs `terragrunt plan`
3. **Review** — teammates review both the code diff and the Terraform plan output in the PR
4. **Approve** the PR
5. **Comment** `atlantis apply -p <project>` — Atlantis runs the apply and posts output
6. **Merge** the PR — git history now records the change with context

No one ever runs `terraform apply` from a laptop. Every change is planned, reviewed, applied, and logged through pull requests.

---

## Conclusion

Vanilla Terraform works until you have more than one environment and more than one engineer. Terragrunt eliminates the boilerplate of managing backends, providers, and repeated module calls across environments. Atlantis closes the loop by making pull requests the single point of control for all infrastructure changes — adding the same review discipline to infra that you already have for application code.

The combination gives you: immutable module versions, isolated state per stack, dependency-ordered applies, and a full audit trail in GitHub with zero manual `terraform apply` runs outside of CI.
