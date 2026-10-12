# Reusable Workflows — Org Secrets

Shared Gitea Actions reusable workflows for the `Infra` org. This is the **alternate** workflow repo — secrets are read directly from **Gitea org-level secrets** rather than from OpenBao. Use this repo when OpenBao is not available or during initial bootstrapping before Vault is set up.

> **Prefer `reusable-workflows-vault` for production use.** This repo is simpler to set up but stores static long-lived credentials as Gitea secrets, which is less secure than dynamic short-lived tokens from OpenBao.

---

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Secret Management — Gitea Org Secrets](#secret-management--gitea-org-secrets)
- [Workflows](#workflows)
- [How to Call These Workflows](#how-to-call-these-workflows)
- [Input Reference](#input-reference)
- [Switching to OpenBao](#switching-to-openbao)
- [Repository Structure Requirements](#repository-structure-requirements)
- [Troubleshooting](#troubleshooting)

---

## Overview

This repo provides `workflow_call`-triggered Gitea Actions workflows covering the full VM lifecycle. It is functionally identical to `reusable-workflows-vault` except secrets are passed as Gitea secrets rather than fetched from OpenBao.

| Workflow | Trigger | What it does |
|---|---|---|
| `reusable-terraform-check.yml` | `workflow_call` | fmt, validate, tflint — no secrets needed |
| `reusable-terraform-plan.yml` | `workflow_call` | init + plan against MinIO backend |
| `reusable-terraform-apply.yml` | `workflow_call` | init + apply against MinIO backend |
| `reusable-terraform-destroy.yml` | `workflow_call` | init + destroy with confirmation gate |
| `reusable-terraform-init.yml` | `workflow_call` | init only — outputs working_dir and state_key |
| `reusable-ansible-check.yml` | `workflow_call` | lint + syntax-check, installs Galaxy roles |
| `reusable-ansible-deploy.yml` | `workflow_call` | init TF state → get VM IP → SSH wait → deploy |

---

## Prerequisites

| Requirement | Notes |
|---|---|
| Gitea org secrets | All credentials listed below set at `Infra` org level |
| MinIO | S3-compatible state backend, bucket `terraform-state` must exist |
| Gitea Act Runner | Docker-based runner registered to the `Infra` org |
| LXD TLS certificates | Client cert/key generated and trusted in LXD |

---

## Secret Management — Gitea Org Secrets

All credentials are stored as static secrets at `Infra` org → Settings → Secrets and Variables. Calling repos inherit them via `secrets: inherit` or pass them explicitly.

### Required Secrets

| Secret | Used By | Description |
|---|---|---|
| `LXD_ADDRESS` | Terraform | LXD API host IP (e.g. `10.0.0.162`) |
| `LXD_CLIENT_CERT` | Terraform | Full PEM content of `gitea-runner.crt` |
| `LXD_CLIENT_KEY` | Terraform | Full PEM content of `gitea-runner.key` |
| `MINIO_ENDPOINT` | Terraform, Ansible | MinIO S3 API URL (e.g. `http://10.248.42.22:9000`) |
| `MINIO_ACCESS_KEY` | Terraform, Ansible | MinIO access key |
| `MINIO_SECRET_KEY` | Terraform, Ansible | MinIO secret key |
| `ANSIBLE_SSH_PUBLIC_KEY` | Terraform | Injected into VM via cloud-init |
| `ANSIBLE_SSH_PRIVATE_KEY` | Ansible | Used by runner to SSH into provisioned VMs |

### Optional Secrets

| Secret | Used By | Description |
|---|---|---|
| `LXD_TRUST_PASSWORD` | Terraform | Only needed if using password auth instead of TLS certs |

### How to Generate LXD Client Certificates

```bash
# On the LXD host — generate cert + key
mkdir -p ~/lxd-certs && cd ~/lxd-certs
openssl req -x509 -newkey ec \
  -pkeyopt ec_paramgen_curve:secp384r1 \
  -sha384 -keyout gitea-runner.key \
  -out gitea-runner.crt \
  -days 3650 -nodes \
  -subj "/CN=gitea-runner"

# Trust the cert in LXD
lxc config trust add ~/lxd-certs/gitea-runner.crt --name gitea-runner

# Verify
lxc config trust list
```

Then copy the full contents of `gitea-runner.crt` → `LXD_CLIENT_CERT` secret, and `gitea-runner.key` → `LXD_CLIENT_KEY` secret.

### How to Generate the Ansible SSH Key Pair

```bash
ssh-keygen -t ed25519 -C "ansible" -f ~/.ssh/ansible_id_ed25519
# No passphrase (required for automation)

# Public key → ANSIBLE_SSH_PUBLIC_KEY
cat ~/.ssh/ansible_id_ed25519.pub

# Private key → ANSIBLE_SSH_PRIVATE_KEY
cat ~/.ssh/ansible_id_ed25519
```

---

## Workflows

### `reusable-terraform-check.yml`

Validates Terraform code. No secrets required — safe to call on any branch.

Steps: `fmt -check` → `init -backend=false` → `validate` → `tflint`

```yaml
jobs:
  check:
    uses: Infra/reusable-workflows/.gitea/workflows/reusable-terraform-check.yml@main
    with:
      terraform_dir: "terraform"   # optional, default: "terraform"
```

---

### `reusable-terraform-plan.yml`

Writes LXD certs from secrets, inits Terraform against MinIO, validates, runs plan.

```yaml
jobs:
  plan:
    uses: Infra/reusable-workflows/.gitea/workflows/reusable-terraform-plan.yml@main
    with:
      environment: ${{ github.event.inputs.environment }}
      vm_name:     ${{ github.event.inputs.vm_name }}
    secrets:
      MINIO_ENDPOINT:           ${{ secrets.MINIO_ENDPOINT }}
      MINIO_ACCESS_KEY:         ${{ secrets.MINIO_ACCESS_KEY }}
      MINIO_SECRET_KEY:         ${{ secrets.MINIO_SECRET_KEY }}
      LXD_ADDRESS:              ${{ secrets.LXD_ADDRESS }}
      LXD_CLIENT_CERT:          ${{ secrets.LXD_CLIENT_CERT }}
      LXD_CLIENT_KEY:           ${{ secrets.LXD_CLIENT_KEY }}
      ANSIBLE_SSH_PUBLIC_KEY:   ${{ secrets.ANSIBLE_SSH_PUBLIC_KEY }}
      # LXD_TRUST_PASSWORD:     ${{ secrets.LXD_TRUST_PASSWORD }}
```

---

### `reusable-terraform-apply.yml`

Same flow as plan, runs `terraform apply -auto-approve`.

```yaml
jobs:
  apply:
    uses: Infra/reusable-workflows/.gitea/workflows/reusable-terraform-apply.yml@main
    with:
      environment: ${{ github.event.inputs.environment }}
      vm_name:     ${{ github.event.inputs.vm_name }}
    secrets:
      MINIO_ENDPOINT:           ${{ secrets.MINIO_ENDPOINT }}
      MINIO_ACCESS_KEY:         ${{ secrets.MINIO_ACCESS_KEY }}
      MINIO_SECRET_KEY:         ${{ secrets.MINIO_SECRET_KEY }}
      LXD_ADDRESS:              ${{ secrets.LXD_ADDRESS }}
      LXD_CLIENT_CERT:          ${{ secrets.LXD_CLIENT_CERT }}
      LXD_CLIENT_KEY:           ${{ secrets.LXD_CLIENT_KEY }}
      ANSIBLE_SSH_PUBLIC_KEY:   ${{ secrets.ANSIBLE_SSH_PUBLIC_KEY }}
```

---

### `reusable-terraform-destroy.yml`

Hard-gates on `confirm_destroy == "yes"` before proceeding.

```yaml
jobs:
  destroy:
    uses: Infra/reusable-workflows/.gitea/workflows/reusable-terraform-destroy.yml@main
    with:
      environment:     ${{ github.event.inputs.environment }}
      vm_name:         ${{ github.event.inputs.vm_name }}
      confirm_destroy: ${{ github.event.inputs.confirm_destroy }}
    secrets:
      MINIO_ENDPOINT:           ${{ secrets.MINIO_ENDPOINT }}
      MINIO_ACCESS_KEY:         ${{ secrets.MINIO_ACCESS_KEY }}
      MINIO_SECRET_KEY:         ${{ secrets.MINIO_SECRET_KEY }}
      LXD_ADDRESS:              ${{ secrets.LXD_ADDRESS }}
      LXD_CLIENT_CERT:          ${{ secrets.LXD_CLIENT_CERT }}
      LXD_CLIENT_KEY:           ${{ secrets.LXD_CLIENT_KEY }}
      ANSIBLE_SSH_PUBLIC_KEY:   ${{ secrets.ANSIBLE_SSH_PUBLIC_KEY }}
```

---

### `reusable-ansible-check.yml`

Installs Ansible, installs Galaxy roles from `ansible/requirements.yml`, runs lint and syntax-check. No infrastructure secrets required.

```yaml
jobs:
  check:
    uses: Infra/reusable-workflows/.gitea/workflows/reusable-ansible-check.yml@main
    with:
      ansible_playbook_path: "ansible/playbook.yml"   # optional
```

---

### `reusable-ansible-deploy.yml`

Inits Terraform to read VM IP from state → installs Galaxy roles → waits for SSH → deploys playbook.

```yaml
jobs:
  deploy:
    uses: Infra/reusable-workflows/.gitea/workflows/reusable-ansible-deploy.yml@main
    with:
      environment:              ${{ github.event.inputs.environment }}
      vm_name:                  ${{ github.event.inputs.vm_name }}
      ansible_playbook_path:    "ansible/playbook.yml"   # optional
      ssh_wait_timeout_seconds: 200                       # optional
    secrets:
      MINIO_ENDPOINT:           ${{ secrets.MINIO_ENDPOINT }}
      MINIO_ACCESS_KEY:         ${{ secrets.MINIO_ACCESS_KEY }}
      MINIO_SECRET_KEY:         ${{ secrets.MINIO_SECRET_KEY }}
      LXD_ADDRESS:              ${{ secrets.LXD_ADDRESS }}
      ANSIBLE_SSH_PUBLIC_KEY:   ${{ secrets.ANSIBLE_SSH_PUBLIC_KEY }}
      ANSIBLE_SSH_PRIVATE_KEY:  ${{ secrets.ANSIBLE_SSH_PRIVATE_KEY }}
```

---

## How to Call These Workflows

Create caller workflows in your repo under `.gitea/workflows/`. Full examples:

**`terraform-check.yml`**
```yaml
name: "Terraform Check"
on:
  pull_request:
  workflow_dispatch:
jobs:
  check:
    uses: Infra/reusable-workflows/.gitea/workflows/reusable-terraform-check.yml@main
    with:
      terraform_dir: "terraform"
```

**`terraform-plan.yml`**
```yaml
name: "Terraform Plan"
on:
  workflow_dispatch:
    inputs:
      environment:
        description: "Environment (dev/prod) — must match folder terraform/env/<env>"
        required: true
        type: string
      vm_name:
        description: "VM name (e.g. dev01) — must match <vm_name>.tfvars"
        required: true
        type: string
jobs:
  plan:
    uses: Infra/reusable-workflows/.gitea/workflows/reusable-terraform-plan.yml@main
    with:
      environment: ${{ github.event.inputs.environment }}
      vm_name:     ${{ github.event.inputs.vm_name }}
    secrets:
      MINIO_ENDPOINT:           ${{ secrets.MINIO_ENDPOINT }}
      MINIO_ACCESS_KEY:         ${{ secrets.MINIO_ACCESS_KEY }}
      MINIO_SECRET_KEY:         ${{ secrets.MINIO_SECRET_KEY }}
      LXD_ADDRESS:              ${{ secrets.LXD_ADDRESS }}
      LXD_CLIENT_CERT:          ${{ secrets.LXD_CLIENT_CERT }}
      LXD_CLIENT_KEY:           ${{ secrets.LXD_CLIENT_KEY }}
      ANSIBLE_SSH_PUBLIC_KEY:   ${{ secrets.ANSIBLE_SSH_PUBLIC_KEY }}
```

**`ansible-deploy.yml`**
```yaml
name: "Ansible Deploy"
on:
  workflow_dispatch:
    inputs:
      environment:
        required: true
        type: string
      vm_name:
        required: true
        type: string
jobs:
  deploy:
    uses: Infra/reusable-workflows/.gitea/workflows/reusable-ansible-deploy.yml@main
    with:
      environment: ${{ github.event.inputs.environment }}
      vm_name:     ${{ github.event.inputs.vm_name }}
    secrets:
      MINIO_ENDPOINT:          ${{ secrets.MINIO_ENDPOINT }}
      MINIO_ACCESS_KEY:        ${{ secrets.MINIO_ACCESS_KEY }}
      MINIO_SECRET_KEY:        ${{ secrets.MINIO_SECRET_KEY }}
      LXD_ADDRESS:             ${{ secrets.LXD_ADDRESS }}
      ANSIBLE_SSH_PUBLIC_KEY:  ${{ secrets.ANSIBLE_SSH_PUBLIC_KEY }}
      ANSIBLE_SSH_PRIVATE_KEY: ${{ secrets.ANSIBLE_SSH_PRIVATE_KEY }}
```

---

## Input Reference

### Terraform Workflows

| Input | Required | Default | Description |
|---|---|---|---|
| `environment` | Yes | — | `dev` or `prod` |
| `vm_name` | Yes | — | Matches `.tfvars` filename (e.g. `dev01`) |
| `terraform_version` | No | latest | Pin a specific Terraform version |
| `terraform_dir` | No | `terraform` | Root dir for fmt/validate (check workflow only) |
| `confirm_destroy` | Yes* | — | Must be `yes` (destroy workflow only) |

### Ansible Workflows

| Input | Required | Default | Description |
|---|---|---|---|
| `environment` | Yes | — | `dev` or `prod` |
| `vm_name` | Yes | — | Target VM name |
| `ansible_playbook_path` | No | `ansible/playbook.yml` | Path to playbook |
| `ssh_wait_timeout_seconds` | No | `200` | Max seconds to wait for SSH |

---

## Switching to OpenBao

When you are ready to move from static Gitea secrets to dynamic OpenBao credentials:

1. Set up OpenBao and enable AppRole auth
2. Populate the KV v2 paths (`homelab/data/lxd`, `homelab/data/minio`, `homelab/data/ansible`)
3. Add three org secrets: `VAULT_ADDR`, `VAULT_ROLE_ID`, `VAULT_SECRET_ID`
4. In each caller workflow, change the `uses:` line:

```yaml
# Before (this repo)
uses: Infra/reusable-workflows/.gitea/workflows/reusable-terraform-plan.yml@main

# After (OpenBao repo)
uses: Infra/reusable-workflows-vault/.gitea/workflows/reusable-terraform-plan.yml@main
```

5. Replace the explicit `secrets:` block with `secrets: inherit` — the Vault workflow fetches everything itself
6. Add `vault_path_*` inputs if your KV paths differ from the defaults

---

## Repository Structure Requirements

```
your-repo/
├── terraform/
│   ├── env/
│   │   ├── dev/
│   │   │   ├── backend.tf      # terraform { backend "s3" {} }
│   │   │   ├── providers.tf
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf      # must expose vm_ip output
│   │   │   └── dev01.tfvars
│   │   └── prod/
│   │       └── prod01.tfvars
└── ansible/
    ├── playbook.yml
    ├── requirements.yml        # Galaxy roles (optional)
    └── roles/                  # local roles (optional)
```

State path convention (auto-derived from inputs):
```
s3://terraform-state/state/<environment>/<vm_name>/terraform.tfstate
```

---

## Troubleshooting

**`Setup LXD Certs` step is skipped**
- The step is conditional on `secrets.LXD_CLIENT_CERT != ''`
- Verify the secret is set and non-empty at the Gitea org level

**Terraform init fails**
- Check `MINIO_ENDPOINT`, `MINIO_ACCESS_KEY`, `MINIO_SECRET_KEY` are correct
- Ensure bucket `terraform-state` exists in MinIO
- MinIO requires `force_path_style=true` — this is already set in the workflow

**`tfvars file not found`**
- `vm_name` input must exactly match a `.tfvars` filename: input `dev01` → `terraform/env/dev/dev01.tfvars`

**SSH wait times out during Ansible deploy**
- cloud-init may still be running — increase `ssh_wait_timeout_seconds` to `300`
- Verify the VM's security group / firewall allows port 22 from the runner

**Missing secrets error in workflow logs**
- Secrets must be set at the **org level** (`Infra` org → Settings), not just repo level, for `secrets: inherit` to work across repos

---

## Infrastructure Created and Maintained By

**Ali Ahmed**  
Building infrastructure, automation, and DvOps workflows

**Contact**

[![GitHub](https://img.shields.io/badge/GitHub-%20ali%20ahmed-black?style=for-the-badge&logo=github)](https://github.com/jeffreyalie)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-%20ali%20ahmed-blue?style=for-the-badge&logo=linkedin)](https://www.linkedin.com/in/ali-ahmed-261755252/)
