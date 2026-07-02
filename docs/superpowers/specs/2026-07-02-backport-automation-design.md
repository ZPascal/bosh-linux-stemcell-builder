# Backport Automation Design

**Date:** 2026-07-02  
**Scope:** Label-triggered cross-branch cherry-pick PRs within `ZPascal/bosh-linux-stemcell-builder`

---

## Goal

When a PR is merged to any active distro branch, contributors can request automatic cherry-pick PRs onto other distro branches by attaching labels before or at merge time. This mirrors the Grafana backport model.

---

## Target Branches

The three active distro branches eligible as backport targets:

- `ubuntu-jammy`
- `ubuntu-noble`
- `ubuntu-resolute`

---

## Labels

Three labels must be created in the repo (one-time setup):

| Label | Meaning |
|---|---|
| `backport/ubuntu-jammy` | Cherry-pick this PR onto `ubuntu-jammy` |
| `backport/ubuntu-noble` | Cherry-pick this PR onto `ubuntu-noble` |
| `backport/ubuntu-resolute` | Cherry-pick this PR onto `ubuntu-resolute` |

Adding a new distro branch in future only requires creating a new label — no workflow changes needed.

---

## Workflow Trigger

File: `.github/workflows/backport.yml`

- **Event:** `pull_request` with type `closed`
- **Guard:** `github.event.pull_request.merged == true`
- **Matrix:** dynamically built from the merged PR's labels, filtered to names matching `backport/*`

If no `backport/*` labels are present, no jobs run.

---

## Per-Target Job Steps

For each matched label, one job runs:

1. Check out the repository
2. Configure `git` identity as `github-actions[bot]`
3. Fetch the target branch (e.g. `ubuntu-noble`)
4. Check if backport branch `backport/<pr-number>-to-<target>` already exists — if yes, skip (idempotency)
5. Create and check out the new branch from the target branch
6. Cherry-pick all commits from the merged PR (`git cherry-pick <sha>...<sha>`)
7. Push the branch to `origin`
8. Open a PR via `gh pr create`:
   - **Title:** `[Backport <target>] <original PR title>`
   - **Base:** the target branch
   - **Body:** links to original PR, lists cherry-picked commits
   - **State:** ready for review (clean) or draft (conflict)

---

## Conflict Handling

If `git cherry-pick` exits non-zero:

1. Stage all conflicted files (`git add -A`)
2. Commit with message `[conflict] cherry-pick from #<pr-number>` 
3. Push the branch
4. Open a **draft** PR with a conflict notice in the body instructing the author to resolve manually
5. Exit job with code 0 (the draft PR is the signal; a failed job would obscure the actual state)

---

## Permissions

The workflow declares:

```yaml
permissions:
  contents: write
  pull-requests: write
```

Uses `GITHUB_TOKEN` — no additional secrets required.

---

## PR Naming Convention

| Field | Format |
|---|---|
| Branch | `backport/<pr-number>-to-<target-branch>` |
| Title | `[Backport ubuntu-noble] <original title>` |
| Body | Auto-generated, links original PR |

---

## Out of Scope

- Upstream (`cloudfoundry`) → fork sync (separate concern)
- `ubuntu-bionic` and `ubuntu-xenial/*` branches (EOL, excluded)
- Auto-merge of backport PRs (always requires human review)
