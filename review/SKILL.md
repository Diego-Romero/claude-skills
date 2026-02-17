---
name: review
description:
  Unified code review — auto-detects mode (re-review, small, full) and project
  environment. Works in any codebase.
---

Provide a code review for local changes on the current branch. This skill
auto-detects everything: review mode, project conventions, verification commands,
and excluded files.

## Pre-flight: Auto-detect Project Environment

Before starting, detect the project environment. Do NOT hardcode anything.

### Step 0.1: Find Instruction File

Check for a project instruction file in this order (stop at first match):

1. `CLAUDE.md`
2. `AGENTS.md`
3. `CONVENTIONS.md`
4. `CONTRIBUTING.md`

Store as `$INSTRUCTION_FILE`. If none found, set to empty (skip convention
checks that require it).

### Step 0.2: Find Project-Specific Review Rules

Check for `.claude/review-rules.md`. If it exists, its contents will be injected
into agent prompts as additional domain-specific rules. This is how projects add
custom review knowledge without modifying this skill.

### Step 0.3: Find Verification Command

Detect the verification command to run before finalizing:

1. **package.json**: Look for scripts in order: `check:ci` → `check` → `lint`
2. **Makefile**: Look for targets: `check` → `lint` → `test`
3. **Cargo.toml**: Use `cargo clippy && cargo test`
4. **pyproject.toml**: Look for `ruff` or `flake8` in dependencies → use
   appropriate command
5. **go.mod**: Use `go vet ./...`

Store as `$VERIFY_CMD`. If nothing found, skip verification step.

### Step 0.4: Determine Excluded Files

Always exclude from diffs:

- `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`
- `*.pbxproj`, `Podfile.lock`
- `Cargo.lock`, `poetry.lock`, `go.sum`

Additionally, read `.gitattributes` for files marked `linguist-generated` and
exclude those too.

Build the git pathspec exclusion string as `$EXCLUDE_PATHS`.

## Phase 1: Auto-detect Review Mode

### Step 1.1: Check for Re-review Trigger

1. Check if a PR exists: `gh pr view --json number,url,title,headRefName 2>&1`
2. If a PR exists, look for a previous `/review` comment:

```bash
REPO=$(gh repo view --json nameWithOwner --jq '.nameWithOwner')
gh api "repos/$REPO/pulls/$PR_NUMBER/reviews" \
  --jq '[.[] | select(.body | contains("Review conducted by") and contains("/review"))] | last'
```

3. If a review exists with unresolved issues, check for commits after
   `submitted_at`:

```bash
git log --after="$REVIEW_SUBMITTED_AT" --format="%H" --reverse HEAD | head -1
```

4. If there are commits after the review → **RE-REVIEW mode**

### Step 1.2: Determine Review Depth (if not re-review)

Get the diff stat:

```bash
git diff main...HEAD --stat -- . $EXCLUDE_PATHS 2>/dev/null || git diff --cached --stat -- . $EXCLUDE_PATHS
```

- If < 200 lines changed AND < 5 files → **SMALL mode** (2 agents + inline
  validation)
- Otherwise → **FULL mode** (4 agents + batch adversarial validation)

Inform the user which mode was selected and why.

---

## FULL Mode (4 agents + batch validation)

### Step F.1: Discover Changes

Check for changes in order:

1. Committed changes on current branch vs main:
   `git log main..HEAD --oneline` and `git diff main...HEAD --stat`
2. Staged changes: `git diff --cached --stat`

If no changes, inform the user and stop.

Get the full diff excluding generated files:

- Committed: `git diff main...HEAD -- . $EXCLUDE_PATHS`
- Staged: `git diff --cached -- . $EXCLUDE_PATHS`

### Step F.2: Launch 4 Review Agents in Parallel

Launch all 4 agents simultaneously using the Task tool. Each agent receives:

- The full diff
- The instruction file contents (if found)
- The review rules contents (if found)
- The project type context

**All agents must return issues in this JSON format:**

```json
{
  "issues": [
    {
      "title": "Short descriptive title",
      "file": "path/to/file.ts",
      "lines": "45-50",
      "description": "What is wrong and why it matters",
      "category": "security|correctness|architecture|performance|test-coverage|plan-compliance",
      "severity": "critical|high|medium|low",
      "suggestion": "How to fix it"
    }
  ]
}
```

**Severity definitions (all agents):**

- **Critical**: Code will crash, lose data, or create a security vulnerability
- **High**: Code will produce wrong results or violate a core project convention
- **Medium**: Code works but deviates from best practices in a meaningful way
- **Low**: Minor improvement that a senior engineer might suggest

---

#### Agent 1: Bugs & Security

**Task**: Scan for bugs, logic errors, and security vulnerabilities in the
changed code.

**Bug detection — look for:**

