# Reusable Workflows Documentation

This repository provides a set of reusable GitHub Actions / Gitea Actions workflows that can be called from other repositories to standardize CI/CD pipelines.

## Overview

The reusable workflows are organized into two categories:

### Terraform Workflows
- **reusable-terraform-check.yml** - Format, validate, and lint Terraform code
- **reusable-terraform-plan.yml** - Plan infrastructure changes with MinIO backend
- **reusable-terraform-apply.yml** - Apply infrastructure changes
- **reusable-terraform-destroy.yml** - Destroy infrastructure
- **reusable-terraform-init.yml** - Initialize Terraform with MinIO backend

### Ansible Workflows
- **reusable-ansible-check.yml** - Lint and syntax check Ansible playbooks
- **reusable-ansible-deploy.yml** - Deploy infrastructure using Ansible

---

## Usage Guide

### For Terraform Workflows

#### 1. Terraform Check (PR Validation)

Call this in a PR workflow to validate Terraform code:

```yaml
name: "Terraform Check"

on:
  pull_request:
  workflow_dispatch:

jobs:
  check:
    uses: <owner>/<repo>/.gitea/workflows/reusable-terraform-check.yml@main
    with:
      terraform_dir: "terraform"  # optional, defaults to "terraform"
```

#### 2. Terraform Plan

Trigger a Terraform plan for a specific environment:

```yaml
name: "Terraform Plan"

on:
  workflow_dispatch:
    inputs:
      environment:
        description: "Environment (dev/prod)"
        required: true
        type: string
      vm_name:
        description: "VM name (e.g., dev01, prod01)"
        required: true

jobs:
  plan:
    uses: <owner>/<repo>/.gitea/workflows/reusable-terraform-plan.yml@main
    with:
      environment: ${{ github.event.inputs.environment }}
      vm_name: ${{ github.event.inputs.vm_name }}
    secrets:
      MINIO_ENDPOINT: ${{ secrets.MINIO_ENDPOINT }}
      MINIO_ACCESS_KEY: ${{ secrets.MINIO_ACCESS_KEY }}
      MINIO_SECRET_KEY: ${{ secrets.MINIO_SECRET_KEY }}
      LXD_ADDRESS: ${{ secrets.LXD_ADDRESS }}
      ANSIBLE_SSH_PUBLIC_KEY: ${{ secrets.ANSIBLE_SSH_PUBLIC_KEY }}
      LXD_TRUST_PASSWORD: ${{ secrets.LXD_TRUST_PASSWORD }}
```

#### 3. Terraform Apply

Apply Terraform changes:

```yaml
name: "Terraform Apply"

on:
  workflow_dispatch:
    inputs:
      environment:
        description: "Environment (dev/prod)"
        required: true
        type: string
      vm_name:
        description: "VM name (e.g., dev01, prod01)"
        required: true

jobs:
  apply:
    uses: <owner>/<repo>/.gitea/workflows/reusable-terraform-apply.yml@main
    with:
      environment: ${{ github.event.inputs.environment }}
      vm_name: ${{ github.event.inputs.vm_name }}
    secrets:
      MINIO_ENDPOINT: ${{ secrets.MINIO_ENDPOINT }}
      MINIO_ACCESS_KEY: ${{ secrets.MINIO_ACCESS_KEY }}
      MINIO_SECRET_KEY: ${{ secrets.MINIO_SECRET_KEY }}
      LXD_ADDRESS: ${{ secrets.LXD_ADDRESS }}
      ANSIBLE_SSH_PUBLIC_KEY: ${{ secrets.ANSIBLE_SSH_PUBLIC_KEY }}
      LXD_TRUST_PASSWORD: ${{ secrets.LXD_TRUST_PASSWORD }}
      LXD_CLIENT_CERT: ${{ secrets.LXD_CLIENT_CERT }}
      LXD_CLIENT_KEY: ${{ secrets.LXD_CLIENT_KEY }}
```

#### 4. Terraform Destroy

Destroy infrastructure (with confirmation):

