---
name: gcp-project-onboarding
description: >
  Generates and maintains a centralised, reusable GitHub Actions workflow
  (workflow_call) that lives in a platform repo and lets any application repo in
  the organisation onboard itself to HCP Terraform, GitHub Environments, and
  repository rulesets with a single job reference. Use this skill whenever the
  user wants to: build a platform/shared repo for GitHub Actions reuse, publish
  a reusable workflow for HCP Terraform workspace registration, create GitHub
  Environments with GCP project variables, set environment-tier deployment
  approval gates (dev/test = no approval, production = required reviewers),
  create or update repository rulesets (branch protection, merge rules, required
  checks), version and release a shared GitHub Actions workflow, set up
  cross-repo secret sharing for Terraform Cloud, or standardise how all repos
  in an org get a Terraform workspace. Also trigger when the user mentions
  "reusable workflow", "called workflow", "workflow_call", "platform repo",
  "central workflow", "shared Actions", "TFE workspace", "Terraform Cloud
  workspace", "HCP workspace automation", "GitHub environment", "deployment
  protection", "approval gate", "required reviewers", "repository ruleset",
  "branch protection", or "GCP project variable".
---

# GCP Project Onboarding — Reusable Workflow Skill

Produces a **centralised reusable GitHub Actions workflow** that lives in a
dedicated platform repo. Any application repo in the organisation fully onboards
itself — HCP Terraform workspace, GitHub Environments with GCP project variables,
environment-tier approval gates, and repository rulesets — by referencing this
workflow with a single `uses:` line. No copy-pasting, no per-repo curl logic.

---

## Clarify before generating

Ask the user (or infer from context) before writing any files.
All project-specific values go into `project-config.json` — not into the workflow file.

| Question | Default / notes |
|---|---|
| Platform repo name | `gcp-project-onboarding` |
| HCP organisation name | required — goes in `config.hcp.organization` |
| HCP workspace name | defaults to repo name if omitted |
| HCP Terraform address (cloud vs self-hosted) | `https://app.terraform.io` |
| Terraform version to pin | `1.7.0` |
| Execution mode (`remote` / `agent` / `local`) | `remote` |
| Working directory (mono-repo support) | `""` (root) |
| Branch to track | `main` |
| Workspace variables to seed | none — add to `config.hcp.workspace_variables[]` |
| Environments list | `dev`, `test`, `production` — each entry in `config.environments[]` |
| GCP project ID per environment | required per-env — `config.environments[].gcp_project_id` |
| Production environment name | `production` — `config.github.production_environment` |
| Reviewer team slug(s) for production | required — `config.github.reviewer_teams[]` |
| Wait timer (minutes) before production deployment | `0` — `config.github.wait_timer` |
| Create repository ruleset? | `true` — `config.ruleset.enabled` |
| Ruleset target branch | `~DEFAULT_BRANCH` |
| Required approvals | `1` |
| Status checks to require | `[]` (none) |
| Block force-pushes / prevent deletion | `true` / `true` |
| Workflow version pin | `@v1` |
| Org secrets configured? | yes — `TFE_TOKEN`, `TF_VCS_OAUTH_TOKEN_ID`, `GITHUB_PAT` |

---

## Repository architecture

The reusable workflow pattern requires **two separate repositories**:

```
<org>/gcp-project-onboarding           ← owned by platform/infra team
├── .github/
│   └── workflows/
│       ├── gcp-project-onboarding.yaml    ← workflow_call definition (all steps)
│       └── release.yaml                    ← moves major-version tag on release
└── CODEOWNERS                             ← protects workflows from unauthorised edits

<org>/my-app-repo                      ← owned by each application team
├── .github/
│   └── workflows/
│       └── onboard-gcp-project.yaml  ← thin caller; reads config + calls reusable workflow
└── project-config.json              ← all project settings; triggers onboarding on change
```

