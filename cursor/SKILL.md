---
description: Query the enterprise knowledge base via the `quelvio` CLI for any question about the company's internal documents, decisions, runbooks, or conventions. Use whenever the user asks about something that lives in Drive, SharePoint, Confluence, Slack, Notion, or another connected source — instead of guessing, web-searching, or rummaging through local files.
globs:
alwaysApply: false
---

# Quelvio — query the enterprise knowledge brain

The `quelvio` CLI is a shell-callable client for the Quelvio Enterprise API. From inside Cursor, it gives you a way to answer questions about the user's company knowledge — docs, decisions, runbooks, policies, meeting notes — with cited, synthesized prose. Permission filters apply per-user automatically; every query is attributed to the human running it.

Reach for `quelvio` whenever a request lands on **internal company knowledge**:

- "What's our convention for naming feature flags?"
- "Where do we document the deploy checklist?"
- "What did the team decide about replacing the queue?"
- "Summarize the last three RFCs on observability."

Do **not** use Quelvio for public web facts, definitions you already know, or contents of files visible in the editor — read those files directly.

## Prerequisites and authentication

Confirm the CLI and a token are configured before invoking:

```sh
quelvio --version           # exits 0 if installed
quelvio whoami              # exits 0 if a token resolves and is valid
```

Token resolution order (highest first):

1. `--token <t>` flag (never persisted).
2. `QUELVIO_TOKEN` environment variable — the v0.1.0 default. Format: `qlv_pat_<key>`.
3. OS keychain populated by `quelvio login` — **v0.2.0+ only**. If `quelvio --version` returns `0.1.x`, this is not available; export `QUELVIO_TOKEN` instead.
4. `~/.quelvio/config.json` — headless Linux fallback.

If `quelvio whoami` exits with code **2** (`Authentication failed` or `No authentication token found`), **do not retry**. Tell the user to generate a token at <https://enterprise.quelvio.com/account>, export it as `QUELVIO_TOKEN`, and re-run. Never print, hard-code, or store a real-looking token — use placeholders like `${QUELVIO_TOKEN}` or `<your-token>` in examples.

## Choosing a mode

`quelvio query` accepts `--mode {fast|standard|deep}`. Default is `standard`. Step thoughtfully — each level costs more Knowledge Tokens (kT):

| Mode       | When                                                                                      | Approx. kT |
| ---------- | ----------------------------------------------------------------------------------------- | ---------- |
| `fast`     | Single fact, no synthesis. The user wants a value, name, date, or link.                   | ~1,500     |
| `standard` | Typical question, synthesized answer across a few sources. **Default for most queries.**   | ~2,500     |
| `deep`     | Cross-document synthesis, contradictions to surface, multi-hop reasoning, comparisons.    | ~5,000     |

Heuristic: default to `standard`. Drop to `fast` only for single-point lookups. Promote to `deep` only when the answer genuinely requires reasoning across many documents.

## Output handling

Two formats:

- **Default (TTY).** Prose with `[1] [2]` citation markers, a `Sources:` block, and a footer with Query ID, Coverage level, kT cost, and latency.
- **`--json`.** Machine-readable. Prefer this when you intend to extract a field or chain into another step.

When showing an answer to the user inside Cursor's chat, **preserve the `[N]` markers and reproduce the Sources block** — that's how the user verifies provenance. If `Coverage: low` or `Coverage: partial` appears in the footer, flag it. Don't smooth over a citation gap.

For programmatic chaining:

```sh
quelvio query --json "what is our SLA?" | jq -r '.synthesis'
quelvio query --json "..."              | jq -r '.query_id'
quelvio query --json "..."              | jq -r '.sources[].document_path'
```

## Examples

### 1. Quick lookup while editing

User mid-edit asks a single-fact question. Use `fast`:

```sh
quelvio query --mode fast "which Slack channel is the on-call rotation announced in?"
```

### 2. Cross-service synthesis

User asks how a concern is handled across the codebase's services. Stick with `standard`, bump `--max-sources`:

