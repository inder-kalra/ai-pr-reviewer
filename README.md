# ai-pr-reviewer

Generic, reusable AI code review for **any** repo — hosted once, called from anywhere.
Auth via a Claude subscription OAuth token (`CLAUDE_CODE_OAUTH_TOKEN`), so there's no
per-run API billing.

Two tracks share this one engine + one secret:

| Track | Workflow | When | Output |
|---|---|---|---|
| **PR reviewer** (pre-merge) | `.github/workflows/review.yml` *(TODO)* | On open PRs | Inline comments + merge gate |
| **Codebase auditor** (post-merge) | `.github/workflows/audit.yml` | Scheduled / on-demand | GitHub Issues |

## Use the codebase auditor in your repo
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

The audit prompt lives in `prompts/audit.md` — edit it centrally and every consumer picks
it up on its next run.

## Versioning
Consumers pin `@v1`. The `audit.yml` second checkout pins `ref: v1` to keep the prompt in
lockstep — bump both together when cutting a new major.
