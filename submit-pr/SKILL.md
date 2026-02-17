---
name: submit-pr
description:
  Create a pull request with proper documentation, conventional commits, and
  complete PR template.
disable-model-invocation: true
---

Create a well-documented pull request following project conventions, including
proper branch naming, conventional commits, and a complete PR description.

## Pre-flight: Auto-detect Environment

### Step 0.1: Find Instruction File

Check for a project instruction file (stop at first match):

1. `CLAUDE.md`
2. `AGENTS.md`
3. `CONVENTIONS.md`
4. `CONTRIBUTING.md`

Store as `$INSTRUCTION_FILE`.

### Step 0.2: Find Verification Command

Detect the verification command:

1. **package.json**: Scripts in order: `check:ci` → `check` → `lint`
2. **Makefile**: Targets: `check` → `lint` → `test`
3. **Cargo.toml**: `cargo clippy && cargo test`
4. **pyproject.toml**: Detect `ruff` or `flake8` → use appropriate command
5. **go.mod**: `go vet ./... && go test ./...`

Store as `$VERIFY_CMD`. If nothing found, skip verification.

### Step 0.3: Find Additional Testing Requirements

If `$INSTRUCTION_FILE` exists, check if it defines files that trigger additional
testing requirements (e.g., testing matrices, checklists). Store these patterns
for Step 2.3.

## Phase 1: Verify Current State

### Step 1.1: Check Git Status

```bash
git status
```

Ensure:

- No uncommitted changes (unless intentional)
- On the correct feature branch (not on main)
- All relevant files are tracked

### Step 1.2: Verify Checks Pass

If `$VERIFY_CMD` was found, run it:

```bash
$VERIFY_CMD
```

If this fails, stop and ask the user if they want to:

- Fix the issues before proceeding
- Proceed anyway (not recommended)
- Abort the PR submission

### Step 1.3: Check Branch Name

```bash
git branch --show-current
```

Verify it follows conventional naming:

- `feat/<name>` — New features
- `fix/<name>` — Bug fixes
- `docs/<name>` — Documentation changes
- `refactor/<name>` — Code refactoring
- `chore/<name>` — Maintenance tasks
- `test/<name>` — Test additions/updates

If the branch name doesn't follow conventions, ask the user if they want to
rename it.

## Phase 2: Review Changes and Gather Information

### Step 2.1: Get Commit History

```bash
git log main..HEAD --oneline
```

Review the commits. Check if they follow Conventional Commits. If not, suggest
cleaning up.

### Step 2.2: Get Detailed Diff

```bash
git diff main...HEAD --stat
git diff main...HEAD
```

Analyze what files were modified, features added/changed, bugs fixed, whether
tests were added, and whether documentation was updated.

### Step 2.3: Check for Additional Testing Requirements

If the instruction file defines files that trigger testing checklists, check if
any changed files match. If so, read the referenced testing documentation and
include the checklist in the PR body.

### Step 2.4: Ask User for PR Details

Ask the user directly:

**Question 1: PR Type** "What type of change is this PR?" Options:

- "Feature - New functionality" (Recommended for features)
- "Bug Fix - Fixes an issue"
- "Documentation - Docs only"
- "Refactor - Code improvements"
- "Chore - Maintenance tasks"

**Question 2: Breaking Changes** "Does this PR introduce breaking changes?"

**Question 3: Testing Status** "What testing has been completed?"

## Phase 3: Generate PR Description

### Step 3.1: Analyze Changes

Based on the diff and commits, generate a comprehensive PR description.

**Auto-fill:**

1. **Description**: 2-3 sentence summary
2. **Type of Change**: Based on user input
3. **Changes Made**: Key changes from the diff
4. **Testing**: Tests run based on user input

### Step 3.2: Create PR Description

Format the PR body. Use a HEREDOC when creating the PR.

## Phase 4: Push and Create PR

### Step 4.1: Ensure Branch is Pushed

```bash
git rev-parse --abbrev-ref --symbolic-full-name @{u} 2>/dev/null || echo "no-upstream"
```

If no upstream:

```bash
git push -u origin $(git branch --show-current)
```

If upstream exists:

```bash
git push
```

### Step 4.2: Get Current User and Org Members

```bash
gh api user --jq '.login'
ORG=$(gh repo view --json owner --jq '.owner.login')
gh api "orgs/$ORG/members" --jq '.[].login' 2>/dev/null || echo ""
```

Filter out the current user to get reviewers.

### Step 4.3: Create Pull Request

```bash
gh pr create --title "<title>" --body "$(cat <<'EOF'
<PR description>
EOF
)" --base main --assignee @me --reviewer <member1>
```

**Title format**: Conventional commits style (`feat: ...`, `fix: ...`, etc.)

**Assignee**: Always `@me`.

**Reviewers**: All other organization members. If the org API fails (personal
repo), skip reviewers.

### Step 4.4: Verify PR Creation

```bash
gh pr view --json number,url,title
```

Display the PR URL to the user.

## Phase 5: Post-PR Tasks

### Step 5.1: Verify CI/CD

```bash
gh pr checks
```

If checks are failing, inform the user and suggest next steps.

### Step 5.2: Final Summary

Display:

```markdown
## Pull Request Submitted

- **Branch**: <branch>
- **Title**: <title>
- **URL**: <url>
- **Type**: <type>
- **Commits**: <count>

### Next Steps

1. Wait for CI checks to complete
2. Address any review comments
3. Merge when approved
```

## Best Practices

### Branch Naming

- Use descriptive, kebab-case names
- Prefix with type: `feat/`, `fix/`, `docs/`, etc.

### Commit Messages

- Present tense, imperative mood ("Add feature" not "Added feature")
- First line 50 characters or less
- Separate subject from body with a blank line

### PR Descriptions

- Be thorough — reviewers need context
- Link to related issues and PRs
- Include screenshots/videos for UI changes
- Document breaking changes clearly

## Troubleshooting

### Git push fails

Check if branch needs rebasing:

```bash
git fetch origin
git rebase origin/main
git push
```

If push is rejected after rebase, ask the user before force-pushing.

### CI/CD checks failing

1. Check logs: `gh pr checks --watch`
2. Fix issues locally
3. Push fixes
4. Wait for checks to re-run