The platform repo is the single source of truth. Application repos never
contain workspace-registration logic — only a `uses:` reference. When the
platform team ships a fix, all callers get it on the next run.

---

## Generating output files

**Always read the templates first — never reconstruct from scratch.**

This skill includes four complete, ready-to-run template files in `templates/`:

| Template file | Destination in repo | What to substitute |
|---|---|---|
| `templates/gcp-project-onboarding.yaml` | `<org>/gcp-project-onboarding/.github/workflows/` | Nothing — canonical workflow; extend only if adding new capability |
| `templates/onboard-gcp-project.yaml` | `<org>/<app-repo>/.github/workflows/` | `<ORG>` in the `uses:` line |
| `templates/project-config.sample.json` | `<org>/<app-repo>/project-config.json` | All placeholder values (org, workspace, GCP project IDs, team slugs) |
| `templates/release.yaml` | `<org>/gcp-project-onboarding/.github/workflows/` | Nothing — generic, no substitutions needed |
| `templates/CODEOWNERS` | `<org>/gcp-project-onboarding/` | `<ORG>` in the team slug |

**Workflow for generating output:**
1. Read the relevant template file(s) from `templates/`.
2. Substitute only the `<PLACEHOLDER>` values using information from the user
   or the clarify table defaults.
3. Present the substituted file(s) — do not rewrite logic, reorder steps, or
   alter variable names.
4. If the user asks to extend the workflow (new step, new input, new API call),
   apply the change to the template copy and note what changed.

---

## Output: four files

Always produce **all four** files together.

### 1. `gcp-project-onboarding/.github/workflows/gcp-project-onboarding.yaml` — the reusable workflow

Lives in the platform repo. Key characteristics:

- Top-level trigger is **`workflow_call` only** (never `push` or `schedule` here —
  those belong on the caller side).
- **Typed `inputs` block** — every caller-tunable value is an explicit input with
  `type`, `required`, and `default`. Do not use `env:` for values callers should
  control.
- **`secrets` block** — declare each secret with `required: true` so GitHub
  validates them before the job starts. Do not use `secrets: inherit` on the
  reusable side; that is the caller's choice.
- **Steps in order:**
  1. `check` — GET the workspace; outputs `exists` (true/false) and `workspace_id`.
  2. `create` — runs only when `exists == false`; POSTs to create the workspace;
     outputs `workspace_id`.
  3. `resolve` — merges create/check IDs into a single `workspace_id` output.
  4. `link-vcs` — PATCHes the workspace with `vcs-repo` attributes.
  5. `set-vars` — posts each variable via a reusable shell function.
  6. `setup-environments` — loops over `inputs.environments`; creates or updates
     each GitHub Environment via the REST API; sets `GCP_PROJECT_ID` as an
     environment variable; applies deployment protection rules based on tier
     (see environment tier logic below).
  7. `setup-ruleset` — runs only when `inputs.create_ruleset == true`; creates
     or updates the repository ruleset via the REST API.
  8. `summary` — writes a `$GITHUB_STEP_SUMMARY` table covering all resources.
- All `curl` calls use `-s -w "\n%{http_code}"` (no `--fail`); capture the HTTP
  status and body separately, then `exit 1` with a descriptive message on non-2xx.
- `jq` is assumed available on `ubuntu-latest`; add an install step only if the
  user specifies a self-hosted runner.

Full `workflow_call` interface:

