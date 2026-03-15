# MCP (Model Context Protocol) Reference

> Source: https://code.claude.com/docs/en/mcp — compiled from official Claude Code documentation.

---

## Overview

MCP (Model Context Protocol) is an open source standard for AI-tool integrations. Claude Code connects to external tools, databases, and APIs via MCP servers. Each server exposes tools, resources, and/or prompts that Claude can discover and invoke during a session.

---

## Server Transport Types

| Transport | Description                                  | Status                         |
| --------- | -------------------------------------------- | ------------------------------ |
| `stdio`   | Local process communication via stdin/stdout | Stable                         |
| `sse`     | Remote Server-Sent Events over HTTP          | Deprecated — use `http`        |
| `http`    | Streamable HTTP (bidirectional)              | Recommended for remote servers |

---

## Installation Methods

All flags (`--transport`, `--env`, `--scope`, `--header`) must appear **before** the server name. `--` separates the server name from the command and its arguments.

```sh
# Remote HTTP server (recommended)
claude mcp add --transport http <name> <url>

# Remote SSE server (deprecated)
claude mcp add --transport sse <name> <url>

# Local stdio server
claude mcp add --transport stdio <name> -- <command> [args...]

# Add from raw JSON
claude mcp add-json <name> '<json>'

# Import from Claude Desktop config
claude mcp add-from-claude-desktop
```

### Common flags

| Flag                 | Description                                               |
| -------------------- | --------------------------------------------------------- |
| `--transport <type>` | Transport type: `stdio`, `sse`, or `http`                 |
| `--scope <scope>`    | Config scope: `local` (default), `project`, or `user`     |
| `--env KEY=VALUE`    | Environment variable passed to the server process         |
| `--header KEY=VALUE` | HTTP header sent with every request (HTTP/SSE transports) |
| `--client-id`        | OAuth 2.0 client ID                                       |
| `--client-secret`    | OAuth 2.0 client secret                                   |
| `--callback-port`    | OAuth 2.0 callback port                                   |

---

## Configuration Scopes

| Scope             | Storage location                    | Visibility                                     |
| ----------------- | ----------------------------------- | ---------------------------------------------- |
| `local` (default) | `~/.claude.json` under project path | Private to user, current project only          |
| `project`         | `.mcp.json` at project root         | Checked into version control, shared with team |
| `user`            | `~/.claude.json` (global section)   | Private to user, all projects                  |

**Precedence**: `local` > `project` > `user`

When the same server name exists at multiple scopes, the higher-precedence entry wins.

---

## `.mcp.json` Schema

Place `.mcp.json` at the project root to share server configuration with your team.

```json
{
  "mcpServers": {
    "server-name": {
      "type": "stdio",
      "command": "path/to/server",
      "args": ["--flag", "value"],
      "env": {
        "API_KEY": "${MY_API_KEY}",
        "BASE_URL": "${BASE_URL:-https://api.example.com}"
      }
    },
    "remote-server": {
      "type": "http",
      "url": "https://mcp.example.com/server",
      "headers": {
        "Authorization": "Bearer ${TOKEN}"
      }
    }
  }
}
```

### Environment variable expansion

Variable expansion is supported in: `command`, `args`, `env` values, `url`, and `headers` values.

| Syntax            | Meaning                                       |
| ----------------- | --------------------------------------------- |
| `${VAR}`          | Expand `VAR`; error if unset                  |
| `${VAR:-default}` | Expand `VAR`; use `default` if unset or empty |

---

## Plugin MCP Integration

Plugins can expose MCP servers by declaring them in `.mcp.json` at the plugin root or inline in `plugin.json`.

- Use `${CLAUDE_PLUGIN_ROOT}` for paths relative to the plugin directory.
- Plugin MCP tools are namespaced as `mcp__<plugin-name>__<tool-name>`.
- Servers start automatically when the plugin is enabled; a session restart is required after configuration changes.
- All transport types (`stdio`, `sse`, `http`) are supported.

Example plugin `.mcp.json`:

```json
{
  "mcpServers": {
    "my-plugin-tools": {
      "type": "stdio",
      "command": "node",
      "args": ["${CLAUDE_PLUGIN_ROOT}/server/index.mjs"]
    }
  }
}
```

---

## Management Commands

| Command                               | Description                                                     |
| ------------------------------------- | --------------------------------------------------------------- |
| `claude mcp list`                     | List all configured servers                                     |
| `claude mcp get <name>`               | Show details for a server                                       |
| `claude mcp remove <name>`            | Remove a server                                                 |
| `claude mcp add-json <name> '<json>'` | Add a server from a JSON string                                 |
| `claude mcp add-from-claude-desktop`  | Import servers from Claude Desktop config                       |
| `claude mcp reset-project-choices`    | Reset per-project server approval decisions                     |
| `/mcp`                                | In-session: show server status and trigger OAuth authentication |

---

## Authentication

Remote MCP servers support OAuth 2.0. Use `/mcp` in-session to initiate the auth flow, or supply credentials at installation time:

```sh
claude mcp add --transport http my-server https://mcp.example.com \
  --client-id <id> \
  --client-secret <secret> \
  --callback-port 8085
```

---

## Output Limits

| Threshold     | Behavior                                        |
| ------------- | ----------------------------------------------- |
| 10,000 tokens | Warning displayed                               |
| 25,000 tokens | Default hard cap (response truncated)           |
| Custom        | Set `MAX_MCP_OUTPUT_TOKENS` env var to override |

---

## Tool Search

When MCP tools exceed 10% of the active context window, tool search is enabled automatically so Claude can locate the right tool without loading all schemas.

| `ENABLE_TOOL_SEARCH` value | Behavior                                |
| -------------------------- | --------------------------------------- |
| `auto` (default)           | Enable when tools exceed 10% of context |
| `auto:N%`                  | Enable when tools exceed N% of context  |
| `true`                     | Always enable                           |
| `false`                    | Always disable                          |

---

## MCP Resources

Resources expose read-only data (files, database records, API responses) that can be referenced directly in prompts.

- Reference syntax: `@server:protocol://resource/path`
- Resources are fuzzy-searchable via autocomplete when typing `@`.

---

## MCP Prompts

Servers can expose reusable prompt templates. Invoke them in-session as:

```
/mcp__<servername>__<promptname>
```

---

## Dynamic Tool Updates

Servers can emit an MCP `list_changed` notification to signal that their tool list has changed. Claude Code handles this automatically by refreshing the tool registry mid-session without requiring a restart.

---

## Managed Configuration

Organizations can push a read-only MCP configuration that applies to all users on a machine.

| Platform | Path                                                       |
| -------- | ---------------------------------------------------------- |
| macOS    | `/Library/Application Support/ClaudeCode/managed-mcp.json` |
| Linux    | `/etc/claude-code/managed-mcp.json`                        |

The file uses the same schema as `.mcp.json`. Managed servers cannot be overridden by user-level or project-level configuration.

---

## Startup Timeout

By default, Claude Code waits a fixed interval for each MCP server to initialize. Override with:

```sh
export MCP_TIMEOUT=30000   # milliseconds
```