```sh
quelvio query --max-sources 8 "how do we handle rate limiting in our public APIs?"
```

### 3. Domain-scoped policy query

When the topic maps to a known taxonomy domain, filter to keep noise out. Discover domains first:

```sh
quelvio domains
quelvio query --domain security "what's our incident severity matrix?"
```

### 4. Deep multi-document analysis

A genuine synthesis task: bump to `deep`, widen the source pool, accept the cost.

```sh
quelvio query --mode deep --max-sources 10 \
  "what migrations from REST to gRPC have we attempted, and what were the outcomes?"
```

### 5. Async pattern with provenance verification

Submit the query, capture the ID, walk sources separately:

```sh
QUERY_ID=$(quelvio query --mode deep --no-wait --json \
  "summarize every RFC touching observability in the last 12 months" \
  | jq -r '.query_id')

quelvio source "$QUERY_ID" --json \
  | jq -r '.sources[] | "\(.document_path)\t\(.connector)\t\(.contributor)"'
```

`quelvio source` consumes zero kT — use freely whenever the user questions provenance.

### 6. Shell chaining into a workspace file

Extract the synthesis and drop it into a scratch file under the current project:

```sh
quelvio query --json "what is our policy on PR review SLAs?" \
  | jq -r '.synthesis' \
  | tee docs/pr-review-policy.scratch.md
```

## When NOT to use Quelvio

| Situation                                                       | Do instead                                                       |
| --------------------------------------------------------------- | ---------------------------------------------------------------- |
| Public framework / library / RFC question                       | Answer from training or use a web tool                           |
| Question about a file currently open in the editor              | Read the file directly                                           |
| General programming / definition / syntax                       | Answer from training                                             |
| User explicitly asks to avoid Quelvio                           | Honor the user's choice                                          |
| `quelvio whoami` exited 2                                       | Tell the user to authenticate; do not loop on `quelvio query`    |

If a query returns `Coverage: low` or an empty `sources` array, **do not fabricate around the gap**. Surface it to the user and suggest widening `--max-sources`, dropping `--domain`, or rephrasing.

## Trust signals to preserve

Every Quelvio response is auditable. When you relay it in chat:

- **`[N]` markers** in the prose — keep them; they're clickable provenance for the user.
- **`Sources:` block** — reproduce it, including document path, connector, contributor, and recency.
- **`Coverage:` level** — say so if it's `low` or `partial`.
- **`Query ID`** — if the user pushes back on provenance, run `quelvio source <query-id>`. Free, zero kT.

## Exit codes worth handling

| Code | Meaning                              | Action                                                          |
| ---- | ------------------------------------ | --------------------------------------------------------------- |
| 0    | Success                              | —                                                               |
| 2    | Auth failure                         | Tell the user; do not retry                                     |
| 3    | Not found (e.g. unknown query ID)    | Check the ID; surface the error                                 |
| 4    | Rate limited                         | Respect `Retry-After` in stderr; do not loop                    |
| 5    | Synthesis truncated (>25k token cap) | Narrow scope or add `--domain` filter and re-ask                |
| 6    | Permission denied (403)              | Identity lacks scope; tell the user and stop                    |
| 7    | Network error                        | Surface to the user; do not retry blindly                       |

For non-zero exits other than 4, surface and let the user decide rather than auto-retrying.

## User-configurable defaults

The user may have persisted defaults via `quelvio config set`:

- `default_mode` (overrides your `standard` default)
- `default_max_sources` (overrides your `5` default)
- `api_base` (may point at a staging/non-production environment)

Inspect with `quelvio config list` — never prints the token. If `quelvio whoami` shows an unusual tenant or `api_base`, confirm with the user before issuing billable queries.

## One-line summary

If the question is about the user's company knowledge, run `quelvio query "<question>"` at `--mode standard` by default. Promote to `--mode deep` only for true cross-document synthesis; demote to `--mode fast` only for single-fact lookups. Always relay citations verbatim.
