# /commit — Create a conventional commit

Follow @git-conventions.mdc for commit message format and branch rules.

## Detect repo

`liftrex-api` and `liftrex-web` are **separate git repos**. Run all git commands in the repo that contains the changed files (check `git rev-parse --show-toplevel`).

## Workflow

1. Run in parallel:
   - `git status`
   - `git diff` (staged + unstaged)
   - `git log -5 --oneline` (match existing style)

2. Verify current branch matches @git-conventions.mdc (`feature/*`, `fix/*`, `chore/*`, or `hotfix/*`). Warn if on `main` or `development` — do not commit directly to protected branches unless the user explicitly asked.

3. Stage only relevant files. Never stage `.env`, credentials, or secrets.

4. Draft a **conventional commit** message from the diff:
   - Format: `type(scope): subject`
   - Subject explains **why** in imperative mood, max 72 chars
   - Body (if needed): 1–3 sentences of context

5. Commit using a HEREDOC:

```bash
git commit -m "$(cat <<'EOF'
type(scope): subject line here

Optional body explaining why this change was made.

EOF
)"
```

6. Run `git status` after commit to confirm success.

## Constraints

- Only create a commit when the user invoked `/commit` — this is explicit permission
- Never use `git commit --amend` unless the user explicitly requests it
- Never skip hooks (`--no-verify`) unless the user explicitly requests it
- Never update git config
- If pre-commit hook fails, fix the issue and create a **new** commit (do not amend)

## Before finishing

Report: branch name, commit hash, files committed, and the full commit message used.