```yaml
name: "Terraform Destroy"

on:
  workflow_dispatch:
    inputs:
      environment:
        description: "Environment (dev/prod)"
        required: true
        type: string
      vm_name:
        description: "VM name (e.g., dev01, prod01)"
        required: true
      confirm_destroy:
        description: "Type 'yes' to confirm destroy"
        required: true

jobs:
  destroy:
    uses: <owner>/<repo>/.gitea/workflows/reusable-terraform-destroy.yml@main
    with:
      environment: ${{ github.event.inputs.environment }}
      vm_name: ${{ github.event.inputs.vm_name }}
      confirm_destroy: ${{ github.event.inputs.confirm_destroy }}
    secrets:
      MINIO_ENDPOINT: ${{ secrets.MINIO_ENDPOINT }}
      MINIO_ACCESS_KEY: ${{ secrets.MINIO_ACCESS_KEY }}
      MINIO_SECRET_KEY: ${{ secrets.MINIO_SECRET_KEY }}
      LXD_ADDRESS: ${{ secrets.LXD_ADDRESS }}
      ANSIBLE_SSH_PUBLIC_KEY: ${{ secrets.ANSIBLE_SSH_PUBLIC_KEY }}
      LXD_TRUST_PASSWORD: ${{ secrets.LXD_TRUST_PASSWORD }}
      LXD_CLIENT_CERT: ${{ secrets.LXD_CLIENT_CERT }}
      LXD_CLIENT_KEY: ${{ secrets.LXD_CLIENT_KEY }}
```

---

### For Ansible Workflows

#### 1. Ansible Check (PR Validation)

Call this in a PR workflow to validate Ansible code:

```yaml
name: "Ansible Check"

on:
  pull_request:
  workflow_dispatch:

jobs:
  check:
    uses: <owner>/<repo>/.gitea/workflows/reusable-ansible-check.yml@main
    with:
      ansible_playbook_path: "ansible/playbook.yml"  # optional
```

#### 2. Ansible Deploy

Deploy using Ansible playbook:

```yaml
name: "Ansible Deploy"

on:
  workflow_dispatch:
    inputs:
      environment:
        description: "Environment (dev/prod)"
        required: true
        type: string
      vm_name:
        description: "VM name (e.g., dev01, prod01)"
        required: true

jobs:
  deploy:
    uses: <owner>/<repo>/.gitea/workflows/reusable-ansible-deploy.yml@main
    with:
      environment: ${{ github.event.inputs.environment }}
      vm_name: ${{ github.event.inputs.vm_name }}
      ansible_playbook_path: "ansible/playbook.yml"
      ssh_wait_timeout_seconds: 200
    secrets:
      MINIO_ENDPOINT: ${{ secrets.MINIO_ENDPOINT }}
      MINIO_ACCESS_KEY: ${{ secrets.MINIO_ACCESS_KEY }}
      MINIO_SECRET_KEY: ${{ secrets.MINIO_SECRET_KEY }}
      LXD_ADDRESS: ${{ secrets.LXD_ADDRESS }}
      ANSIBLE_SSH_PUBLIC_KEY: ${{ secrets.ANSIBLE_SSH_PUBLIC_KEY }}
      ANSIBLE_SSH_PRIVATE_KEY: ${{ secrets.ANSIBLE_SSH_PRIVATE_KEY }}
```

---

## Required Secrets

When using these reusable workflows, ensure the following secrets are configured in your repository:

### Terraform Workflows
- `MINIO_ENDPOINT` - MinIO S3 endpoint for Terraform state
- `MINIO_ACCESS_KEY` - MinIO access key
- `MINIO_SECRET_KEY` - MinIO secret key
- `LXD_ADDRESS` - LXD server address
- `ANSIBLE_SSH_PUBLIC_KEY` - Ansible SSH public key
- `LXD_TRUST_PASSWORD` - LXD trust password
- `LXD_CLIENT_CERT` - (Optional) LXD client certificate
- `LXD_CLIENT_KEY` - (Optional) LXD client key

