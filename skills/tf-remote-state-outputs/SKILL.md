---
name: tf-remote-state-outputs
description: >-
  Extract all Terraform outputs from a repo and generate a REMOTE_STATE_OUTPUTS.md documenting
  them for consumers who reference this project via terraform_remote_state. Use when setting up
  a new Terraform project, after adding/removing outputs, or when you need a quick inventory of
  what a project exports. Works on any Terraform repo with output blocks.
allowed-tools: Read, Grep, Glob, Write, Edit
---

# Extract Terraform Remote State Outputs

Scan all `.tf` files in the repo, extract every `output` block, and generate (or update) a `REMOTE_STATE_OUTPUTS.md` file documenting what this project exports for remote state consumers.

## Instructions

### 1. Identify the Project

- Read `_backend.tf` (or any file containing a `backend` block) to extract:
  - **S3 bucket** (e.g. `rimu-tf-state`)
  - **State key** (e.g. `ritdu-data-governance-eu1-tf-env/terraform.tfstate`)
  - **Project name** — derive from the `app_name` variable default, the state key, or the directory name
- If no backend config exists, use the directory name as the project name and note that backend is not configured.

### 2. Find All Outputs

- Glob for `**/*.tf` (exclude `.terraform/` directories).
- Grep for `^output\s+"` across all matched files to locate every output block.
- For each output found, read the surrounding block to extract:
  - **Output name** (the string in quotes after `output`)
  - **Source file** (the `.tf` filename)
  - **Value expression** (the `value = ...` line)
  - **Description** (the `description = ...` line if present)
  - **Sensitive** flag if present

### 3. Generate Descriptions

For each output:
- If a `description` attribute exists in the output block, use it verbatim.
- Otherwise, infer a concise description from the value expression:
  - `module.main_vpc.vpc_id` → "VPC ID"
  - `aws_kms_key.s3.arn` → "S3 KMS key ARN"
  - `aws_ecs_cluster.main.arn` → "Main ECS cluster ARN"
  - Keep descriptions short (< 80 chars), focusing on what the value represents.

### 4. Write REMOTE_STATE_OUTPUTS.md

Generate the file at the repo root with this structure:

```markdown
# Remote State Outputs

## Outputs Exported by This Project (`<project-name>`)

State location: `s3://<bucket>/<key>`

| Output | Source | Description |
|--------|--------|-------------|
| `output_name` | `source_file.tf` | Description of the output |
```

**Formatting rules:**
- Group outputs by source file — outputs from the same `.tf` file should be adjacent in the table.
- Sort groups by filename alphabetically.
- Within each group, preserve the order outputs appear in the source file.
- Wrap output names in backticks.
- Wrap source filenames in backticks.
- If any outputs are marked `sensitive = true`, append " (sensitive)" to the description.

### 5. Handle Updates

- If `REMOTE_STATE_OUTPUTS.md` already exists, **overwrite it entirely** with the freshly generated content. This file is auto-generated and should always reflect current state.

## Example Output

```markdown
# Remote State Outputs

## Outputs Exported by This Project (`ritdu-data-governance-eu1-tf-base`)

State location: `s3://rimu-tf-state/ritdu-data-governance-tf-base/terraform.tfstate`

| Output | Source | Description |
|--------|--------|-------------|
| `switch_role_administrators_arn` | `switch-role-administrators.tf` | ARN of the administrators IAM role for cross-account access |
| `switch_role_developers_arn` | `switch-role-developers.tf` | ARN of the developers IAM role for cross-account access |
| `kms_s3_arn` | `kms.tf` | S3 KMS key ARN |
| `kms_db_arn` | `kms.tf` | RDS KMS key ARN |
| `main_vpc_id` | `vpc.tf` | VPC ID |
| `main_vpc_cidr` | `vpc.tf` | VPC CIDR block |
```

## Rules

- Only document `output` blocks — ignore variables, locals, data sources, and resources.
- Exclude any files under `.terraform/` directory.
- Do NOT fabricate outputs — every entry must correspond to an actual `output` block in the code.
- If no outputs exist in the repo, write a note saying "This project does not export any outputs."
