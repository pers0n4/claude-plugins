# Claude Code Hooks Reference

> Source: https://code.claude.com/docs/en/hooks — Official Claude Code documentation.

---

## Overview

Hooks are user-defined shell commands, HTTP endpoints, or LLM prompts that execute at specific lifecycle points during a Claude Code session. They let you intercept, validate, observe, or extend Claude's behavior without modifying any project code.

A hook can:

- Block an action (e.g. deny a tool call)
- Inject additional context into Claude's context window
- Post notifications to external services
- Log activity or enforce policy

Hooks that can block execution must complete within their timeout. Hooks that cannot block run asynchronously (or have very short timeouts) and their results are informational only.

---

## Configuration Locations

Hooks are registered in JSON settings files or plugin manifests. Claude Code loads them from multiple scopes and merges them:

| Scope            | File                                           | Notes                               |
| ---------------- | ---------------------------------------------- | ----------------------------------- |
| User             | `~/.claude/settings.json`                      | Applies to all projects             |
| Project (shared) | `.claude/settings.json`                        | Checked into version control        |
| Project (local)  | `.claude/settings.local.json`                  | Gitignored; overrides for local dev |
| Plugin           | `hooks/hooks.json` inside the plugin directory | Loaded when the plugin is active    |
| Skill/Agent      | Frontmatter of the skill or agent file         | Scoped to that skill/agent          |
| Managed policy   | Policy settings file                           | Org-level enforcement               |

---

## hooks.json Schema

```json
{
  "hooks": {
    "EventName": [
      {
        "matcher": "regex_pattern",
        "hooks": [
          {
            "type": "command",
            "command": "node \"${CLAUDE_PLUGIN_ROOT}/hooks/my-hook.mjs\"",
            "timeout": 600,
            "statusMessage": "Running check...",
            "async": false,
            "once": false
          }
        ]
      }
    ]
  }
}
```

### Top-level fields

| Field   | Type   | Description                                 |
| ------- | ------ | ------------------------------------------- |
| `hooks` | object | Map of event name → array of matcher groups |

### Matcher group fields

| Field     | Type   | Description                                                                                                |
| --------- | ------ | ---------------------------------------------------------------------------------------------------------- |
| `matcher` | string | Regular expression matched against the event's matcher value (tool name, source, etc.). Omit to match all. |
| `hooks`   | array  | List of hook handler objects to run when the matcher fires                                                 |

### Hook handler fields

| Field           | Type    | Default  | Description                                     |
| --------------- | ------- | -------- | ----------------------------------------------- |
| `type`          | string  | required | One of `command`, `http`, `prompt`, `agent`     |
| `command`       | string  | —        | Shell command (type=command) or URL (type=http) |
| `timeout`       | number  | 600      | Maximum seconds to wait before killing the hook |
| `statusMessage` | string  | —        | Message shown in the UI while the hook runs     |
| `async`         | boolean | false    | If true, runs in background and cannot block    |
| `once`          | boolean | false    | If true, fires only once per session            |

---

## Hook Handler Types

### command

Runs a shell command. Input is written to stdin as a JSON object. The hook communicates results via exit code and stdout JSON.

```json
{ "type": "command", "command": "node \"${CLAUDE_PLUGIN_ROOT}/hooks/pre-tool.mjs\"" }
```

- Use `${CLAUDE_PLUGIN_ROOT}` for portable paths inside plugins.
- Use `CLAUDE_ENV_FILE` (available on `SessionStart`) to source environment.
- Prefix the command with a unique env var when multiple plugins register the same script template to avoid Claude Code deduplication: `NOTIFIER=slack node "..."`

### http

POSTs the input JSON as the request body to the given URL. The response body is parsed as the output JSON.

```json
{ "type": "http", "command": "https://example.com/webhook" }
```

### prompt

Sends the event data to a fast LLM for evaluation. Returns `{ "ok": true/false, "reason": "..." }`. Useful for natural-language policy checks.

```json
{ "type": "prompt", "command": "Deny any request that touches files outside the project directory." }
```

### agent

Launches a limited multi-turn agent with access to read-only tools (Read, Grep, Glob) for complex verification that needs to inspect files before deciding. Returns the same output JSON format as other types.

```json
{ "type": "agent", "command": "Verify the changed file has a matching test file." }
```

---

## Supported Events

### Event table

