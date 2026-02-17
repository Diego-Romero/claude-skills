---
name: create-prd
description:
  Interactively create a PRD by walking the user through questions and
  generating a structured draft. Reads project instruction file for context.
---

Create a Product Requirements Document by gathering information from the user
and generating a structured PRD.

## Pre-flight: Auto-detect Environment

### Step 0.1: Find Instruction File

Check for a project instruction file (stop at first match):

1. `CLAUDE.md`
2. `AGENTS.md`
3. `CONVENTIONS.md`
4. `CONTRIBUTING.md`

Store as `$INSTRUCTION_FILE`. Read it to understand the repo's stack,
conventions, and architecture.

### Step 0.2: Find PRD Template

Check for an existing PRD template:

1. `docs/prds/prd-template.md`
2. `docs/prd-template.md`
3. `.github/prd-template.md`

If a template exists, use it as the structure. If not, use the built-in template
in Step 4.

### Step 0.3: Find PRD Directory

Check where PRDs should be saved:

1. `docs/prds/` (if it exists)
2. `docs/` (if it exists)
3. Create `docs/prds/`

## Steps

### 1. Read Context

If `$INSTRUCTION_FILE` exists, read it to understand the repo's stack,
conventions, and architecture. Use this to inform questions and pre-fill
technical context.

If a PRD template was found, read it to understand the expected structure.

### 2. Ask for the Feature Overview

Start by asking the user a single open-ended question:

> **What feature do you want to build? Describe the problem and what you have in
> mind.**

Let the user explain in their own words. Don't ask for structured input yet —
this is the brainstorming phase.

### 3. Ask Targeted Follow-Up Questions

Based on their answer, ask follow-up questions to fill gaps. Ask directly to
keep it interactive. Cover these areas (skip any the user already answered):

1. **Problem & motivation** — Who has this problem? Why now?
2. **Success criteria** — How will we know it works? Any metrics?
3. **Scope boundaries** — What's explicitly out of scope?
4. **User flow** — Walk through the main interaction step by step
5. **Cross-repo impact** — Does this need changes in other repos or services?
6. **Edge cases** — Offline, errors, empty states, permissions?
7. **Phasing** — Can this be shipped incrementally?

Don't ask all questions at once. Ask 2-3 at a time based on what's still
unclear. Two rounds of questions is usually enough.

### 4. Generate the PRD

If a project PRD template was found (Step 0.2), use it as the structure.
Otherwise, use this template:

```markdown
# PRD: [Feature Name]

> **Status:** Open for Discussion
> **Author:** [Name]
> **Created:** [today's date]
> **Last Updated:** [today's date]
> **GitHub Issue:** #TBD
> **Effort:** [estimate]

---

## 1. Problem Statement

_What problem are we solving? Who has this problem? Why does it matter now?_

## 2. Goals & Success Metrics

| Goal | Metric | Target |
| ---- | ------ | ------ |
| ...  | ...    | ...    |

### Non-Goals

- ...

## 3. User Stories

- **As a** [user type], **I want to** [action] **so that** [outcome].

## 4. Scope

### In Scope

- ...

### Out of Scope

- ...

## 5. Functional Requirements

- [ ] FR-1: [Requirement]
- [ ] FR-2: [Requirement]

## 6. Non-Functional Requirements

- **Performance:** [constraints]
- **Error handling:** [approach]

## 7. UX & Design

### User Flow

1. [Step]
2. [Step]

### Edge Cases

| Scenario | Expected Behavior |
| -------- | ----------------- |
| ...      | ...               |

## 8. Technical Approach

_High-level guidance. For stack details refer to the project's instruction file._

### Cross-Repo Impact

| Repo | Changes Needed |
| ---- | -------------- |
| ...  | ...            |

### Key Technical Decisions

- ...

## 9. Acceptance Criteria

- [ ] AC-1: [Testable condition]
- [ ] AC-2: [Testable condition]

## 10. Risks & Dependencies

| Risk / Dependency | Impact | Mitigation |
| ----------------- | ------ | ---------- |
| ...               | ...    | ...        |

## 11. Implementation Phases

### Phase 1: [Name]

- [ ] [Deliverable]

## 12. Open Questions

- [ ] [Question]
```

Fill in every section based on the conversation:

- Use the instruction file to inform the Technical Approach (don't duplicate
  stack info — reference it)
- Set Status to `Open for Discussion`
- Use today's date
- Include an Effort estimate: `1-2 days`, `3-5 days`, `1-2 weeks`, `2-4 weeks`,
  `4+ weeks`
- Leave Open Questions for anything genuinely unresolved

### 5. Save the PRD

Save the generated PRD to the PRD directory with a descriptive filename:

```
docs/prds/[feature-name].md
```

Use kebab-case for the filename.

### 6. Present for Review

Show the user the generated PRD and ask if they want to adjust anything. Make
edits as requested until they're satisfied.

### 7. Commit, Push, and Create Issue

Once the user is happy with the PRD:

1. **Ask who to assign the GitHub issue to.** Fetch org members dynamically:

   ```bash
   ORG=$(gh repo view --json owner --jq '.owner.login')
   gh api "orgs/$ORG/members" --jq '.[].login' 2>/dev/null || echo ""
   ```

   Present members as options (include "No assignee"). Only ask this once.

2. **Commit and push the PRD:**

   ```bash
   git add docs/prds/[filename].md
   git commit -m "docs: add [feature name] PRD"
   git push
   ```

3. **Create a GitHub issue:**
   - Title: short feature name
   - Body: summary of the PRD with a link to the file, key features, and
     implementation phases
   - Assign to the user specified in step 1
   - Use `gh issue create`

4. **Update the PRD** to replace `#TBD` in the GitHub Issue field with the
   actual issue number returned by `gh issue create`.

5. **Commit the PRD issue reference update:**

   ```bash
   git add docs/prds/[filename].md
   git commit -m "docs: link PRD to issue #[number]"
   git push
   ```

Remind them:

- The PRD is now **Open for Discussion** — share with the team for review
- Update the Status field as it progresses
- Once approved, reference the PRD when starting implementation work

## Guidelines

- **Keep it conversational.** The user shouldn't feel like they're filling out a
  form. Extract structure from natural conversation.
- **Right-size the detail.** Be thorough on scope, requirements, and acceptance
  criteria. Be light on process overhead.
- **Pre-fill what you can.** If you can infer technical approach, edge cases, or
  non-functional requirements from the conversation + instruction file, fill them
  in rather than asking.
- **Flag cross-repo work early.** If the feature needs changes in other repos,
  make that prominent.
- **Don't over-specify implementation.** The PRD describes WHAT, not HOW.
  Implementation details belong in the plan/code, not the PRD.