- Logic errors: Wrong operators, incorrect conditionals, off-by-one
- Null/undefined dereferences on nullable values
- Wrong async patterns: Missing `await`, unhandled promise rejections, race
  conditions
- Missing imports, incorrect module paths, undefined variables
- State mutation bugs (framework-specific: React state mutation, etc.)
- Stale closures in hooks/callbacks with incorrect dependency arrays

**Security — look for:**

- Sensitive data exposure: Secrets, tokens, API keys logged or displayed
- Injection vulnerabilities: SQL injection, XSS, command injection
- Insecure data storage (secrets in plaintext, localStorage for tokens)
- Auth token handling issues (leaked to logs, sent to untrusted endpoints)
- Unvalidated external input used in security-sensitive operations

**If review rules exist**, also check for domain-specific correctness patterns
described there.

**Do NOT flag:**

- Issues in code not modified by this changeset
- Theoretical attacks requiring physical device access
- Issues a linter or type checker would catch
- Style or formatting issues

**Only flag if confident** the code will fail, produce wrong results, or create a
vulnerability.

---

#### Agent 2: Project Conventions

**Task**: Check for meaningful deviations from project conventions.

**Process:**

1. Read `$INSTRUCTION_FILE` to understand the project's patterns and conventions
2. If `.claude/review-rules.md` exists, read it for additional rules
3. Review the changed code for violations

**Flag only:**

- Violations of documented architectural patterns
- Violations of documented naming/path conventions
- Violations of documented anti-patterns
- For each issue, **quote the exact rule being violated** from the instruction
  file

**Do NOT flag:**

- Minor deviations that achieve the same goal
- Code that predates the changeset
- Style preferences not documented in the instruction file
- Issues already caught by other agents

If no instruction file exists, skip this agent and return zero issues.

---

#### Agent 3: Performance

**Task**: Scan for performance issues in the changed code.

**Look for (adapt to detected stack):**

- N+1 query patterns (database queries or API calls in loops)
- Unnecessary re-computation (missing memoization where expensive)
- Resource leaks (unclosed connections, intervals not cleaned up, event listeners
  not removed)
- Large inline allocations in hot paths
- Expensive operations in render/request paths

**Do NOT flag:**

- Micro-optimizations that don't affect user experience
- Missing memoization where re-renders/recomputation is cheap
- Performance issues in code not modified by this changeset

---

#### Agent 4: Coverage & Planning

**Task**: Check test coverage for modified code AND plan compliance.

**Test coverage:**

1. Check if test files exist in the project (look for `__tests__/`, `test/`,
   `*_test.*`, `*.test.*`, `*.spec.*` patterns)
2. If NO tests exist, skip coverage checks
3. If tests exist, check whether modified utility/logic functions have
   corresponding test updates

**Flag only:**

- Modified functions that have existing tests which were NOT updated
- New pure logic/utility functions that should have tests

**Do NOT flag:**

- Missing tests for UI components, hooks, or config files
- Missing tests when no test infrastructure exists

**Plan compliance:**

1. Check if plan files exist (`.plans/`, `docs/plans/`, `plans/`)
2. If plan files exist, read the most recent one and verify compliance
3. If no plans exist, skip

**Flag only:**

- Required plan items not implemented
- Implementation significantly deviating from planned approach

### Step F.3: Batch Adversarial Validation

After all 4 agents complete, if there are any issues found, launch **1 batch
validation agent** that receives ALL issues together.

**The validator applies this 5-point skepticism checklist to each issue:**

1. **Is it actually in the diff?** — Must be in code introduced or modified by
   this changeset, not pre-existing
2. **Is the assessment correct?** — Read the actual code and verify the claimed
   problem exists
3. **Is there protecting context?** — Check surrounding code for guards, type
   checks, upstream validation
4. **Would a senior engineer care?** — Real problem or pedantic nitpick?
5. **Is the severity right?** — Could this be downgraded?

**Validation result format per issue:**

```json
{
  "valid": true | false,
  "adjusted_severity": "critical|high|medium|low",
  "reason": "Why the issue stands or why it's a false positive"
}
```

Remove issues where `valid: false`. Use `adjusted_severity` for remaining
issues.

### Step F.4: Verification

Run `$VERIFY_CMD` if detected. Note any failures in the report.

### Step F.5: Generate Report

**If NO issues found:**

```markdown
## Code Review ✅

No issues found. Checked for:

- Security vulnerabilities and bugs
- Project convention compliance
- Performance issues
- Test coverage and plan compliance

Changes look good to proceed.
```

**If issues found**, group by severity (Critical > High > Medium > Low):

```markdown
## Code Review

Found {count} issue(s) requiring attention:

### Critical

**{title}** `{file}:{lines}`

{description}

**Suggestion:** {suggestion}

---

### High

...
```

