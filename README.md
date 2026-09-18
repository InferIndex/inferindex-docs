# InferIndex

InferIndex finds the cheapest provider for a given LLM, across 70+ tracked pricing sources — direct provider
APIs and aggregators/routers — with price history and promotion detection.

**Live API**: https://api.inferindex.dev

**Service status**: https://status.inferindex.dev (external uptime monitoring)

This repository holds the public documentation, examples and (eventually) a client SDK for that API. The
backend itself — collection, parsers, data model — lives in a separate private repository.

```bash
curl "https://api.inferindex.dev/cheapest?model=deepseek/deepseek-v3.2"
```

_Example response as of 2026-09-18 — prices, providers and statuses change._

```json
{
  "query": "deepseek/deepseek-v3.2",
  "model": { "id": "deepseek/deepseek-v3.2", "name": "DeepSeek: DeepSeek V3.2" },
  "other_matches": ["deepseek/deepseek-v3.2-exp", "deepseek/deepseek-v3.2-exp-thinking"],
  "sort": "blended",
  "cheapest": {
    "provider": "Nous Portal",
    "via": "nous",
    "tier": "standard",
    "input_per_1M": 0.2088,
    "output_per_1M": 0.3096,
    "cache_read_per_1M": 0.0216,
    "blended_per_1M": 0.234,
    "currency": "USD",
    "quantization": "unknown",
    "quantization_source": "unknown",
    "context_length": 163840,
    "promo": false,
    "also_via": [],
    "unverified": []
  },
  "offers": [
    { "provider": "Nous Portal", "via": "nous", "blended_per_1M": 0.234, "…": "…" },
    { "provider": "GMICloud", "via": "aggregator", "blended_per_1M": 0.234, "…": "…" }
  ],
  "hidden_tiers": { "flex": 1 },
  "filters_unknown": {},
  "strict": false
}
```

Prices are in USD per million tokens, converted at the day's ECB rate when the source publishes another
currency. `blended_per_1M` is `(3 × input + output) / 4`, a rough proxy for a typical chat workload.
Each offer appears once, at its cheapest source: `via` is `"direct"` or the aggregator it comes from, and
`also_via` lists other sources selling the same offer. `promo` is `false` unless a promotion was detected, in
which case `confidence` and `price_before_promo` are filled in.

## Status

**Early / actively developed.** The API is public and free to use, without authentication, but there is no
uptime guarantee, no versioned API contract yet, and endpoints or response shapes may change. If you build
something on it, expect to adjust when they do. Feedback and issues are welcome.

Service status: https://status.inferindex.dev (external uptime monitoring). It shows current and past
availability; it is not an uptime guarantee.

## Why

Provider pricing pages disagree, change without notice, and rarely show what resellers actually charge for
the same model. InferIndex collects prices directly from 70+ sources on their own schedule (5 minutes to 24
hours depending on the source), stores every change, and answers "who is cheapest for model X right now" from
that history — client requests never trigger a live call to a provider.

## Data freshness and limits

- **Cache**: successful responses carry `Cache-Control: public, max-age=300` (5 minutes) (except `/health/ready`, never cached). On the custom domain, responses are additionally held in Cloudflare's
  edge cache for the same 5 minutes.
