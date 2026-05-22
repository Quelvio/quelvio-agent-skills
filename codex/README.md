# OpenAI Codex CLI — Quelvio skill

Codex CLI reads `AGENTS.md` files for tool and behavior guidance. This skill is shipped as portable Markdown — drop it in as `AGENTS.md` (project-scoped) or as part of your global Codex instructions.

## Install (project-scoped, recommended)

Place at the repo root so every Codex session in this repo picks it up:

```sh
curl -fsSL https://raw.githubusercontent.com/Quelvio/quelvio-agent-skills/main/codex/SKILL.md \
  -o AGENTS.md
```

Commit `AGENTS.md` so teammates running Codex inside the repo benefit.

If your repo already has an `AGENTS.md` with other instructions, append the Quelvio skill rather than overwriting:

```sh
echo "" >> AGENTS.md
curl -fsSL https://raw.githubusercontent.com/Quelvio/quelvio-agent-skills/main/codex/SKILL.md \
  >> AGENTS.md
```

## Install (global, all projects)

Codex resolves `AGENTS.md` files up the directory tree. To make this skill available everywhere, drop it in your home directory or your Codex config directory (consult your Codex CLI's own docs for the exact path on your version — paths have shifted across releases):

```sh
mkdir -p ~/.codex
curl -fsSL https://raw.githubusercontent.com/Quelvio/quelvio-agent-skills/main/codex/SKILL.md \
  -o ~/.codex/AGENTS.md
```

> Codex CLI's instruction-file conventions are still evolving. If your Codex version uses a different filename or location (e.g. `~/.codex/instructions.md`), adapt the path — the Markdown body is portable and runtime-agnostic.

## Reload

Codex re-reads `AGENTS.md` at session start. Restart your Codex session after installing or updating the skill.

## Prerequisites

The skill assumes the Quelvio CLI is installed and a token is configured:

```sh
npm i -g @quelvio/cli
export QUELVIO_TOKEN=qlv_pat_<your-token>   # generate at https://enterprise.quelvio.com/account
quelvio whoami                              # should print your identity
```

Put `export QUELVIO_TOKEN=...` in your shell rc (`~/.zshrc`, `~/.bashrc`) so Codex's shell sub-processes inherit it.

If `quelvio --version` reports `0.1.x`, browser-based `quelvio login` is not yet available — use the env var.

## Troubleshooting

| Symptom                                          | Fix                                                                                       |
| ------------------------------------------------ | ----------------------------------------------------------------------------------------- |
| Codex doesn't invoke `quelvio`                   | Confirm `AGENTS.md` is at the repo root (or in your global Codex config path).            |
| `command not found: quelvio` in Codex shell      | `npm i -g @quelvio/cli`; verify `which quelvio` returns a path.                           |
| Auth fails inside Codex but works in plain shell | The Codex-spawned shell isn't inheriting `QUELVIO_TOKEN`. Add it to your shell rc.        |
| Codex over-uses `quelvio` for local file work    | The skill explicitly forbids this — file an issue with the transcript.                    |

## Updating

```sh
curl -fsSL https://raw.githubusercontent.com/Quelvio/quelvio-agent-skills/main/codex/SKILL.md \
  -o AGENTS.md
```

Restart your Codex session.
