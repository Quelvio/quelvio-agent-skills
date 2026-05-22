# Cursor — Quelvio skill

A drop-in rule that teaches Cursor when to reach for `quelvio query` instead of guessing or web-searching. The file uses MDC (Markdown Cursor) frontmatter — Cursor's `.cursor/rules/*.mdc` convention.

## Install (project-scoped, recommended)

```sh
mkdir -p .cursor/rules
curl -fsSL https://raw.githubusercontent.com/Quelvio/quelvio-agent-skills/main/cursor/SKILL.md \
  -o .cursor/rules/quelvio.mdc
```

Commit `.cursor/rules/quelvio.mdc` so every teammate working in the repo benefits.

## Install (workspace-wide alternative)

If your team prefers `AGENTS.md` over `.cursor/rules/`, drop the file at the repo root:

```sh
curl -fsSL https://raw.githubusercontent.com/Quelvio/quelvio-agent-skills/main/cursor/SKILL.md \
  -o AGENTS.md
```

Cursor reads `AGENTS.md` (and falls back to `.cursorrules` for legacy configs). Frontmatter is ignored in `AGENTS.md` mode — the body still drives behavior.

## Frontmatter notes

The file ships with `alwaysApply: false` and an empty `globs:` field, meaning Cursor activates the rule based on its `description` when the agent decides a question is relevant. To force the rule to always load in this project, edit the frontmatter to `alwaysApply: true`. To scope to a file pattern, set `globs:` (e.g. `globs: docs/**/*.md`).

## Reload

Cursor reloads rules on file save. If a fresh rule isn't taking effect, restart the chat window or reload the workspace.

## Prerequisites

The skill assumes the Quelvio CLI is installed and a token is configured:

```sh
npm i -g @quelvio/cli
export QUELVIO_TOKEN=qlv_pat_<your-token>   # generate at https://enterprise.quelvio.com/account
quelvio whoami                              # should print your identity
```

Put `export QUELVIO_TOKEN=...` in your shell rc (`~/.zshrc`, `~/.bashrc`) so the integrated terminal Cursor spawns inherits it.

If `quelvio --version` reports `0.1.x`, browser-based `quelvio login` is not yet available — use the env var.

## Troubleshooting

| Symptom                                          | Fix                                                                                          |
| ------------------------------------------------ | -------------------------------------------------------------------------------------------- |
| Cursor doesn't invoke `quelvio`                  | Confirm `.cursor/rules/quelvio.mdc` exists. Try `alwaysApply: true` in the frontmatter.      |
| `command not found: quelvio` in integrated term  | `npm i -g @quelvio/cli`, restart Cursor so PATH is re-read.                                  |
| Auth fails inside Cursor but works in terminal   | The integrated terminal isn't inheriting `QUELVIO_TOKEN`. Add it to your shell rc.           |
| Cursor over-uses `quelvio` for editor-local work | The skill explicitly forbids this — file an issue with the transcript.                       |

## Updating

```sh
curl -fsSL https://raw.githubusercontent.com/Quelvio/quelvio-agent-skills/main/cursor/SKILL.md \
  -o .cursor/rules/quelvio.mdc
```

Reload the workspace to pick up changes.
