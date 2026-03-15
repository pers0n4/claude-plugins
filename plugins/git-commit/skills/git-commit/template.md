# Commit Message Templates

Use the HEREDOC format below. Pick the variation that matches the commit's complexity.

## Simple (subject only)

```bash
git commit -m "$(cat <<'EOF'
type(scope): message
EOF
)"
```

## With body

```bash
git commit -m "$(cat <<'EOF'
type(scope): message

Body explaining why this change was made.
Wrap at 72 characters.
EOF
)"
```

## With issue reference

```bash
git commit -m "$(cat <<'EOF'
type(scope): message

Fixes #123
EOF
)"
```

## With body and issue reference

```bash
git commit -m "$(cat <<'EOF'
type(scope): message

Body explaining why.

Fixes #123
EOF
)"
```

## Breaking change

```bash
git commit -m "$(cat <<'EOF'
type(scope)!: message

Body explaining the breaking change.

BREAKING CHANGE: description of what breaks and migration path.
EOF
)"
```

## Revert

```bash
git commit -m "$(cat <<'EOF'
revert: message describing what is reverted

Refs: <original-commit-SHA>
EOF
)"
```

---

## Completed Examples

Real-world examples showing how each template is filled in.

```
feat(auth): add OAuth2 login flow

Closes #42
```

```
fix(api): handle null response from payment endpoint

The payment gateway occasionally returns null on timeout.
Previously this caused an unhandled exception in the webhook handler.

Fixes #189
```

```
chore(deps): upgrade react to v19
```

```
fix: remove unused imports
```

```
feat(api)!: change response format to JSON:API

All endpoints now return a JSON:API envelope with `data`, `meta`,
and `errors` fields instead of raw objects.

BREAKING CHANGE: response shape changed from `{ users: [...] }` to
`{ data: [...], meta: { total: N } }`.

Fixes #301
```

```
revert: undo session timeout change

Refs: a1b2c3d
```
