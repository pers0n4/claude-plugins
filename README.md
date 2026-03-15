# claude-plugins-pers0n4

Directory of Claude Code extensions — development tools and productivity plugins.

## Overview

This repository serves two purposes:

- **Marketplace registry** — `.claude-plugin/marketplace.json` lists plugins available for installation via the Claude Code plugin marketplace
- **Plugin source** — `plugins/` contains plugin implementations ready to use or fork

## Included Plugins

| Plugin                            | Description                                                                                              | Version |
| --------------------------------- | -------------------------------------------------------------------------------------------------------- | ------- |
| [git-workflow](plugins/git-workflow/) | Git workflow tools — commit with Conventional Commits v1.0.0, type/scope analysis, and HEREDOC templates | 0.1.0   |

## Quick Start

### Add marketplace

```bash
claude plugin marketplace add pers0n4/claude-plugins
```

### Install a plugin

```bash
# From marketplace
claude plugin install git-workflow@pers0n4

# Or load directly from local path
claude --plugin-dir ./plugins/git-workflow
```

## Repository Structure

```
.claude-plugin/
  marketplace.json            # Marketplace manifest (registers plugins for distribution)
plugins/
  git-workflow/               # Git workflow skill plugin
    .claude-plugin/plugin.json
    skills/commit/            # SKILL.md + references + templates
```

## Development

### Testing

```bash
# Validate plugin structure
claude plugin validate ./plugins/git-workflow

# Load with debug output
claude --plugin-dir ./plugins/git-workflow --debug
```
