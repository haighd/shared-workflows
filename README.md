# shared-workflows

Reusable GitHub Actions workflows for CI/CD and PR automation. Used by [golden-paws-photography](https://github.com/golden-paws-photography/golden-paws-photography) and [trader](https://github.com/predictive-trader/trader).

## Workflows

| Workflow                    | Purpose                                                          |
| --------------------------- | ---------------------------------------------------------------- |
| `copilot-review.yml`        | Request Copilot as PR reviewer                                   |
| `severity-check.yml`        | Categorize review threads by severity, block CI on critical/high |
| `auto-resolve-outdated.yml` | Resolve outdated review threads when new commits push            |

## Adding to a new project

### 1. Create wrapper workflows

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

The severity-check workflow is called from within your project's `ci.yml`:

```yaml
severity-check:
  if: github.event_name == 'pull_request'
  uses: haighd/shared-workflows/.github/workflows/severity-check.yml@68f1c4b # v1.0.1
  with:
    pr_number: ${{ github.event.pull_request.number }}
```

### 2. Create your project CI workflow

Create `.github/workflows/ci.yml` with branch-aware tiering. CI triggers directly on
`pull_request` events — no labels, no `repository_dispatch`, no review-gate orchestration.

The CI workflow determines the tier based on the PR's target branch:

- **Full tier** (PR targets `main`): all checks including E2E / rehearsals
- **Light tier** (PR targets `sprint/*` or other): core checks only (lint, typecheck, tests)

Include a final aggregation job named `CI` that evaluates all upstream results:

```yaml
ci:
  name: CI
  if: always()
  needs: [severity-check, lint, typecheck, tests]
  runs-on: ubuntu-latest
  steps:
    - name: Evaluate results
      run: |
        for result in "${{ needs.severity-check.result }}" "${{ needs.lint.result }}" "${{ needs.typecheck.result }}" "${{ needs.tests.result }}"; do
          if [[ "$result" == "failure" ]]; then
            exit 1
          fi
        done
```

### 3. Apply GitHub rulesets

**Copilot Reviews** (all branches — no deletion block):

```bash
gh api repos/OWNER/REPO/rulesets --method POST --input - <<'EOF'
{
  "name": "Copilot Reviews",
  "target": "branch",
  "enforcement": "active",
  "conditions": {"ref_name": {"include": ["~ALL"], "exclude": []}},
  "rules": [
    {"type": "copilot_code_review", "parameters": {"review_on_push": true, "review_draft_pull_requests": true}}
  ]
}
EOF
```

**main-protection** (main branch only):

```bash
gh api repos/OWNER/REPO/rulesets --method POST --input - <<'EOF'
{
  "name": "main-protection",
  "target": "branch",
  "enforcement": "active",
  "conditions": {"ref_name": {"include": ["refs/heads/main"], "exclude": []}},
  "rules": [
    {"type": "deletion"},
    {"type": "non_fast_forward"},
    {"type": "pull_request", "parameters": {
      "dismiss_stale_reviews_on_push": true,
      "require_code_owner_review": false,
      "require_last_push_approval": false,
      "required_approving_review_count": 0,
      "required_review_thread_resolution": true,
      "allowed_merge_methods": ["merge", "squash", "rebase"]
    }},
    {"type": "required_status_checks", "parameters": {
      "strict_required_status_checks_policy": false,
      "do_not_enforce_on_create": false,
      "required_status_checks": [{"context": "CI / CI", "integration_id": 15368}]
    }}
  ],
  "bypass_actors": [
    {"actor_id": YOUR_USER_ID, "actor_type": "User", "bypass_mode": "always"}
  ]
}
EOF
```

Get your user ID with `gh api user --jq '.id'`.

### 4. Enable auto-delete head branches

```bash
gh api repos/OWNER/REPO -X PATCH -f delete_branch_on_merge=true
```

### 5. Add secrets

| Secret                 | Purpose                                                          |
| ---------------------- | ---------------------------------------------------------------- |
| `COPILOT_REVIEW_TOKEN` | Token with `pull-requests: write` for requesting Copilot reviews |

## Pinning

All consumer projects should pin to a specific commit SHA, not `@main`:

```yaml
uses: haighd/shared-workflows/.github/workflows/copilot-review.yml@68f1c4b # v1.0.1
```

After updating shared-workflows, bump the SHA in each consumer project.

## PR lifecycle

```
PR opened / pushed
  -> Copilot requested as reviewer (copilot-review.yml)
  -> CI runs automatically (ci.yml, branch-aware tiering)
  -> Auto-resolve outdated threads (auto-resolve-outdated.yml)

Developer addresses Copilot feedback
  -> Pushes fixes -> CI re-runs, Copilot re-reviews

Merge requirements (GitHub ruleset on main):
  -> CI / CI status check passes
  -> All review threads resolved
  -> No force push, no branch deletion on main
```
