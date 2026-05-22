# Quelvio — query the enterprise knowledge brain

`quelvio` is a shell-callable CLI for the Quelvio Enterprise API. From inside the OpenAI Codex CLI, it is the right tool whenever the user asks a question about the company's internal knowledge — runbooks, decisions, conventions, meeting notes, policies, RFCs, on-call docs — anything stored in the company's connected sources (Google Drive, SharePoint, Confluence, Slack, Notion, and more). It returns cited, synthesized answers in prose or JSON. Permission filters apply per-user automatically; every query is attributed to the human running the session.

**Reach for `quelvio` when the question lands on internal knowledge:**

- "What is our deploy pipeline?"
- "Where do we document the on-call rotation?"
- "What did we decide about migrating off Kafka?"
- "Summarize the Q1 architecture review."

**Do NOT use Quelvio for:**

- Public web facts → use a web tool or answer from training
- Files in the working directory → read them directly
- General programming / definitions / syntax → answer from training

## Prerequisites and authentication

Before invoking `quelvio`, confirm it's installed and a token resolves:

```sh
quelvio --version    # exits 0 if installed; if absent, run: npm i -g @quelvio/cli
quelvio whoami       # exits 0 if a token is configured and valid
```

Token resolution, highest precedence first:

1. `--token <t>` flag (never persisted).
2. `QUELVIO_TOKEN` environment variable — the v0.1.0 default. Format: `qlv_pat_<key>`.
3. OS keychain populated by `quelvio login` — **v0.2.0+ only**. If `quelvio --version` reports `0.1.x`, this path is unavailable; the user must export `QUELVIO_TOKEN`.
4. `~/.quelvio/config.json` — fallback for headless Linux without `libsecret`.

If `quelvio whoami` exits **2** (`Authentication failed` or `No authentication token found`), **stop**. Tell the user to generate a token at <https://enterprise.quelvio.com/account>, export it as `QUELVIO_TOKEN`, and re-run. Do not retry the call. Never print or hard-code real-looking tokens — use literal placeholders like `${QUELVIO_TOKEN}` or `<your-token>`.

## Choosing a mode

`quelvio query` accepts `--mode {fast|standard|deep}`. Default is `standard`. Choose deliberately — each step up consumes more Knowledge Tokens (kT):

| Mode       | When                                                                                      | ~kT     |
| ---------- | ----------------------------------------------------------------------------------------- | ------- |
| `fast`     | Single fact, no synthesis. User wants a value, name, date, or link.                       | ~1,500  |
| `standard` | Typical question synthesizing across a few sources. **Default for most queries.**          | ~2,500  |
| `deep`     | Cross-document synthesis, contradictions, multi-hop reasoning, comparisons.               | ~5,000  |

Default to `standard`. Step down to `fast` only when the question is a single-point lookup. Step up to `deep` only when the answer truly requires reasoning across many documents.

## Output handling

Two formats:

- **Default (TTY).** Prose answer with `[1] [2]` citation markers, a `Sources:` block, and a footer with Query ID, Coverage level, kT cost, and latency.
- **`--json`.** Machine-readable response. Prefer this when extracting fields or chaining into subsequent steps.

When you relay an answer to the user, **preserve `[N]` markers and the Sources block verbatim** — that's how the user audits provenance. If the footer reports `Coverage: low` or `Coverage: partial`, surface it. Do not smooth over a citation gap.

Programmatic extraction examples:

```sh
quelvio query --json "what is our SLA?" | jq -r '.synthesis'
quelvio query --json "..."              | jq -r '.query_id'
quelvio query --json "..."              | jq -r '.sources[].document_path'
```

## Examples

### 1. Simple one-shot lookup

```sh
quelvio query "what's the rollback procedure for production deploys?"
```

Return the prose with citations intact.

### 2. Multi-source synthesis across services

```sh
quelvio query --max-sources 8 "how do our backend services authenticate users?"
```

### 3. Domain-filtered policy question

When the topic maps to a known taxonomy domain, scope to it. List domains first:

```sh
quelvio domains
quelvio query --domain hr "what's our policy on remote work eligibility?"
```

### 4. Deep cross-document synthesis

True synthesis tasks — comparisons, contradictions, multi-doc summaries:

```sh
quelvio query --mode deep --max-sources 10 \
  "compare our incident response practices in Q3 vs Q4 — what changed?"
```

### 5. Async + provenance verification

For long-running deep queries, return immediately, then walk sources separately:

```sh
QUERY_ID=$(quelvio query --mode deep --no-wait --json \
  "summarize every RFC touching the data plane in 2026" \
  | jq -r '.query_id')

quelvio source "$QUERY_ID" --json \
  | jq -r '.sources[] | "\(.document_path)\t\(.connector)\t\(.contributor)"'
```

`quelvio source` consumes zero kT. Use freely whenever the user questions provenance.

### 6. Shell chaining for an artifact

```sh
quelvio query --json "what's our policy on PR-review SLAs?" \
  | jq -r '.synthesis' \
  > docs/pr-review-policy.scratch.md
```

## When NOT to use Quelvio

| Situation                                                       | Do instead                                                       |
| --------------------------------------------------------------- | ---------------------------------------------------------------- |
| Public framework / library / RFC question                       | Web tool or training                                             |
| Question about a file in the working directory                  | Read the file directly                                           |
| General programming / definition / syntax                       | Answer from training                                             |
| User explicitly asks to skip Quelvio                            | Honor the user's choice                                          |
| `quelvio whoami` exited 2                                       | Tell the user to authenticate; do not loop on `quelvio query`    |

If a query returns `Coverage: low` or an empty `sources` array, **do not fabricate around the gap**. Surface it and offer to widen `--max-sources`, drop `--domain`, or rephrase.

## Trust signals to preserve

When you relay a Quelvio answer to the user:

- **Keep `[N]` markers** in the prose — they're how the user clicks through to provenance.
- **Reproduce the `Sources:` block** — document path, connector, contributor, recency.
- **Surface `Coverage:` level** if it's `low` or `partial`.
- **Hold onto the `Query ID`** — if the user later asks "where did that come from?" run `quelvio source <query-id>` (zero kT cost).

## Exit codes to handle

| Code | Meaning                                | Action                                                          |
| ---- | -------------------------------------- | --------------------------------------------------------------- |
| 0    | Success                                | —                                                               |
| 2    | Auth failure                           | Tell the user; do not retry the same call                       |
| 3    | Not found                              | Check the resource ID; surface the error                        |
| 4    | Rate limited                           | Respect `Retry-After` from stderr; do not loop                  |
| 5    | Synthesis truncated (>25k token cap)   | Narrow scope or add `--domain`, re-ask                          |
| 6    | Permission denied (403)                | Identity lacks scope; tell the user and stop                    |
| 7    | Network error                          | Surface to the user; do not retry blindly                       |

For non-zero exits other than 4, surface to the user rather than auto-retrying.

## User-configurable defaults

The user may have set persisted defaults via `quelvio config set`:

- `default_mode` (overrides your `standard` default)
- `default_max_sources` (overrides your `5` default)
- `api_base` (may point at staging or another non-production environment)

Inspect with `quelvio config list` — token is never printed. If `quelvio whoami` shows an unfamiliar tenant or `api_base`, confirm with the user before issuing billable queries.

## One-line summary

If the question is about the company's own knowledge, run `quelvio query "<question>"` at `--mode standard`. Promote to `--mode deep` only for true cross-document synthesis. Demote to `--mode fast` only for single-fact lookups. Always relay citations verbatim.