```yaml
on:
  workflow_call:
    inputs:
      workspace_name:
        description: 'HCP Terraform workspace name (defaults to calling repo name)'
        type: string
        required: false
        default: ''
      terraform_version:
        description: 'Terraform version to pin in the workspace'
        type: string
        required: false
        default: '1.7.0'
      execution_mode:
        description: 'remote | agent | local'
        type: string
        required: false
        default: 'remote'
      working_directory:
        description: 'Terraform working directory (for mono-repos)'
        type: string
        required: false
        default: ''
      branch:
        description: 'VCS branch to track'
        type: string
        required: false
        default: 'main'
      auto_apply:
        description: 'Enable auto-apply on the workspace'
        type: boolean
        required: false
        default: false
      tfe_address:
        description: 'HCP Terraform address (override for self-hosted TFE)'
        type: string
        required: false
        default: 'https://app.terraform.io'
      # --- GitHub Environments ---
      environments:
        description: 'Comma-separated list of environment names to create (e.g. dev,test,production)'
        type: string
        required: false
        default: 'dev,test,production'
      production_environment:
        description: 'Name of the environment that requires approval (exact match)'
        type: string
        required: false
        default: 'production'
      reviewer_teams:
        description: 'Comma-separated GitHub team slugs used as required reviewers on production'
        type: string
        required: false
        default: ''
      wait_timer:
        description: 'Minutes to wait before a production deployment can proceed (0 = no wait)'
        type: number
        required: false
        default: 0
      gcp_project_ids:
        description: >
          JSON object mapping each environment name to its GCP project ID.
          Example: {"dev":"my-project-dev","test":"my-project-test","production":"my-project-prod"}
        type: string
        required: false
        default: '{}'
      # --- Repository ruleset ---
      create_ruleset:
        description: 'Create or update a repository ruleset on the calling repo'
        type: boolean
        required: false
        default: true
      ruleset_target_branch:
        description: 'Branch pattern the ruleset targets (e.g. ~DEFAULT_BRANCH or refs/heads/main)'
        type: string
        required: false
        default: '~DEFAULT_BRANCH'
      require_pr:
        description: 'Require a pull request before merging'
        type: boolean
        required: false
        default: true
      required_approvals:
        description: 'Minimum number of required PR approvals'
        type: number
        required: false
        default: 1
      dismiss_stale_reviews:
        description: 'Dismiss stale reviews when new commits are pushed'
        type: boolean
        required: false
        default: true
      required_status_checks:
        description: 'JSON array of status check contexts to require. Example: ["ci/build","ci/test"]'
        type: string
        required: false
        default: '[]'
      block_force_pushes:
        description: 'Block force-pushes to the target branch'
        type: boolean
        required: false
        default: true
      prevent_deletion:
        description: 'Prevent deletion of the target branch'
        type: boolean
        required: false
        default: true
    secrets:
      TFE_TOKEN:
        required: true
      TF_VCS_OAUTH_TOKEN_ID:
        required: true
      GITHUB_PAT:
        required: true   # Fine-grained PAT: repo environments + rulesets write scopes
    outputs:
      workspace_id:
        description: 'The HCP Terraform workspace ID'
        value: ${{ jobs.register.outputs.workspace_id }}
      workspace_created:
        description: 'true if the workspace was newly created, false if it already existed'
        value: ${{ jobs.register.outputs.workspace_created }}
```

### 2. `gcp-project-onboarding/CODEOWNERS` — protect the platform repo

```
# Platform workflow owners — required reviewers for any change
.github/workflows/   @<org>/platform-team
```

### 3. `my-app-repo/.github/workflows/onboard-gcp-project.yaml` — the caller

A thin file in each application repo. The caller:

- Triggers on `push` to `main` (path-filtered) and `workflow_dispatch`.
- References the platform workflow by **tag** (e.g. `@v1`) not `@main`, so
  callers are insulated from in-progress platform changes.
- Uses `secrets: inherit` to forward organisation secrets automatically (no
  per-repo secret configuration needed when org-level secrets are set).

```yaml
name: Onboard GCP Project

on:
  push:
    branches: [main]
    paths: ['.github/workflows/onboard-gcp-project.yaml']
  workflow_dispatch:

jobs:
  onboard:
    uses: <org>/gcp-project-onboarding/.github/workflows/gcp-project-onboarding.yaml@v1
    with:
      workspace_name: ${{ github.event.repository.name }}
      terraform_version: "1.7.0"
      environments: "dev,test,production"
      production_environment: "production"
      reviewer_teams: "platform-team"
      wait_timer: 5
      gcp_project_ids: '{"dev":"my-app-dev","test":"my-app-test","production":"my-app-prod"}'
      create_ruleset: true
      required_approvals: 1
    secrets: inherit
```

