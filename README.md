# shared-workflows

Reusable GitHub Actions workflows for CI/CD and PR automation. Used by [golden-paws-photography](https://github.com/golden-paws-photography/golden-paws-photography) and [trader](https://github.com/predictive-trader/trader).

## Workflows

| Workflow                    | Purpose                                                          |
| --------------------------- | ---------------------------------------------------------------- |
| `copilot-review.yml`        | Request Copilot as PR reviewer                                   |
| `severity-check.yml`        | Categorize review threads by severity, block CI on critical/high |
| `auto-resolve-outdated.yml` | Resolve outdated review threads when new commits push            |

## Adding to a new project

### 1. Create thin wrapper workflows

Each project needs wrapper workflows that call the shared ones. Create these in `.github/workflows/`:

**`.github/workflows/copilot-review.yml`**

```yaml
name: Request Copilot Review
on:
  pull_request:
    types: [opened, synchronize, reopened]
permissions:
  pull-requests: write
jobs:
  copilot:
    uses: haighd/shared-workflows/.github/workflows/copilot-review.yml@68f1c4b # v1.0.1
    with:
      pr_number: ${{ github.event.pull_request.number }}
    secrets:
      copilot_review_token: ${{ secrets.COPILOT_REVIEW_TOKEN }}
```

**`.github/workflows/auto-resolve-outdated.yml`**

```yaml
name: Auto-Resolve Outdated Threads
on:
  pull_request:
    types: [synchronize]
permissions:
  contents: read
  pull-requests: write
jobs:
  resolve:
    uses: haighd/shared-workflows/.github/workflows/auto-resolve-outdated.yml@68f1c4b # v1.0.1
    with:
      pr_number: ${{ github.event.pull_request.number }}
```

The severity-check workflow is called from within `ci-pipeline.yml` (not as a standalone wrapper):

```yaml
severity-check:
  needs: gate
  uses: haighd/shared-workflows/.github/workflows/severity-check.yml@68f1c4b # v1.0.1
  with:
    pr_number: ${{ needs.gate.outputs.pr_number }}
```

### 2. Create reviewer config

Create `.github/reviewer-config.yml`:

```yaml
required_reviewers:
  copilot:
    login: copilot-pull-request-reviewer[bot]
    completion_mode: explicit
    done_patterns:
      - "Pull request overview"

optional_reviewers: {}
timeout_minutes: 30
```

### 3. Copy project-specific workflows

Copy these from an existing project and adapt:

- `review-gate.yml` -- manages `ready-for-ci` / `ready-to-merge` labels
- `ci-pipeline.yml` -- triggered by `repository_dispatch: ci-ready`, runs your test suite
- `bypass.yml` -- passphrase-protected emergency bypass via `workflow_dispatch`

### 4. Add secrets

| Secret                 | Purpose                                                                  |
| ---------------------- | ------------------------------------------------------------------------ |
| `COPILOT_REVIEW_TOKEN` | Token with `pull-requests: write` for requesting Copilot reviews         |
| `BYPASS_PASSPHRASE`    | Passphrase for the bypass workflow (human-only, never share with agents) |

### 5. Apply GitHub ruleset

```bash
gh api repos/OWNER/REPO/rulesets --method POST --input - <<'EOF'
{
  "name": "main-protection",
  "target": "branch",
  "enforcement": "active",
  "conditions": {
    "ref_name": { "include": ["refs/heads/main"], "exclude": [] }
  },
  "rules": [
    {"type": "deletion"},
    {"type": "non_fast_forward"},
    {"type": "pull_request", "parameters": {
      "dismiss_stale_reviews_on_push": true,
      "require_code_owner_review": false,
      "require_last_push_approval": false,
      "required_approving_review_count": 1,
      "required_review_thread_resolution": true
    }},
    {"type": "required_status_checks", "parameters": {
      "strict_required_status_checks_policy": false,
      "required_status_checks": [{"context": "CI Pipeline"}]
    }}
  ],
  "bypass_actors": [
    {"actor_id": YOUR_USER_ID, "actor_type": "User", "bypass_mode": "always"}
  ]
}
EOF
```

Get your user ID with `gh api user --jq '.id'`.

## Pinning

All consumer projects should pin to a specific commit SHA, not `@main`:

```yaml
uses: haighd/shared-workflows/.github/workflows/copilot-review.yml@68f1c4b # v1.0.1
```

After updating shared-workflows, bump the SHA in each consumer project.

## PR lifecycle

```
PR opened
  -> Copilot requested as reviewer
  -> ready-for-ci and ready-to-merge labels stripped

Bot reviews complete + 0 unresolved threads
  -> ready-for-ci label added
  -> ci-ready repository_dispatch event fired

CI pipeline runs automatically
  -> severity check -> lightweight CI -> full tests
  -> post-CI thread re-check (race window protection)
  -> ready-to-merge label added

Merge
  -> Ruleset enforces: 1 approval + CI Pipeline passing + 0 unresolved threads
```