- **Collection frequency**: most sources are polled hourly; a few (exchange rates, model catalog) once a day.
- **Rate limits**: 60 requests per minute per IP on `/cheapest`, `/history` and `/resellers`, counting only
  requests not answered from cache. An optional API key (`x-api-key` header) raises this to 300 requests per
  minute per key; free keys will be offered, self-service sign-up isn't available yet. An invalid key returns
  401. Over the limit, requests get `429`. The MCP server has its own limits, see
  [Use with AI assistants](#use-with-ai-assistants-mcp).
- **Cost estimate**: add `prompt_tokens`, `output_tokens`, `cached_ratio` and `requests_per_day` to `/cheapest`
  or `/resellers` to get an estimated cost per request and per month for each offer, and
  `sort=estimated_cost` to rank by it. See [docs/api.md](docs/api.md#cost-estimate).
- **Hidden tiers by default**: degraded service tiers (`flex`, `batch` — lower priority, variable latency, in
  exchange for a lower price) are excluded from `/cheapest` and `/resellers` unless you ask for them with
  `include_tiers=flex,batch`. The number hidden is reported in `hidden_tiers`.

## Reliability of the prices — read this before trusting a number

InferIndex is a comparison tool, not a source of truth. Treat every price as "what our collector last saw",
not as a guarantee of what you will be billed:

- Prices come from public APIs and, for some sources, HTML pages parsed with source-specific code — a page
  redesign can silently break a parser until it's caught.
- A price that looks unusually low may be a promotion, a misparsed unit, or a provider that hasn't updated its
  page yet. InferIndex applies guard rails (rejecting implausible prices, quarantining sudden jumps, flagging
  outliers against a second reference source) and reports detected promotions with a confidence level, but
  these are heuristics, not certainties.
- Reseller aggregators can add their own margin. When
  a source's pricing model isn't fully understood, it is either excluded or flagged.
- **Always confirm the price on the provider's own page or dashboard before committing to real spend.**

## Endpoints

| Route | What it does |
|---|---|
| `GET /cheapest?model=deepseek/deepseek-v3.2` | Cheapest offer across all active sources, and every matching offer deduplicated and sorted |
| `GET /resellers?model=deepseek/deepseek-v3.2` | Current prices across every reseller (direct and via aggregators), one line per source, paginated beyond 100 offers |
| `GET /history?model=deepseek/deepseek-v3.2&days=30&granularity=day` | Price history, paginated by cursor |
| `GET /models?search=deepseek` | Search tracked models |
| `GET /index?series=market` | Weekly market index, every Monday from 2026-10-05 — see the [methodology](docs/market-index.md) |
| `GET /health/live` | Liveness only: the service is up, no database access |
| `GET /health/ready` | Readiness: data freshness and scheduler health, see below |
| `/mcp` | MCP server for AI assistants, see [Use with AI assistants](#use-with-ai-assistants-mcp) |

### `/cheapest` filters

- `sort=blended\|input\|output` — sort order (default `blended`)
- `min_context=100000` — minimum context window
- `quantization=fp8,bf16` — accepted quantizations
- `min_uptime=95` — minimum 30-minute uptime (%), as observed by an aggregator (not measured by InferIndex)
- `tools=true`, `json=true`, `vision=true` — only offers declared to support tool calling, JSON output, image input
- `region=eu` — only offers processed in that region (`eu, us, cn, uk, ch, sg, id, my, vn, kr, jp, in, ca, au`), per the providers' official documents
- `no_training=true` — only offers whose providers state they don't train on prompts
- `no_waitlist=true` — only providers whose signup is open to everyone
- `strict=true` — also drop offers for which the filtered data is unknown (by default they're kept and flagged
  in `unverified`, counted in `filters_unknown`)
- `include_tiers=flex,batch` — include degraded service tiers (excluded by default, see above)
- `explain=true` — list every excluded offer with its reason codes (an `explanation` summary is always included)

Offers also carry `conditions` (data region, retention, training on prompts, and what it takes to open an
account — each with its official source link), `quantization_source` (whether the compute precision is declared
by the source or inferred), and, on `/resellers`, `reliability` from the providers' official status pages. See
[docs/api.md](docs/api.md).

### `/history`

`granularity=raw` returns one row per price change; `day` / `week` (weeks start Monday) return one point per
period and per offer (provider, quantization, variant) with min, max and last price. Use `days` or `from`/`to`
for a period, or `at=2026-09-15` for the prices in force at a given moment. Prices are converted to USD at the
ECB rate of the day of the price.

Model search (`/models`):

```bash
curl "https://api.inferindex.dev/models?search=deepseek"
```

_Example response as of 2026-09-15 — prices, providers and statuses change._

```json
{
  "count": 19,
  "models": [
    { "id": "deepseek/deepseek-v3.2", "name": "DeepSeek: DeepSeek V3.2" },
    { "id": "deepseek/deepseek-v4.1-flash", "name": "DeepSeek: DeepSeek V4.1 Flash" }
  ]
}
```

### `/health/ready`

_Example response as of 2026-09-15 — prices, providers and statuses change._

```json
{
  "ready": true,
  "status": "ok",
  "failures": [],
  "degraded": [],
  "warnings": ["2 price source(s) in error", "1 announcement source(s) in error"],
  "prices": { "current_offers": 5664, "latest_check_at": "2026-09-15T05:01:05.639Z", "latest_check_age_minutes": 1.5 },
  "watcher": { "active_sources": 84, "errors": 1, "persistent_errors": 0, "latest_check_at": "2026-09-15T05:02:08.890Z", "latest_check_age_minutes": 0.4 },
  "schedulers": { "active": 62, "ok": 62, "starting": 0, "late": 0, "not_started": 0 },
  "source_errors": { "count": 1, "persistent": 0 },
  "version": { "id": "…", "tag": "", "deployed_at": "…" },
  "checked_at": "2026-09-15T05:02:34.383Z"
}
```

`status` is `ok`, `degraded` (a non-critical part is having trouble — see the `degraded` array for which) or
`unavailable` (no usable price data). HTTP 200 for `ok`/`degraded`, 503 for `unavailable`. Useful as an
uptime-monitor target if you depend on this API.

Full route and parameter reference: [docs/api.md](docs/api.md).

## Use with AI assistants (MCP)

InferIndex is also available as a remote [MCP](https://modelcontextprotocol.io) server, so an AI assistant can
look up prices for you:

- **Endpoint**: `https://mcp.inferindex.dev/mcp` (Streamable HTTP)
  (`https://api.inferindex.dev/mcp` also works, as an alias)
- Listed in the official MCP Registry as `io.github.InferIndex/inferindex`.
- **Read-only, no authentication.** An API key can be passed in the `x-api-key` header.
- **Limits**: 120 requests per minute per IP on `/mcp`, at most 10 JSON-RPC messages per batch and 64 KB per
  request (otherwise `400` or `413` with JSON-RPC error `-32600`). Over the rate limit: `429` with JSON-RPC error
  `-32000`. Each tool call also counts toward the limit of the API route it uses (`/cheapest`, `/resellers`,
  `/history`).

| Tool | What it does |
|---|---|
| `search_models(query)` | Find the exact id of a model |
| `cheapest(model, …)` | Cheapest offers, with the same filters as `/cheapest` (`min_context`, `tools`, `json`, `vision`, `region`, `no_training`, `no_waitlist`, `strict`, `include_tiers`), optional usage (`prompt_tokens`, `output_tokens`, `cached_ratio`, `requests_per_day`) and `limit` |
| `compare_providers(model, sort, limit, …)` | Every offer, one line per provider, with usage conditions and reliability |
| `price_history(model, days \| from + to \| at, granularity, provider, limit)` | Offer price history, plus the lab's official prices |
| `estimate_cost(model, prompt_tokens, output_tokens, cached_ratio, requests_per_day)` | Estimated cost per request and per month, sorted |

Every result includes `api_url`, the equivalent API call, so you can check or reuse it.

### Setup

For Cline and other agents that install servers themselves, see [llms-install.md](llms-install.md).

**Claude Code**

```bash
claude mcp add --transport http inferindex https://mcp.inferindex.dev/mcp
```

With an API key, add `--header "x-api-key: YOUR_KEY"`.

**Claude Desktop / claude.ai**: Settings → Connectors → Add custom connector, URL `https://mcp.inferindex.dev/mcp`.
Older Claude Desktop versions, in `claude_desktop_config.json`:

```json
{ "mcpServers": { "inferindex": { "command": "npx", "args": ["-y", "mcp-remote", "https://mcp.inferindex.dev/mcp"] } } }
```

**Cursor**: one click, or add it by hand.

[![Add to Cursor](https://cursor.com/deeplink/mcp-install-dark.svg)](https://cursor.com/en/install-mcp?name=inferindex&config=eyJ1cmwiOiJodHRwczovL21jcC5pbmZlcmluZGV4LmRldi9tY3AifQ%3D%3D)

In `~/.cursor/mcp.json`:

```json
{ "mcpServers": { "inferindex": { "url": "https://mcp.inferindex.dev/mcp" } } }
```

With an API key, add `"headers": { "x-api-key": "YOUR_KEY" }` next to `"url"`.

**Codex CLI**

```bash
codex mcp add inferindex --url https://mcp.inferindex.dev/mcp
```

Or in `~/.codex/config.toml`:

```toml
[mcp_servers.inferindex]
url = "https://mcp.inferindex.dev/mcp"
```

With an API key, add `env_http_headers = { "x-api-key" = "INFERINDEX_API_KEY" }` to that table and set the
`INFERINDEX_API_KEY` environment variable.

## Contributing

Documentation fixes and example contributions are welcome — open a pull request or an issue. This repository
covers the public API surface only; the backend itself is not open source.

## License

- **Documentation** (this README, `docs/`): [CC BY 4.0](LICENSE-docs). Reuse and adapt it freely, with credit
  to InferIndex.
- **Code** (examples, snippets, client SDK): [MIT](LICENSE).

These licenses cover this repository only, not the InferIndex backend or its price data.
