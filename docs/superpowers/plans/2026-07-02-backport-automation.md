# Backport Automation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a GitHub Actions workflow that automatically cherry-picks merged PRs onto other distro branches when `backport/*` labels are present.

**Architecture:** A single workflow file fires on `pull_request closed` events, dynamically builds a matrix from the PR's `backport/*` labels, and runs one job per target branch — cherry-picking, pushing a new branch, and opening a ready-for-review (or draft on conflict) PR.

**Tech Stack:** GitHub Actions, `gh` CLI (pre-installed on `ubuntu-latest`), `git`, `GITHUB_TOKEN`

---

## File Structure

| File | Action | Responsibility |
|---|---|---|
| `.github/workflows/backport.yml` | Create | The entire backport workflow |

No other files need touching. Labels are created via `gh` CLI in Task 1.

---

### Task 1: Create the three backport labels in the GitHub repo

**Files:**
- No file changes — one-time `gh` CLI setup

- [ ] **Step 1: Create the labels**

Run (from the repo root):
```bash
gh label create "backport/ubuntu-jammy"    --color "0075ca" --description "Cherry-pick this PR onto ubuntu-jammy"
gh label create "backport/ubuntu-noble"    --color "e4e669" --description "Cherry-pick this PR onto ubuntu-noble"
gh label create "backport/ubuntu-resolute" --color "d93f0b" --description "Cherry-pick this PR onto ubuntu-resolute"
```

Expected: each command prints `✓ Label "backport/ubuntu-*" created in ZPascal/bosh-linux-stemcell-builder`

- [ ] **Step 2: Verify labels exist**

```bash
gh label list | grep backport
```

Expected output (order may vary):
```
backport/ubuntu-jammy     Cherry-pick this PR onto ubuntu-jammy     #0075ca
backport/ubuntu-noble     Cherry-pick this PR onto ubuntu-noble     #e4e669
backport/ubuntu-resolute  Cherry-pick this PR onto ubuntu-resolute  #d93f0b
```

---

### Task 2: Write the backport workflow file

**Files:**
- Create: `.github/workflows/backport.yml`

- [ ] **Step 1: Create the workflow file**

Create `.github/workflows/backport.yml` with this exact content:

```yaml
name: Backport

on:
  pull_request:
    types: [closed]

permissions:
  contents: write
  pull-requests: write

jobs:
  build-matrix:
    if: github.event.pull_request.merged == true
    runs-on: ubuntu-latest
    outputs:
      targets: ${{ steps.labels.outputs.targets }}
    steps:
      - id: labels
        name: Extract backport targets from labels
        env:
          LABELS: ${{ toJson(github.event.pull_request.labels.*.name) }}
        run: |
          targets=$(echo "$LABELS" | jq -c '[.[] | select(startswith("backport/")) | ltrimstr("backport/")]')
          echo "targets=$targets" >> "$GITHUB_OUTPUT"

  backport:
    needs: build-matrix
    if: needs.build-matrix.outputs.targets != '[]'
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        target: ${{ fromJson(needs.build-matrix.outputs.targets) }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Configure git identity
        run: |
          git config user.name  "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"

      - name: Check if backport branch already exists
        id: check
        run: |
          branch="backport/${{ github.event.pull_request.number }}-to-${{ matrix.target }}"
          if git ls-remote --exit-code origin "refs/heads/$branch" > /dev/null 2>&1; then
            echo "exists=true" >> "$GITHUB_OUTPUT"
          else
            echo "exists=false" >> "$GITHUB_OUTPUT"
          fi
          echo "branch=$branch" >> "$GITHUB_OUTPUT"

      - name: Cherry-pick onto target branch
        if: steps.check.outputs.exists == 'false'
        id: cherrypick
        env:
          PR_NUMBER: ${{ github.event.pull_request.number }}
          PR_TITLE:  ${{ github.event.pull_request.title }}
          TARGET:    ${{ matrix.target }}
          BRANCH:    ${{ steps.check.outputs.branch }}
          MERGE_SHA: ${{ github.event.pull_request.merge_commit_sha }}
          BASE_SHA:  ${{ github.event.pull_request.base.sha }}
        run: |
          git fetch origin "$TARGET"
          git checkout -b "$BRANCH" "origin/$TARGET"

          # Collect all commits in the PR (base..merge_commit)
          mapfile -t commits < <(git log --reverse --pretty=format:'%H' "$BASE_SHA..$MERGE_SHA")

          conflict=false
          for sha in "${commits[@]}"; do
            if ! git cherry-pick "$sha"; then
              git add -A
              git commit --no-edit -m "[conflict] cherry-pick $sha from #$PR_NUMBER"
              conflict=true
            fi
          done

          git push origin "$BRANCH"
          echo "conflict=$conflict" >> "$GITHUB_OUTPUT"

      - name: Open backport PR (clean)
        if: steps.check.outputs.exists == 'false' && steps.cherrypick.outputs.conflict == 'false'
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
          PR_TITLE:  ${{ github.event.pull_request.title }}
          PR_URL:    ${{ github.event.pull_request.html_url }}
          TARGET:    ${{ matrix.target }}
          BRANCH:    ${{ steps.check.outputs.branch }}
        run: |
          gh pr create \
            --title "[Backport $TARGET] $PR_TITLE" \
            --body "Automated backport of #$PR_NUMBER to \`$TARGET\`.

          Original PR: $PR_URL" \
            --base "$TARGET" \
            --head "$BRANCH"

      - name: Open backport PR (conflict — draft)
        if: steps.check.outputs.exists == 'false' && steps.cherrypick.outputs.conflict == 'true'
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
          PR_TITLE:  ${{ github.event.pull_request.title }}
          PR_URL:    ${{ github.event.pull_request.html_url }}
          TARGET:    ${{ matrix.target }}
          BRANCH:    ${{ steps.check.outputs.branch }}
        run: |
          gh pr create \
            --title "[Backport $TARGET] $PR_TITLE" \
            --body "Automated backport of #$PR_NUMBER to \`$TARGET\`.

          Original PR: $PR_URL

          > [!WARNING]
          > **This backport has conflicts.** Check out the branch \`$BRANCH\`, resolve the conflict markers, and mark this PR as ready for review." \
            --base "$TARGET" \
            --head "$BRANCH" \
            --draft

      - name: Skip (branch already exists)
        if: steps.check.outputs.exists == 'true'
        run: |
          echo "Backport branch ${{ steps.check.outputs.branch }} already exists — skipping."
```