The caller should be **fewer than 20 lines**. If it grows larger, logic has
leaked out of the reusable workflow.

---

## Required GitHub secrets

Always include this table in a comment block at the top of both files and in
any written explanation:

| Secret | Where to get it |
|---|---|
| `TFE_TOKEN` | HCP Terraform → User/Team Settings → API Tokens |
| `TF_VCS_OAUTH_TOKEN_ID` | `GET /api/v2/organizations/{org}/oauth-tokens` — see snippet below |
| `GITHUB_PAT` | GitHub → Settings → Developer settings → Fine-grained tokens (see scopes below) |

**Required PAT scopes** (fine-grained token, scoped to the application repo):
- `environments: read and write` — create/update GitHub Environments and protection rules
- `administration: write` — create/update repository rulesets

**Snippet to retrieve the OAuth token ID:**
```bash
curl -s \
  --header "Authorization: Bearer $TFE_TOKEN" \
  "https://app.terraform.io/api/v2/organizations/YOUR_ORG/oauth-tokens" \
  | jq '.data[] | {id: .id, service: .attributes["service-provider"]}'
```

---

## GitHub Environments — tier logic

Each environment is created (or updated) via `PUT /repos/{owner}/{repo}/environments/{env_name}`.

**Approval gate rules (derive from environment name, not a separate input):**

```
if environment_name == inputs.production_environment:
    → deployment_branch_policy: null (any branch can deploy — lock down via ruleset instead)
    → protection_rules:
        - type: required_reviewers
          reviewers: [ each slug in inputs.reviewer_teams resolved to team node_id ]
        - type: wait_timer (if inputs.wait_timer > 0)
else:
    → deployment_branch_policy: null
    → protection_rules: []    ← no approval gate for dev / test
```

The tier check must be a **string equality test** against `inputs.production_environment`,
not a substring match. A repo named `my-production-clone` must not accidentally
get reviewers.

**Resolving team node IDs** — the protection rules API takes node IDs, not slugs.
Resolve each slug before the environment loop:

```bash
TEAM_SLUG="platform-team"
NODE_ID=$(curl -s \
  --header "Authorization: Bearer $GITHUB_PAT" \
  --header "Accept: application/vnd.github+json" \
  "https://api.github.com/orgs/$ORG/teams/$TEAM_SLUG" \
  | jq -r '.node_id')
```

**Environment variable — GCP project ID:**

After creating each environment, set the `GCP_PROJECT_ID` variable using:

```bash
GCP_ID=$(echo '${{ inputs.gcp_project_ids }}' | jq -r --arg env "$ENV_NAME" '.[$env] // empty')

if [ -n "$GCP_ID" ]; then
  curl -s -w "\n%{http_code}" \
    --request POST \
    --header "Authorization: Bearer $GITHUB_PAT" \
    --header "Accept: application/vnd.github+json" \
    --data "{\"name\":\"GCP_PROJECT_ID\",\"value\":\"$GCP_ID\"}" \
    "https://api.github.com/repos/$REPO/environments/$ENV_NAME/variables"
fi
```

Use `PATCH` instead of `POST` when the variable already exists (HTTP 409 on POST
means it exists; retry with `PATCH /repos/{owner}/{repo}/environments/{env}/variables/{name}`).

**Full environment PUT payload:**

```json
{
  "wait_timer": 0,
  "prevent_self_review": false,
  "reviewers": [
    { "type": "Team", "id": <team_node_id_integer> }
  ],
  "deployment_branch_policy": null
}
```

