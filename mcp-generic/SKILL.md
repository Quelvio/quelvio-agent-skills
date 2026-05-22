# Quelvio CLI — generic agent skill

This is a portable skill document for any MCP-compatible agent runtime (or any runtime that lets you inject a system prompt addendum / tool-context file). It teaches the agent when and how to invoke the [Quelvio CLI](https://github.com/Quelvio/quelvio-cli) — a shell-callable client that returns cited, synthesized answers from the user's enterprise knowledge base.

If your runtime supports YAML frontmatter for skills, prepend:

```yaml
---
name: quelvio
description: Query the enterprise knowledge brain via the `quelvio` shell command for questions about the user's internal company knowledge.
---
```

Otherwise, paste the body below into whatever instruction or system-prompt slot your runtime exposes.

---

# Quelvio — query the enterprise knowledge brain

The `quelvio` CLI is a shell-callable client for the Quelvio Enterprise API. It searches every source the user's company has connected — Google Drive, SharePoint, Confluence, Slack, Notion, and more — and returns a synthesized, cited answer in prose or JSON. Permission filters apply per-user automatically; every query is attributed to the human running it.

Reach for `quelvio` whenever the user's request touches **internal company knowledge** — runbooks, decisions, conventions, meeting notes, policies, RFCs:

- "What is our deployment process?"
- "Where is the on-call rotation documented?"
- "What did we decide about migrating off MongoDB?"
- "Summarize the architecture review from last quarter."

Do **not** use Quelvio for public web facts, file contents in the user's working directory, or general programming knowledge.

## Prerequisites and authentication

Before invoking, confirm the CLI is installed and a token resolves:

```sh
quelvio --version    # exits 0 if installed; otherwise: npm i -g @quelvio/cli
quelvio whoami       # exits 0 if a token is configured and valid
```

Token resolution, highest precedence first:

1. `--token <t>` flag — overrides everything; never persisted.
2. `QUELVIO_TOKEN` environment variable — the v0.1.0 default. Format: `qlv_pat_<key>`.
3. OS keychain populated by `quelvio login` — **v0.2.0+ only**. If `quelvio --version` returns `0.1.x`, this is not available; the user must export `QUELVIO_TOKEN`.
4. `~/.quelvio/config.json` — fallback for headless Linux without `libsecret`.

If `quelvio whoami` exits **2** (`Authentication failed` or `No authentication token found`), **stop**. Tell the user to generate a token at <https://enterprise.quelvio.com/account>, export it as `QUELVIO_TOKEN`, and re-run. Do not retry the call. Never log, echo, or hard-code real-looking tokens — use literal placeholders like `${QUELVIO_TOKEN}` or `<your-token>`.

## Choosing a mode

`quelvio query` accepts `--mode {fast|standard|deep}`. Default is `standard`. Pick deliberately — each step up costs more Knowledge Tokens (kT):

| Mode       | When                                                                                       | ~kT     |
| ---------- | ------------------------------------------------------------------------------------------ | ------- |
| `fast`     | Single fact, no synthesis. The user wants a value, name, date, or link.                    | ~1,500  |
| `standard` | Typical question synthesized across a handful of sources. **Default for most queries.**     | ~2,500  |
| `deep`     | Cross-document synthesis, contradictions to surface, multi-hop reasoning, comparisons.     | ~5,000  |

Default to `standard`. Step down to `fast` only when the question is a single-point lookup. Step up to `deep` only when the answer genuinely needs reasoning across many documents.

## Output handling

Two formats:

- **Default (TTY).** Prose with `[1] [2]` citation markers, a `Sources:` block, and a footer showing Query ID, Coverage level, kT cost, and latency.
- **`--json`.** Machine-readable JSON. Prefer this for programmatic chaining or field extraction.

When relaying the answer, **preserve `[N]` markers and the Sources block verbatim** — that's how the user audits provenance. If the footer shows `Coverage: low` or `Coverage: partial`, surface it. Do not smooth over a citation gap.

Programmatic extraction:

```sh
quelvio query --json "what is our SLA?" | jq -r '.synthesis'
quelvio query --json "..."              | jq -r '.query_id'
quelvio query --json "..."              | jq -r '.sources[].document_path'
```

## Examples

### 1. Simple one-shot lookup

```sh
quelvio query "what is our rollback procedure for production deploys?"
```

Return the prose answer with `[N]` markers and Sources intact.

### 2. Multi-source synthesis

```sh
quelvio query --max-sources 8 "how do we handle authentication across our backend services?"
```

### 3. Domain-scoped policy question

When the topic maps to a known taxonomy domain, scope to it for cleaner results:

```sh
quelvio domains
quelvio query --domain customer-support "what's our refund policy for annual subscriptions?"
```

### 4. Deep cross-document synthesis

For genuine multi-doc reasoning — comparisons, contradictions, multi-hop synthesis:

```sh
quelvio query --mode deep --max-sources 10 \
  "compare our incident response practices in Q3 vs Q4 — what changed and why?"
```

### 5. Async + provenance verification chain

Capture the query ID immediately, then walk sources separately:

```sh
QUERY_ID=$(quelvio query --mode deep --no-wait --json \
  "summarize every RFC touching the data plane in 2026" \
  | jq -r '.query_id')

quelvio source "$QUERY_ID" --json \
  | jq -r '.sources[] | "\(.document_path)\t\(.connector)\t\(.contributor)"'
```

`quelvio source` consumes zero kT — use freely whenever the user questions provenance.

### 6. Shell chaining into a file

```sh
quelvio query --json "what is our PR-review SLA?" \
  | jq -r '.synthesis' \
  | tee /tmp/pr-review-sla.md
```

## When NOT to use Quelvio

| Situation                                                       | Do instead                                                       |
| --------------------------------------------------------------- | ---------------------------------------------------------------- |
| Public framework / library / RFC question                       | Web tool or training knowledge                                   |
| Question about a file in the working directory                  | Read the file directly                                           |
| General programming / definition / syntax                       | Answer from training                                             |
| User explicitly asks to skip Quelvio                            | Honor the user's choice                                          |
| `quelvio whoami` exited 2                                       | Tell the user to authenticate; do not loop on `quelvio query`    |

If a query returns `Coverage: low` or an empty `sources` array, **do not fabricate around the gap**. Surface it and offer to widen `--max-sources`, drop `--domain`, or rephrase.

## Trust signals to preserve

When relaying a Quelvio answer:

- **Keep `[N]` markers** in the prose — they're how the user clicks through to provenance.
- **Reproduce the `Sources:` block** — document path, connector, contributor, recency.
- **Surface `Coverage:` level** if it's `low` or `partial`.
- **Hold onto the `Query ID`** — if the user later asks "where did that come from?", run `quelvio source <query-id>` (zero kT cost).

## Exit codes worth handling

| Code | Meaning                                | Action                                                          |
| ---- | -------------------------------------- | --------------------------------------------------------------- |
| 0    | Success                                | —                                                               |
| 2    | Auth failure                           | Tell the user; do not retry                                     |
| 3    | Not found                              | Check the resource ID; surface the error                        |
| 4    | Rate limited                           | Respect `Retry-After` in stderr; do not loop                    |
| 5    | Synthesis truncated (>25k token cap)   | Narrow scope or add `--domain`; re-ask                          |
| 6    | Permission denied (403)                | Identity lacks scope; tell the user and stop                    |
| 7    | Network error                          | Surface to the user; do not retry blindly                       |

For non-zero exits other than 4, surface to the user rather than auto-retrying.

## User-configurable defaults

The user may have persisted defaults via `quelvio config set`:

- `default_mode` (overrides your `standard` default)
- `default_max_sources` (overrides your `5` default)
- `api_base` (may point at staging or another non-production environment)

Inspect with `quelvio config list` — token is never printed. If `quelvio whoami` shows an unfamiliar tenant or `api_base`, confirm with the user before issuing billable queries.

## Five commands at a glance

| Command                     | Purpose                                                              | Costs kT? |
| --------------------------- | -------------------------------------------------------------------- | --------- |
| `quelvio query <text>`      | Ask a natural-language question; returns synthesis + citations.       | Yes       |
| `quelvio domains`           | List taxonomy domains and their coverage levels.                     | No        |
| `quelvio source <query-id>` | Walk per-chunk provenance for a previous query.                      | No        |
| `quelvio whoami`            | Show signed-in identity, tenant, auth method, redacted token prefix.  | No        |
| `quelvio config <op>`       | List/get/set/unset persisted defaults in `~/.quelvio/config.json`.   | No        |

## One-line summary

If the question is about the user's company knowledge, run `quelvio query "<question>"` at `--mode standard`. Promote to `--mode deep` only for true cross-document synthesis. Demote to `--mode fast` only for single-fact lookups. Always relay citations verbatim.
