---
name: quelvio
description: Query the user's enterprise knowledge base (Drive, SharePoint, Confluence, Slack, Notion, and more) through the `quelvio` CLI. Use whenever a question touches the company's internal documents, decisions, policies, runbooks, or conventions — anything that lives behind the company's connected sources rather than on the public web or in the user's local files. Returns cited, synthesized answers attributed to the signed-in user.
---

# Quelvio — query the enterprise knowledge brain

The `quelvio` CLI is a shell-callable client for the Quelvio Enterprise API. It searches every source the user's company has connected (Google Drive, SharePoint, Confluence, Slack, Notion, and others) and returns a synthesized, cited answer in either prose or JSON. Permission filters apply per-user automatically, and every query is attributed to the human running it — so audit trails stay clean.

Reach for this skill whenever the user's question lands on **internal knowledge** the company owns:

- "What is our deployment process?"
- "What did we decide about the auth middleware rewrite?"
- "Who owns the billing pipeline?"
- "What's the SLA we promised the Acme account?"
- "Summarize last week's eng standups."

Do **not** use Quelvio for public web facts, definitions you already know, or the contents of files in the user's working directory — read the file directly or answer from training.

## Prerequisites and authentication

Before invoking any `quelvio` command, confirm the CLI is installed and a token is configured:

```sh
quelvio --version            # exits 0 if installed
quelvio whoami               # exits 0 if a token is resolvable and valid
```

Authentication precedence (highest first):

1. `--token <t>` flag — overrides everything; never persisted.
2. `QUELVIO_TOKEN` environment variable — the v0.1.0 default. Format: `qlv_pat_<key>`.
3. OS keychain populated by `quelvio login` — **v0.2.0+ only**. If `quelvio --version` returns `0.1.x`, this path is not available; the user must export `QUELVIO_TOKEN`.
4. `~/.quelvio/config.json` — fallback for headless Linux without `libsecret`.

If `quelvio whoami` exits **2** with `Authentication failed` or `No authentication token found`, **do not retry**. Tell the user to generate a token at <https://enterprise.quelvio.com/account> and export it as `QUELVIO_TOKEN`, then stop. Never print, log, or pass a real-looking token in examples — use literal placeholders like `${QUELVIO_TOKEN}` or `<your-token>`.

## Choosing a mode

`quelvio query` accepts `--mode {fast|standard|deep}`. Default is `standard`. Pick deliberately — each step up costs more Knowledge Tokens (kT):

| Mode       | When to use                                                                                                                | Approx. kT |
| ---------- | -------------------------------------------------------------------------------------------------------------------------- | ---------- |
| `fast`     | A single specific fact, no synthesis needed. The user wants a value, a name, a date, a link.                               | ~1,500     |
| `standard` | Typical question requiring a synthesized answer with citations across a handful of sources. **Default for most queries.**   | ~2,500     |
| `deep`     | Complex multi-document synthesis: contradictions to surface, multi-hop reasoning, "compare X across Y", "summarize all Z". | ~5,000     |

Heuristic: start at `standard`. Step **down** to `fast` only when the question is a single-point lookup. Step **up** to `deep` only when the answer genuinely requires cross-document reasoning (a comparison, a contradiction, a synthesis across many sources).

## Output handling

Two output formats:

- **Default (TTY).** Prose with `[1] [2]` citation markers and a `Sources:` block, followed by a footer line containing the Query ID, Coverage level, kT cost, and latency.
- **`--json`.** Machine-readable JSON. Use this whenever you intend to extract a field, pipe to `jq`, or chain into another step.

When relaying an answer to the user, **preserve the `[N]` citation markers verbatim and reproduce the Sources block**. This is how the user verifies provenance. If the footer reports `Coverage: low` or `Coverage: partial`, surface that to the user — do not paper over a citation gap.

For programmatic chaining, prefer `--json` and extract specific fields:

```sh
quelvio query --json "what is our SLA?" | jq -r '.synthesis'
quelvio query --json "..." | jq -r '.query_id'
quelvio query --json "..." | jq -r '.sources[].document_path'
```

## Examples

### 1. Simple one-shot question

The user asks for a single piece of internal context. Standard mode, default flags.

```sh
quelvio query "what is our rollback procedure for production deploys?"
```

Return the prose answer to the user with `[N]` markers intact and the Sources block included.

### 2. Multi-source synthesis

