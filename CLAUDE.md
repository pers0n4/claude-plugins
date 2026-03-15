# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Claude Code plugin marketplace registry and plugin source repository. Plugins are installed via `claude plugin install <name>@pers0n4` or loaded locally with `claude --plugin-dir ./plugins/<name>`.

## Development Tools

Managed via [mise](https://mise.jdx.dev/). Run `mise install` to set up:

- **dprint** — Markdown formatter (`dprint fmt`)
- **lefthook** — Git hooks manager
- **cocogitto** — Conventional commits enforcement (`cog verify`)

Setup after clone:

```bash
mise install && lefthook install
```

## Git Hooks (lefthook)

- **pre-commit**: `dprint fmt` on staged `.md` files (auto-fixes and re-stages)
- **commit-msg**: `cog verify` enforces Conventional Commits

## Commands

```bash
dprint fmt              # Format all markdown files
dprint check            # Check formatting without writing
cog verify --file <f>   # Verify a commit message file
claude plugin validate ./plugins/<name>  # Validate plugin manifest
```

## Architecture

`docs/references/` — Claude Code plugin development references (plugins, skills, hooks, MCP specs). Consult before implementing plugin features.

```
.claude-plugin/
  marketplace.json          # Registry manifest — lists all plugins for marketplace distribution
plugins/
  <name>/
    .claude-plugin/plugin.json  # Plugin manifest (name, version, author, keywords)
    skills/<skill>/
      SKILL.md                  # Skill definition (instructions for Claude)
      references/               # Supporting reference docs for the skill
      template.md               # Output templates used by the skill
```

Each plugin is self-contained under `plugins/<name>/`. The marketplace registry at `.claude-plugin/marketplace.json` references plugins by relative `source` path.

When adding a new plugin: create `plugins/<name>/.claude-plugin/plugin.json`, then register in `.claude-plugin/marketplace.json` under `plugins[]`.

## Conventions

- Commit messages follow Conventional Commits (`feat:`, `fix:`, `chore:`, `docs:`, etc.)
- Indent: 2 spaces, LF line endings, UTF-8, final newline (see `.editorconfig`)
- Skills reference supporting files via `${CLAUDE_SKILL_DIR}`