For non-production environments the `reviewers` array must be `[]` and
`wait_timer` must be `0`.

---

## Repository rulesets

Create or update via `POST /repos/{owner}/{repo}/rulesets` (create) or
`PUT /repos/{owner}/{repo}/rulesets/{ruleset_id}` (update).

**Check if a ruleset already exists** before creating:

```bash
EXISTING=$(curl -s \
  --header "Authorization: Bearer $GITHUB_PAT" \
  --header "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/$OWNER/$REPO/rulesets" \
  | jq -r '.[] | select(.name == "standard-branch-protection") | .id')
```

**Ruleset payload template:**

```json
{
  "name": "standard-branch-protection",
  "target": "branch",
  "enforcement": "active",
  "conditions": {
    "ref_name": {
      "include": ["~DEFAULT_BRANCH"],
      "exclude": []
    }
  },
  "rules": [
    { "type": "deletion" },
    { "type": "non_fast_forward" },
    {
      "type": "pull_request",
      "parameters": {
        "required_approving_review_count": 1,
        "dismiss_stale_reviews_on_push": true,
        "require_code_owner_review": false,
        "require_last_push_approval": false,
        "required_review_thread_resolution": false
      }
    },
    {
      "type": "required_status_checks",
      "parameters": {
        "required_status_checks": [],
        "strict_required_status_checks_policy": false
      }
    }
  ]
}
```

Map inputs to payload fields:

| Input | JSON field |
|---|---|
| `inputs.ruleset_target_branch` | `conditions.ref_name.include[0]` |
| `inputs.require_pr` | include/exclude the `pull_request` rule object |
| `inputs.required_approvals` | `rules[pull_request].parameters.required_approving_review_count` |
| `inputs.dismiss_stale_reviews` | `rules[pull_request].parameters.dismiss_stale_reviews_on_push` |
| `inputs.required_status_checks` | `rules[required_status_checks].parameters.required_status_checks` — parse JSON array from input string |
| `inputs.block_force_pushes` | include/exclude the `non_fast_forward` rule object |
| `inputs.prevent_deletion` | include/exclude the `deletion` rule object |

When `inputs.required_status_checks` is `[]` (the default), omit the
`required_status_checks` rule entirely rather than passing an empty array —
GitHub will error on an empty `required_status_checks` array.

**Idempotency:** if `$EXISTING` is non-empty, use `PUT …/rulesets/$EXISTING`;
otherwise use `POST …/rulesets`. Log which branch was taken in the step output.

---

## Versioning and release strategy for the platform repo

Callers pin to a **tag** (`@v1`, `@v2`), not `@main`. Explain this pattern
whenever generating the caller file:

```
main  ──●──●──●──●──●──●──●   (platform team commits here)
              │        │
             v1       v2       (tags cut when ready; callers pin these)
```

Include a GitHub Actions release workflow in the platform repo that moves the
major-version tag forward on every release:

```yaml
# gcp-project-onboarding/.github/workflows/release.yaml
name: Release
on:
  push:
    tags: ['v*.*.*']
jobs:
  tag:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
      - name: Move major version tag
        run: |
          MAJOR=$(echo "${{ github.ref_name }}" | cut -d. -f1)
          git tag -f "$MAJOR"
          git push origin "$MAJOR" --force
```

Callers on `@v1` automatically get all `v1.x.x` patches without changing their
workflow file.

---

## Cross-repo secret sharing

Reusable workflows work best with **organisation-level secrets** so callers need
zero per-repo configuration. Show this guidance when the user asks about secrets:

```
GitHub Org Settings → Secrets and variables → Actions → New organisation secret
  TFE_TOKEN              → All repositories (or selected repos)
  TF_VCS_OAUTH_TOKEN_ID  → All repositories
  GITHUB_PAT             → All repositories (fine-grained PAT; see scopes in secrets table)

Note: TF_CLOUD_ORG is no longer a secret. It lives in project-config.json
under config.hcp.organization alongside all other project identity fields.
```

