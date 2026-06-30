---
name: ship
description: Full lifecycle skill — opens a PR with review, or merges & cleans up when the PR is ready. Follows the git workflow from apple-app-scaffolder's AGENTS.md (step 14).
---

# Ship

Handles the complete PR lifecycle: creating a feature branch, pushing, opening a PR with review, and post-merge cleanup (squash merge, delete branch, prune).

## When to Use This Skill

Invoke this skill when the user says things like:
- "ship it"
- "commit push pr"
- "ship this"
- "create a pr"
- "push and open a pr"
- "commit and ship"
- "merge and cleanup"

## Detection

Before starting, detect which mode to run:

```bash
gh pr status --json number,state,headRefName,isDraft,reviews
```

If the current branch already has an open PR → **Mode B** (ship & cleanup).
Otherwise → **Mode A** (create PR).

---

## Mode A — Create PR

### Step 1 — Sanity checks

Run these in parallel:

```bash
git status
git diff
git log --oneline -5
git remote -v
```

Use the output to understand: what branch we're on, what's changed, what the recent commit style looks like, and what the remote is.

If currently on `main` or `master`, create a feature branch first:

```bash
git checkout -b <branch-name-derived-from-changes>
```

Derive the branch name from the diff (e.g. `fix/login-crash`, `feat/onboarding`). Use `git diff --stat` to understand scope.

### Step 2 — Run tests (if available)

If the project has tests, run them before proceeding:

- Xcode project: `xcodebuild test -scheme <SchemeName>` (derive scheme from project.yml or `xcodebuild -list`)
- Node: `npm test` / `pnpm test`
- Go: `go test ./...`
- Rust: `cargo test`

Abort if tests fail. Report failures clearly.

### Step 3 — Fetch and rebase

```bash
git fetch origin
git rebase origin/main
```

If there are conflicts, resolve them (keeping both sides where appropriate), then continue the rebase.

### Step 4 — Stage files

Stage all changed/new files that are relevant to the work. Explicitly exclude:
- `.DS_Store`
- `*.db`, `*.db-shm`, `*.db-wal`
- Any file that looks like it contains secrets (`.env`, `credentials.*`, etc.)
- Prototype/demo directories unless they are part of the deliverable

```bash
git add <relevant files>
git status   # verify staging looks right
```

### Step 5 — Commit

Write a commit message that:
- Uses the imperative mood (`feat:`, `fix:`, `ci:`, `docs:`, `refactor:`)
- First line ≤ 72 characters summarising the **why**, not the what
- Body bullet points for non-obvious details (optional but preferred for large changes)
- Matches the style of recent commits in `git log`

```bash
git commit -m "<message>"
```

### Step 6 — Push

If the branch has no upstream yet:

```bash
git push -u origin <branch>
```

Otherwise:

```bash
git push
```

### Step 7 — Open PR

Derive the PR title and body from the changes. Use `git diff` and `git log` to summarise what changed and why.

```bash
gh pr create \
  --title "<concise title derived from changes>" \
  --body "$(cat <<'EOF'
## Summary

<2-4 bullet points describing what changed and why, derived from the diff>

## Notes

<any reviewer context, migration steps, known limitations — omit section if none>
EOF
)" \
  --base main
```

Capture the PR URL from the output.

### Step 8 — Trigger opencode review

```bash
gh pr comment <PR number> --body "/oc please review this PR and approve if you find it ready to merge"
```

### Step 9 — Report back

Tell the user:
- The branch name
- The commit hash and message
- The PR URL
- That opencode review has been triggered

---

## Mode B — Ship & Cleanup

Use this when a PR already exists for the current branch and the user wants to merge and clean up.

### Step 1 — Verify PR state

```bash
gh pr view --json number,state,mergeable,reviews
```

If the PR is a draft, ask the user if they want to mark it ready first. If reviews haven't approved, ask confirmation.

### Step 2 — Squash merge to main

```bash
gh pr merge <PR number> --squash --delete-branch
```

The `--delete-branch` flag deletes the remote branch automatically.

### Step 3 — Switch to main and pull

```bash
git checkout main
git pull origin main
```

### Step 4 — Prune local branches

```bash
git fetch --prune
```

Clean up the local branch that was just merged:

```bash
git branch -d <branch-name>
```

If the branch has unmerged changes (shouldn't happen after squash merge), use `-D` and warn.

### Step 5 — Report back

Tell the user:
- The PR was squash-merged
- The branch was deleted (remote + local)
- Current branch is `main` with latest changes

---

## Rules

- **Never force-push** to `main` or `master`
- **Never commit** `.env`, secrets, or credential files — warn the user if they're staged
- **Always rebase** onto `origin/main` before pushing to minimise conflicts
- If `gh` is not authenticated, tell the user to run `gh auth login` first
- If the build/tests are known to exist, run them before committing in Mode A and abort if they fail
- In Mode B, verify the PR is approved before merging unless the user explicitly says to skip review
