# /branch — Create a new git branch

Follow @git-conventions.mdc for branch naming, base branch, and repo rules.

## Purpose

Create a correctly named branch from the right base in **liftrex-api** or **liftrex-web**.

## Inputs

Infer from the user's message, or ask if missing:

| Input | Required | Default |
|-------|----------|---------|
| Repo | Yes | Detect from changed files or context; ask if unclear |
| Type | Yes | `feature` |
| Description | Yes | Short kebab-case slug from the task |

Valid types: `feature`, `fix`, `chore`, `hotfix`

## Detect repo

`liftrex-api` and `liftrex-web` are **separate git repos**. Run all git commands in the target app repo (`cd liftrex-api` or `cd liftrex-web`). Never create branches from the monorepo root.

## Preconditions

- Branch name follows `type/short-kebab-description` per @git-conventions.mdc
- Base branch: `development` for `feature/*`, `fix/*`, `chore/*`; `main` for `hotfix/*` only
- If uncommitted changes exist on the current branch, warn the user — offer to stash, commit (`/commit`), or discard before switching

## Workflow

1. Resolve repo, type, and description. Build branch name:

```
{type}/{short-kebab-description}
```

2. `cd` into the target repo.

3. Run in parallel:

```bash
git status
git branch --show-current
git fetch origin development main
```

4. If dirty working tree, stop and ask how to handle uncommitted changes.

5. Check out and update the base branch:

```bash
# feature, fix, chore
git checkout development
git pull origin development

# hotfix only
git checkout main
git pull origin main
```

6. Create the branch:

```bash
git checkout -b {type}/{short-kebab-description}
```

7. Push and set upstream (unless user asked for local-only):

```bash
git push -u origin HEAD
```

## Constraints

- Never create branches named `main` or `development`
- Never branch `feature/*`, `fix/*`, or `chore/*` from `main`
- Lowercase kebab-case only; max 50 chars after the prefix
- Never update git config
- Never force-push
- Only create the branch when the user invoked `/branch` or explicitly asked to create a branch

## Before finishing

Report:

- Repo (`liftrex-api` or `liftrex-web`)
- Base branch used
- New branch name
- Whether it was pushed to `origin`
- Suggested next steps: make changes → `/commit` → `/pr-development` → merge → `/pr-main` (same branch → `main`)
