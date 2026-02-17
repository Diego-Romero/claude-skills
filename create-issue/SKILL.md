---
name: create-issue
description:
  Create a well-structured GitHub issue with technical context from the
  codebase.
argument-hint: "[optional: brief description of the issue]"
---

Create a GitHub issue enriched with codebase-derived technical context so anyone
picking it up knows exactly what files to touch and how.

## Pre-flight: Auto-detect Environment

### Step 0.1: Find Instruction File

Check for a project instruction file (stop at first match):

1. `CLAUDE.md`
2. `AGENTS.md`
3. `CONVENTIONS.md`
4. `CONTRIBUTING.md`

Store as `$INSTRUCTION_FILE`. Read it to understand the repo's architecture,
file layout, and conventions.

## Steps

### 1. Read Context

If `$INSTRUCTION_FILE` exists, read it to understand the repo's architecture.
This informs codebase exploration in step 5.

### 2. Get Description

If `$ARGUMENTS` is non-empty, use it as the starting description.

Otherwise, ask the user:

> **What issue do you want to create? Describe the problem, feature, or task.**

Let them explain in their own words.

### 3. Classify Type

Auto-detect the issue type from the description. Do not ask the user — infer it:

| Type          | Label           | Signal                                           |
| ------------- | --------------- | ------------------------------------------------ |
| Bug           | `bug`           | "broken", "wrong", "crash", "error", regression  |
| Feature       | `enhancement`   | "add", "new", "support", "implement", "improve"  |
| Documentation | `documentation` | "docs", "readme", "document", "guide"            |
| Task          | _(no label)_    | Chore, refactor, config, cleanup, infrastructure |

### 4. Ask Follow-ups

Based on the type, ask **1-2 targeted follow-up questions** (one round max)
using direct questions. Skip questions the user already answered. Examples:

- **Bug**: "What's the expected vs actual behavior?" / "Can you describe the
  steps to reproduce?"
- **Feature**: "What's the user-facing goal?" / "Any constraints or scope
  boundaries?"
- **Docs**: "Which part of the system needs documenting?"
- **Task**: "Any deadline or priority context?"

If the initial description is detailed enough, skip follow-ups entirely.

### 5. Explore Codebase

This is the core value of the skill. Use Glob, Grep, and Read to find:

- **Affected files** — What files need to change and why
- **Related patterns/utilities** — Existing code to reuse or extend
- **Entry points** — Where the feature/fix integrates into the app
- **Root cause** (bugs) — The specific code path causing the issue

Read 3-8 files to produce specific, actionable technical context. Don't just
list file names — explain what each file does and what needs to change.

### 6. Generate Issue

Compose the issue body using the template below, adapted by type.

**Title**: Short, imperative summary (e.g., "Fix progress ring resetting on
pause", "Add push notification sound settings").

**Body template:**

```markdown
## Problem / Motivation

[2-3 sentences explaining why this issue exists]

## Steps to Reproduce ← bugs only

1. [Step]
2. [Step]
3. [Observe: ...]

## User Flow ← features only

1. [User does X]
2. [App responds with Y]
3. [Result: Z]

## Technical Context

**Affected files:**

- `path/to/file.ts` — [role and what needs to change]
- `path/to/other.ts` — [role and what needs to change]

**Related code:**

- [Existing patterns, utilities, or modules to reuse]

## Proposed Solution

[High-level approach in 2-4 sentences]

### Implementation Steps

1. [Specific step with file path]
2. [Specific step with file path]
3. ...

## Acceptance Criteria

- [ ] [Testable condition]
- [ ] [Testable condition]
```

**Type-specific adjustments:**

- **Bugs**: Include "Steps to Reproduce", omit "User Flow"
- **Features**: Include "User Flow", omit "Steps to Reproduce"
- **Tasks**: Omit both — use simplified template with Problem/Motivation,
  Technical Context, Implementation Steps, and Acceptance Criteria
- **Docs**: Omit Technical Context if it's a pure documentation task

### 7. Present for Review

Display the full issue (title, labels, body) to the user. Ask if they want to
adjust anything. Make edits as requested.

### 8. Create Issue

Once the user is happy:

1. **Ask who to assign the issue to.** Fetch org members dynamically:

   ```bash
   ORG=$(gh repo view --json owner --jq '.owner.login')
   gh api "orgs/$ORG/members" --jq '.[].login' 2>/dev/null || echo ""
   ```

   Present members as options (include "No assignee" as an option). If the org
   API fails (e.g., personal repo), ask for a GitHub username directly.

2. **Create the issue with `gh`:**

   ```bash
   gh issue create \
     --title "<title>" \
     --body "<body>" \
     --label "<label>" \
     --assignee "<assignee>"
   ```

   Omit `--label` if type is Task (no label). Omit `--assignee` if user chose
   "No assignee".

### 9. Confirm

Display the issue number and URL:

```
Created issue #<number>: <url>
```

## Guidelines

- **Specificity over generality.** The whole point is that the agent explores
  the codebase to give actionable file paths and implementation guidance — not
  vague descriptions anyone could write.
- **Don't over-scope.** Issues should be completable in a single PR. If the
  description implies a multi-PR effort, suggest breaking it into multiple
  issues.
- **Match existing conventions.** Look at recent issues in the repo
  (`gh issue list`) to match the team's style if there are existing patterns.
- **Keep it concise.** Implementation steps should be specific but not
  exhaustive — leave room for the implementer's judgment.
