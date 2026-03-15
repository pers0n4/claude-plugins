# Conventional Commits v1.0.0 Reference

Reference material for the commit skill. Read this file when you need details on types, scopes, footers, or breaking changes.

## Types

Pick the single most accurate type based on what the diff actually does:

| Type       | When to use                                              |
| ---------- | -------------------------------------------------------- |
| `feat`     | A new capability that didn't exist before (SemVer MINOR) |
| `fix`      | Corrects a bug or wrong behavior (SemVer PATCH)          |
| `refactor` | Restructures code without changing behavior              |
| `chore`    | Maintenance (deps, config, build, tooling)               |
| `docs`     | Documentation only                                       |
| `style`    | Formatting, whitespace, semicolons — no logic change     |
| `test`     | Adding or fixing tests                                   |
| `perf`     | Performance improvement                                  |
| `ci`       | CI/CD pipeline changes                                   |
| `build`    | Build system or external dependency changes              |
| `revert`   | Reverts a previous commit (add `Refs: <SHA>` footer)     |

Only `feat` and `fix` are defined by the spec. The others come from the Angular convention and are widely adopted. Types other than `feat` and `fix` have no implicit SemVer effect unless they also include a BREAKING CHANGE.

If the diff spans multiple types (e.g. a feature + a refactor), prefer the primary intent. If they're truly independent changes, suggest splitting into separate commits.

## Scope Heuristics

Derive scope from the area of the codebase that changed. Scope should be a short noun (1-2 words, lowercase, no spaces), wrapped in parentheses after the type.

- Changes in a single directory/module → use that module name (e.g. `auth`, `api`, `components`, `store`)
- Changes across many directories with no clear focus → omit scope
- Single file change → scope from the file's purpose, not its name
- Root config files → `config`
- Package dependency changes only → `deps`

Check the project's CLAUDE.md for project-specific scope conventions if available.

When in doubt, omit the scope. A missing scope is better than a misleading one.

## Footers

Footers appear one blank line after the body. Each footer uses one of two separator formats:

- `token: value` — colon + space (e.g. `Reviewed-by: Alice`)
- `token #value` — space + hash (e.g. `Fixes #123`)

Footer tokens use `-` in place of spaces (e.g. `Acked-by`, `Reviewed-by`). The sole exception is `BREAKING CHANGE` which keeps its space.

A footer's value may span multiple lines; parsing terminates when the next valid token/separator pair is observed.

### Common footer patterns

- `Fixes #<issue>` or `Closes #<issue>` — links commit to an issue tracker
- `Refs: <SHA>` — references related commits (especially for `revert` type; note: `git revert` auto-generates `This reverts commit <SHA>.` as body text — using a `Refs:` footer is an alternative convention)
- `Reviewed-by: <name>` — attribution
- `BREAKING CHANGE: <description>` — see section below

Include issue references when the commit directly resolves a tracked issue and the user mentions it.

## Breaking Changes

Breaking changes correlate to a SemVer MAJOR bump. They MUST be indicated in one of two ways (or both):

**Method A — `!` in the type/scope prefix:**

- Place `!` immediately before the `:`
- Example: `feat!: drop Node 14 support` or `feat(api)!: change response format`
- When `!` is used, the `BREAKING CHANGE:` footer MAY be omitted; the description itself describes the breaking change

**Method B — Footer token:**

- `BREAKING CHANGE: <description>` (MUST be uppercase — the only case-sensitive token in the spec)
- `BREAKING-CHANGE` (with hyphen) is synonymous

Both methods MAY be used together for maximum clarity.

## Subject Line Rules

- Start with a lowercase verb in imperative mood (`add`, `fix`, `remove`, not `added`, `fixes`, `removing`)
- Be specific about what changed, not vague ("update code" is useless)
- Focus on _what_ the change does, not _how_
- Keep the full line (including type and scope) under 72 characters (widely adopted convention, not spec-mandated)
- No period at the end
- English only

## Body Rules

- Separate from subject with a blank line
- Explain _why_ the change was made, not _what_ (the diff shows _what_)
- Wrap at 72 characters
- Free-form; may consist of any number of newline-separated paragraphs
