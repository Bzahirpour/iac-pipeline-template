# iac-pipeline-template

A GitHub template repository for AWS Terraform projects. Provides a hardened CI/CD pipeline (lint, validate, plan, Trivy + Checkov security scans with SARIF upload to GitHub Code Scanning, Infracost estimate, sticky PR comment, and Claude security review on every PR; environment-gated apply for `dev` and `prod` on push to `main`), a bootstrap module for the S3 remote state bucket and OIDC IAM role, scaffolded `dev` and `prod` Terraform workspaces, and pinned Dependabot updates for both GitHub Actions and Terraform providers.

## Using this as a GitHub template

1. On this repo's GitHub page, click **Use this template** → **Create a new repository**.
2. Clone your new repo and work through the [New project setup checklist](#new-project-setup-checklist) below.

## New project setup checklist

### a. Bootstrap state bucket and OIDC role

```bash
cd bootstrap
terraform init
terraform apply
```

Edit `variables.tf` defaults (or pass via `-var`):

| Variable | What to set |
|---|---|
| `project_name` | Your project name, e.g. `my-project` |
| `github_org` | Your GitHub username or org |
| `github_repo` | The new repo name |
| `create_oidc_provider` | `false` if you already have the GitHub OIDC provider in this AWS account (only one is allowed per account — import the existing one with `terraform import aws_iam_openid_connect_provider.github <arn>`) |

Note the outputs — `tf_state_bucket` and `github_actions_role_arn`. You'll need them for the next steps. Paste `tf_state_bucket` into both `infra/envs/dev/primary/backend.tf` and `infra/envs/prod/primary/backend.tf` (replacing the `REPLACE-ME` placeholder).

### b. Set GitHub Variables

In your new repo → **Settings → Secrets and variables → Actions → Variables tab**:

| Variable | Value |
|---|---|
| `AWS_REGION` | The AWS region you bootstrapped in (e.g. `us-east-1`) |
| `AWS_ROLE_ARN` | The `github_actions_role_arn` output from bootstrap |

### c. Set GitHub Secrets — in BOTH the Actions scope and the Dependabot scope

In your new repo → **Settings → Secrets and variables**, add the following to **both** the **Actions** tab and the **Dependabot** tab:

| Secret | Used by | Where to get one |
|---|---|---|
| `CLAUDE_API_KEY` | `claude-review` job in `pr-checks.yml` | https://console.anthropic.com |
| `INFRACOST_API_KEY` | `infracost` job in `pr-checks.yml` | https://www.infracost.io/ (free tier available) |

> **GitHub has a separate secret scope for Dependabot.** Secrets added to the Actions scope are not visible to Dependabot-triggered workflow runs. Dependabot opens PRs that re-run `pr-checks.yml`, so it needs the same secrets. **Missing the Dependabot scope is the single most common failure mode** — Dependabot PRs will fail with confusing "API key not found" errors until you add the secrets there too.

### d. Configure GitHub Environments

In your new repo → **Settings → Environments**, create two environments:

- `dev` — add **Required reviewers** (yourself) to enforce the approval gate before applying to dev.
- `prod` — add **Required reviewers** (yourself) to enforce the approval gate before applying to prod.

Both environments must exist for `apply.yml` to run; both should have required reviewers.

### e. Adjust the env layout if you change the defaults

This template ships with `infra/envs/dev/primary/` and `infra/envs/prod/primary/`. The `/primary` layer is intentional — it leaves room to add sibling workspaces (e.g. `/secondary`, `/global`) alongside `/primary` in the same env without restructuring later.

If you rename, remove, or add workspaces, update `.github/dependabot.yml`'s `terraform` `directories` list and the working-directory paths in `.github/workflows/pr-checks.yml` and `.github/workflows/apply.yml` accordingly.

### f. Scope down the bootstrap IAM role (optional but recommended)

`bootstrap/main.tf` attaches `AdministratorAccess` to the GitHub Actions role for simplicity. Once you know which AWS services your project actually uses, replace this with a least-privilege policy.

## CI/CD gate

Two workflows replaced the original `terraform.yml`:

### `pr-checks.yml` — every PR targeting `main` that touches `infra/**`

```
lint → validate (dev + prod) → plan (dev + prod) → scan (dev + prod)
                                                 ↘ infracost (dev + prod)
                                                       ↓
                                                    comment (sticky)
claude-review (independent lane)
```

| Job | What it does | Gates merge? |
|---|---|---|
| `lint` | `terraform fmt -check` + tflint | Yes |
| `validate` | `init -backend=false` + `validate` (no AWS creds) | Yes |
| `plan` | Full plan via OIDC; uploads plan.json + plan.txt as artifacts | Yes |
| `scan` | Trivy + Checkov against resolved `plan.json`; SARIF → Code Scanning | Yes (HIGH/CRITICAL) |
| `infracost` | Cost estimate from plan JSON | No |
| `comment` | Single sticky comment: plan + Trivy + Checkov + cost per env | No |
| `claude-review` | Claude posts its own security review comment | No |

### `apply.yml` — push to `main` that touches `infra/**`

```
plan-dev → apply-dev (env gate) → plan-prod → apply-prod (env gate)
```

Each apply job is gated by a GitHub Environment with required reviewers. Both dev and prod require approval. `concurrency: cancel-in-progress: false` ensures in-flight applies are never cancelled by a subsequent push.

> **Plan drift:** The plan shown in the PR comment is generated from the PR head commit. `apply.yml` re-plans from scratch at merge time. If another PR merged concurrently and changed shared state, the applied plan may differ from what was reviewed. The approval gate before each apply is the moment to catch this.

### Secrets required

| Secret | Used by | Notes |
|---|---|---|
| _(none — OIDC)_ | All AWS calls | Role assumed via `<project>-github-actions` |
| `CLAUDE_API_KEY` | `claude-review` job | Anthropic API key |
| `INFRACOST_API_KEY` | `infracost` job | Free tier available at infracost.io |

## Repository layout

```
bootstrap/                # S3 state bucket + OIDC IAM role (one-time, local apply)
infra/
  modules/
    _example/             # Scaffold module — rename or copy as a starting point
  envs/
    dev/
      primary/            # Default dev workspace (add /secondary, /global, etc. as siblings)
    prod/
      primary/            # Default prod workspace (add /secondary, /global, etc. as siblings)
.github/
  workflows/
    pr-checks.yml         # lint → validate → plan → scan → infracost → comment + claude-review
    apply.yml             # plan-dev → apply-dev (gate) → plan-prod → apply-prod (gate)
  dependabot.yml          # Weekly grouped updates for GitHub Actions + Terraform providers
.checkov.yaml             # Checkov skip-check list (empty by default)
.trivyignore              # Trivy suppression list (empty by default)
.tflint.hcl               # tflint plugin + ruleset configuration
```
