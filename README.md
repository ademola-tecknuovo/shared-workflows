# shared-workflows

A central repository of reusable GitHub Actions workflows for Terraform deployments on AWS. Rather than duplicating pipeline logic across every project repository, teams can call these workflows directly and pass in their own configuration.

---

## How it works

Instead of each repository containing a full pipeline, a calling repository contains a thin workflow file that references this repo. The heavy lifting — Terraform init, plan, apply, and AWS authentication — is handled here.

```
Your Project Repo                    This Repo (shared-workflows)
─────────────────────                ─────────────────────────────
terraform.yml (caller)    ────────►  terraform-reusable.yml
  │                                    │
  │  passes in:                        │  runs:
  │  - aws_region                      │  - Terraform Init
  │  - tf_version                      │  - Terraform Format Check
  │  - lambda_function_name            │  - Terraform Validate
  │  - sns_topic_name                  │  - Terraform Plan
  │  - log_retention_days              │  - Terraform Apply (main only)
  │                                    │
  │  secrets:                          │  authenticates via:
  │  - aws_role_arn        ──────────► │  - OIDC (short-lived AWS token)
  │  - bucket_name                     │    No static keys stored
  └  - notification_email              └
```

---

## Pipeline flow

```
Push / PR / Manual trigger
         │
         ▼
┌─────────────────────┐
│   Terraform Plan    │  ◄── runs on every trigger (push, PR, manual)
│                     │
│  • Init             │
│  • Format check     │
│  • Validate         │
│  • Plan             │
└────────┬────────────┘
         │
         │  only continues if:
         │  branch = main
         │  event = push or manual
         │
         ▼
┌─────────────────────┐
│   Terraform Apply   │  ◄── deploys to AWS
│                     │
│  • Init             │
│  • Apply            │
└─────────────────────┘
```

> **Note:** Pull requests only run the Plan job. The Apply job is intentionally blocked on PRs — it only runs once code has been merged or pushed to `main`.

---

## AWS Authentication (OIDC)

This workflow uses **OpenID Connect (OIDC)** rather than long-lived AWS access keys. When the pipeline runs, GitHub requests a short-lived token from AWS that expires when the job finishes (roughly 15 minutes). No permanent credentials are stored anywhere.

```
GitHub Actions runner
        │
        │  1. requests token
        ▼
GitHub OIDC Provider
        │
        │  2. token sent to AWS
        ▼
AWS IAM (verifies token)
        │
        │  3. issues temporary credentials (~15 mins)
        ▼
Pipeline assumes IAM Role
        │
        │  4. runs Terraform with temporary credentials
        ▼
AWS Resources deployed
```

For this to work, each calling repository must have an IAM role set up in AWS with a trust relationship to GitHub. See the **Setup** section below.

---

## Available workflows

| Workflow | Description |
|---|---|
| `terraform-reusable.yml` | Generic Terraform plan and apply pipeline for AWS |

---

## Setup

### 1. AWS — Create an OIDC Identity Provider (one-time per AWS account)

1. Go to **IAM → Identity providers → Add provider**
2. Select **OpenID Connect**
3. Provider URL: `https://token.actions.githubusercontent.com` → click **Get thumbprint**
4. Audience: `sts.amazonaws.com`
5. Click **Add provider**

### 2. AWS — Create an IAM Role for your project repo

1. Go to **IAM → Roles → Create role**
2. Select **Web identity**
3. Identity provider: `token.actions.githubusercontent.com`
4. Audience: `sts.amazonaws.com`
5. Set the GitHub organisation, repository, and branch to scope the role to your specific repo
6. Attach the permissions your Terraform project requires
7. Name the role (e.g. `github-actions-your-project-role`)
8. Copy the **Role ARN** — you will need this in the next step

### 3. GitHub — Add secrets to your project repo

In your project repository, go to **Settings → Secrets and variables → Actions** and add:

| Secret name | Value |
|---|---|
| `ADMIN_ROLE_ARN` | The IAM Role ARN from step 2 |
| `BUCKET_NAME` | Your S3 bucket name |
| `NOTIFICATION_EMAIL` | Email address for SNS notifications |

### 4. GitHub — Create the caller workflow

In your project repository, create `.github/workflows/terraform.yml` with the following content, adjusting the `with` values for your project:

```yaml
name: Terraform CI/CD

on:
  pull_request:
    branches:
      - main
  push:
    branches:
      - main
  workflow_dispatch:

jobs:

  terraform:
    name: Terraform
    uses: ademola-tecknuovo/shared-workflows/.github/workflows/terraform-reusable.yml@main
    permissions:
      id-token: write
      contents: read
    with:
      aws_region: eu-west-2
      tf_version: "1.10.0"
      lambda_function_name: "your-lambda-function-name"
      sns_topic_name: "your-sns-topic-name"
      log_retention_days: "14"
    secrets:
      aws_role_arn: ${{ secrets.ADMIN_ROLE_ARN }}
      bucket_name: ${{ secrets.BUCKET_NAME }}
      notification_email: ${{ secrets.NOTIFICATION_EMAIL }}
```

---

## Inputs reference

| Input | Required | Default | Description |
|---|---|---|---|
| `aws_region` | Yes | — | AWS region to deploy to (e.g. `eu-west-2`) |
| `tf_version` | No | `1.10.0` | Terraform version to use |
| `working_directory` | No | `.` | Path to Terraform files if not in repo root |
| `lambda_function_name` | No | `""` | Name of the Lambda function |
| `sns_topic_name` | No | `""` | Name of the SNS topic |
| `log_retention_days` | No | `14` | CloudWatch log retention period in days |

## Secrets reference

| Secret | Required | Description |
|---|---|---|
| `aws_role_arn` | Yes | ARN of the IAM role to assume via OIDC |
| `bucket_name` | No | S3 bucket name |
| `notification_email` | No | Email address for SNS notifications |

---

## Repo structure

```
shared-workflows/
└── .github/
    └── workflows/
        └── terraform-reusable.yml   # reusable Terraform pipeline
```
