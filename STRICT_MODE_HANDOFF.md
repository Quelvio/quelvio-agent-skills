# Strict-Mode Sentinel Handoff (FE-13)

> **Docs PR — informational only.** Unlike the other seven Quelvio SDK
> repos, this repo ships **no runtime HTTP code** — it's a collection of
> agent skill bundles (Claude Code, Codex, Cursor, generic MCP). There
> is therefore no place to "wire" a header detector here. This doc
> exists so the skill authors are aware of the new strict-mode UX and
> can mention it in skill copy where relevant.

## What backend PR #643 ships

Backend PR #643 emits two response headers globally on the search /
retrieval endpoints:

| Header | Value | Meaning |
| --- | --- | --- |
| `X-Quelvio-API-Version` | `2.0` | API contract version. Informational. |
| `X-Quelvio-Sentinel-Set` | `closed-v1` | Tenant is on the strict (closed) permission model. Some results may be filtered. |

When the sentinel header is present, the consuming SDK (`mcp-server`,
`cli`, `vercel-ai-sdk`, `langchain-*`, `llama-index`, `crewai`) will
emit a one-shot warning. Skill authors should be aware that under
strict mode, search may return fewer chunks than the user expects —
prompts that say "you have access to all company knowledge" should
soften to "you have access to the knowledge for which you have explicit
permissions" when running against a strict-mode tenant.

## Reference implementations

- TypeScript / Cloudflare Workers (canonical):
  [`quelvio-mcp-server` `src/sentinel.ts`](https://github.com/Quelvio/quelvio-mcp-server/blob/main/src/sentinel.ts)
- Python (httpx event hooks pattern):
  [`quelvio-langchain-python` handoff doc](https://github.com/Quelvio/quelvio-langchain-python/blob/main/STRICT_MODE_HANDOFF.md)

## Contract (for context — not implemented here)

Every SDK that touches the Quelvio API MUST:

1. Log once per process when `X-Quelvio-Sentinel-Set` is observed.
2. Use the structured event token `quelvio_sentinel_set_detected`.
3. Surface via the SDK's existing logger / stderr. Never throw.

## Owner

FE-13 / antonis@rolle.io. Backend counterpart: PR #643 on
`Quelvio/quelvio-platform`.
