# Skills Reference

> Sourced from official Claude Code documentation at https://code.claude.com/docs/en/skills

---

## Overview

Skills extend Claude's capabilities. Create a `SKILL.md` file with instructions, and Claude adds it to its toolkit. Claude uses skills when relevant, or you can invoke one directly with `/skill-name`.

Claude Code skills follow the [Agent Skills open standard](https://agentskills.io), which works across multiple AI tools. Claude Code extends the standard with additional features including invocation control, subagent execution, and dynamic context injection.

**Custom commands merged into skills.** A file at `.claude/commands/deploy.md` and a skill at `.claude/skills/deploy/SKILL.md` both create `/deploy` and work the same way. Existing `.claude/commands/` files continue to work. Skills add optional features: a directory for supporting files, frontmatter to control whether you or Claude invokes them, and the ability for Claude to load them automatically when relevant.

---

## Where Skills Live

Where you store a skill determines who can use it. When skills share the same name across levels, higher-priority locations win.

| Priority (high to low) | Location   | Path                                     | Applies to                     |
| :--------------------- | :--------- | :--------------------------------------- | :----------------------------- |
| 1                      | Enterprise | Managed settings                         | All users in your organization |
| 2                      | Personal   | `~/.claude/skills/<skill-name>/SKILL.md` | All your projects              |
| 3                      | Project    | `.claude/skills/<skill-name>/SKILL.md`   | This project only              |
| 4                      | Plugin     | `<plugin>/skills/<skill-name>/SKILL.md`  | Where plugin is enabled        |

Plugin skills use a `plugin-name:skill-name` namespace, so they cannot conflict with other levels. If a skill and a `.claude/commands/` file share the same name, the skill takes precedence.

### Nested Discovery

When you work with files in subdirectories, Claude Code automatically discovers skills from nested `.claude/skills/` directories. For example, if you are editing a file in `packages/frontend/`, Claude Code also looks for skills in `packages/frontend/.claude/skills/`. This supports monorepo setups where packages have their own skills.

### Skills from Additional Directories

Skills defined in `.claude/skills/` within directories added via `--add-dir` are loaded automatically and picked up by live change detection, so you can edit them during a session without restarting.

---

## Skill Directory Layout

Each skill is a directory with `SKILL.md` as the required entrypoint. Additional files are optional.

```
my-skill/
├── SKILL.md           # Main instructions (required)
├── reference.md       # Detailed docs — loaded when needed
├── examples/
│   └── sample.md      # Example output showing expected format
└── scripts/
    └── validate.sh    # Script Claude can execute
```

Keep `SKILL.md` under 500 lines. Move detailed reference material to separate files and reference them from `SKILL.md` so Claude knows what each file contains and when to load it.

---

## SKILL.md Frontmatter Schema

Configure skill behavior using YAML frontmatter between `---` markers at the top of `SKILL.md`. All fields are optional; `description` is strongly recommended.

```yaml
---
name: my-skill
description: What this skill does and when to use it
allowed-tools: Read, Grep
---

Your skill instructions here...
```

| Field                      | Required    | Description                                                                                                                                                             |
| :------------------------- | :---------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `name`                     | No          | Display name for the skill. Lowercase letters, numbers, and hyphens only, max 64 characters. If omitted, uses the directory name.                                       |
| `description`              | Recommended | What the skill does and when to use it. Claude uses this to decide when to apply the skill automatically. If omitted, uses the first paragraph of markdown content.     |
| `argument-hint`            | No          | Hint shown during autocomplete to indicate expected arguments. Example: `[issue-number]` or `[filename] [format]`.                                                      |
| `disable-model-invocation` | No          | Set to `true` to prevent Claude from automatically loading this skill. Use for workflows you want to trigger manually. Default: `false`.                                |
| `user-invocable`           | No          | Set to `false` to hide from the `/` menu. Use for background knowledge users should not invoke directly. Default: `true`.                                               |
| `allowed-tools`            | No          | Tools Claude can use without asking permission when this skill is active.                                                                                               |
| `model`                    | No          | Model to use when this skill is active.                                                                                                                                 |
| `context`                  | No          | Set to `fork` to run in a forked subagent context.                                                                                                                      |
| `agent`                    | No          | Which subagent type to use when `context: fork` is set. Options: `Explore`, `Plan`, `general-purpose`, or any custom subagent name. If omitted, uses `general-purpose`. |
| `hooks`                    | No          | Hooks scoped to this skill's lifecycle.                                                                                                                                 |

---

## String Substitutions

Skills support string substitution for dynamic values in skill content.

| Variable               | Description                                                                                                                                                                                                                                   |
| :--------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `$ARGUMENTS`           | All arguments passed when invoking the skill. If not present in the content, arguments are appended as `ARGUMENTS: <value>`.                                                                                                                  |
| `$ARGUMENTS[N]`        | Access a specific argument by 0-based index. Example: `$ARGUMENTS[0]` for the first argument.                                                                                                                                                 |
| `$N`                   | Shorthand for `$ARGUMENTS[N]`. Example: `$0` for the first argument, `$1` for the second.                                                                                                                                                     |
| `${CLAUDE_SESSION_ID}` | The current session ID. Useful for logging, creating session-specific files, or correlating skill output with sessions.                                                                                                                       |
| `${CLAUDE_SKILL_DIR}`  | The directory containing the skill's `SKILL.md` file. For plugin skills, this is the skill's subdirectory within the plugin, not the plugin root. Use this to reference bundled scripts or files regardless of the current working directory. |

Example:

```yaml
---
name: session-logger
description: Log activity for this session
---

Log the following to logs/${CLAUDE_SESSION_ID}.log:

$ARGUMENTS
```

---

## Invocation Control

By default, both you and Claude can invoke any skill. Two frontmatter fields let you restrict this.

| Frontmatter                      | User can invoke | Claude can invoke | When loaded into context                                     |
| :------------------------------- | :-------------- | :---------------- | :----------------------------------------------------------- |
| (default)                        | Yes             | Yes               | Description always in context; full skill loads when invoked |
| `disable-model-invocation: true` | Yes             | No                | Description not in context; full skill loads when you invoke |
| `user-invocable: false`          | No              | Yes               | Description always in context; full skill loads when invoked |

**`disable-model-invocation: true`** — Use for workflows with side effects or timing requirements you want to control, such as `/commit`, `/deploy`, or `/send-slack-message`.

**`user-invocable: false`** — Use for background knowledge that is not actionable as a command. For example, a `legacy-system-context` skill that explains an old system: Claude should know this when relevant, but invoking it as a command is not meaningful.

Note: `user-invocable` only controls menu visibility, not Skill tool access. Use `disable-model-invocation: true` to block programmatic invocation by Claude.

---

## Dynamic Context Injection

The `` !`command` `` syntax runs shell commands before the skill content is sent to Claude. The command output replaces the placeholder, so Claude receives actual data rather than the command itself. This is preprocessing — Claude only sees the final rendered result.

Example — fetch live pull request data before Claude sees the prompt:

```yaml
---
name: pr-summary
description: Summarize changes in a pull request
context: fork
agent: Explore
allowed-tools: Bash(gh *)
---

## Pull request context
- PR diff: !`gh pr diff`
- PR comments: !`gh pr view --comments`
- Changed files: !`gh pr diff --name-only`

## Your task
Summarize this pull request...
```

When this skill runs, each `` !`command` `` executes immediately, its output replaces the placeholder, and Claude receives the fully-rendered prompt with actual PR data.

---

## Subagent Execution

Add `context: fork` to frontmatter to run a skill in an isolated subagent. The skill content becomes the prompt that drives the subagent. The subagent has no access to your conversation history.

`context: fork` only makes sense for skills with explicit instructions. If your skill contains guidelines without an actionable task, the subagent returns without meaningful output.

The `agent` field specifies which subagent configuration to use. Options: `Explore`, `Plan`, `general-purpose`, or any custom subagent from `.claude/agents/`. If omitted, uses `general-purpose`.

| Approach                     | System prompt                             | Task                | Also loads                   |
| :--------------------------- | :---------------------------------------- | :------------------ | :--------------------------- |
| Skill with `context: fork`   | From agent type (`Explore`, `Plan`, etc.) | SKILL.md content    | CLAUDE.md                    |
| Subagent with `skills` field | Subagent's markdown body                  | Claude's delegation | Preloaded skills + CLAUDE.md |

Example — research skill using the Explore agent:

```yaml
---
name: deep-research
description: Research a topic thoroughly
context: fork
agent: Explore
---

Research $ARGUMENTS thoroughly:

1. Find relevant files using Glob and Grep
2. Read and analyze the code
3. Summarize findings with specific file references
```

---

## Supporting Files

`SKILL.md` is the required entrypoint. Additional files in the skill directory are optional and let you build more powerful skills. Large reference docs, API specifications, or example collections do not need to load into context every time the skill runs.

Reference supporting files from `SKILL.md` so Claude knows what each file contains and when to load it:

```markdown
## Additional resources

- For complete API details, see [reference.md](reference.md)
- For usage examples, see [examples.md](examples.md)
```

Keep `SKILL.md` under 500 lines. Move detailed reference material to separate files.

---

## Permission Restrictions

By default, Claude can invoke any skill that does not have `disable-model-invocation: true` set. Your permission settings govern baseline approval behavior for all tools. Skills that define `allowed-tools` grant Claude access to those tools without per-use approval when the skill is active.

Three ways to control which skills Claude can invoke:

**Disable all skills** by denying the Skill tool in permissions:

```
Skill
```

**Allow or deny specific skills** using permission rules:

```
# Allow only specific skills
Skill(commit)
Skill(review-pr *)

# Deny specific skills
Skill(deploy *)
```

Permission syntax:

| Syntax          | Match behavior                                   |
| :-------------- | :----------------------------------------------- |
| `Skill(name)`   | Exact match                                      |
| `Skill(name *)` | Prefix match — matches `name` with any arguments |

**Hide individual skills** by adding `disable-model-invocation: true` to their frontmatter. This removes the skill from Claude's context entirely.

---

## Skill Budget

Skill descriptions are loaded into context at runtime so Claude knows what is available. The budget scales dynamically at **2% of the context window**, with a fallback of **16,000 characters**.

If you have many skills, they may exceed the character budget and some will be excluded. Run `/context` to check for a warning about excluded skills.

Override the limit with the `SLASH_COMMAND_TOOL_CHAR_BUDGET` environment variable.

In a regular session, only skill descriptions are loaded into context; full skill content loads when invoked. Subagents with preloaded skills work differently: the full skill content is injected at startup.

---

## Related

- [Subagents](https://code.claude.com/docs/en/sub-agents) — delegate tasks to specialized agents
- [Plugins](https://code.claude.com/docs/en/plugins) — package and distribute skills with other extensions
- [Hooks](https://code.claude.com/docs/en/hooks) — automate workflows around tool events
- [Memory](https://code.claude.com/docs/en/memory) — manage CLAUDE.md files for persistent context
- [Built-in commands](https://code.claude.com/docs/en/commands) — reference for built-in `/` commands
- [Permissions](https://code.claude.com/docs/en/permissions) — control tool and skill access