### Step F.6: Submit Review to GitHub PR

Check if a PR exists: `gh pr view --json number,url,title 2>&1`

If no PR exists, skip and inform the user.

If a PR exists, submit as a review:

- **APPROVE** (`--approve`): Zero issues
- **COMMENT** (`--comment`): Only low/medium issues
- **REQUEST_CHANGES** (`--request-changes`): Any critical/high issues

If `--request-changes` fails (own PR), fall back to `--comment`.

Use a HEREDOC for the body. Append signature:

```
---

_Review conducted by 4-agent review system via `/review` (full mode)_
```

---

## SMALL Mode (2 agents + inline validation)

### Step S.1: Discover Changes

Same as Step F.1.

### Step S.2: Launch 2 Review Agents in Parallel

Launch **Agent 1 (Bugs & Security)** and **Agent 2 (Project Conventions)** only.
Same prompts as FULL mode.

### Step S.3: Inline Validation

For each issue, apply a quick **3-point skepticism check** (no sub-agent):

1. **Is it actually in the diff?** — Not pre-existing code
2. **Is the assessment correct?** — Read the actual code if needed
3. **Would a senior engineer care?** — Real problem vs pedantic nitpick

Remove any issues that fail. Adjust severity if needed.

### Step S.4: Verification

Run `$VERIFY_CMD` if detected.

### Step S.5: Generate Report

Same format as FULL mode, but with signature:

```
---

_Review conducted by 2-agent review system via `/review` (small mode)_
```

### Step S.6: Submit to PR

Same logic as Step F.6.

---

## RE-REVIEW Mode (auto-triggered)

### Step R.1: Load Context

Parse the original review from the PR (found in Step 1.1). Extract each issue
from the review body using the standard format:

```
### {Severity}
**{title}** `{file}:{lines}`
{description}
**Suggestion:** {suggestion}
```

If the review was "Code Review ✅" (no issues), inform the user and stop.

### Step R.2: Get Changes Since Review

```bash
FIRST_FIX_COMMIT=$(git log --after="$REVIEW_SUBMITTED_AT" --format="%H" --reverse HEAD | head -1)
git diff ${FIRST_FIX_COMMIT}^..HEAD -- . $EXCLUDE_PATHS
```

If no commits since review, inform the user and stop.

### Step R.3: Launch 2 Verification Agents in Parallel

#### Agent A: Fix Verification

For each original high/medium issue:

1. Read the current state of the file
2. Check if the problem still exists
3. Classify as: **fixed**, **partially-fixed**, or **not-fixed**

Be generous: different but valid approaches count as fixed.

#### Agent B: Regression Scan

Quick scan of the post-review diff for new bugs introduced by the fixes. Same
checks as Agent 1 (Bugs & Security) but scoped only to the new delta.

### Step R.4: Verification

Run `$VERIFY_CMD` if detected.

### Step R.5: Generate Report

**If ALL issues fixed and NO regressions:**

```markdown
## Re-Review ✅ - All Issues Resolved

All {count} high/medium issue(s) from the original review have been addressed.

- [x] ~~**{title}** `{file}:{lines}` — {description}~~

No regressions detected.

---

_Re-review conducted by `/review` (re-review mode)_
```

**If some issues remain OR regressions found:**

```markdown
## Re-Review - {remaining_count} Issue(s) Remaining

**Original issues:** {total} checked, {fixed} fixed, {partial} partially fixed,
{not_fixed} not fixed

### Fix Status

- [x] ~~**{title}** `{file}:{lines}`~~ **Fixed**
- [ ] **{title}** `{file}:{lines}` **Not Fixed**
  > {explanation}

### New Regressions (if any)

**{title}** `{file}:{lines}`
{description}
**Suggestion:** {suggestion}

---

_Re-review conducted by `/review` (re-review mode)_
```

### Step R.6: Submit to PR

- **APPROVE**: All high/medium fixed, no critical/high regressions
- **COMMENT**: Some remain but no critical/high regressions
- **REQUEST_CHANGES**: New critical/high regressions (fall back to COMMENT on own
  PR)

---

## False Positives to Avoid

**Do NOT flag these (applies to all modes):**

- Pre-existing issues not introduced in this changeset
- Code that appears buggy but is correct given surrounding context
- Pedantic nitpicks a senior engineer wouldn't flag
- Issues a linter or type checker will catch
- Minor code quality concerns not documented in the instruction file

**If not certain, do not flag it.** False positives waste reviewer time and erode
trust.

## Agent Behavior Notes

- All tools are functional. Do not make exploratory or test calls.
- Only call a tool if required. Every tool call should have a clear purpose.
- Launch agents in parallel wherever the skill says to.
- Use the `general-purpose` subagent_type for all review and validation agents.
