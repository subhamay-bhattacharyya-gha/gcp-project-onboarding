# GCP Project Onboarding — Reusable Workflow

![Built with Claude Code](https://img.shields.io/badge/Built_with-Claude_Code-blueviolet?logo=anthropic&logoColor=white)&nbsp;![Terraform](https://img.shields.io/badge/Terraform-7B42BC?logo=terraform&logoColor=white)&nbsp;![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-2088FF?logo=githubactions&logoColor=white)&nbsp;![Release](https://github.com/subhamay-bhattacharyya-gha/gcp-project-onboarding/actions/workflows/release.yaml/badge.svg)&nbsp;![Commit Activity](https://img.shields.io/github/commit-activity/t/subhamay-bhattacharyya-gha/gcp-project-onboarding)&nbsp;![Last Commit](https://img.shields.io/github/last-commit/subhamay-bhattacharyya-gha/gcp-project-onboarding)&nbsp;![Release Date](https://img.shields.io/github/release-date/subhamay-bhattacharyya-gha/gcp-project-onboarding)&nbsp;![Repo Size](https://img.shields.io/github/repo-size/subhamay-bhattacharyya-gha/gcp-project-onboarding)&nbsp;![File Count](https://img.shields.io/github/directory-file-count/subhamay-bhattacharyya-gha/gcp-project-onboarding)&nbsp;![Issues](https://img.shields.io/github/issues/subhamay-bhattacharyya-gha/gcp-project-onboarding)&nbsp;![Top Language](https://img.shields.io/github/languages/top/subhamay-bhattacharyya-gha/gcp-project-onboarding)&nbsp;![Custom Endpoint](https://img.shields.io/endpoint?url=https://gist.githubusercontent.com/bsubhamay/8258f1e702044c00c3d090add48c0f51/raw/gcp-project-onboarding.json?)

A centralised, reusable GitHub Actions workflow (`workflow_call`) that lets any application repo in the organisation onboard itself to HCP Terraform, GitHub Environments, and repository rulesets with a single `env.json` file.

## Features

- **Single config file** — all project settings live in `env.json` at the repo root
- **HCP Terraform integration** — creates workspaces, links VCS, and seeds variables
- **GitHub Environments** — creates environments with `GCP_PROJECT_ID` per environment
- **Production approval gate** — configurable reviewers and wait timer (exact match only)
- **Repository rulesets** — branch protection with PR reviews, status checks, force-push blocking
- **Fully idempotent** — every step is safe to re-run on an already-onboarded repo
- **Reusable design** — consumed via `workflow_call` by any repo in the org

## How it works

The workflow runs 9 sequential steps:

| Step | Name                   | Description                                         |
| ---- | ---------------------- | --------------------------------------------------- |
| 0    | `parse`                | Validate and extract all fields from `env.json`     |
| 1    | `check`                | Check if the HCP Terraform workspace already exists |
| 2    | `create`               | Create workspace (skipped if it exists)             |
| 3    | `resolve`              | Merge workspace ID from create or check             |
| 4    | `link-vcs`             | Attach VCS repository to the workspace              |
| 5    | `set-vars`             | Seed workspace variables                            |
| 6    | `setup-environments`   | Create GitHub Environments + set `GCP_PROJECT_ID`   |
| 7    | `setup-ruleset`        | Create or update repository ruleset                 |
| 8    | `summary`              | Write `$GITHUB_STEP_SUMMARY`                        |

## Quick start

### 1. Add `env.json` to your application repo root

Copy `env.json` from this platform repo and fill in your values:

```json
{
  "hcp": {
    "organization": "my-hcp-org",
    "workspace_name": "my-app",
    "terraform_version": "1.7.0",
    "execution_mode": "remote",
    "branch": "main",
    "auto_apply": false,
    "workspace_variables": [
      { "key": "TF_VAR_region", "value": "europe-west2", "sensitive": false, "category": "terraform" }
    ]
  },
  "environments": [
    { "name": "dev",        "gcp_project_id": "my-app-dev-123456" },
    { "name": "test",       "gcp_project_id": "my-app-test-789012" },
    { "name": "production", "gcp_project_id": "my-app-prod-345678" }
  ],
  "github": {
    "production_environment": "production",
    "reviewer_teams": ["platform-team"],
    "wait_timer": 5
  },
  "ruleset": {
    "enabled": true,
    "target_branch": "~DEFAULT_BRANCH",
    "require_pr": true,
    "required_approvals": 1,
    "dismiss_stale_reviews": true,
    "required_status_checks": ["ci / build", "ci / test"],
    "block_force_pushes": true,
    "prevent_deletion": true
  }
}
```

### 2. Add the caller workflow

Copy `templates/onboard-gcp-project.yaml` to `.github/workflows/onboard-gcp-project.yaml` in your app repo:

```yaml
name: Onboard GCP Project

on:
  push:
    branches: [main]
    paths:
      - 'env.json'
      - '.github/workflows/onboard-gcp-project.yaml'
  workflow_dispatch:

jobs:
  onboard:
    uses: <ORG>/gcp-project-onboarding/.github/workflows/gcp-project-onboarding.yaml@v1
    secrets: inherit
```

### 3. Configure org-level secrets

| Secret                   | Description                                                      |
| ------------------------ | ---------------------------------------------------------------- |
| `TFE_TOKEN`              | HCP Terraform API token                                          |
| `TF_VCS_OAUTH_TOKEN_ID`  | OAuth token ID for VCS provider                                  |
| `GITHUB_PAT`             | Fine-grained PAT (environments: read/write, administration: write) |

### 4. Push and go

Push `env.json` and the caller workflow to `main`. The onboarding runs automatically.

## Idempotency

| Resource                       | Mechanism                                             |
| ------------------------------ | ----------------------------------------------------- |
| HCP workspace                  | Existence check before create; step skipped if exists |
| HCP workspace variables        | `POST` — on `422` fetch var ID and `PATCH`            |
| GitHub Environments            | `PUT` is always an upsert                             |
| GitHub Environment variables   | `POST` — on `409` use `PATCH`                         |
| Repository ruleset             | List by name — `PUT` if found, `POST` if not          |

## Repository structure

```text
gcp-project-onboarding/
├── .github/workflows/
│   ├── gcp-project-onboarding.yaml   # The reusable workflow
│   ├── create-branch.yaml
│   └── release.yaml
├── templates/
│   ├── gcp-project-onboarding.yaml   # Template copy (kept in sync)
│   └── onboard-gcp-project.yaml      # Caller workflow template
├── env.json                           # Sample configuration
├── CLAUDE.md                          # AI development conventions
└── README.md
```

## Versioning

Releases follow semver. The `release.yaml` workflow moves the major-version tag (`v1`) forward automatically.

- **Patch** (`v1.0.x`) — bug fixes, no interface changes
- **Minor** (`v1.x.0`) — new optional config fields, backward compatible
- **Major** (`v2.0.0`) — breaking changes requiring migration

## License

MIT

---

**Built with [Claude Code](https://claude.ai/code)**
