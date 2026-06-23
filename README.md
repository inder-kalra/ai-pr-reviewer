# ai-pr-reviewer

Generic, reusable AI code review for **any** repo — hosted once, called from anywhere.
Auth via a Claude subscription OAuth token (`CLAUDE_CODE_OAUTH_TOKEN`), so there's no
per-run API billing.

Two tracks share this one engine + one secret:

| Track | Workflow | When | Output |
|---|---|---|---|
| **PR reviewer** (pre-merge) | `.github/workflows/review.yml` | On open PRs | Inline comments + soft merge gate |
| **Codebase auditor** (post-merge) | `.github/workflows/audit.yml` | Scheduled / on-demand | GitHub Issues |

## Track 1 — the PR reviewer (pre-merge)

On every pull request, a senior-staff-engineer Claude reviews **only the PR diff** (with
surrounding file context as needed) for what would break in production — security holes,
correctness bugs, systemic risk, regressions. It is architecture-focused, not a linter.

It posts **inline review comments** on the changed lines (via the GitHub pull-request
reviews API) plus one summary review, then emits a deterministic verdict.

### Soft merge gate (and the private-repo caveat)
The reviewer's last stdout line is exactly `VERDICT: CRITICAL` or `VERDICT: CLEAN`. A
final workflow step greps that line and `exit 1`s on a CRITICAL (or if no verdict is
found at all, so silent breakage stays visible). That turns the check red.

This is a **soft gate**: a red ❌ check that is visible but does NOT physically block the
merge button. Hard blocking needs a *required status check* in branch protection, which
needs GitHub Pro on **private** repos. On a private repo without Pro, treat the red check
as a stop sign — read the inline CRITICAL comments before merging. On a public repo (or
with Pro), add `review` as a required status check to make it a hard gate.

### Onboard your repo
Copy `templates/ai-review.yml` to `.github/workflows/ai-review.yml` in your repo, then set
the `CLAUDE_CODE_OAUTH_TOKEN` secret. It calls:

```yaml
on:
  pull_request: {}
jobs:
  review:
    uses: inder-kalra/ai-pr-reviewer/.github/workflows/review.yml@v1
    permissions: { contents: read, pull-requests: write }
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
    with:
      focus: ""   # optional project-specific emphasis
```

The review prompt lives in `prompts/review.md` — edit it centrally and every consumer
picks it up on its next PR.

## Track 2 — the codebase auditor (post-merge)

Copy `templates/ai-audit.yml` to `.github/workflows/ai-audit.yml` in your repo, then set
the `CLAUDE_CODE_OAUTH_TOKEN` secret. It calls:

```yaml
jobs:
  audit:
    uses: inder-kalra/ai-pr-reviewer/.github/workflows/audit.yml@v1
    permissions: { contents: read, issues: write }
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

It runs on a schedule, sweeps the whole default branch, and files **GitHub Issues** for
accumulated drift (no PR, no gate). The audit prompt lives in `prompts/audit.md` — edit it
centrally and every consumer picks it up on its next run.

## Versioning
Consumers pin `@v1`. Each reusable workflow's second checkout also pins `ref: v1` to keep
its prompt in lockstep — bump both tracks together when cutting a new major.
