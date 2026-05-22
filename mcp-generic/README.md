# Generic MCP agent — Quelvio skill

A runtime-agnostic Markdown skill suitable for any MCP-compatible agent — or any agent runtime that accepts a system-prompt addendum or instruction file. Use this when your agent is not Claude Code, Cursor, or OpenAI Codex CLI (each of which has its own variant in a sibling directory).

## How to use

The file is plain Markdown. Inject it however your runtime expects:

- **System-prompt addendum.** Concatenate it into your agent's system prompt at session start.
- **Instruction file.** Some runtimes read a designated `.md` file (e.g. `agent-instructions.md`, `INSTRUCTIONS.md`). Drop it there.
- **Tool description.** If your runtime accepts a per-tool description string for the `quelvio` shell tool, paste the body into the description field.

## Fetch

```sh
curl -fsSL https://raw.githubusercontent.com/Quelvio/quelvio-agent-skills/main/mcp-generic/SKILL.md \
  -o quelvio-skill.md
```

Or pin to a specific commit for reproducible deployments:

```sh
curl -fsSL https://raw.githubusercontent.com/Quelvio/quelvio-agent-skills/<commit-sha>/mcp-generic/SKILL.md \
  -o quelvio-skill.md
```

## Prerequisites

The skill assumes the Quelvio CLI is installed and a token is configured **in the same shell environment the agent invokes commands from**:

```sh
npm i -g @quelvio/cli
export QUELVIO_TOKEN=qlv_pat_<your-token>   # generate at https://enterprise.quelvio.com/account
quelvio whoami                              # should print your identity
```

For agents that spawn isolated shells, make sure `QUELVIO_TOKEN` is available in each spawned environment (shell rc, container env, or whatever your runtime uses for env injection).

If `quelvio --version` reports `0.1.x`, browser-based `quelvio login` is not yet available — use the env var.

## Frontmatter

The skill includes a YAML frontmatter snippet near the top. Runtimes that consume the body directly can drop the snippet. Runtimes that key on `name`/`description` can lift it into their expected location.

## Troubleshooting

| Symptom                                                | Fix                                                                       |
| ------------------------------------------------------ | ------------------------------------------------------------------------- |
| Agent never invokes `quelvio`                          | Confirm the skill content actually reached the runtime's prompt context.  |
| `command not found: quelvio` in agent's shell          | Ensure `@quelvio/cli` is installed in the agent's PATH.                   |
| Auth fails inside agent but works in your terminal     | The agent's shell isn't inheriting `QUELVIO_TOKEN`. Inject it explicitly. |
| Agent over-uses `quelvio` for local file work          | The skill forbids this — open an issue with the transcript.               |

## Updating

Re-fetch and re-inject. Most runtimes pick up the new prompt on session restart.