With org secrets, the caller uses `secrets: inherit` and needs no secrets block
of its own. Mention that `secrets: inherit` only forwards secrets the caller
repo has access to — private repos outside the org cannot inherit org secrets.

For enterprises with strict secret scoping, show the explicit secrets block
alternative:

```yaml
jobs:
  onboard:
    uses: <org>/gcp-project-onboarding/.github/workflows/gcp-project-onboarding.yaml@v1
    with:
      workspace_name: ${{ github.event.repository.name }}
    secrets:
      TFE_TOKEN: ${{ secrets.TFE_TOKEN }}
      TF_VCS_OAUTH_TOKEN_ID: ${{ secrets.TF_VCS_OAUTH_TOKEN_ID }}
```

---

## project-config.json schema

All project-specific settings live in `project-config.json` at the root of
each application repo. This file is the single source of truth for a project's
identity. Copy `templates/project-config.sample.json` as a starting point.

The workflow reads and validates this file at run time. Fields marked
**required** will cause the workflow to fail with a clear error message if
absent or null.

### Top-level structure

```
{
  "hcp":          { ... }   HCP Terraform workspace settings
  "environments": [ ... ]   GitHub Environments (one object per env)
  "github":       { ... }   Environment protection settings
  "ruleset":      { ... }   Repository ruleset (branch protection) settings
}
```

### `hcp` object

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `organization` | string | ✅ | — | HCP Terraform organisation name |
| `workspace_name` | string | ❌ | repo name | HCP workspace name |
| `terraform_version` | string | ❌ | `"1.7.0"` | Terraform version to pin |
| `execution_mode` | string | ❌ | `"remote"` | `remote` / `agent` / `local` |
| `working_directory` | string | ❌ | `""` | Terraform root inside repo |
| `branch` | string | ❌ | `"main"` | VCS branch to track |
| `auto_apply` | boolean | ❌ | `false` | Auto-apply on plan success |
| `tfe_address` | string | ❌ | `"https://app.terraform.io"` | Override for self-hosted TFE |
| `workspace_variables` | array | ❌ | `[]` | Seed variables (see below) |

Each entry in `workspace_variables`:

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `key` | string | ✅ | — | Variable name |
| `value` | string | ✅ | — | Variable value |
| `sensitive` | boolean | ❌ | `false` | Mark as sensitive (write-only in UI) |
| `category` | string | ❌ | `"terraform"` | `"terraform"` or `"env"` |

### `environments` array

Each object in the array:

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `name` | string | ✅ | — | GitHub Environment name (e.g. `"production"`) |
| `gcp_project_id` | string | ❌ | — | GCP project ID set as `GCP_PROJECT_ID` env var |

### `github` object

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `production_environment` | string | ❌ | `"production"` | Exact env name that gets approval gate |
| `reviewer_teams` | array[string] | ❌ | `[]` | GitHub team slugs for required reviewers |
| `wait_timer` | number | ❌ | `0` | Minutes to wait before prod deployment proceeds |

### `ruleset` object

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `enabled` | boolean | ❌ | `true` | Create/update the ruleset |
| `target_branch` | string | ❌ | `"~DEFAULT_BRANCH"` | Branch pattern |
| `require_pr` | boolean | ❌ | `true` | Require PR before merge |
| `required_approvals` | number | ❌ | `1` | Minimum approving reviews |
| `dismiss_stale_reviews` | boolean | ❌ | `true` | Dismiss stale reviews on push |
| `required_status_checks` | array[string] | ❌ | `[]` | Status check contexts to require |
| `block_force_pushes` | boolean | ❌ | `true` | Block force-pushes |
| `prevent_deletion` | boolean | ❌ | `true` | Prevent branch deletion |

---

## HCP Terraform API reference (key endpoints)

