# API reference

All routes are `GET`, work without authentication, and are served from `https://api.inferindex.dev`. Responses are JSON.
Successful responses carry `Cache-Control: public, max-age=300` (5 minutes) unless noted otherwise; do not
poll faster than that, it won't return fresher data. See the [README](../README.md) for rate limits, cache
behavior and a warning about price reliability before you go further.

This page documents the public read routes only. It is generated from the backend's route table and checked
against the production API on 2026-09-15.

## Authentication and rate limits

No key is needed. Requests without a key are limited to **60 requests per minute per IP** on `/cheapest`,
`/history` and `/resellers` (requests answered from cache don't count). Over the limit, the API returns **429**.
Other routes have no limit today, which may change. The MCP server's limits are in the
[README](../README.md#use-with-ai-assistants-mcp).

You can optionally send an API key in the `x-api-key` header to get **300 requests per minute per key** on
those routes:

```bash
curl -H "x-api-key: YOUR_KEY" "https://api.inferindex.dev/resellers?model=deepseek/deepseek-v3.2"
```

An unknown or revoked key returns **401** — the request is not silently served as anonymous, so remove the
header rather than sending an invalid key. Free API keys will be offered; self-service sign-up isn't available
yet.

## Model identifiers

Use an explicit model id, as listed by `/models` (for example `deepseek/deepseek-v4-pro-0813`). `/models` and
`other_matches` only list real models: no "latest" aliases and no service-level variants.

- **Explicit ids** (`org/model`) must exist: an unknown one returns **404** (`No model with id '…'`) with
  `suggestions`, never a different model.
- **Short or partial names** (`deepseek-v3`, `gpt`) are resolved to the closest match. When the name doesn't point
  to exactly one model, the response adds `"ambiguous": true`, and `other_matches` lists the other candidates — on
  `/cheapest`, `/resellers` and `/history`. Check `model.id` in the response, or send an explicit id.

- **"Latest" aliases** (ids ending in `-latest`) point to whichever version is newest, so they aren't accepted by
  `/cheapest`, `/resellers` or `/history`. They return **404** with a `suggestions` array of explicit ids to use
  instead. Response shape (the alias you sent appears in place of `<alias>`):

  ```json
  {
    "error": "'<alias>' is an alias that points to whichever version is latest, not a model: request an explicit model id",
    "suggestions": ["deepseek/deepseek-v4-pro-0813", "deepseek/deepseek-v4-pro"]
  }
  ```

- **Service-level suffixes** `:batch`, `:flex` and `:priority` are accepted: they return the base model with that
  tier included, like `include_tiers=`. The recommended form is still `model=<id>&include_tiers=batch`.
- **Any other `:suffix`** returns **404**, with the base model in `suggestions`.

## Errors

Errors return JSON with an English `error` message, for example:

```json
{ "error": "Missing 'model' parameter, e.g. /cheapest?model=deepseek/deepseek-v3.2" }
```

| Status | When | Example message |
|---|---|---|
| 400 | Missing or invalid parameter | `sort must be blended, input, output or estimated_cost`, `from must be before to`, `granularity must be raw, day or week`, `region must be one of: eu, us, …` |
| 401 | Unknown or revoked `x-api-key` | `Invalid or revoked API key` |
| 404 | Model not found, no price for the model, alias or unsupported suffix, or unknown route | `No model found for '…'`, `No price recorded for … (model not tracked?)`, `'…' is not a model id: use '…'` (with `suggestions`), `Unknown route` |
| 405 | Method other than `GET` | `Method not allowed` |
| 429 | Rate limit exceeded | `Too many requests, retry in a minute` |
| 503 | Service not ready yet | `Model catalog not collected yet, retry in a few minutes` |

Rely on the HTTP status and on field names; message wording may still be refined.

## `GET /`

Returns the API name, `version` (currently `"0.6.0"`), a short English description of each public endpoint,
`filters`: the accepted parameters of `/cheapest`, `/resellers` and `/history`, and `signals`: a short official
description of each offer signal (`backend_unknown`, `aggregated_price`, `points_based`, `below_official_list`).

## Offer fields

`/cheapest` and `/resellers` return offers with the same core fields:

| Field | Meaning |
|---|---|
| `input_per_1M`, `output_per_1M`, `cache_read_per_1M`, `blended_per_1M` | Prices in USD per million tokens. `blended` = (3 × input + output) / 4. Offers are always compared and sorted on these USD values |
| `currency` | Always `"USD"` for the fields above |
| `original_currency`, `original_input_per_1M`, `original_output_per_1M`, `original_cache_read_per_1M` | Price as published by the source, in its own currency (e.g. `EUR`, `CHF`, `CNY`), converted to USD at the day's ECB rate. An offer whose currency has no known rate is left out |
| `variant` | Service variant published by the source (e.g. `flex`, `fast`, a region), `null` for the default offer |
| `tier` | `standard`, `flex`, `batch` or `priority`, derived from `variant`. `flex` and `batch` (lower priority, variable latency, lower price) are hidden by default: add `include_tiers=flex`, `include_tiers=flex,batch` or `include_tiers=all`. `hidden_tiers` counts what was hidden |
| `tiered` | `true` when the price depends on request size (pricing brackets); the price shown is the **first bracket**, so large requests may cost more. The cost estimate picks the right bracket, see [Cost estimate](#cost-estimate) |
| `quantization` | `fp4`, `int4`, `int8`, `fp8`, `fp16`, `bf16` or `unknown` |
| `quantization_source` | Where that precision comes from: `declared` (published by the source, either in a dedicated field or in the offer's identifier such as `qwen3-max-fp8` — either way we only read it), `inferred` (worked out by InferIndex, never presented as declared), `not_applicable` (closed-weight model, where compute precision isn't a characteristic of the offer — not missing data), `unknown` (neither published nor derivable). With `strict=true`, `inferred` offers are dropped (reason code `quantization_inferred_strict`); `not_applicable` and `unknown` are not dropped on this ground. An offer with an inferred precision is also flagged in its `unverified` array and counted in `filters_unknown.quantization` |
| `context_length`, `max_output_tokens` | Context window and maximum output, in tokens, when published |
| `supports_tools`, `supports_json`, `supports_vision` | Tool calling, JSON output (JSON mode, `response_format`, structured outputs) and image input, **as declared by the source**: `true`/`false` only when the source says so explicitly, `null` otherwise |
| `uptime_30m`, `uptime_provenance` | (`/cheapest` only) 30-minute availability in percent, **as observed by an aggregator** — never an independent measurement by InferIndex. `uptime_provenance` is `"aggregator"` when `uptime_30m` is set, `null` otherwise |
| `conditions` | Usage conditions from the provider's official documents (data region, retention, training on prompts…), see [Usage conditions](#usage-conditions) |
| `checked_at` | Last time the offer's price was confirmed at its source. For a price that doesn't change, it is refreshed at most every 2 hours, so it can lag by up to 2 hours plus the source's collection interval |
| `stale` | `true` when the offer hasn't been re-checked for more than 3 × its source's collection interval (6 hours minimum): its price may be out of date. Stale offers are hidden from `/cheapest` (unless `include_stale=true`) and listed last in `/resellers` |
| `via_name` | Readable name of the aggregator the offer goes through, when it is disclosed (e.g. `"Eden AI"` for `via: "edenai"`); `null` for direct offers and for aggregators that aren't identified by name |
| `peak_pricing` | For providers that charge more at peak hours: `{ input_per_1M, output_per_1M, hours }`, the peak rate in USD and the provider's description of peak hours (`hours` may be `null`). The main prices are the off-peak rate. `null` otherwise |
| `official_list_price` | The lab's own list price for this model — `{ input_per_1M, output_per_1M, source_url, checked_at }` in USD — or `null` when the lab doesn't publish one. An offer that doesn't specify a region is compared with the lab's cheapest region; an offer for a given region (e.g. variant `eu` or `global`) with that same region |
| `below_official_list` | `true` when an offer resold by someone other than the lab (including a closed model sold under the reseller's own name) is clearly below what the lab itself charges, with no discount declared. It's a signal to double-check the offer, not a promotion: the offer is still listed and can still be bought |
| `served_model`, `requested_model`, `requested_models`, `redirect_note` | Only on offers where the lab has announced that a model id is now served by another model. The offer is listed under the model actually served (`served_model`); `requested_model` is the id to use with that provider (`requested_models` lists every id that leads to this same offer), and `redirect_note` explains the change. Such an offer is compared with the served model, including its list price |
| `aggregated_price` | `true` when a gateway publishes a single price for several backends it doesn't name (e.g. "cheapest available" or a default backend). The offer is still listed, but is never picked as the cheapest in `/cheapest` (reason code `aggregated_price`). Always `false` as of 2026-09-18 |
| `backend_unknown` | `true` when a router publishes a price under its own name without saying which provider serves the model. A signal only: the offer is compared normally |
| `points_based` | `true` when the source publishes its price in points or credits, converted to USD: what you actually pay depends on how you buy those points. The offer is still listed in `/resellers`, but is never picked as the cheapest (reason code `points_based`) |

`inferred` stays rare in responses by design: when the same provider is seen through several sources and one of
them publishes the precision, the published value wins.

**Gateways.** When a gateway routes to several providers, each backend it names is a separate offer: `provider`
is the provider that serves the model and `via` the gateway (for example `"provider": "DeepInfra", "via": "vercel"`).
When the gateway doesn't name the backend, `provider` is the gateway itself and `backend_unknown` is `true`.

## `GET /cheapest`

Cheapest offer for a tracked model across every active source (direct providers and aggregators), and every
matching offer sorted.

Offers are **deduplicated**: the same offer (provider + variant + quantization) seen through several sources
appears once, at its lowest price for the chosen `sort`; at equal price, the direct offer wins. `via` tells you
where that price comes from and `also_via` lists the other sources selling the same offer. `context_length`,
`uptime_30m`, `supports_tools`, `supports_json` and `supports_vision` are filled in from those other sources when the winning one doesn't publish
them. For one line per source, without deduplication, use `/resellers`.

| Parameter | Required | Meaning |
|---|---|---|
| `model` | yes | Model id, e.g. `deepseek/deepseek-v3.2`. A partial name also works but may be `ambiguous`, see [Model identifiers](#model-identifiers) |
| `sort` | no | `blended` (default), `input`, `output` or `estimated_cost` (needs usage parameters) |
| `min_context` | no | Minimum context window, in tokens |
| `min_uptime` | no | Minimum 30-minute uptime, percent (aggregator-observed, see `uptime_provenance`) |
| `tools` | no | `true` to keep only offers that support tool calling |
| `json` | no | `true` to keep only offers that support JSON output |
| `vision` | no | `true` to keep only offers that accept image input |
| `region` | no | Keep only offers whose requests are processed in that region: `eu, us, cn, uk, ch, sg, id, my, vn, kr, jp, in, ca, au` — provider **and** aggregator must both qualify |
| `no_training` | no | `true` to keep only offers whose provider **and** aggregator state they don't train on prompts |
| `no_waitlist` | no | `true` to keep only offers from providers whose signup is open to everyone, see [Access conditions](#access-conditions) |
| `strict` | no | `true` to also drop offers whose source doesn't publish the data a filter needs (see below) |
| `quantization` | no | Comma-separated accepted quantizations, e.g. `fp8,bf16` |
| `include_tiers` | no | Comma-separated degraded tiers to include (`flex`, `batch`), or `all`. Hidden by default — see README |
| `include_stale` | no | `true` to include stale offers (see `stale` in Offer fields), hidden by default |
| `prompt_tokens` | no | Tokens sent per request (integer, 0–10,000,000). Enables the cost estimate, see below |
| `output_tokens` | no | Tokens generated per request (integer, 0–10,000,000) |
| `cached_ratio` | no | Share of `prompt_tokens` read from cache, between 0 and 1 (default 0) |
| `requests_per_day` | no | Requests per day, to get a monthly estimate |
| `explain` | no | `true` to list every excluded offer with its reasons, see [Explained response](#explained-response) |

```bash
curl "https://api.inferindex.dev/cheapest?model=deepseek/deepseek-v3.2"
```

_Example response as of 2026-09-15 — prices, providers and statuses change._

```json
{
  "query": "deepseek/deepseek-v3.2",
  "model": { "id": "deepseek/deepseek-v3.2", "name": "DeepSeek: DeepSeek V3.2" },
  "other_matches": ["deepseek/deepseek-v3.2-exp", "deepseek/deepseek-v3.2-exp-thinking"],
  "sort": "blended",
  "cheapest": { "provider": "Nous Portal", "via": "nous", "blended_per_1M": 0.234, "…": "same shape as offers" },
  "offers": [
    {
      "provider": "AtlasCloud",
      "via": "direct",
      "variant": null,
      "tier": "standard",
      "input_per_1M": 0.26,
      "output_per_1M": 0.38,
      "cache_read_per_1M": 0.13,
      "blended_per_1M": 0.29,
      "currency": "USD",
      "original_currency": "USD",
      "original_input_per_1M": 0.26,
      "original_output_per_1M": 0.38,
      "original_cache_read_per_1M": 0.13,
      "quantization": "fp8",
      "quantization_source": "declared",
      "context_length": 163840,
      "max_output_tokens": 147456,
      "uptime_30m": 99.9,
      "uptime_provenance": "aggregator",
      "supports_tools": true,
      "supports_json": false,
      "supports_vision": false,
      "conditions": { "provider": { "…": "see Usage conditions" }, "via": null },
      "promo": false,
      "promo_source": null,
      "confidence": null,
      "promo_since": null,
      "promo_ends_at": null,
      "promo_expired": false,
      "price_before_promo": null,
      "price_since": "2026-09-14T18:02:11.965Z",
      "checked_at": "2026-09-15T04:40:42.431Z",
      "also_via": ["aggregator"],
      "unverified": [],
      "stale": false
    }
  ],
  "hidden_tiers": { "flex": 1 },
  "filters_unknown": {},
  "strict": false,
  "stale_hidden": 0,
  "explanation": { "considered": 54, "eligible": 42, "excluded": { "tier_hidden": 1, "duplicate": 11 }, "reason": "cheapest eligible offer by blended_per_1M" },
  "fetched_at": "2026-09-15T05:25:27.766Z"
}
```

- `via`: `"direct"` when the provider publishes the price itself, `"aggregator"` for an aggregator that isn't
  identified by name, otherwise the id of the aggregator/router. `also_via` uses the same values.
- **Filters on data that may be unknown** (`min_context`, `min_uptime`, `tools`, `json`, `vision`, `region`,
  `no_training`): by default an offer for which the data isn't known is kept, with the filter name in its
  `unverified` array (e.g. `["min_uptime"]`). `filters_unknown` counts those offers per filter, e.g.
  `{ "region": 41, "no_training": 22 }`. With `strict=true` they are excluded; `strict` echoes the mode used.
- `promo`, `promo_source`, `confidence`, `promo_since`, `promo_ends_at` and `price_before_promo` describe a
  detected promotion; `promo` is `false` (and the rest `null`) on most offers. `promo_ends_at` is always an ISO
  date-time (e.g. `2026-09-18T23:59:59Z`): when the source publishes only a date, the promotion runs until
  23:59:59 UTC that day.
- `promo_expired` is `true` when the promotion's published end has passed: the discounted price is then no
  longer presented as the price you pay (reason code `promo_expired` in the explanation).
- `hidden_tiers` counts offers excluded by the default tier filter, by tier name.
- `stale_hidden` counts stale offers left out. For example, `/cheapest?model=z-ai/glm-5.2` returned
  `"stale_hidden": 1` on 2026-09-15; with `include_stale=true` that offer came back with `"stale": true` and a
  `checked_at` from the previous day.

Only active sources are included. 404 with `other_matches` (up to 5 close model ids) if `model` doesn't
resolve, or if it resolves but has no tracked price.

## Explained response

Every `/cheapest` response includes an `explanation` block saying how the result was reached:

| Field | Meaning |
|---|---|
| `considered` | Offers looked at for this model, including those left out for a missing exchange rate |
| `eligible` | Offers left after every rule and filter |
| `excluded` | Number of offers excluded, per reason code |
| `reason` | Why `cheapest` was chosen, e.g. `cheapest eligible offer by blended_per_1M` (or the `sort` field, such as `estimated_cost_per_request`), or `no eligible offer` |

With `explain=true`, the response also lists each excluded offer in `excluded_offers`:

```bash
curl "https://api.inferindex.dev/cheapest?model=deepseek/deepseek-v3.2&min_context=128000&explain=true"
```

_Example response as of 2026-09-15 — prices, providers and statuses change._

```json
{
  "explanation": {
    "considered": 54,
    "eligible": 39,
    "excluded": { "tier_hidden": 1, "duplicate": 11, "context_too_small": 3 },
    "reason": "cheapest eligible offer by blended_per_1M"
  },
  "excluded_offers": [
    {
      "provider": "AtlasCloud",
      "via": "aggregator",
      "variant": null,
      "quantization": "fp8",
      "input_per_1M": 0.26,
      "output_per_1M": 0.38,
      "blended_per_1M": 0.29,
      "reasons": ["duplicate"],
      "detail": "same offer kept via direct"
    }
  ]
}
```

`excluded_offers` entries also carry the estimated cost when usage parameters are given. Rules are applied in
this order: stale → hidden tier → duplicate → filters. An offer excluded at one step isn't evaluated by the next
steps, but it can collect several filter codes at the filter step — so the counts in `excluded` can add up to
more than `considered − eligible`.

| Code | Meaning |
|---|---|
| `stale` | Not re-checked recently, see `stale` in [Offer fields](#offer-fields) |
| `aggregated_price` | Single price for unnamed backends, see [Offer fields](#offer-fields) |
| `promo_expired` | The promotion's published end date has passed |
| `points_based` | Price converted from points or credits, see [Offer fields](#offer-fields) |
| `tier_hidden` | `flex` or `batch` tier, hidden by default |
| `duplicate` | Same offer kept through another source (cheaper, or direct at equal price) |
| `no_fx_rate` | (count only) no exchange rate for the source's currency |
| `quantization_mismatch` | Quantization not in `quantization` |
| `context_too_small` / `context_unknown_strict` | Context below `min_context` / unknown, with `strict=true` |
| `uptime_below_min` / `uptime_unknown_strict` | Uptime below `min_uptime` / unknown, with `strict=true` |
| `tools_unsupported` / `tools_unknown_strict` | No tool calling / unknown, with `strict=true` |
| `json_unsupported` / `json_unknown_strict` | No JSON output / unknown, with `strict=true` |
| `vision_unsupported` / `vision_unknown_strict` | No image input / unknown, with `strict=true` |
| `region_mismatch` / `region_unknown_strict` | Processing region doesn't match `region` / unknown, with `strict=true` |
| `training_not_excluded` / `training_unknown_strict` | Prompts may be used for training / unknown, with `strict=true` |
| `signup_restricted` / `signup_unknown_strict` | Signup isn't open to everyone / unknown, with `strict=true` |
| `quantization_inferred_strict` | Quantization inferred rather than declared, with `strict=true` |
| `unpublishable_provider` | Offer from a provider that InferIndex does not list publicly |

## Usage conditions

Each offer of `/cheapest` and `/resellers` carries `conditions`, read from the provider's official documents
(legal pages, documentation, trust centers — never third-party sites):

- `conditions.provider`: the provider's conditions;
- `conditions.via`: the same conditions for the aggregator the offer goes through, or `null` for direct offers.
  For an aggregator that isn't identified by name, only the codes are given (`value` repeats the code,
  `source_url` is `null`).

| Condition | Codes |
|---|---|
| `regions` | Where requests are **processed**, e.g. `EU`, `US`, `UK`, `CH`, `global` (processing may leave a region). Several regions are comma-separated, e.g. `SG,ID,US` |
| `data_hosting_region` | Where data is **hosted or stored**, e.g. `EU`, `US`, `SG`, `global` — informational, not used by `region=` |
| `data_retention` | `none`, `limited`, `stored` |
| `training_on_prompts` | `no`, `yes`, `opt_out`, `opt_in`, `conditional` |
| `rate_limits` | `published`, `none_published` |
| `sla` | `published`, `enterprise_only`, `none` |
| `tools`, `json`, `vision` | `yes`, `no` — as declared in the source's data |
| `access` | How to get an account with the provider, see [Access conditions](#access-conditions) |

Every condition can also be `unknown`. Each value has this shape:

_Example response as of 2026-09-15 — prices, providers and statuses change._

```json
{
  "training_on_prompts": {
    "value": "no (except Google/Anthropic models per their policies)",
    "code": "no",
    "source_url": "https://deepinfra.com/terms",
    "checked_at": "2026-09-15T00:00:00Z"
  }
}
```

**Deployment region of an offer.** When the source publishes where a given offer is deployed (for example a
cloud region such as `eu-west-1`, `francecentral` or `fr-par`), `regions` gives that region for this offer
rather than the provider's general statement, and `region=` uses it. London counts as `UK` and Zurich as `CH`,
never `EU`; a vague or unknown region name (`europe`, `global`) gives no region. Example, as of 2026-09-18:

```json
{ "regions": { "value": "Deployment region 'eu' published by the source for this offer", "code": "EU", "source_url": null, "checked_at": "…" } }
```

`value` is the provider's wording, `code` the normalized value, `source_url` the page it was read from and
`checked_at` the last check that the statement is still on that page. A condition that isn't published, or is
ambiguous or contradictory, is `unknown` — never an implicit yes. Conditions change: check `source_url` before
relying on one.

**Filters** (`/cheapest` and `/resellers`): `region=` (`eu, us, cn, uk, ch, sg, id, my, vn, kr, jp, in, ca, au`) keeps offers processed in that region, `no_training=true`
offers that don't train on prompts. Provider **and** aggregator must both qualify: an offer resold by an
aggregator with no guaranteed region (`global`) is unknown for `region=`, and dropped with `strict=true`, even when
the backend itself is in that region. Unknown values follow the
same rule as other filters (kept and flagged in `unverified` / `filters_unknown`, excluded with `strict=true`).

## Access conditions

`conditions.provider.access` describes what it takes to open an account with the provider — useful before you
switch to a cheaper offer you can't actually sign up for. Each field uses the same
`{ value, code, source_url, checked_at }` shape as the other conditions:

| Field | Codes |
|---|---|
| `signup` | `open`, `waitlist`, `invite_only`, `closed` |
| `minimum_spend` | Whether a minimum spend or prepaid credit is required |
| `payment_card` | Whether a payment card is required to start |
| `excluded_countries` | Countries the provider says it doesn't serve |
| `identity_verification` | Whether identity or phone verification is required |

Any field can be `unknown`, which means the provider doesn't publish it clearly — not that the answer is no.

**Filter**: `no_waitlist=true` on `/cheapest` and `/resellers` keeps only providers whose signup is open to
everyone. Reason codes `signup_restricted` and, with `strict=true`, `signup_unknown_strict`.

These fields were introduced on 2026-09-16 and are being filled in progressively, so most are still `unknown`
today; `no_waitlist=true&strict=true` therefore excludes almost everything for now.

## Cost estimate

`/cheapest` and `/resellers` can estimate what a workload would cost at each offer. Pass `prompt_tokens` and/or
`output_tokens` (plus optionally `cached_ratio` and `requests_per_day`); without them, responses are unchanged.
Add `sort=estimated_cost` to rank offers by estimated cost per request instead of list price.

```bash
curl "https://api.inferindex.dev/cheapest?model=deepseek/deepseek-v3.2&prompt_tokens=2000&output_tokens=500&cached_ratio=0.5&requests_per_day=1000&sort=estimated_cost"
```

_Example response as of 2026-09-15 — prices, providers and statuses change._

```json
{
  "sort": "estimated_cost",
  "usage": { "prompt_tokens": 2000, "output_tokens": 500, "cached_ratio": 0.5, "requests_per_day": 1000, "note": "…" },
  "cheapest": {
    "input_per_1M": 0.2088,
    "output_per_1M": 0.3096,
    "cache_read_per_1M": 0.0216,
    "estimated_cost_per_request": 0.000385,
    "estimated_cost_per_month": 11.56,
    "estimate_approx": false,
    "estimate_tier_applied": false,
    "estimate_notes": [],
    "…": "…"
  },
  "offers": [ "… same fields on every offer" ]
}
```

Here: 1,000 uncached prompt tokens × $0.2088/M + 1,000 cached tokens × $0.0216/M + 500 output tokens × $0.3096/M
≈ $0.000385 per request, × 1,000 requests × 30 days ≈ **$11.56 per month**.

| Field | Meaning |
|---|---|
| `usage` | The usage parameters taken into account, echoed back |
| `estimated_cost_per_request` | Estimated cost of one request, in USD |
| `estimated_cost_per_month` | Estimated cost over 30 days, in USD — only with `requests_per_day` |
| `estimate_tier_applied` | `true` when the offer has pricing brackets and the bracket matching both `prompt_tokens` and `output_tokens` was used |
| `estimate_approx` | `true` when the estimate is approximate — see `estimate_notes` for why |
| `estimate_notes` | Why, when relevant (see below) |

| Note | Meaning |
|---|---|
| `tiers_not_detailed` | The source doesn't detail its brackets: the first bracket is used |
| `beyond_published_tiers` | The request is larger than the last bracket the source publishes: the price of the highest bracket reached is used |
| `cache_price_unknown` | No cache price published: the input price is used for the cached part |
| `peak_pricing` | The price depends on the time of the call (peak / off-peak): the estimate uses the normal rate |
| `exceeds_context` | Prompt + output is larger than the offer's context window — the offer is still listed |

Brackets can depend on prompt size, on output size, or both; bracket prices published in another currency are
converted to USD at the current rate.

Estimates use the listed prices only: they ignore minimum charges, taxes, free quotas and anything billed
outside tokens. Invalid values (e.g. a negative token count, `cached_ratio` above 1, or `sort=estimated_cost`
without `prompt_tokens`/`output_tokens`) return **400** with an `error` message.

## `GET /resellers`

Current prices for a tracked model across every active source (direct providers and aggregators/routers), one
line per source — no deduplication, unlike `/cheapest`.
Paginated beyond 100 offers.

| Parameter | Required | Meaning |
|---|---|---|
| `model` | yes | Same matching as `/cheapest` |
| `sort` | no | Same as `/cheapest`, including `estimated_cost` |
| `limit` | no | Page size, default 100, max 500 |
| `cursor` | no | Opaque cursor from a previous response's `next_cursor` |
| `include_tiers` | no | Same as `/cheapest` |
| `region`, `no_training`, `no_waitlist`, `strict` | no | Same as `/cheapest`, see [Usage conditions](#usage-conditions) and [Access conditions](#access-conditions) |
| `prompt_tokens`, `output_tokens`, `cached_ratio`, `requests_per_day` | no | Cost estimate, same as `/cheapest` |

```bash
curl "https://api.inferindex.dev/resellers?model=deepseek/deepseek-v3.2&limit=2"
```

_Example response as of 2026-09-15 — prices, providers and statuses change._

```json
{
  "model": { "id": "deepseek/deepseek-v3.2", "name": "DeepSeek: DeepSeek V3.2" },
  "sort": "blended",
  "cheapest": {
    "provider": "Nous Portal",
    "via": "nous",
    "variant": null,
    "tier": "standard",
    "input_per_1M": 0.2088,
    "output_per_1M": 0.3096,
    "currency": "USD",
    "quantization": "unknown",
    "context_length": 163840,
    "promo": false,
    "price_since": "2026-09-14T19:24:43.680Z",
    "checked_at": "2026-09-15T03:42:23.752Z"
  },
  "total_offers": 53,
  "hidden_tiers": { "flex": 1 },
  "next_cursor": "WzIsImJsZW5kZWQiXQ",
  "offers": [ "… up to `limit` offers" ]
}
```

`via` is `"direct"` when the provider publishes the price itself, `"aggregator"` for an offer resold by an
aggregator that isn't identified by name, or otherwise the id of the aggregator/router reselling it (e.g.
`"nous"`) — the same model/provider pair can appear more than once via different
routes, often at different prices. Stale offers (`"stale": true`) are not hidden here but always listed after
every fresh offer, whatever the `sort`. `cheapest` is always the single cheapest offer across *all* pages, not just
the current one, and follows the same rules as `/cheapest`: it is never a stale offer, an expired promotion, a
price converted from points, or a gateway's aggregated price. Keep paging with `cursor` while `next_cursor` is
non-null.

When `region` or `no_training` is set, the response adds `filters_unknown` and `excluded` (number of offers
excluded, per reason code — same codes as [Explained response](#explained-response)).

**Reliability.** Each offer carries `reliability` (the provider's) and `via_reliability` (the aggregator's, when
the offer goes through one that publishes a status page), read from official status pages:

_Example response as of 2026-09-15 — prices, providers and statuses change._

```json
{
  "status": "operational",
  "status_page": "https://status.streamlake.ai/",
  "checked_at": "2026-09-15T14:37:00.359Z",
  "open_incidents": [],
  "incidents_30d": 0,
  "major_incidents_30d": 0,
  "last_incident_at": null,
  "source_url": "https://status.streamlake.ai/history.rss"
}
```

`status` is `operational`, `incident` (at least one incident in progress, listed in `open_incidents` with title,
impact, start and link) or `unknown`. **`unknown` means no status page could be read — not "no incident".**
Scheduled maintenance isn't counted. This is what providers publish about themselves, not an independent
latency or availability measurement.

## `GET /history`

Price history for a tracked model, paginated by cursor. Two ways to ask:

- over a lookback window (`days`) or a period (`from` / `to`), row by row or aggregated by day or week;
- the prices in force at a given moment (`at`).

| Parameter | Required | Meaning |
|---|---|---|
| `model` | yes | Same matching as `/cheapest` |
| `days` | no | Lookback window, default 7. Max depends on `granularity`: 90 (`raw`), 400 (`day`), 1500 (`week`) |
| `from`, `to` | no | Period instead of `days`: `YYYY-MM-DD` or ISO 8601 date-time. A date alone for `to` means the end of that day (UTC); `to` defaults to now. Same maximum length as `days` — a longer period is shortened from `from`, with `days_capped` |
| `at` | no | `YYYY-MM-DD` or ISO 8601 date-time: prices in force at that moment (a date alone means the end of that day, UTC; a future moment means now) |
| `granularity` | no | `raw` (default): one row per price change. `day` / `week`: one point per period and per offer. Not used with `at` |
| `series` | no | `cheapest`: the cheapest offer of each day instead of the full history, see [Cheapest offer of each day](#cheapest-offer-of-each-day). Only with `days` or `from`/`to` |
| `provider` | no | Filter to one provider name (case-insensitive) |
| `limit` | no | Page size, default 1000 |
| `cursor` | no | Opaque cursor from a previous response's `next_cursor` |

`at` can't be combined with `days`, `from` or `to`, and `days` can't be combined with `from` or `to`; `from` must
be before `to`. These cases, and unreadable dates, return **400** with an `error` message.

### Over a period

```bash
curl "https://api.inferindex.dev/history?model=deepseek/deepseek-v3.2&from=2026-09-14&to=2026-09-15&granularity=day&limit=1"
```

_Example response as of 2026-09-15 — prices, providers and statuses change._

```json
{
  "model": { "id": "deepseek/deepseek-v3.2", "name": "DeepSeek: DeepSeek V3.2" },
  "from": "…",
  "to": "…",
  "tracking_since": "2026-09-14T18:02:05Z",
  "granularity": "day",
  "points": 1,
  "next_cursor": "…",
  "truncated": true,
  "history": [
    {
      "period": "2026-09-14",
      "provider": "Alibaba",
      "quantization": "fp8",
      "variant": "",
      "sources": 2,
      "input_min": 0.3705,
      "input_max": 0.3705,
      "input_last": 0.3705,
      "output_min": 1.1115,
      "output_max": 1.1115,
      "output_last": 1.1115,
      "original_currency": "USD",
      "original_input_last": 0.3705,
      "original_output_last": 1.1115
    }
  ]
}
```

With `day`/`week`, each row aggregates one offer (provider + quantization + variant) over one period, in USD,
with `original_currency` and the last prices in that currency. Weeks start on Monday.

With `granularity=raw`, each row is a single price snapshot: `captured_at`, `last_checked_at`, `ended_at`,
`source`, `provider`, `variant`, `quantization`, prices in the source's currency (`currency`, `input_per_1m`,
`output_per_1m`, `cache_read_per_1m`) and in USD (`input_usd_per_1M`, `output_usd_per_1M`,
`cache_read_usd_per_1M`). `source` is the id of the pricing source, or `"aggregator"` for an aggregator that isn't
identified by name. `ended_at` is when the price was replaced; `null` means the price is still in force.
`last_checked_at` may lag by up to 2 hours (see `checked_at` in [Offer fields](#offer-fields)).

### Prices at a given moment

```bash
curl "https://api.inferindex.dev/history?model=deepseek/deepseek-v3.2&at=2026-09-15"
```

_Example response as of 2026-09-15 — prices, providers and statuses change._

```json
{
  "model": { "id": "deepseek/deepseek-v3.2", "name": "DeepSeek: DeepSeek V3.2" },
  "at": "2026-09-15T23:59:59.999Z",
  "tracking_since": "2026-09-14T18:02:05Z",
  "count": 1,
  "next_cursor": null,
  "offers": [
    {
      "source": "atlascloud",
      "provider": "AtlasCloud",
      "variant": null,
      "quantization": "fp8",
      "price_since": "2026-09-14T18:02:11.965Z",
      "price_until": null,
      "input_per_1M": 0.26,
      "output_per_1M": 0.38,
      "cache_read_per_1M": 0.13,
      "currency": "USD",
      "fx_rate_date": null,
      "original_currency": "USD",
      "original_input_per_1M": 0.26,
      "original_output_per_1M": 0.38,
      "original_cache_read_per_1M": 0.13
    }
  ]
}
```

One row per offer (source, provider, quantization, variant): the last price captured at or before `at` that was
still valid at that moment (`price_since` / `price_until`; `price_until` is `null` when the price is still in force).

### Cheapest offer of each day

With `series=cheapest`, `/history` returns one point per day at 00:00 UTC: the offer `/cheapest` would have
selected at that instant with its default filters. Use `days` or `from`/`to` (not `at`); the period is limited to
**90 days** — a longer request is shortened and flagged with `days_capped: true` and `requested_days`.

```bash
curl "https://api.inferindex.dev/history?model=deepseek/deepseek-v3.2&series=cheapest&days=3"
```

_Example response as of 2026-09-18 — prices, providers and statuses change._

```json
{
  "model": { "id": "deepseek/deepseek-v3.2", "name": "DeepSeek: DeepSeek V3.2" },
  "series": "cheapest",
  "from": "2026-09-15T23:16:55.521Z",
  "to": "2026-09-18T23:16:55.521Z",
  "tracking_since": "2026-09-14T18:02:05Z",
  "days": 3,
  "count": 3,
  "note": "…",
  "cheapest_by_day": [
    {
      "date": "2026-09-16",
      "at": "2026-09-16T00:00:00.000Z",
      "cheapest": {
        "provider": "GMICloud",
        "via": "aggregator",
        "variant": null,
        "quantization": "fp8",
        "input_per_1M": 0.2088,
        "output_per_1M": 0.3096,
        "blended_per_1M": 0.234,
        "promo": true
      },
      "providers": 41
    }
  ]
}
```

Prices are in USD at the latest ECB rate published before each instant. `cheapest` is `null` on a day when no offer
was eligible; `providers` is the number of providers considered that day. Any other `series` value, or
`series=cheapest` with `at`, returns **400**.

### Exchange rates

History prices are converted to USD at the **ECB rate of the day of the price**, not today's rate: the day of
`at`, the day of each raw snapshot, or the day of each `day`/`week` period. The rate used is the latest one
published on or before that day (no rates on weekends); `fx_rate_date` gives it for `at` (`null` for a price
already in USD). Prices as published stay in `original_*`. `/cheapest` and `/resellers` use the current rate.

### Official prices before tracking

For models with a lab that publishes list prices (for example OpenAI, Anthropic or Google), `/history` also returns
`official_prices`: the lab's own list price over time, for the period before offers were tracked. Each entry is
one interval:

```bash
curl "https://api.inferindex.dev/history?model=openai/gpt-4o&days=2&limit=1"
```

_Example response as of 2026-09-18 — prices, providers and statuses change._

```json
{
  "official_prices": [
    { "provider": "OpenAI", "effective_from": "2024-05-13", "effective_to": "2024-08-06", "input_per_1M": 5, "output_per_1M": 15, "cache_read_per_1M": null, "currency": "USD", "note": null },
    { "provider": "OpenAI", "effective_from": "2024-08-06", "effective_to": "2026-08-28", "input_per_1M": 2.5, "output_per_1M": 10, "cache_read_per_1M": null, "currency": "USD", "note": null },
    { "provider": "OpenAI", "effective_from": "2026-08-28", "effective_to": "2026-09-14", "input_per_1M": 2.5, "output_per_1M": 10, "cache_read_per_1M": 1.25, "currency": "USD", "note": null }
  ]
}
```

| Field | Meaning |
|---|---|
| `provider` | The lab publishing the price |
| `effective_from`, `effective_to` | Dates the price applied (`YYYY-MM-DD`) |
| `input_per_1M`, `output_per_1M`, `cache_read_per_1M` | List price in USD per million tokens; `cache_read_per_1M` is `null` when no cache price applied |
| `currency` | Always `"USD"` |
| `note` | Optional remark on the interval, otherwise `null` |

With `at`, only the interval covering that date is returned — so a date before tracking started still gets the
lab's price, next to `"before_tracking": true`. With `days` or `from`/`to`, all intervals are returned. These are
list prices, not offers: they aren't included in `offers`, `history`, `/cheapest` or `/resellers`. The field is
absent for models without published lab prices (most open-weight models).

### Before tracking started

Price tracking started on **2026-09-14T18:02:05Z** (`tracking_since`). An `at` or `to` before that returns 200 with
no offer data and `"before_tracking": true`, not an error (plus `official_prices` when available, see above):

_Example response as of 2026-09-15 — prices, providers and statuses change._

```json
{ "at": "2026-09-01T23:59:59.999Z", "tracking_since": "2026-09-14T18:02:05Z", "before_tracking": true, "count": 0, "next_cursor": null }
```

404 if the model resolves but has no history at all (as opposed to an empty page from pagination, which returns
200 with no rows).

## `GET /index`

The weekly InferIndex market index: the median price of one million tokens (USD, blended 3:1) per model family,
published every Monday at 00:00 UTC from **2026-10-05**. How it is computed:
[market index methodology](market-index.md).

| Parameter | Required | Meaning |
|---|---|---|
| `series` | no | `market` (default): from the offers we collect. `official`: from the labs' list prices we collect |
| `at` | no | `YYYY-MM-DD`: the latest publication on or before that date. Default: the latest publication |

```bash
curl "https://api.inferindex.dev/index?series=market"
```

Response shape (values in `<…>` are placeholders, not published data):

```json
{
  "index": "InferIndex market index",
  "series": "market",
  "published_at": "<Monday>T00:00:00.000Z",
  "method_version": "1.0",
  "unit": "USD per 1M tokens, blended 3:1 (input:output)",
  "families": [
    { "family": "frontier_closed", "value": "<number>", "models": "<n>", "model_ids": ["…"] },
    { "family": "compact_closed", "value": null, "models": 2, "model_ids": ["…", "…"] }
  ],
  "available_dates": ["<Monday>", "…"],
  "corrections": []
}
```

- `families` lists `frontier_closed`, `compact_closed`, `code`, `open_large`, `open_medium` and `open_small`, in that
  order. `value` is `null` when the family had fewer than 3 models that week; `models` and `model_ids` give the models
  counted, so the value can be recomputed from their reference prices.
- `method_version` is the methodology version used for that publication.
- `available_dates` lists the published Mondays of the series (up to 52, most recent first).
- Values are stored at publication and served as published. A published week is only recomputed through an explicit
  correction, listed in `corrections` for the week and series served (empty when there is none). Each entry:
  `{ "family", "previous_value", "value", "previous_models", "models", "reason", "corrected_at" }`, where
  `previous_*` are the values before the correction and `value` / `models` those after it (also shown in
  `families`).

Before the first publication, the route returns **404**:

```json
{
  "error": "The InferIndex market index is published every Monday at 00:00 UTC, starting 2026-10-05",
  "first_publication": "2026-10-05T00:00:00.000Z"
}
```

An unknown `series` or a malformed `at` returns **400**.

## `GET /models`

Search tracked models by name or id fragment.

| Parameter | Required | Meaning |
|---|---|---|
| `search` | no | Query string; omit to list up to 50 tracked models |

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

`count` is the total number of matches; `models` is capped at 50 even if `count` is higher. Only real models are
listed — see [Model identifiers](#model-identifiers).

## `GET /health/live`

Liveness only: confirms the service is up and returns the deployed version. Does not touch the database —
use `/health/ready` to check data freshness.

_Example response as of 2026-09-15 — prices, providers and statuses change._

```json
{ "status": "ok", "version": { "id": "…", "deployed_at": "…" } }
```

## `GET /health/ready`

Readiness: data freshness and collection health. Never cached. Suitable as an uptime-monitor target if your
integration depends on this API.

Service status: https://status.inferindex.dev (external uptime monitoring) shows current and past availability.
The API still comes with no uptime guarantee.

```bash
curl "https://api.inferindex.dev/health/ready"
```

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

`status` is `ok`, `degraded` (something non-critical is having trouble — see `degraded`) or `unavailable` (no
usable price data, or the readiness check itself failed). `ready` is `true` with HTTP 200 for `ok`/`degraded`,
`false` with HTTP 503 for `unavailable`. `failures`, `degraded` and `warnings` are human-readable English strings (e.g. `"1 scheduler(s) late"`,
`"more than 10% of price sources in error"`) and `source_errors`
only gives counts. `schedulers` counts the price-collection jobs and how many are running on time. Treat
everything except `ready`, `status` and `degraded` as informational and subject to change.