| Event                | Matcher Values                                                                         | Supported Types | Can Block?           | Notes                                                          |
| -------------------- | -------------------------------------------------------------------------------------- | --------------- | -------------------- | -------------------------------------------------------------- |
| `SessionStart`       | `startup` / `resume` / `clear` / `compact`                                             | command         | No                   | Fires when a session is created or resumed                     |
| `SessionEnd`         | `clear` / `logout` / `prompt_input_exit` / `bypass_permissions_disabled` / `other`     | command         | No (1.5 s timeout)   | Very short timeout; keep handlers fast                         |
| `InstructionsLoaded` | _(none)_                                                                               | command         | No                   | Fires after CLAUDE.md and system prompts are loaded            |
| `UserPromptSubmit`   | _(none)_                                                                               | all             | Yes                  | Fires when the user submits a prompt                           |
| `PreToolUse`         | Tool names (see below)                                                                 | all             | Yes                  | Fires before Claude executes a tool; can deny or modify input  |
| `PermissionRequest`  | Tool names (see below)                                                                 | all             | Yes                  | Fires when Claude requests permission for a restricted tool    |
| `PostToolUse`        | Tool names (see below)                                                                 | all             | No (can add context) | Fires after a tool succeeds; output becomes additional context |
| `PostToolUseFailure` | Tool names (see below)                                                                 | all             | No                   | Fires after a tool fails                                       |
| `Notification`       | `permission_prompt` / `idle_prompt` / `auth_success` / `elicitation_dialog`            | command         | No                   | UI notification events                                         |
| `SubagentStart`      | Agent type names                                                                       | command         | No                   | Fires when a sub-agent is spawned                              |
| `SubagentStop`       | Agent type names                                                                       | all             | Yes                  | Fires when a sub-agent finishes                                |
| `Stop`               | _(none)_                                                                               | all             | Yes                  | Fires when Claude stops generating                             |
| `TeammateIdle`       | _(none)_                                                                               | all             | Yes                  | Fires when a teammate agent becomes idle                       |
| `TaskCompleted`      | _(none)_                                                                               | all             | Yes                  | Fires when an assigned task is marked complete                 |
| `ConfigChange`       | `user_settings` / `project_settings` / `local_settings` / `policy_settings` / `skills` | command         | Yes                  | Fires when a settings file changes                             |
| `PreCompact`         | `manual` / `auto`                                                                      | command         | No                   | Fires before context compaction                                |
| `PostCompact`        | `manual` / `auto`                                                                      | command         | No                   | Fires after context compaction                                 |
| `WorktreeCreate`     | _(none)_                                                                               | command         | Yes                  | Fires when a git worktree is created                           |
| `WorktreeRemove`     | _(none)_                                                                               | command         | No                   | Fires when a git worktree is removed                           |
| `Elicitation`        | MCP server name                                                                        | all             | Yes                  | Fires when an MCP server requests user input                   |
| `ElicitationResult`  | MCP server name                                                                        | all             | Yes                  | Fires after elicitation input is submitted                     |

### Tool name matcher values

For `PreToolUse`, `PermissionRequest`, `PostToolUse`, and `PostToolUseFailure`, the matcher is compared against the tool name:

| Matcher                 | Matches                                |
| ----------------------- | -------------------------------------- |
| `Bash`                  | Shell command execution                |
| `Edit`                  | File editing                           |
| `Write`                 | File creation/overwrite                |
| `Read`                  | File reading                           |
| `Glob`                  | File pattern search                    |
| `Grep`                  | Content search                         |
| `Agent`                 | Sub-agent invocation                   |
| `WebFetch`              | HTTP fetch                             |
| `WebSearch`             | Web search                             |
| `mcp__.*`               | Any MCP tool (regex)                   |
| `mcp__<server>__<tool>` | Specific MCP tool on a specific server |

---

## Common Input Fields

Every hook receives at minimum the following fields in its stdin JSON payload:

```json
{
  "session_id": "abc123",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/path/to/project",
  "permission_mode": "default",
  "hook_event_name": "PreToolUse",
  "agent_id": "agent-uuid",
  "agent_type": "executor"
}
```

| Field             | Description                                                        |
| ----------------- | ------------------------------------------------------------------ |
| `session_id`      | Unique identifier for the current session                          |
| `transcript_path` | Path to the JSONL file containing the full conversation transcript |
| `cwd`             | Working directory of the Claude Code process                       |
| `permission_mode` | Current permission mode (`default`, `bypass`, etc.)                |
| `hook_event_name` | Name of the event that fired this hook                             |
| `agent_id`        | Identifier of the agent that triggered the hook                    |
| `agent_type`      | Type/role of the triggering agent                                  |

Additional fields are provided per event. For example:

- `SessionStart`: `source` (startup/resume/clear/compact), `CLAUDE_ENV_FILE` path
- `UserPromptSubmit`: `prompt` (the user's message text)
- `PreToolUse` / `PostToolUse`: `tool_name`, `tool_input`, `tool_response`
- `SubagentStart` / `SubagentStop`: `agent_type`, `agent_id`
- `Notification`: `notification_type`, `message`
- `PreCompact` / `PostCompact`: `compact_reason` (manual/auto)

---

## Exit Code Behavior

Applies to `command` type hooks. Other types use response body or model output.

| Exit Code | Meaning                                                                                                                                          |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `0`       | Success. Claude parses stdout as JSON for output fields.                                                                                         |
| `2`       | Blocking error. stderr content is injected into Claude's context as an error message. Claude halts the current action and considers the message. |
| Any other | Non-blocking error. stderr is shown only in verbose/debug mode. Execution continues.                                                             |

---

## Output JSON Format

A hook handler may write a JSON object to stdout. All fields are optional. Unrecognised fields are ignored.

```json
{
  "continue": true,
  "stopReason": "Policy violation: file outside project",
  "suppressOutput": false,
  "systemMessage": "Note: this file was linted automatically.",
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "additionalContext": "..."
  }
}
```

| Field                | Type    | Description                                                        |
| -------------------- | ------- | ------------------------------------------------------------------ |
| `continue`           | boolean | If `false`, Claude stops the current action. Requires exit code 0. |
| `stopReason`         | string  | Human-readable explanation shown when `continue` is false          |
| `suppressOutput`     | boolean | If `true`, the hook's stdout is not echoed to the UI               |
| `systemMessage`      | string  | Text injected into Claude's system context for this turn           |
| `hookSpecificOutput` | object  | Event-specific additional output (see below)                       |

---

## PreToolUse Output

`PreToolUse` and `PermissionRequest` hooks support additional output fields inside `hookSpecificOutput`:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow",
    "permissionDecisionReason": "Approved by policy check.",
    "updatedInput": { "command": "echo sanitised" },
    "additionalContext": "Input was normalised."
  }
}
```

| Field                      | Values                   | Description                                                      |
| -------------------------- | ------------------------ | ---------------------------------------------------------------- |
| `permissionDecision`       | `allow` / `deny` / `ask` | Override the permission decision                                 |
| `permissionDecisionReason` | string                   | Reason shown to the user or Claude                               |
| `updatedInput`             | object                   | Replacement tool input; Claude uses this instead of the original |
| `additionalContext`        | string                   | Extra context appended to Claude's view of the tool result       |

---

## Environment Variables

| Variable             | Available In        | Description                                                                                                                                            |
| -------------------- | ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `CLAUDE_PROJECT_DIR` | All hooks           | Absolute path to the project root directory                                                                                                            |
| `CLAUDE_PLUGIN_ROOT` | Plugin hooks        | Absolute path to the plugin's root directory. Use in `hooks.json` command strings for portable paths: `node "${CLAUDE_PLUGIN_ROOT}/hooks/my-hook.mjs"` |
| `CLAUDE_ENV_FILE`    | `SessionStart` only | Path to a file whose contents should be sourced as environment variables for the session                                                               |

---

## Async Hooks

Setting `"async": true` on a hook handler causes it to run in the background after the event fires. Async hooks:

- Cannot block Claude's execution (no exit code 2 blocking)
- Cannot return `continue: false`
- Are suitable for notifications, logging, and side effects
- Do not delay the user experience

```json
{
  "type": "command",
  "command": "node \"${CLAUDE_PLUGIN_ROOT}/hooks/notify.mjs\"",
  "async": true
}
```

---

## MCP Tool Matching

MCP (Model Context Protocol) tools follow the naming pattern `mcp__<server>__<tool>`. Use this in matchers to target specific servers or tools:

| Matcher pattern             | What it matches                          |
| --------------------------- | ---------------------------------------- |
| `mcp__.*`                   | All tools on all MCP servers             |
| `mcp__my-server__.*`        | All tools on `my-server`                 |
| `mcp__my-server__read_file` | Only the `read_file` tool on `my-server` |

Matchers are regular expressions, so `mcp__.*` is a valid regex that matches all MCP tool events.

---

## Example: Minimal command hook

`hooks/hooks.json` inside a plugin:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "NOTIFIER=myplugin node \"${CLAUDE_PLUGIN_ROOT}/hooks/pre-tool-use.mjs\"",
            "timeout": 10
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "NOTIFIER=myplugin node \"${CLAUDE_PLUGIN_ROOT}/hooks/session-start.mjs\"",
            "async": true
          }
        ]
      }
    ]
  }
}
```

`hooks/pre-tool-use.mjs`:

```js
import { readFileSync } from 'fs';

const input = JSON.parse(readFileSync('/dev/stdin', 'utf8'));

if (input.tool_input?.command?.includes('rm -rf')) {
  // Exit 2 = blocking: stderr goes to Claude
  process.stderr.write('Destructive rm -rf is not allowed.');
  process.exit(2);
}

// Exit 0 = success, no output needed
process.exit(0);
```

---

_Last updated: 2026-03-15. Sourced from https://code.claude.com/docs/en/hooks._