- [ ] **Step 2: Verify the file was created**

```bash
cat .github/workflows/backport.yml
```

Expected: the full YAML content above is printed without errors.

- [ ] **Step 3: Commit the workflow**

```bash
git add .github/workflows/backport.yml
git commit -m "ci: add label-triggered backport workflow"
```

Expected: `[ubuntu-jammy <sha>] ci: add label-triggered backport workflow`

---

### Task 3: Push and verify the workflow registers on GitHub

**Files:**
- No changes — push and observe

- [ ] **Step 1: Push the branch**

```bash
git push origin ubuntu-jammy
```

Expected: `Branch 'ubuntu-jammy' set up to track remote branch 'ubuntu-jammy' from 'origin'.`

- [ ] **Step 2: Verify the workflow appears in GitHub Actions**

```bash
gh workflow list
```

Expected output includes:
```
Backport    active    backport.yml
```

---

### Task 4: Smoke-test the workflow with a real PR

**Files:**
- No changes — end-to-end test

- [ ] **Step 1: Create a trivial test branch and PR targeting ubuntu-jammy**

```bash
git checkout -b test/backport-smoke-test
echo "# backport smoke test" >> README.md
git add README.md
git commit -m "test: trivial commit for backport smoke test"
git push origin test/backport-smoke-test
```

- [ ] **Step 2: Open the PR with a backport label**

```bash
gh pr create \
  --title "test: backport smoke test" \
  --body "Testing backport automation." \
  --base ubuntu-jammy \
  --head test/backport-smoke-test \
  --label "backport/ubuntu-noble"
```

Note the PR number printed (e.g. `#10`).

- [ ] **Step 3: Merge the PR**

```bash
gh pr merge --squash --delete-branch
```

- [ ] **Step 4: Wait for the backport workflow to complete**

```bash
gh run list --workflow backport.yml --limit 5
```

Wait until the run shows `completed` / `success`. Re-run the command if it still shows `in_progress`.

- [ ] **Step 5: Verify the backport PR was created**

```bash
gh pr list --base ubuntu-noble | grep "Backport ubuntu-noble"
```

Expected: a PR titled `[Backport ubuntu-noble] test: backport smoke test` is listed.

- [ ] **Step 6: Close and clean up the smoke-test backport PR**

```bash
# Get the PR number from the previous step output, then:
gh pr close <backport-pr-number> --delete-branch
```

- [ ] **Step 7: Revert the README change on ubuntu-jammy**

```bash
git checkout ubuntu-jammy
git revert HEAD --no-edit
git push origin ubuntu-jammy
```
