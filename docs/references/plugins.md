# Claude Code Plugins Reference

> Sourced from official Anthropic documentation at [code.claude.com/docs/en/plugins](https://code.claude.com/docs/en/plugins) and [code.claude.com/docs/en/plugins-reference](https://code.claude.com/docs/en/plugins-reference). Retrieved 2026-03-15.

This document is a practical technical reference for the Claude Code plugin system. It covers directory structure, manifest schema, installation scopes, CLI commands, caching, the marketplace registry format, plugin lifecycle, version management, and auto-discovery.

---

## Table of Contents

1. [Plugin Directory Structure](#plugin-directory-structure)
2. [Plugin Manifest Schema](#plugin-manifest-schema)
3. [Plugin Installation Scopes](#plugin-installation-scopes)
4. [CLI Commands](#cli-commands)
5. [Environment Variables](#environment-variables)
6. [Plugin Caching](#plugin-caching)
7. [Marketplace Registry Schema](#marketplace-registry-schema)
8. [Plugin Lifecycle](#plugin-lifecycle)
9. [Version Management](#version-management)
10. [Auto-Discovery](#auto-discovery)

---

## Plugin Directory Structure

A plugin is a self-contained directory. The manifest file is the only component that belongs inside `.claude-plugin/`. All other component directories live at the plugin root.

```
plugin-name/
├── .claude-plugin/           # Metadata directory (optional)
│   └── plugin.json           # Plugin manifest
├── commands/                 # Slash commands (legacy; prefer skills/ for new work)
│   ├── status.md
│   └── logs.md
├── agents/                   # Subagent definitions
│   ├── security-reviewer.md
│   └── performance-tester.md
├── skills/                   # Agent Skills (SKILL.md per subdirectory)
│   ├── code-reviewer/
│   │   └── SKILL.md
│   └── pdf-processor/
│       ├── SKILL.md
│       └── scripts/
├── hooks/                    # Hook event handlers
│   ├── hooks.json            # Main hook configuration
│   └── extra-hooks.json      # Additional hooks (optional)
├── settings.json             # Default settings applied when plugin is enabled
├── .mcp.json                 # MCP server definitions
├── .lsp.json                 # LSP server configurations
├── scripts/                  # Hook and utility scripts
│   ├── format-code.sh
│   └── process.js
├── LICENSE
└── CHANGELOG.md
```

### Default File Locations

| Component   | Default Location             | Notes                                                         |
| :---------- | :--------------------------- | :------------------------------------------------------------ |
| Manifest    | `.claude-plugin/plugin.json` | Optional; Claude Code auto-discovers components without it    |
| Commands    | `commands/`                  | Markdown files; legacy approach, use `skills/` for new skills |
| Agents      | `agents/`                    | Markdown files with YAML frontmatter                          |
| Skills      | `skills/`                    | One subdirectory per skill, each containing `SKILL.md`        |
| Hooks       | `hooks/hooks.json`           | JSON configuration with event matchers and actions            |
| MCP servers | `.mcp.json`                  | Standard MCP server configuration                             |
| LSP servers | `.lsp.json`                  | Language server configurations                                |
| Settings    | `settings.json`              | Only the `agent` key is currently supported                   |

> **Warning**: Do not put `commands/`, `agents/`, `skills/`, or `hooks/` inside `.claude-plugin/`. Only `plugin.json` belongs there.

---

## Plugin Manifest Schema

The manifest lives at `.claude-plugin/plugin.json`. It is **optional** — when absent, Claude Code auto-discovers components in default locations and derives the plugin name from the directory name. Include a manifest when you need to provide metadata or specify custom component paths.

### Complete Example

```json
{
  "name": "my-plugin",
  "version": "2.1.0",
  "description": "Brief plugin description",
  "author": {
    "name": "Author Name",
    "email": "author@example.com",
    "url": "https://github.com/author"
  },
  "homepage": "https://docs.example.com/plugin",
  "repository": "https://github.com/author/my-plugin",
  "license": "MIT",
  "keywords": ["keyword1", "keyword2"],
  "commands": ["./custom/commands/special.md"],
  "agents": "./custom/agents/",
  "skills": "./custom/skills/",
  "hooks": "./config/hooks.json",
  "mcpServers": "./mcp-config.json",
  "outputStyles": "./styles/",
  "lspServers": "./.lsp.json"
}
```

### Required Fields

If a manifest is present, `name` is the only required field.

| Field  | Type   | Description                                                                                                                                                                | Example              |
| :----- | :----- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :------------------- |
| `name` | string | Unique plugin identifier (kebab-case, no spaces). Used as the namespace prefix for all components. Skills in a plugin named `my-plugin` appear as `/my-plugin:skill-name`. | `"deployment-tools"` |

### Metadata Fields

| Field         | Type   | Required | Description                                                                                  | Example                                            |
| :------------ | :----- | :------- | :------------------------------------------------------------------------------------------- | :------------------------------------------------- |
| `version`     | string | No       | Semantic version string. If also set in the marketplace entry, `plugin.json` takes priority. | `"2.1.0"`                                          |
| `description` | string | No       | Brief explanation of plugin purpose                                                          | `"Deployment automation tools"`                    |
| `author`      | object | No       | Author information. Supported subfields: `name` (string), `email` (string), `url` (string)   | `{"name": "Dev Team", "email": "dev@example.com"}` |
| `homepage`    | string | No       | Documentation or project URL                                                                 | `"https://docs.example.com"`                       |
| `repository`  | string | No       | Source code URL                                                                              | `"https://github.com/user/plugin"`                 |
| `license`     | string | No       | SPDX license identifier                                                                      | `"MIT"`, `"Apache-2.0"`                            |
| `keywords`    | array  | No       | String tags for discovery                                                                    | `["deployment", "ci-cd"]`                          |

### Component Path Fields

Custom paths **supplement** default directories; they do not replace them. If `commands/` exists at the plugin root, it is always loaded regardless of what is declared here. All paths must be relative to the plugin root and must start with `./`.

| Field          | Type                      | Required | Description                                                        | Example                                       |
| :------------- | :------------------------ | :------- | :----------------------------------------------------------------- | :-------------------------------------------- |
| `commands`     | string \| array           | No       | Additional command files or directories                            | `"./custom/cmd.md"` or `["./a.md", "./b.md"]` |
| `agents`       | string \| array           | No       | Additional agent files                                             | `"./custom/agents/reviewer.md"`               |
| `skills`       | string \| array           | No       | Additional skill directories                                       | `"./custom/skills/"`                          |
| `hooks`        | string \| array \| object | No       | Hook config file paths or an inline hook configuration object      | `"./my-extra-hooks.json"`                     |
| `mcpServers`   | string \| array \| object | No       | MCP config file paths or an inline MCP server configuration object | `"./my-extra-mcp-config.json"`                |
| `outputStyles` | string \| array           | No       | Additional output style files or directories                       | `"./styles/"`                                 |
| `lspServers`   | string \| array \| object | No       | LSP server config file paths or an inline LSP configuration object | `"./.lsp.json"`                               |

### LSP Server Fields

When `lspServers` is specified inline (as an object), each key is a language identifier. Required and optional subfields:

| Field                   | Required | Description                                                        |
| :---------------------- | :------- | :----------------------------------------------------------------- |
| `command`               | Yes      | LSP binary to execute (must be in PATH)                            |
| `extensionToLanguage`   | Yes      | Object mapping file extensions to language identifiers             |
| `args`                  | No       | Array of command-line arguments for the LSP server                 |
| `transport`             | No       | Communication transport: `stdio` (default) or `socket`             |
| `env`                   | No       | Object of environment variables set when starting the server       |
| `initializationOptions` | No       | Options passed to the server during initialization                 |
| `settings`              | No       | Settings passed via `workspace/didChangeConfiguration`             |
| `workspaceFolder`       | No       | Workspace folder path for the server                               |
| `startupTimeout`        | No       | Max milliseconds to wait for server startup                        |
| `shutdownTimeout`       | No       | Max milliseconds to wait for graceful shutdown                     |
| `restartOnCrash`        | No       | Boolean; whether to automatically restart the server if it crashes |
| `maxRestarts`           | No       | Maximum restart attempts before giving up                          |

---

## Plugin Installation Scopes

When installing a plugin, a scope determines which settings file receives the entry and who can use the plugin.

| Scope     | Settings File                 | Description                                             |
| :-------- | :---------------------------- | :------------------------------------------------------ |
| `user`    | `~/.claude/settings.json`     | Personal; available across all projects. Default scope. |
| `project` | `.claude/settings.json`       | Shared with team via version control                    |
| `local`   | `.claude/settings.local.json` | Project-specific; gitignored, not shared                |
| `managed` | Managed settings (read-only)  | Set by administrators; users can update but not remove  |

---

## CLI Commands

All plugin management commands follow the pattern `claude plugin <subcommand>`. The `plugin@marketplace` syntax targets a specific marketplace when multiple are configured.

### plugin install

```
claude plugin install <plugin> [options]
```

| Argument / Option     | Description                                                               | Default |
| :-------------------- | :------------------------------------------------------------------------ | :------ |
| `<plugin>`            | Plugin name, or `plugin-name@marketplace-name` for a specific marketplace | —       |
| `-s, --scope <scope>` | Installation scope: `user`, `project`, or `local`                         | `user`  |
| `-h, --help`          | Display help                                                              | —       |

```bash
# Install to user scope (default)
claude plugin install formatter@my-marketplace

# Install to project scope (shared with team via git)
claude plugin install formatter@my-marketplace --scope project

# Install to local scope (gitignored)
claude plugin install formatter@my-marketplace --scope local
```

### plugin uninstall

Aliases: `remove`, `rm`

```
claude plugin uninstall <plugin> [options]
```

| Argument / Option     | Description                                            | Default |
| :-------------------- | :----------------------------------------------------- | :------ |
| `<plugin>`            | Plugin name or `plugin-name@marketplace-name`          | —       |
| `-s, --scope <scope>` | Scope to uninstall from: `user`, `project`, or `local` | `user`  |
| `-h, --help`          | Display help                                           | —       |

### plugin enable

```
claude plugin enable <plugin> [options]
```

| Argument / Option     | Description                                   | Default |
| :-------------------- | :-------------------------------------------- | :------ |
| `<plugin>`            | Plugin name or `plugin-name@marketplace-name` | —       |
| `-s, --scope <scope>` | Scope: `user`, `project`, or `local`          | `user`  |
| `-h, --help`          | Display help                                  | —       |

### plugin disable

Disables a plugin without uninstalling it.

```
claude plugin disable <plugin> [options]
```

| Argument / Option     | Description                                   | Default |
| :-------------------- | :-------------------------------------------- | :------ |
| `<plugin>`            | Plugin name or `plugin-name@marketplace-name` | —       |
| `-s, --scope <scope>` | Scope: `user`, `project`, or `local`          | `user`  |
| `-h, --help`          | Display help                                  | —       |

### plugin update

```
claude plugin update <plugin> [options]
```

| Argument / Option     | Description                                     | Default |
| :-------------------- | :---------------------------------------------- | :------ |
| `<plugin>`            | Plugin name or `plugin-name@marketplace-name`   | —       |
| `-s, --scope <scope>` | Scope: `user`, `project`, `local`, or `managed` | `user`  |
| `-h, --help`          | Display help                                    | —       |

### plugin validate

Validates plugin JSON syntax and structure.

```bash
claude plugin validate ./path/to/plugin
```

Can also be run from within the Claude Code TUI:

```
/plugin validate .
```

---

## Environment Variables

### `${CLAUDE_PLUGIN_ROOT}`

Contains the absolute path to the plugin's installation directory. Use this variable in all paths inside `hooks.json` and MCP server configurations. This is required because marketplace plugins are copied to a cache location on install; absolute paths baked at development time will be wrong in the cache.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PLUGIN_ROOT}/scripts/format-code.sh"
          }
        ]
      }
    ]
  }
}
```

```json
{
  "mcpServers": {
    "my-server": {
      "command": "${CLAUDE_PLUGIN_ROOT}/servers/run.js",
      "args": ["--config", "${CLAUDE_PLUGIN_ROOT}/config.json"]
    }
  }
}
```

---

## Plugin Caching

### Cache Location

Marketplace plugins are copied to a local cache directory rather than used in-place:

```
~/.claude/plugins/cache/
```

Plugins loaded via `--plugin-dir` are used directly (no cache copy). Only marketplace-installed plugins go through the cache.

### Path Traversal Limitations

Installed plugins cannot reference files outside their plugin directory. Paths that traverse upward (e.g. `../shared-utils`) will fail because those files are not copied into the cache.

### Symlink Workaround

If a plugin needs to incorporate external files, create symlinks to those files inside the plugin directory before distributing. Symlinks are followed during the cache copy:

```bash
# Inside the plugin directory
ln -s /path/to/shared-utils ./shared-utils
```

The symlinked content is copied into the cache alongside the rest of the plugin files.

### Cache and Version Detection

Claude Code uses the version field to determine whether a plugin needs updating. If plugin code changes without a version bump, existing users will not receive the update due to caching. Always increment the version in `plugin.json` before distributing a new release.

---

## Marketplace Registry Schema

A marketplace is a repository or directory containing `.claude-plugin/marketplace.json`. This file lists available plugins and where to find them.

### Top-Level Fields

| Field      | Type   | Required | Description                                                                                                     |
| :--------- | :----- | :------- | :-------------------------------------------------------------------------------------------------------------- |
| `name`     | string | Yes      | Marketplace identifier (kebab-case, no spaces). Users reference it as `plugin@marketplace-name` during install. |
| `owner`    | object | Yes      | Maintainer information (see below)                                                                              |
| `plugins`  | array  | Yes      | Array of plugin entry objects (see below)                                                                       |
| `metadata` | object | No       | Additional marketplace metadata (see below)                                                                     |

### Owner Fields

| Field   | Type   | Required | Description                      |
| :------ | :----- | :------- | :------------------------------- |
| `name`  | string | Yes      | Name of the maintainer or team   |
| `email` | string | No       | Contact email for the maintainer |

### Metadata Fields

| Field                  | Type   | Description                                                                                                                                                              |
| :--------------------- | :----- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `metadata.description` | string | Brief marketplace description                                                                                                                                            |
| `metadata.version`     | string | Marketplace version                                                                                                                                                      |
| `metadata.pluginRoot`  | string | Base directory prepended to relative plugin source paths. For example, `"./plugins"` lets you write `"source": "formatter"` instead of `"source": "./plugins/formatter"` |

### Plugin Entry Fields

Each object in the `plugins` array supports all fields from the plugin manifest schema plus the marketplace-specific fields below.

#### Required

| Field    | Type             | Description                                                                                                           |
| :------- | :--------------- | :-------------------------------------------------------------------------------------------------------------------- |
| `name`   | string           | Plugin identifier (kebab-case). Public-facing; users see it when installing.                                          |
| `source` | string \| object | Where to fetch the plugin. A string starting with `./` is a relative path. Objects specify typed sources (see below). |

#### Optional Metadata

| Field         | Type    | Description                                                              |
| :------------ | :------ | :----------------------------------------------------------------------- |
| `description` | string  | Brief plugin description                                                 |
| `version`     | string  | Plugin version. Overridden by `plugin.json` version if both are present. |
| `author`      | object  | Author info: `name` (required), `email` (optional)                       |
| `homepage`    | string  | Plugin homepage or documentation URL                                     |
| `repository`  | string  | Source code repository URL                                               |
| `license`     | string  | SPDX license identifier                                                  |
| `keywords`    | array   | Tags for discovery                                                       |
| `category`    | string  | Plugin category for organization                                         |
| `tags`        | array   | Additional searchability tags                                            |
| `strict`      | boolean | Controls component authority (see below). Default: `true`                |

#### Optional Component Paths

| Field        | Type             | Description                                          |
| :----------- | :--------------- | :--------------------------------------------------- |
| `commands`   | string \| array  | Custom paths to command files or directories         |
| `agents`     | string \| array  | Custom paths to agent files                          |
| `hooks`      | string \| object | Custom hooks configuration or path to hooks file     |
| `mcpServers` | string \| object | MCP server configurations or path to MCP config file |
| `lspServers` | string \| object | LSP server configurations or path to LSP config file |

#### Strict Mode

The `strict` field controls which source is authoritative for component definitions.

| Value            | Behavior                                                                                                                                                          |
| :--------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `true` (default) | `plugin.json` is the authority. The marketplace entry can supplement it with additional components; both sources are merged.                                      |
| `false`          | The marketplace entry is the entire definition. If the plugin also has a `plugin.json` that declares components, that is a conflict and the plugin fails to load. |

### Plugin Source Types

The `source` field in each plugin entry tells Claude Code where to fetch the plugin.

| Source type   | Format                          | Required fields            | Optional fields       |
| :------------ | :------------------------------ | :------------------------- | :-------------------- |
| Relative path | `"./relative/path"` (string)    | —                          | —                     |
| GitHub        | `{"source": "github", ...}`     | `repo`                     | `ref`, `sha`          |
| Git URL       | `{"source": "url", ...}`        | `url` (must end in `.git`) | `ref`, `sha`          |
| Git subdir    | `{"source": "git-subdir", ...}` | `url`, `path`              | `ref`, `sha`          |
| npm package   | `{"source": "npm", ...}`        | `package`                  | `version`, `registry` |
| pip package   | `{"source": "pip", ...}`        | `package`                  | `version`, `registry` |

**Relative paths** resolve relative to the marketplace root (the directory containing `.claude-plugin/`). Must start with `./`. Only work when the marketplace is added via Git; URL-based distribution requires an external source type.

**GitHub source fields:**

| Field  | Type   | Required | Description                                          |
| :----- | :----- | :------- | :--------------------------------------------------- |
| `repo` | string | Yes      | Repository in `owner/repo` format                    |
| `ref`  | string | No       | Branch or tag; defaults to repository default branch |
| `sha`  | string | No       | Full 40-character commit SHA for exact pinning       |

**Git URL source fields:**

| Field | Type   | Required | Description                                             |
| :---- | :----- | :------- | :------------------------------------------------------ |
| `url` | string | Yes      | Full git repository URL. The `.git` suffix is optional. |
| `ref` | string | No       | Branch or tag; defaults to repository default branch    |
| `sha` | string | No       | Full 40-character commit SHA for exact pinning          |

**Git subdir source fields:**

| Field  | Type   | Required | Description                                                                        |
| :----- | :----- | :------- | :--------------------------------------------------------------------------------- |
| `url`  | string | Yes      | Git URL, GitHub `owner/repo` shorthand, or SSH URL                                 |
| `path` | string | Yes      | Subdirectory path within the repo containing the plugin (e.g. `"tools/my-plugin"`) |
| `ref`  | string | No       | Branch or tag; defaults to repository default branch                               |
| `sha`  | string | No       | Full 40-character commit SHA for exact pinning                                     |

**npm source fields:**

| Field      | Type   | Required | Description                                                                         |
| :--------- | :----- | :------- | :---------------------------------------------------------------------------------- |
| `package`  | string | Yes      | Package name or scoped package (e.g. `@org/plugin`)                                 |
| `version`  | string | No       | Version or version range (e.g. `2.1.0`, `^2.0.0`, `~1.5.0`)                         |
| `registry` | string | No       | Custom npm registry URL. Defaults to the system npm registry (typically npmjs.org). |

### Minimal Marketplace Example

```json
{
  "name": "my-plugins",
  "owner": {
    "name": "Your Name"
  },
  "plugins": [
    {
      "name": "code-formatter",
      "source": "./plugins/formatter",
      "description": "Automatic code formatting on save"
    },
    {
      "name": "deployment-tools",
      "source": {
        "source": "github",
        "repo": "owner/deploy-plugin",
        "ref": "v2.0.0"
      },
      "description": "Deployment automation tools",
      "version": "2.0.0"
    }
  ]
}
```

---

## Plugin Lifecycle

```
install → cache → enable/disable → update → uninstall
```

| Stage         | Description                                                                                                                                      |
| :------------ | :----------------------------------------------------------------------------------------------------------------------------------------------- |
| **Install**   | `claude plugin install <plugin>` fetches the plugin from its source and writes its name to the chosen scope's `enabledPlugins` in settings.      |
| **Cache**     | The plugin directory is copied to `~/.claude/plugins/cache/`. Path-based plugins (`--plugin-dir`) skip this step and are used directly.          |
| **Enable**    | Plugin is active and its components (commands, agents, hooks, MCP servers) are loaded by Claude Code.                                            |
| **Disable**   | `claude plugin disable <plugin>` deactivates the plugin without removing the cached copy. Re-enable with `claude plugin enable`.                 |
| **Update**    | `claude plugin update <plugin>` fetches the latest version from the source and replaces the cached copy. Version must differ to trigger a fetch. |
| **Uninstall** | `claude plugin uninstall <plugin>` removes the plugin from settings and deletes the cached copy.                                                 |

Marketplace plugins force-enabled by managed settings cannot be overridden by `--plugin-dir`. All other installed plugins can be temporarily overridden by loading a local copy with the same name via `--plugin-dir`.

---

## Version Management

Plugin versions follow [Semantic Versioning](https://semver.org/) (`MAJOR.MINOR.PATCH`):

| Segment | When to increment                            |
| :------ | :------------------------------------------- |
| MAJOR   | Breaking changes (incompatible API changes)  |
| MINOR   | New features (backward-compatible additions) |
| PATCH   | Bug fixes (backward-compatible fixes)        |

Version is declared in `plugin.json`:

```json
{
  "name": "my-plugin",
  "version": "2.1.0"
}
```

**Priority rule**: If `version` is set in both `plugin.json` and the marketplace entry, `plugin.json` always wins silently. To avoid confusion, set the version in only one place. For relative-path plugins that live inside the marketplace repository, prefer setting it in the marketplace entry. For externally hosted plugins, prefer `plugin.json`.

**Update detection**: Claude Code compares the installed cached version against the version reported by the source. If the versions are identical, no update is performed — even if the underlying files changed. Always increment the version before distributing new code.

Pre-release identifiers are supported: `2.0.0-beta.1`, `1.5.0-rc.2`, etc.

---

## Auto-Discovery

The plugin manifest is optional. When `.claude-plugin/plugin.json` is absent, Claude Code auto-discovers components using the following rules:

| Component   | Auto-discovered from                                                   |
| :---------- | :--------------------------------------------------------------------- |
| Plugin name | Directory name of the plugin root                                      |
| Commands    | `commands/` at plugin root — all `.md` files                           |
| Agents      | `agents/` at plugin root — all `.md` files                             |
| Skills      | `skills/` at plugin root — subdirectories each containing a `SKILL.md` |
| Hooks       | `hooks/hooks.json` at plugin root                                      |
| MCP servers | `.mcp.json` at plugin root                                             |
| LSP servers | `.lsp.json` at plugin root                                             |
| Settings    | `settings.json` at plugin root                                         |

Auto-discovery scans these default locations unconditionally. When a manifest is present, any custom paths declared in it are loaded **in addition to** whatever exists at the default locations — they do not suppress default discovery.

To skip auto-discovery for specific components, the manifest must not declare those component paths and the default directories must not exist. There is no explicit opt-out flag.
