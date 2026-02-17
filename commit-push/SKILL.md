---
name: commit-push
description:
  Stage changes, create a commit with a conventional message, and push to the
  current branch.
disable-model-invocation: true
argument-hint: "[optional commit message]"
---

Stage, commit, and push changes on the current branch.

If the user provides $ARGUMENTS, use it as the commit message. Otherwise,
analyze the changes and generate a conventional commit message.

## Pre-flight: Auto-detect Environment

### Step 0.1: Find Pre-commit Hook Info

Check if the project uses a pre-commit hook that auto-formats code:

- Look for `husky` in `package.json` (Node.js projects)
- Look for `.pre-commit-config.yaml` (Python/general projects)
- Look for `.husky/` directory

If a pre-commit hook exists that runs formatting, there's no need to run checks
manually before committing.

### Step 0.2: Find Verification Command (optional)

If no pre-commit hook is found, detect a verification command:

1. **package.json**: Look for scripts: `check` → `lint`
2. **Makefile**: Look for targets: `check` → `lint`
3. **Cargo.toml**: `cargo clippy`
4. **pyproject.toml**: `ruff check`
5. **go.mod**: `go vet ./...`

Store as `$VERIFY_CMD`. If nothing found, skip verification.

## Steps

### 1. Review changes

```bash
git status
git diff
git diff --cached
```

Identify what changed. Exclude files that shouldn't be committed:

- `.env` or credentials files
- Vendored/dead files that are no longer used
- Build output or generated code that should be gitignored

### 2. Stage relevant files

Stage files individually by name. Do not use `git add -A` or `git add .`.

### 3. Create commit

Use a conventional commit message following this format:

```
<type>: <short description>

<optional body with details>
```

Types: `feat`, `fix`, `docs`, `refactor`, `chore`, `test`, `perf`

Pass the message via HEREDOC:

```bash
git commit -m "$(cat <<'EOF'
<message here>

EOF
)"
```

### 4. Push

```bash
git push
```

If the branch has no upstream, use:

```bash
git push -u origin $(git branch --show-current)
```

If push is rejected (non-fast-forward), stop and ask the user how to proceed. Do
not force push without explicit permission.

### 5. Confirm

Show the commit hash, message, and branch. Confirm the push succeeded.
