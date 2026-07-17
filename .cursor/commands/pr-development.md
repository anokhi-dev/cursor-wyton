# /pr-development — Open a PR into `development`

Follow @git-conventions.mdc for branch naming and @.cursor/rules/architecture-flow.mdc for repo context.

## Purpose

Create a pull request from the current feature branch into **`development`** for testing on dev.

Flow (step 1 of 2):

```
feature/login
      │
      ├── PR → development ✅   (/pr-development — this command)
      │      └── merge
      │
      └── PR → main ✅          (/pr-main — same branch, after dev merge)
```

## Detect repo

`liftrex-api` and `liftrex-web` are **separate git repos**. Run all commands in the repo with the changes.

## Preconditions

- Current branch is **not** `main` or `development`
- Branch name follows `feature/*`, `fix/*`, `chore/*`, or `hotfix/*` per @git-conventions.mdc
- Changes are committed (run `/commit` first if there are uncommitted changes — ask user)
- Branch is pushed to remote

## Workflow

1. Run in parallel:
   - `git status`
   - `git branch --show-current`
   - `git log development..HEAD --oneline`
   - `git remote -v` and check upstream tracking

2. If uncommitted changes exist, stop and tell the user to commit first (`/commit`).

3. Push branch if not on remote:

```bash
git push -u origin HEAD
```

4. Draft PR title from commits using conventional commit style:
   - `feat(scope): short description` or `fix(scope): short description`

5. Draft PR body:

```markdown
## Summary
- [1–3 bullets describing what changed and why]

## Test plan
- [ ] Steps to verify on dev environment
```

6. Create PR with `gh`:

```bash
gh pr create --base development --head "$(git branch --show-current)" --title "TITLE" --body "$(cat <<'EOF'
## Summary
...

## Test plan
- [ ] ...

EOF
)"
```

7. Return the PR URL.

## Constraints

- Base branch must be **`development`**
- Never force-push
- Never update git config
- Do not push unless branch is unpushed or user explicitly asked to push

## Before finishing

Report: repo name (`liftrex-api` or `liftrex-web`), source branch, PR URL, and suggested next step: merge on GitHub → test on dev → `/pr-main` from the **same branch** → `main`.
