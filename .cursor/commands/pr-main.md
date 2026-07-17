# /pr-main — Open a PR into `main`

Follow @git-conventions.mdc and the branch flow in `liftrex-api/.github/BRANCH_PROTECTION.md` (or `liftrex-web/.github/BRANCH_PROTECTION.md`).

## Purpose

Promote tested work to **`main`** from the **same feature branch** — after it is merged into `development` first.

```
feature/login
      │
      ├── PR → development ✅   (/pr-development)
      │      └── merge
      │
      └── PR → main ✅          (/pr-main)
```

Both PRs use the **same branch** as head (e.g. `feature/login`). Do **not** open `development → main` for feature work.

## Detect repo

`liftrex-api` and `liftrex-web` are **separate git repos**. Run all commands in the repo with the changes.

## Preconditions

- Current branch is **not** `main` or `development`
- Branch name follows `feature/*`, `fix/*`, `chore/*`, or `hotfix/*` per @git-conventions.mdc
- **`/pr-development` PR is already merged** into `origin/development`
- Changes tested on dev before promoting to main
- All commits follow @git-conventions.mdc

Verify the branch tip is in `development` history (required by `verify-development-merge`):

```bash
git fetch origin development main
git merge-base --is-ancestor HEAD origin/development && echo "OK" || echo "BLOCKED — merge /pr-development first"
```

If blocked, stop and instruct the user to:
1. Open `/pr-development` and merge to `development` first
2. Then run `/pr-main` from the **same feature branch**

## Workflow

1. Run in parallel:
   - `git status`
   - `git branch --show-current`
   - `git fetch origin development main`
   - `git log origin/main..HEAD --oneline`

2. Confirm current branch is a feature/fix/chore branch (not `development` or `main`).

3. Run the ancestor check above. If blocked, stop.

4. Push branch if not on remote:

```bash
git push -u origin HEAD
```

5. Draft PR title from commits (conventional commit style):
   - `feat(scope): short description` or `fix(scope): short description`

6. Draft PR body:

```markdown
## Summary
- [What is being promoted to production]

## Test plan
- [ ] Verified on development environment
- [ ] CI / lint checks pass

## Branch flow
- [ ] Merged to development via /pr-development first
```

7. Create PR with `gh` — head is the **current feature branch**:

```bash
gh pr create --base main --head "$(git branch --show-current)" --title "TITLE" --body "$(cat <<'EOF'
## Summary
...

## Test plan
- [ ] ...

EOF
)"
```

8. Return the PR URL. GitHub runs **Verify Development Merge** (`check-development`); it passes when the feature branch tip is already in `development` history.

## Constraints

- Base branch must be **`main`**
- Head branch must be the **current feature/fix/chore branch** — not `development`
- Never open `development → main` for single-feature promotion (use this two-PR flow instead)
- Never force-push to `main` or `development`
- Never update git config

## Before finishing

Report: repo name, source branch (feature branch name), PR URL, whether `check-development` is expected to pass, and reminder that merged feature branches are auto-deleted by `delete-merged-branch.yml`.