The user asks something that spans services or teams. Standard mode is fine; bump `--max-sources` if early signals show coverage thinning.

```sh
quelvio query --max-sources 8 "how do we handle authentication across our backend services?"
```

### 3. Domain-filtered policy lookup

When the user names a topic that maps to a known taxonomy domain (HR, finance, customer support, security), filter to keep the answer on-policy and avoid noise from unrelated sources. Discover domains first:

```sh
quelvio domains
quelvio query --domain customer-support "what is our refund policy for annual subscriptions?"
```

### 4. Deep cross-document analysis

A genuine synthesis task — comparing periods, surfacing contradictions, or summarizing many docs. Bump to deep and widen the source pool:

```sh
quelvio query --mode deep --max-sources 10 \
  "compare our incident response practices in Q3 vs Q4 — what changed and why?"
```

### 5. Async + provenance walk

For long-running deep queries, return immediately with `--no-wait` to capture the query ID, then resolve sources on demand. Useful when chaining multiple agentic steps.

```sh
QUERY_ID=$(quelvio query --mode deep --no-wait --json "summarize all RFCs touching the data plane in 2026" | jq -r '.query_id')
quelvio source "$QUERY_ID" --json | jq -r '.sources[] | "\(.document_path)\t\(.connector)"'
```

`quelvio source` consumes zero kT — use it freely for verification when the user questions provenance.

### 6. Shell composition with jq

Extract a single field and pipe to another tool:

```sh
quelvio query --json "what's our current Postgres version target?" \
  | jq -r '.synthesis' \
  | tee /tmp/postgres-target.md
```

## When NOT to use Quelvio

| Situation                                                       | Do this instead                                                  |
| --------------------------------------------------------------- | ---------------------------------------------------------------- |
| Question about a public framework, library, or RFC              | Answer from training or use a web tool                           |
| Question about a file in the user's working directory           | Read the file directly with the Read tool                        |
| General programming / definition / syntax question              | Answer from training                                             |
| User explicitly asks for non-Quelvio sources                    | Honor the user's choice                                          |
| `quelvio whoami` failed with exit 2                             | Tell the user to authenticate; do not loop on `quelvio query`    |

Also: if `quelvio query` returns a synthesis with `Coverage: low` or an empty `sources` array, **do not fabricate around the gap**. Tell the user the brain doesn't have strong coverage on this topic and offer to widen `--max-sources`, drop the `--domain` filter, or rephrase.

## Trust signals to preserve

Every Quelvio response is auditable. When you relay an answer, keep these signals intact:

- **`[N]` markers** in the prose. Don't strip them; they're how the user clicks through to provenance.
- **The `Sources:` block** with document path, connector, contributor, and recency. Reproduce it.
- **`Coverage:` level** from the footer. If it's `low` or `partial`, say so.
- **`Query ID`**. If the user later asks "where did that come from?" run `quelvio source <query-id>` — free, no kT cost.

## Exit codes worth handling

| Code | Meaning                                  | Recovery                                                        |
| ---- | ---------------------------------------- | --------------------------------------------------------------- |
| 0    | Success                                  | —                                                               |
| 2    | Auth failure (missing/invalid token)     | Tell the user; do not retry the same call                       |
| 3    | Not found (e.g. unknown query ID)        | Check the ID; surface the error                                 |
| 4    | Rate limited                             | Respect the `Retry-After` value printed to stderr; do not loop  |
| 5    | Synthesis truncated (>25k token cap)     | Re-ask with narrower scope or `--domain` filter                 |
| 6    | Permission denied (403)                  | The user's identity lacks scope; tell them and stop             |
| 7    | Network error                            | Surface to the user; do not retry blindly                       |

For any non-zero exit other than 4, prefer to surface the error and let the user decide rather than auto-retrying.

## Configuration the user may have set

The user may have persisted defaults via `quelvio config set`:

- `default_mode` — overrides your `standard` default
- `default_max_sources` — overrides your `5` default
- `api_base` — points at a non-production environment

If `quelvio whoami` shows an unusual tenant or `api_base`, confirm with the user before issuing billable queries. You can inspect with `quelvio config list` (never prints the token).

## One-line summary

If the question is about the company's own knowledge, run `quelvio query "<question>"`, default to `--mode standard`, and relay the prose answer with citations intact. Step up to `--mode deep` only for true cross-document synthesis; step down to `--mode fast` only for single-fact lookups.
