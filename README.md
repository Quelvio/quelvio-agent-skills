# quelvio-agent-skills

Skill files that teach terminal-resident AI agents — Claude Code, Cursor, OpenAI Codex CLI, and any MCP-compatible agent — when and how to reach for the [Quelvio CLI](https://github.com/Quelvio/quelvio-cli) instead of guessing, searching the web, or rummaging through local files.

## Why this repo exists

The Quelvio CLI (`@quelvio/cli`) puts every connected enterprise source — Drive, SharePoint, Confluence, Slack, Notion, and more — behind a single scriptable command that returns cited, per-user-permissioned answers. But a great tool is only useful if the agent next to the developer knows it exists. These skill files are the bridge: drop the right one into your agent's runtime, and the agent will reach for `quelvio query` the moment a question lands on internal knowledge instead of public facts or local code.

## What's inside

| Path                       | Target runtime                              | Format                                             |
| -------------------------- | ------------------------------------------- | -------------------------------------------------- |
| `claude-code/SKILL.md`     | Anthropic Claude Code (CLI, desktop, web)   | YAML-frontmatter skill                             |
| `cursor/SKILL.md`          | Cursor IDE                                  | Adaptable to `.cursor/rules/*.mdc` or `AGENTS.md`  |
| `codex/SKILL.md`           | OpenAI Codex CLI                            | `AGENTS.md`-compatible Markdown                    |
| `mcp-generic/SKILL.md`     | Any MCP-compatible agent runtime            | Portable Markdown / system prompt addendum         |

Each subdirectory contains its own short `README.md` with installation steps, where to drop the file, how to reload the agent, and per-runtime troubleshooting.

## Quick install

### Claude Code

```sh
mkdir -p ~/.claude/skills/quelvio
curl -fsSL https://raw.githubusercontent.com/Quelvio/quelvio-agent-skills/main/claude-code/SKILL.md \
  -o ~/.claude/skills/quelvio/SKILL.md
```

See [`claude-code/README.md`](./claude-code/README.md) for project-scoped install and reload notes.

### Cursor

```sh
mkdir -p .cursor/rules
curl -fsSL https://raw.githubusercontent.com/Quelvio/quelvio-agent-skills/main/cursor/SKILL.md \
  -o .cursor/rules/quelvio.mdc
```

See [`cursor/README.md`](./cursor/README.md) for frontmatter tweaks Cursor's runtime expects.

### OpenAI Codex CLI

```sh
curl -fsSL https://raw.githubusercontent.com/Quelvio/quelvio-agent-skills/main/codex/SKILL.md \
  -o AGENTS.md
```

See [`codex/README.md`](./codex/README.md) for project-vs-global placement.

### Generic MCP agent

Treat [`mcp-generic/SKILL.md`](./mcp-generic/SKILL.md) as a system prompt addendum, tool-description file, or instruction document — whichever your runtime supports. See the per-directory README.

## Prerequisites

All skill files assume the user has already installed the Quelvio CLI:

```sh
npm i -g @quelvio/cli      # v0.1.0 on npm as of writing
quelvio --version
export QUELVIO_TOKEN=qlv_pat_<your-token>
quelvio whoami
```

v0.1.0 of the CLI authenticates via the `QUELVIO_TOKEN` environment variable (a Personal Access Token from [enterprise.quelvio.com/account](https://enterprise.quelvio.com/account)). Browser-based `quelvio login` lands in v0.2.0.

## How these skills behave

Each `SKILL.md` teaches its host agent four things:

1. **When to reach for Quelvio.** Internal company knowledge — runbooks, decisions, meeting notes, code conventions, policies — instead of the public web or local files.
2. **Which `--mode` to choose.** `fast` for one-fact lookups, `standard` (default) for typical synthesis, `deep` for cross-document reasoning.
3. **How to compose with shell.** Pipe `--json` into `jq`, chain `--no-wait` queries with `quelvio source`, surface citation markers verbatim to the user.
4. **What to refuse.** Public facts, local file contents, general definitions, and any case where Quelvio returns a low-coverage answer the agent shouldn't fabricate around.

## Authoritative sources

- **Quelvio CLI repo:** <https://github.com/Quelvio/quelvio-cli>
- **CLI documentation:** <https://quelvio.com/docs/cli>
- **Enterprise dashboard / PAT generation:** <https://enterprise.quelvio.com/account>

If a skill file ever drifts from CLI behavior, the [CLI README](https://github.com/Quelvio/quelvio-cli/blob/main/README.md) is the source of truth — file an issue and we'll re-sync.

## Contributing

Pull requests welcome, especially for additional runtimes (GitHub Copilot CLI, Windsurf, Aider, Continue, Cline) once their tool-context conventions stabilize. Keep skill files substantive: real example queries, accurate flag names, no placeholder sections.

## License

MIT. See [LICENSE](./LICENSE).
