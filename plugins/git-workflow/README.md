# git-workflow

A Claude Code plugin for git workflows. Currently includes a commit skill following [Conventional Commits v1.0.0](https://www.conventionalcommits.org/).

## What It Does

Analyzes staged changes and composes a well-structured commit message with:

- **Type** detection (`feat`, `fix`, `refactor`, `chore`, `docs`, etc.)
- **Scope** inference from the changed area of the codebase
- **Imperative mood** subject lines under 72 characters
- **HEREDOC format** for multi-line messages with proper git trailer parsing
- **Breaking change** and **issue reference** support

## Installation

```bash
# From marketplace
claude plugin install git-workflow@pers0n4

# From local directory (for development)
claude --plugin-dir ./plugins/git-workflow
```

## Usage

```
/git-workflow:commit              # Analyze staged changes and compose commit
/git-workflow:commit fix login    # Use hint to guide commit message
```

The skill reads `git diff --cached` to determine what will be committed, then presents a complete `git commit` command for your confirmation.

If nothing is staged, it shows `git status` and asks what to stage.

## Workflow

1. Checks staged changes (`git diff --cached --stat`, `git status --short`)
2. Analyzes the diff to determine type, scope, and message
3. Selects the appropriate HEREDOC template (simple, with body, with issue ref, breaking change, revert)
4. Presents the completed `git commit` command
5. Waits for your confirmation before executing

## Safety

- Never uses `--no-verify` or skips hooks
- Never uses `git add -A` or `git add .`
- If a pre-commit hook fails, creates a **new** commit (never `--amend`)
- Suggests splitting when the diff contains unrelated changes

## Supported Commit Types

| Type       | Description                            |
| ---------- | -------------------------------------- |
| `feat`     | New capability (SemVer MINOR)          |
| `fix`      | Bug fix (SemVer PATCH)                 |
| `refactor` | Code restructuring, no behavior change |
| `chore`    | Maintenance, deps, config              |
| `docs`     | Documentation only                     |
| `style`    | Formatting, whitespace                 |
| `test`     | Adding or fixing tests                 |
| `perf`     | Performance improvement                |
| `ci`       | CI/CD pipeline changes                 |
| `build`    | Build system changes                   |
| `revert`   | Reverts a previous commit              |
