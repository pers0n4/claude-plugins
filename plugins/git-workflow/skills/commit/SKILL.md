---
name: commit
description: This skill should be used when the user asks to commit changes, says "/commit", "커밋", or "커밋해줘", wants to save work to git, mentions conventional commits, or asks about commit message format. It creates git commits following Conventional Commits v1.0.0 by analyzing staged changes to determine type, scope, and message. It should also be triggered when a commit is needed as part of a larger workflow.
allowed-tools: Bash(git *)
argument-hint: "[message hint]"
---

# Git Commit — Conventional Commits

Format: `<type>[optional scope]: <description>`

Supporting files (read on demand):

- `${CLAUDE_SKILL_DIR}/references/conventions.md` — type table, scope heuristics, footer formats, breaking change rules
- `${CLAUDE_SKILL_DIR}/template.md` — HEREDOC commit command templates and completed examples

## Input

If `$ARGUMENTS` is provided (e.g. `/commit fix typo in login page`), use it as a hint for the commit message. The hint guides type, scope, and message decisions — but the actual diff always takes precedence. If the hint says "fix" but the diff adds a new feature, the diff wins.

## Workflow

### 1. Check staged changes

Run in parallel:

```bash
git diff --cached --stat
git diff --cached
git status --short
git log --oneline -5
```

If nothing is staged, show `git status` and ask the user what to stage. Do not stage files without asking.

### 2. Analyze and compose

Determine **type**, **scope**, and **message** from the diff:

- **Type**: Pick the most accurate one (`feat`, `fix`, `refactor`, `chore`, `docs`, `style`, `test`, `perf`, `ci`, `build`, `revert`). Only `feat` and `fix` have SemVer significance.
- **Scope**: Short noun from the changed area. Omit when unclear. Check project CLAUDE.md for conventions.
- **Message**: Imperative mood, lowercase verb, under 72 chars total (convention), no period. English only (Korean triggers are supported for invoking the skill, but commit message text should be in English).
- **Body**: Optional. Explain _why_, not _what_. Blank line after subject, wrap at 72 chars.
- **Footers**: Add `Fixes #N` / `Closes #N` when resolving a tracked issue. For breaking changes, use `!` before `:` and/or a `BREAKING CHANGE:` footer.

For complex commits (breaking changes, multi-paragraph bodies, revert with refs), read `${CLAUDE_SKILL_DIR}/references/conventions.md`.

If the diff spans multiple unrelated changes, suggest splitting into separate commits.

### 3. Present and confirm

Pick the appropriate template variation from `${CLAUDE_SKILL_DIR}/template.md` and fill it in. Present the completed `git commit` command to the user.

Wait for user confirmation before executing. Revise if the user wants changes.

## Rules

- Never use `--no-verify` or skip hooks unless the user explicitly asks
- Never use `git add -A` or `git add .` — stage specific files
- If a pre-commit hook fails, fix the issue and create a **new** commit (do not `--amend` — the failed commit never happened)
- One logical change per commit
