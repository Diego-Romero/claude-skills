# claude-skills

Global [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skills that
work in any codebase. Clone this repo directly as `~/.claude/skills/` and the
skills are immediately available in all your projects.

## Install

```bash
git clone git@github.com:diego-romero/claude-skills.git ~/.claude/skills
```

## Update

```bash
cd ~/.claude/skills && git pull
```

## Uninstall

```bash
rm -rf ~/.claude/skills
```

---

## Available Skills

### `/review` — Unified Code Review

One command that auto-detects the right review mode:

| Mode        | When                                       | Agents | Validation            |
| ----------- | ------------------------------------------ | ------ | --------------------- |
| **Full**    | >= 200 lines or >= 5 files changed         | 4      | Batch adversarial     |
| **Small**   | < 200 lines AND < 5 files                  | 2      | Inline 3-point check  |
| **Re-review** | Prior `/review` on PR + new commits since | 2      | Fix verification      |

**What it auto-detects:**

- **Review mode** — re-review vs small vs full (you never pick)
- **Instruction file** — CLAUDE.md, AGENTS.md, CONVENTIONS.md, CONTRIBUTING.md
- **Verification command** — from package.json, Makefile, Cargo.toml, etc.
- **Excluded files** — lock files, generated files (via .gitattributes)
- **Project-specific rules** — from `.claude/review-rules.md` if present

**Usage:**

```
/review
```

That's it. No arguments, no mode selection.

**Adding project-specific rules:**

Create `.claude/review-rules.md` in your project with domain-specific review
knowledge. For example:

```markdown
## Duration Fields

- `work_period_duration` is stored in MINUTES, not seconds
- Convert to ms: multiply by 60 * 1000

## Architecture

- This is a client-only repo — no backend functions here
- Native modules: JS in src/native-modules/, Swift in plugins/native-sources/
```

These rules are injected into agent prompts alongside the instruction file.

### `/commit-push` — Stage, Commit, Push

One command to stage changes, create a conventional commit, and push.

```
/commit-push                    # Auto-generates commit message
/commit-push fix the login bug  # Uses your message
```

Auto-detects pre-commit hooks (Husky, pre-commit) and verification commands.
Stages files individually (never `git add -A`). Handles upstream tracking.

### `/submit-pr` — Create Pull Request

Creates a well-documented PR with conventional commits, proper description, and
auto-assigned reviewers.

```
/submit-pr
```

Auto-detects verification command, instruction file, and org members for
reviewer assignment. Gathers PR details interactively.

### `/create-issue` — GitHub Issue with Codebase Context

Creates a GitHub issue enriched with technical context from the codebase —
affected files, related code, implementation steps.

```
/create-issue
/create-issue "add dark mode support"
```

Explores the codebase to provide actionable file paths and guidance, not just
vague descriptions.

### `/create-prd` — Product Requirements Document

Interactively creates a PRD by asking questions and generating a structured
draft.

```
/create-prd
```

Auto-detects existing PRD templates. Commits the PRD, creates a linked GitHub
issue, and guides the team workflow.

---

## Adding New Skills

Create a new directory with a `SKILL.md` file:

```
my-skill/
  SKILL.md
```

The `SKILL.md` must have frontmatter:

```markdown
---
name: my-skill
description: What it does
---

Instructions for the skill...
```

Then invoke with `/my-skill` in any project.

## License

MIT
