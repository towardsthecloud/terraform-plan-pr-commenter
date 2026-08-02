# Terraform Plan PR Commenter

A GitHub Action that posts the output of `terraform plan` as a comment on Pull Requests. This action helps teams review infrastructure changes directly within their PR workflow, making it easier to catch potential issues before applying Terraform changes.

![Terraform Plan PR Comment Example](./images/terraform-plan-pr-comment-example.png)

## Features

- Automatically posts formatted Terraform plan output to PR comments
- Updates existing comments instead of creating duplicates
- Optionally skips posting when there are no changes
- Supports custom headers for better organization in multi-environment setups
- Works with binary plan files for accurate change detection
- This GitHub Action is developed using native JavaScript, so it executes way faster compared to an action build using Docker.

<!-- TIP-LIST:START -->
> [!TIP]
> **Now you can see _what's_ changing in your infrastructure. But what about _how much it will cost_?**
>
> We developed a GitHub App called [CloudBurn](https://cloudburn.io) that automatically analyzes your Terraform plans and adds cost impact analysis right in your PR comments. Catch expensive decisions before they hit production, not weeks later on your AWS bill.
>
> <a href="https://github.com/apps/cloudburn-io"><img alt="Install CloudBurn from GitHub Marketplace" src="https://img.shields.io/badge/Install%20CloudBurn-GitHub%20Marketplace-brightgreen.svg?style=for-the-badge&logo=github"/></a>
>
> <details>
> <summary>💰 <strong>Two-minute setup</strong></summary>
> <br/>
>
> 1. **[Install CloudBurn](https://github.com/apps/cloudburn-io)** on the same repository where you use this action
> 2. **Open a PR** – This action posts the Terraform plan, then CloudBurn reads it and adds a separate comment with cost analysis
>
> **What's included:**
>
> - Monthly cost deltas showing exactly how much your changes will increase or decrease your AWS bill
> - Real-time pricing from AWS Pricing API based on your infrastructure's region
> - Per-resource cost breakdowns with old vs. new monthly costs
> - Free forever for 1 repository with unlimited users
>
> </details>
<!-- TIP-LIST:END -->

## Inputs

| Input               | Description                                                                         | Required | Default               |
| ------------------- | ----------------------------------------------------------------------------------- | -------- | --------------------- |
| `planfile`          | Path to the Terraform plan file to post as comment in the Pull Request              | Yes      | -                     |
| `terraform-cmd`     | Command to execute for calling the Terraform binary                                 | No       | `terraform`           |
| `working-directory` | Directory where the Terraform binary should be called                               | No       | `.`                   |
| `token`             | The GitHub or PAT token to use for posting comments to Pull Requests                | No       | `${{ github.token }}` |
| `header`            | Header to use for the Pull Request comment                                          | No       | -                     |
| `aws-region`        | The AWS region where the infrastructure changes are being applied (e.g., us-east-1) | No       | -                     |

## Outputs

| Output     | Description                                                        |
| ---------- | ------------------------------------------------------------------ |
| `markdown` | The raw markdown output of the `terraform plan` command            |
| `empty`    | Whether the `terraform plan` contains any changes (`true`/`false`) |

## Usage

### Example 1: Direct Usage in Workflow

```yaml
name: Terraform Plan and Comment on PR

on:
  pull_request:
    branches:
      - main

permissions:
  pull-requests: write
  contents: read

jobs:
  plan-and-comment:
    name: Run Terraform Plan and Post PR Comment
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v5

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Terraform Init
        run: terraform init

      - name: Terraform Plan
        run: terraform plan -out=tfplan.binary

      # Add this action to your workflow ↓
      - name: Post Terraform Plan Comment in PR
        uses: towardsthecloud/terraform-plan-pr-commenter@v1
        with:
          planfile: tfplan.binary
          aws-region: us-east-1
```

### Example 2: Reusable Workflow Call

Create a reusable workflow in `.github/workflows/terraform-plan-comment.yml`:

```yaml
name: Reusable Terraform Plan PR Comment

on:
  workflow_call:
    inputs:
      planfile:
        description: 'Path to the Terraform plan file'
        type: string
        required: true
      working-directory:
        description: 'Terraform working directory'
        type: string
        required: true
      aws-region:
        description: 'AWS Region where resources will be deployed'
        type: string

jobs:
  comment-terraform-plan:
    name: Post Terraform Plan as PR Comment
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
      contents: read

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v5

      - name: Download Plan Artifact
        uses: actions/download-artifact@v5
        with:
          name: terraform-plan-artifact
          path: ${{ inputs.working-directory }}

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Terraform Init
        run: terraform init -backend=false
        working-directory: ${{ inputs.working-directory }}

      # Add this action to your workflow ↓
      - name: Post Terraform Plan Comment in PR
        uses: towardsthecloud/terraform-plan-pr-commenter@v1
        with:
          planfile: ${{ inputs.planfile }}
          working-directory: ${{ inputs.working-directory }}
          aws-region: ${{ inputs.aws-region }}
```

Then call this workflow from your main Terraform workflow:

```yaml
name: Terraform Plan with Artifact Upload

on:
  pull_request:
    branches:
      - main

jobs:
  plan-infrastructure:
    name: Generate and Upload Terraform Plan
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v5

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Terraform Init
        run: terraform init
        working-directory: ./infrastructure

      - name: Terraform Plan
        run: terraform plan -out=tfplan.binary
        working-directory: ./infrastructure

      - name: Upload Plan Artifact
        uses: actions/upload-artifact@v5
        with:
          name: terraform-plan-artifact
          path: ./infrastructure/tfplan.binary
          retention-days: 1

  post-plan-comment:
    needs: plan-infrastructure
    uses: ./.github/workflows/terraform-plan-comment.yml
    with:
      planfile: tfplan.binary
      working-directory: ./infrastructure
      aws-region: us-east-1
```

Want to test this out first? Check out the [AWS Terraform Starter Kit](https://github.com/towardsthecloud/aws-terraform-starter-kit) we created. It's a production-ready Terraform template that has a GitHub workflow already configured to use this GitHub Action.

## Permissions

This action requires the following permissions:

```yaml
permissions:
  pull-requests: write  # Required to post comments on PRs
  contents: read        # Required to read repository contents
```

## Usage-based cost assumptions

CloudBurn prices S3, SQS, and SNS from monthly usage you declare, since a plan shows that a bucket or queue exists but never how much traffic it will carry.

Put those numbers in `.cloudburn/usage-assumptions.json` at the root of your repository:

```json
{
  "schemaVersion": 2,
  "resources": {
    "module.storage.aws_s3_bucket.reports": {
      "s3": { "storage": { "standardGbMonth": 500 } }
    }
  }
}
```

Keys under `resources` are the resource addresses Terraform prints in the plan.

When the file is present, this action appends it to the plan comment inside a collapsed **CloudBurn usage assumptions** block, together with the version of the file at the pull request base commit. CloudBurn reads both from the comment and never reads your repository, so editing only the assumptions still produces a cost delta.

Repositories without the file get the same comment as before. A file over 64 KB or with invalid JSON isn't embedded; CloudBurn reports it as a configuration error instead.

The [usage assumptions schema reference](https://cloudburn.io/docs/github-app/usage-assumptions) covers the full schema: supported fields per service, repository-wide and per-resource-type defaults, account usage for graduated pricing tiers, and free-tier handling.

## Documentation

For complete documentation, including advanced configuration options and integration with CloudBurn for cost analysis, visit:

[Full Documentation on CloudBurn.io](https://cloudburn.io/docs/terraform-plan-github-action)

## Author

Maintained by [Towards the Cloud](https://towardsthecloud.com)