Use these exact paths. The base URL is always `$TFE_ADDRESS` (default:
`https://app.terraform.io`).

| Action | Method | Path |
|---|---|---|
| Check workspace exists | GET | `/api/v2/organizations/{org}/workspaces/{name}` |
| Create workspace | POST | `/api/v2/organizations/{org}/workspaces` |
| Update/link VCS | PATCH | `/api/v2/workspaces/{workspace_id}` |
| Create variable | POST | `/api/v2/vars` |
| Assign to project | POST | `/api/v2/projects/{project_id}/relationships/workspaces` |
| Grant team access | POST | `/api/v2/team-workspaces` |

All HCP Terraform requests must include:
```
Authorization: Bearer $TFE_TOKEN
Content-Type: application/vnd.api+json
```

## GitHub REST API reference (key endpoints)

Base URL: `https://api.github.com`. All requests must include:
```
Authorization: Bearer $GITHUB_PAT
Accept: application/vnd.github+json
X-GitHub-Api-Version: 2022-11-28
```

| Action | Method | Path |
|---|---|---|
| Create/update environment | PUT | `/repos/{owner}/{repo}/environments/{env_name}` |
| Create environment variable | POST | `/repos/{owner}/{repo}/environments/{env_name}/variables` |
| Update environment variable | PATCH | `/repos/{owner}/{repo}/environments/{env_name}/variables/{name}` |
| Get team (for node ID) | GET | `/orgs/{org}/teams/{team_slug}` |
| List rulesets | GET | `/repos/{owner}/{repo}/rulesets` |
| Create ruleset | POST | `/repos/{owner}/{repo}/rulesets` |
| Update ruleset | PUT | `/repos/{owner}/{repo}/rulesets/{ruleset_id}` |

---

## Workspace create payload template

```json
{
  "data": {
    "type": "workspaces",
    "attributes": {
      "name": "<WORKSPACE_NAME>",
      "terraform-version": "<TF_VERSION>",
      "auto-apply": false,
      "execution-mode": "remote",
      "working-directory": "",
      "description": "Managed by GitHub Actions — repo: <ORG/REPO>"
    }
  }
}
```

## VCS link PATCH payload template

```json
{
  "data": {
    "type": "workspaces",
    "attributes": {
      "vcs-repo": {
        "identifier": "<ORG/REPO>",
        "oauth-token-id": "<OAUTH_TOKEN_ID>",
        "branch": "main",
        "default-branch": true
      },
      "working-directory": "",
      "trigger-prefixes": []
    }
  }
}
```

## Variable create payload template

```json
{
  "data": {
    "type": "vars",
    "attributes": {
      "key": "<KEY>",
      "value": "<VALUE>",
      "sensitive": false,
      "category": "terraform",
      "hcl": false
    },
    "relationships": {
      "workspace": {
        "data": { "type": "workspaces", "id": "<WORKSPACE_ID>" }
      }
    }
  }
}
```

`category` must be `"terraform"` (for `TF_VAR_*` Terraform variables) or
`"env"` (for environment variables available to the runner).

---

## Idempotency rules

The workflow must be safe to re-run at any time:

**HCP Terraform workspace**
- If the workspace already exists, skip creation but still run the VCS link and
  variable steps (PATCH/POST are idempotent for the same values).
- If a variable already exists, catch the `422` from the API and use
  `PATCH /api/v2/vars/{var_id}` to update. Add a comment explaining this.

**GitHub Environments**
- `PUT /repos/{owner}/{repo}/environments/{env_name}` is always an upsert —
  safe to call on every run. No existence check needed.
- For environment variables: attempt `POST`; on HTTP `409` retry with `PATCH`.
  Log which path was taken.

**Repository ruleset**
- Check for an existing ruleset by name before creating (see ruleset section).
  Use `PUT` to update if found, `POST` to create if not.

**Summary**
- The `$GITHUB_STEP_SUMMARY` must always show whether each resource was
  created or already existed / updated.