### Ansible Workflows
- `MINIO_ENDPOINT` - MinIO S3 endpoint
- `MINIO_ACCESS_KEY` - MinIO access key
- `MINIO_SECRET_KEY` - MinIO secret key
- `LXD_ADDRESS` - LXD server address
- `ANSIBLE_SSH_PUBLIC_KEY` - Ansible SSH public key
- `ANSIBLE_SSH_PRIVATE_KEY` - Ansible SSH private key

---

## Repository Structure Requirements

When using these workflows, ensure your repository follows this structure:

```
your-repo/
├── terraform/
│   ├── env/
│   │   ├── dev/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   ├── providers.tf
│   │   │   └── dev01.tfvars
│   │   └── prod/
│   │       ├── main.tf
│   │       ├── variables.tf
│   │       ├── outputs.tf
│   │       ├── providers.tf
│   │       └── prod01.tfvars
│   └── modules/
│       └── (your modules)
├── ansible/
│   ├── playbook.yml
│   ├── inventory.yml
│   └── roles/
└── .gitea/workflows/
    ├── terraform-check.yml
    ├── terraform-plan.yml
    ├── terraform-apply.yml
    ├── terraform-destroy.yml
    ├── ansible-check.yml
    ├── ansible-deploy.yml
    └── reusable-*.yml
```

---

## Input Parameters

### Terraform Workflows

| Input | Required | Type | Description |
|-------|----------|------|-------------|
| `environment` | Yes | string | Environment name (dev, prod) |
| `vm_name` | Yes | string | VM name matching tfvars file (dev01, prod01) |
| `terraform_version` | No | string | Terraform version (defaults to latest) |
| `terraform_dir` | No | string | Terraform directory (defaults to "terraform") |
| `confirm_destroy` | Yes* | string | Must be "yes" for destroy workflows |

*Only required for destroy workflow

### Ansible Workflows

| Input | Required | Type | Description |
|-------|----------|------|-------------|
| `environment` | Yes | string | Environment name (dev, prod) |
| `vm_name` | Yes | string | VM name matching tfvars file |
| `ansible_playbook_path` | No | string | Path to playbook (defaults to "ansible/playbook.yml") |
| `ssh_wait_timeout_seconds` | No | number | SSH wait timeout in seconds (default: 200) |

---

## Workflow Validation

All workflows validate inputs before execution:

- **Environment validation**: Only allows "dev" or "prod"
- **tfvars file validation**: Checks that the tfvars file exists
- **Terraform validation**: Runs format, init, and validate checks
- **Ansible validation**: Runs lint and syntax checks
- **Destroy confirmation**: Requires "yes" confirmation for safety

---

## How to Import Into Your Repository

To use these reusable workflows in your repository:

1. **Update your caller workflows** to reference the reusable workflows:
   ```yaml
   jobs:
     check:
       uses: <owner>/<repo>/.gitea/workflows/reusable-terraform-check.yml@<branch>
   ```

2. **Replace `<owner>/<repo>`** with the actual repository reference

3. **Use `@main`** or a specific branch/tag for version pinning

4. **Configure secrets** in your repository settings

5. **Ensure directory structure** matches the requirements above

---

## Customization

To customize these workflows for your use case:

1. Copy the reusable workflow YAML to your repository
2. Modify the environment validation, paths, or steps as needed
3. Update the `uses:` directive in your caller workflows

---

## Troubleshooting

### Common Issues

**Reusable workflow not found**
- Ensure the path includes `.gitea/workflows/` or `.github/workflows/`
- Verify the workflow file exists in the referenced branch
- Check that the branch name in `@branch` is correct

**Missing secrets**
- Verify all required secrets are set in repository settings
- Check secret names match exactly (case-sensitive)
- Ensure secrets are inherited if using organization-level secrets

**Terraform init failures**
- Verify MinIO credentials and endpoint
- Ensure MinIO bucket "terraform-state" exists
- Check network connectivity to MinIO server

**Ansible deployment failures**
- Verify SSH keys are correct and have proper permissions
- Ensure VM IP is accessible from runner
- Check Ansible playbook syntax with the check workflow first

---

## Support

For issues or questions, refer to the original workflow implementations or contact the repository maintainer.
