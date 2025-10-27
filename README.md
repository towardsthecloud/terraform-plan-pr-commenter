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
        default: 'tfplan.binary'
      working-directory:
        description: 'Terraform working directory'
        type: string
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
          name: 'terraform-plan'
          path: ${{ inputs.working-directory }}

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Terraform Init
        run: terraform init -backend=false
        working-directory: ${{ inputs.working-directory }}

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
          name: terraform-plan
          path: ./infrastructure/tfplan.binary
          retention-days: 5

  post-plan-comment:
    needs: plan-infrastructure
    uses: ./.github/workflows/terraform-plan-comment.yml
    with:
      planfile: tfplan.binary
      working-directory: ./infrastructure
      aws-region: us-east-1
```

### Example 3: Multi-Environment Setup

```yaml
name: Terraform Plan for Multiple Environments

on:
  pull_request:
    branches:
      - main

permissions:
  pull-requests: write
  contents: read

jobs:
  plan-production-env:
    name: Plan and Comment - Production (us-east-1)
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v5

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Terraform Init
        run: terraform init
        working-directory: ./environments/production

      - name: Terraform Plan
        run: terraform plan -out=tfplan.binary
        working-directory: ./environments/production

      - name: Post Production Plan Comment
        uses: towardsthecloud/terraform-plan-pr-commenter@v1
        with:
          planfile: tfplan.binary
          working-directory: ./environments/production
          aws-region: us-east-1

  plan-staging-env:
    name: Plan and Comment - Staging (us-west-2)
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v5

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Terraform Init
        run: terraform init
        working-directory: ./environments/staging

      - name: Terraform Plan
        run: terraform plan -out=tfplan.binary
        working-directory: ./environments/staging

      - name: Post Staging Plan Comment
        uses: towardsthecloud/terraform-plan-pr-commenter@v1
        with:
          planfile: tfplan.binary
          working-directory: ./environments/staging
          aws-region: us-west-2
```

## Permissions

This action requires the following permissions:

```yaml
permissions:
  pull-requests: write  # Required to post comments on PRs
  contents: read        # Required to read repository contents
```

## Notes

- The action will update the same comment on subsequent pushes to the PR, avoiding comment spam
- The `planfile` should be a binary plan file generated with `terraform plan -out=<filename>`
- Make sure to run `terraform init` before using this action, as it needs to read the plan file

## Author

Maintained by [Towards the Cloud](https://github.com/towardsthecloud)