---

## Optional extensions

Include these as commented-out steps or a follow-up section when the user
indicates they need them:

**Assign to HCP Project**
```bash
curl -s --fail \
  --request POST \
  --header "Authorization: Bearer $TFE_TOKEN" \
  --header "Content-Type: application/vnd.api+json" \
  --data '{"data":[{"type":"workspaces","id":"'"$WORKSPACE_ID"'"}]}' \
  "$TFE_ADDRESS/api/v2/projects/$PROJECT_ID/relationships/workspaces"
```

**Grant team access**
```bash
curl -s --fail \
  --request POST \
  --header "Authorization: Bearer $TFE_TOKEN" \
  --header "Content-Type: application/vnd.api+json" \
  --data '{
    "data": {
      "type": "team-workspaces",
      "attributes": {"access": "write"},
      "relationships": {
        "team":      {"data":{"type":"teams","id":"'"$TEAM_ID"'"}},
        "workspace": {"data":{"type":"workspaces","id":"'"$WORKSPACE_ID"'"}}
      }
    }
  }' \
  "$TFE_ADDRESS/api/v2/team-workspaces"
```

---

## Quality checklist (verify before presenting output)

**Platform repo (reusable workflow)**
- [ ] Top-level trigger is `workflow_call` only — no `push`, `schedule`, etc.
- [ ] All caller-tunable values are explicit `inputs` with `type`, `required`, `default`
- [ ] `secrets` block declares `TFE_TOKEN`, `TF_VCS_OAUTH_TOKEN_ID`, `GITHUB_PAT` with `required: true` (TF_CLOUD_ORG is now in config JSON)
- [ ] `outputs` block exposes `workspace_id` and `workspace_created`
- [ ] All `curl` calls capture HTTP status separately and `exit 1` on non-2xx
- [ ] The `resolve` step correctly merges create vs. fetch IDs
- [ ] `setup-environments` step loops over all environments in `inputs.environments`
- [ ] Approval gate is applied **only** when env name equals `inputs.production_environment` (exact string match)
- [ ] Reviewer team slugs are resolved to node IDs before building the protection rules payload
- [ ] `GCP_PROJECT_ID` environment variable is set per environment from `inputs.gcp_project_ids` JSON
- [ ] Environment variable upsert handles HTTP 409 (POST → PATCH fallback)
- [ ] `setup-ruleset` step is guarded by `if: inputs.create_ruleset == true`
- [ ] Ruleset existence check by name before create vs. update
- [ ] `required_status_checks` rule is omitted when input array is empty
- [ ] Idempotency behaviour documented in code comments for all three resource types
- [ ] `$GITHUB_STEP_SUMMARY` covers workspace, environments, and ruleset rows
- [ ] `CODEOWNERS` file generated for `.github/workflows/`
- [ ] Release workflow generated to move major-version tag
- [ ] Self-hosted runner note added if user specified non-`ubuntu-latest`

**Application repo (caller)**
- [ ] Caller file is under 25 lines — no logic, only `uses:` + `with:` + `secrets:`
- [ ] `project-config.json` present in repo root with all `<PLACEHOLDER>` values filled
- [ ] `config` input passes the parsed and comment-stripped JSON from `project-config.json`
- [ ] Pinned to a tag (`@v1`), not `@main`
- [ ] Triggered on `push` (path-filtered) + `workflow_dispatch`
- [ ] Uses `secrets: inherit` (or explicit secrets block for strict scoping)

**Explanation**
- [ ] Required secrets table included with PAT scope requirements
- [ ] OAuth token ID retrieval snippet included
- [ ] Org-level secret setup instructions include `GITHUB_PAT` (TF_CLOUD_ORG noted as moved to config JSON)
- [ ] `project-config.sample.json` schema explained to user
- [ ] Approval gate tier logic explained (exact-match rule called out)
- [ ] Versioning/tag strategy explained