# CLAUDE.md

This file defines conventions for AI-assisted development on the
`gcp-project-onboarding` platform repo. Follow these rules when modifying,
extending, or reviewing any file in this repository.

---

## Repository purpose

This is a **platform repo**, not an application repo. Its sole output is
`.github/workflows/gcp-project-onboarding.yaml` — a reusable `workflow_call`
workflow consumed by application repos across the organisation. Every decision
here has downstream impact on all callers.

---

## Workflow authoring rules

### Trigger

The reusable workflow must only ever have `workflow_call` as its trigger.
Never add `push`, `schedule`, or `workflow_dispatch` to `gcp-project-onboarding.yaml`.
Those belong in the caller (`onboard-gcp-project.yaml` in the application repo).

### Inputs

All project-specific configuration is passed as a single `config` JSON string
input. **Do not add new individual inputs to the `workflow_call` interface.**
If new configuration is needed, extend the `project-config.json` schema and
parse the new field in the `parse` step (Step 0).

### Step ordering

Steps must always follow this sequence:

```text
0. parse          — validate + extract all fields from config JSON
1. check          — does the HCP workspace already exist?
2. create         — create workspace (skipped if exists)
3. resolve        — merge workspace ID from create or check
4. link-vcs       — attach VCS repository to workspace
5. set-vars       — seed workspace variables
6. setup-environments — create GitHub Environments + set GCP_PROJECT_ID
7. setup-ruleset  — create or update repository ruleset
8. summary        — write $GITHUB_STEP_SUMMARY
```

Never reorder steps. Never merge two steps into one.

### curl conventions

Every `curl` call must:

- Use `-s -w "\n%{http_code}"` — capture status and body separately
- **Never** use `--fail` — always check the HTTP status code explicitly
- `exit 1` with a descriptive message and the response body on any non-2xx

```bash
# Correct pattern
RESPONSE=$(curl -s -w "\n%{http_code}" --request POST ...)
HTTP_STATUS=$(echo "$RESPONSE" | tail -1)
BODY=$(echo "$RESPONSE" | head -n -1)
if [ "$HTTP_STATUS" != "201" ]; then
  echo "ERROR: <description> (HTTP $HTTP_STATUS)"
  echo "$BODY" | jq .
  exit 1
fi
```

### JSON building

Always use `jq -n` with `--arg` / `--argjson` to construct JSON payloads.
Never build JSON by concatenating shell strings — it breaks on special
characters and is impossible to review safely.

```bash
# Correct
PAYLOAD=$(jq -n --arg name "$WS_NAME" --argjson auto "$AUTO_APPLY" \
  '{data:{type:"workspaces",attributes:{name:$name,"auto-apply":$auto}}}')

# Wrong — do not do this
PAYLOAD="{\"data\":{\"type\":\"workspaces\",\"attributes\":{\"name\":\"$WS_NAME\"}}}"
```

### Secrets

The three declared secrets are `TFE_TOKEN`, `TF_VCS_OAUTH_TOKEN_ID`, and
`GITHUB_PAT`. Do not add new secrets to the `workflow_call` interface without
a platform team discussion — every caller repo must have the secret configured.

---

## Idempotency — non-negotiable

Every step must be safe to re-run on an already-onboarded repo. The rules are:

| Resource | Idempotency mechanism |
| --- | --- |
| HCP workspace | Existence check before create; step skipped if exists |
| HCP workspace variables | `POST` → on `422` fetch var ID and `PATCH` |
| GitHub Environments | `PUT` is always an upsert — no existence check needed |
| GitHub Environment variables | `POST` → on `409` use `PATCH` |
| Repository ruleset | List by name → `PUT` if found, `POST` if not |

If you add a new resource provisioning step, it must follow one of these
patterns. Document which pattern it uses in a code comment.

---

## Approval gate rule

The production approval gate uses an **exact string equality check**:

```bash
if [ "$ENV_NAME" = "$PROD_ENV" ]; then
```

Never change this to a substring match (`contains`, `grep`, `==*`).
A repo named `my-production-mirror` must not receive approval gates.

---

## config JSON schema

The `parse` step (Step 0) is the single place where `config` JSON is read and
validated. All subsequent steps read from `steps.parse.outputs.*` — they must
never reference `inputs.config` directly. This keeps JSON parsing centralised
and makes the step outputs the stable interface inside the job.

When adding a new config field:

1. Add it to `project-config.sample.json` with a `_comment` and a sensible default
2. Parse it in the `parse` step with a `// "default"` fallback using `jq`
3. Export it as a step output
4. Reference it in the relevant step via `steps.parse.outputs.<field>`

---

## File extension

All workflow files use `.yaml` (not `.yml`). This applies to files in
`.github/workflows/` and in `templates/`.

---

## templates/ directory

The `templates/` directory contains the canonical copies of files application
teams copy into their own repos. When `gcp-project-onboarding.yaml` changes,
update `templates/gcp-project-onboarding.yaml` to match. They must always
be in sync.

---

## Versioning

Releases follow semver. The `release.yaml` workflow moves the major-version
tag (`v1`) forward automatically on every `v1.x.x` push.

- **Patch** (`v1.0.x`) — bug fixes, no interface changes
- **Minor** (`v1.x.0`) — new optional config fields, backward compatible
- **Major** (`v2.0.0`) — breaking changes to `workflow_call` interface or
  `project-config.json` required fields; requires platform team sign-off and
  a migration guide for existing callers

---

## What not to change without platform team review

- The `workflow_call` inputs/secrets/outputs interface
- The step ordering or step IDs
- The `project-config.json` required fields (adding optional fields is fine)
- The `CODEOWNERS` file
- The approval gate exact-match logic
