# Claude Code — Quelvio skill

A drop-in skill that teaches Claude Code when to reach for `quelvio query` instead of guessing or searching the web. Once installed, Claude Code's runtime ingests the YAML frontmatter at session start; the body is loaded into context when the skill is triggered.

## Install (user-scoped, available in every project)

```sh
mkdir -p ~/.claude/skills/quelvio
curl -fsSL https://raw.githubusercontent.com/Quelvio/quelvio-agent-skills/main/claude-code/SKILL.md \
  -o ~/.claude/skills/quelvio/SKILL.md
```

## Install (project-scoped, lives in the repo)

```sh
mkdir -p .claude/skills/quelvio
curl -fsSL https://raw.githubusercontent.com/Quelvio/quelvio-agent-skills/main/claude-code/SKILL.md \
  -o .claude/skills/quelvio/SKILL.md
```

Commit `.claude/skills/quelvio/SKILL.md` so every teammate working in the repo gets it.

## Reload

Skills are picked up at session start. Restart your Claude Code session (or run `/clear`) after installing.

## Prerequisites

The skill assumes the Quelvio CLI is installed and a token is configured:

```sh
npm i -g @quelvio/cli
export QUELVIO_TOKEN=qlv_pat_<your-token>   # generate at https://enterprise.quelvio.com/account
quelvio whoami                              # should print your identity
```

If `quelvio --version` reports `0.1.x`, OAuth-based `quelvio login` is not yet available — use the env var.

## Troubleshooting

| Symptom                                          | Fix                                                                                          |
| ------------------------------------------------ | -------------------------------------------------------------------------------------------- |
| Claude doesn't seem to use `quelvio`             | Confirm the skill file is at `~/.claude/skills/quelvio/SKILL.md` and restart the session.    |
| Claude calls `quelvio` but auth fails            | `quelvio whoami` in your shell. Re-export `QUELVIO_TOKEN` if needed.                         |
| Claude over-uses `quelvio` for local file work   | Skill explicitly forbids this — file an issue with the transcript so we can sharpen the cue. |
| `command not found: quelvio`                     | Run `npm i -g @quelvio/cli` and confirm `which quelvio`.                                     |

## Updating

```sh
curl -fsSL https://raw.githubusercontent.com/Quelvio/quelvio-agent-skills/main/claude-code/SKILL.md \
  -o ~/.claude/skills/quelvio/SKILL.md
```

Restart your session to pick up changes.
