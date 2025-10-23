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

| Input               | Description                                                            | Required | Default               |
| ------------------- | ---------------------------------------------------------------------- | -------- | --------------------- |
| `planfile`          | Path to the Terraform plan file to post as comment in the Pull Request | Yes      | -                     |
| `token`             | The GitHub or PAT token to use for posting comments to Pull Requests   | Yes      | `${{ github.token }}` |
| `header`            | Header to use for the Pull Request comment                             | Yes      | `📝 Terraform Plan`    |
| `terraform-cmd`     | Command to execute for calling the Terraform binary                    | Yes      | `terraform`           |
| `working-directory` | Directory where the Terraform binary should be called                  | Yes      | `.`                   |

## Outputs

| Output     | Description                                                        |
| ---------- | ------------------------------------------------------------------ |
| `markdown` | The raw markdown output of the `terraform plan` command            |
| `empty`    | Whether the `terraform plan` contains any changes (`true`/`false`) |

## Usage

### Example 1: Direct Usage in Workflow

```yaml
name: Terraform Plan

on:
  pull_request:
    branches:
      - main

permissions:
  pull-requests: write
  contents: read

jobs:
  terraform-plan:
    name: Terraform Plan
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
          header: |
            ## Terraform Plan for production in us-east-1
```

### Example 2: Reusable Workflow Call

Create a reusable workflow in `.github/workflows/terraform-plan-comment.yml`:

```yaml
name: Terraform Plan Comment

on:
  workflow_call:
    inputs:
      aws-region:
        description: 'AWS Region where resources will be deployed'
        type: string
      environment:
        description: 'Environment name (e.g., production, staging)'
        type: string
      working-directory:
        description: 'Terraform working directory'
        type: string

jobs:
  post-plan-comment:
    name: Post Terraform Plan Comment
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
          planfile: tfplan.binary
          working-directory: ${{ inputs.working-directory }}
          header: |
            ## Terraform Plan for ${{ inputs.environment }} in ${{ inputs.aws-region }}
```

Then call this workflow from your main Terraform workflow:

```yaml
name: Terraform CI

on:
  pull_request:
    branches:
      - main

jobs:
  terraform-plan:
    name: Terraform Plan
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

  comment-plan:
    needs: terraform-plan
    uses: ./.github/workflows/terraform-plan-comment.yml
    with:
      aws-region: us-east-1
      environment: production
      working-directory: ./infrastructure
```

### Example 3: Multi-Environment Setup

```yaml
name: Multi-Environment Terraform Plan

on:
  pull_request:
    branches:
      - main

permissions:
  pull-requests: write
  contents: read

jobs:
  plan-production:
    name: Plan Production
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

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
          header: |
            ## :rocket: Production Environment (us-east-1)
          skip-empty: true

  plan-staging:
    name: Plan Staging
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

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
          header: |
            ## :test_tube: Staging Environment (us-west-2)
          skip-empty: true
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
- When `skip-empty` is set to `true`, no comment will be posted if the plan shows no changes
- The `planfile` should be a binary plan file generated with `terraform plan -out=<filename>`
- Make sure to run `terraform init` before using this action, as it needs to read the plan file

## Author

Maintained by [Towards the Cloud](https://github.com/towardsthecloud)
