# API reference

All routes are `GET`, work without authentication, and are served from `https://api.inferindex.dev`. Responses are JSON.
Successful responses carry `Cache-Control: public, max-age=300` (5 minutes) unless noted otherwise; do not
poll faster than that, it won't return fresher data. See the [README](../README.md) for rate limits, cache
behavior and a warning about price reliability before you go further.

This page documents the public read routes only. It is generated from the backend's route table and checked
against the production API on 2026-09-15.

## Authentication and rate limits

No key is needed. Requests without a key are limited to **60 requests per minute per IP** on `/cheapest`,
`/history`, `/resellers`, `/self-host`, `/gpus` and `/gpu-rentals` (requests answered from cache don't count). Over the limit, the API returns **429**.
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
| `offer_url` | The official page where the offer can be checked and bought: the aggregator's page when the offer is sold through a named aggregator, otherwise the provider's pricing page. `null` when no reliable public page is known |
| `peak_pricing` | For providers that charge more at peak hours: `{ input_per_1M, output_per_1M, hours }`, the peak rate in USD and the provider's description of peak hours (`hours` may be `null`). The main prices are the off-peak rate. `null` otherwise |
| `price_context` | What the displayed price is, as a list of entries — empty for a plain price. Every percentage names its reference. See [Price context](#price-context) |
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

### Price context

`price_context` explains the displayed price. The existing fields `promo`,
`price_before_promo`, `peak_pricing` and `below_official_list` keep their meaning. An offer can carry several entries.

| `type` | Meaning | Other fields |
|---|---|---|
| `scheduled` | Off-peak price of a published time-of-use schedule. Never a promotion | `tariff` (`"off_peak"`), `hours` (provider's description, may be `null`), `peak` (`{ input_per_1M, output_per_1M }`), `discount`, `reference: "peak"` |
| `promotion` | Discount announced by the provider (`source: "provider"`), or confirmed when the price came back up (`source: "history"`) | `source`, `ends_at` (`null` when no end date is published), `discount`, `reference: "provider_usual"` |
| `price_drop_observed` | Drop seen in the price history, with no announced promotion, or a discount declared by a provider with no announced end date for 14 days or more. Never shown as a promotion | `discount`, `reference: "previous_price"` |
| `below_lab_list` | Sold below the lab's official list price, with no declared discount | `discount`, `reference: "lab_list"` |
| `service_tier` | Non-standard service level | `tier` (`flex`, `batch` or `priority`) |
| `cached_input` | Price of cached input tokens, kept apart from `input_per_1M` | `cache_read_per_1M` |

`discount` is the drop in blended price, from 0 to 1 (4 decimals), measured against its `reference`: the peak rate
(`peak`), the provider's usual price (`provider_usual`), the previous observed price (`previous_price`) or the lab's
list price (`lab_list`).

_Example entries as of 2026-09-19 — prices, providers and statuses change._

```json
"price_context": [
  { "type": "promotion", "source": "provider", "ends_at": null, "discount": 0.3, "reference": "provider_usual" },
  { "type": "cached_input", "cache_read_per_1M": 0.0216 }
]
```

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
| `offers` | no | `false` to return only `cheapest` and the counts, without the `offers` list |
| `limit` | no | Return only the first N offers, 1 to 500 |
| `detail` | no | `full` (default) or `compact`: a shorter response for AI assistants, see [Compact view](#compact-view) |

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
  23:59:59 UTC that day. A discount declared by a provider with no announced end date for 14 days or more is shown as a
  price drop rather than a promotion: `promo` is `"probable"`, `price_before_promo` holds the previous price and
  `price_context` has a `price_drop_observed` entry. The price and the ranking don't change.
- `promo_expired` is `true` when the promotion's published end has passed: the discounted price is then no
  longer presented as the price you pay (reason code `promo_expired` in the explanation).
- `hidden_tiers` counts offers excluded by the default tier filter, by tier name.
- `offers_total` is the number of eligible offers, whatever `offers` or `limit`; `offers_truncated` is `true` when
  `limit` cut the list. Invalid `offers` or `limit` values return **400**.
- `stale_hidden` counts stale offers left out. For example, `/cheapest?model=z-ai/glm-5.2` returned
  `"stale_hidden": 1` on 2026-09-15; with `include_stale=true` that offer came back with `"stale": true` and a
  `checked_at` from the previous day.

Only active sources are included. 404 with `other_matches` (up to 5 close model ids) if `model` doesn't
resolve, or if it resolves but has no tracked price.

## Compact view

`detail=compact` on `/cheapest` and `/resellers` returns a shorter response, made for AI assistants: the same winner and
the same prices as the default view, but the winner without its empty fields, and **one short line** for each other
offer. `detail=full` is the default and changes nothing. Any other value returns **400** (`detail must be full or compact`).

```bash
curl "https://api.inferindex.dev/cheapest?model=deepseek/deepseek-v3.2&detail=compact&limit=3"
```

The response adds `"detail": "compact"` and a `note`. Each line for the other offers carries `provider`, `via`,
`input_per_1M`, `output_per_1M`, `blended_per_1M`, the estimated cost when the request asks for one, `signals` and
`training_on_prompts`. Example line, as of 2026-10-05 — prices and providers change:

```json
{ "provider": "Avian", "via": "direct", "input_per_1M": 0.23, "output_per_1M": 0.33, "blended_per_1M": 0.255,
  "training_on_prompts": { "provider": "no" } }
```

How to read it:

- A field or condition that is absent was not published by the provider: it never means "no".
- `signals` lists the flags that are true (for example `stale`, `promo`, `points_based`, `backend_unknown`); a flag that is
  not listed is false.
- For everything else (every condition, reliability, original currency, context), use the default view or the
  `/cheapest` response without `detail`.

## Explained response

Every `/cheapest` response includes an `explanation` block saying how the result was reached:

| Field | Meaning |
|---|---|
| `considered` | Offers looked at for this model, including those left out for a missing exchange rate |
| `eligible` | Offers left after every rule and filter |
| `excluded` | Number of offers excluded, per reason code |
| `reason` | Why `cheapest` was chosen, e.g. `cheapest eligible offer by blended_per_1M` (or the `sort` field, such as `estimated_cost_per_request`), or `no eligible offer` |

With `explain=true`, the response also lists each excluded offer in `excluded_offers` (except offers from providers
InferIndex does not list publicly, which are only counted):

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
| `unpublishable_provider` | Offer from a provider that InferIndex does not list publicly. Only counted in `/cheapest`'s `explanation`: such offers never appear in `excluded_offers`, `/resellers`, `/history` or MCP results |

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
| `signup` | `open`, `waitlist`, `invite_only`, `restricted` (sign-up possible, but reserved for some countries or profiles), `closed` |
| `minimum_spend` | Whether a minimum spend or prepaid credit is required |
| `payment_card` | Whether a payment card is required to start |
| `excluded_countries` | Countries the provider says it doesn't serve |
| `identity_verification` | Whether identity or phone verification is required |

Any field can be `unknown`, which means the provider doesn't publish it clearly — not that the answer is no.

**Filter**: `no_waitlist=true` on `/cheapest` and `/resellers` keeps only providers whose signup is open to
everyone: no waitlist, invitation or country restriction. Reason codes `signup_restricted` and, with `strict=true`, `signup_unknown_strict`.

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
| `detail` | no | `full` (default) or `compact`, see [Compact view](#compact-view) |
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
  "method_version": "1.1",
  "unit": "USD per 1M tokens, blended 3:1 (input:output)",
  "families": [
    { "family": "frontier_closed", "value": "<number>", "models": "<n>", "model_ids": ["…"] },
    { "family": "compact_closed", "value": null, "models": 2, "model_ids": ["…", "…"] }
  ],
  "available_dates": ["<Monday>", "…"],
  "archives": [
    { "kind": "publication", "url": "https://api.inferindex.dev/index/archive/<Monday>/publication.json", "sha256": "<hex>", "bytes": "<n>", "archived_at": "<timestamp>" }
  ],
  "corrections": []
}
```

- `families` lists `frontier_closed`, `compact_closed`, `code`, `open_large`, `open_medium` and `open_small`, in that
  order. `value` is `null` when the family had fewer than 3 models that week; `models` and `model_ids` give the models
  counted, so the value can be recomputed from their reference prices.
- `method_version` is the methodology version used for that publication.
- `available_dates` lists the published Mondays of the series (up to 52, most recent first).
- `archives` lists the archived JSON files of the week and series served: the publication itself, and one more file for
  each later correction. Each entry gives the file's `url`, its `sha256` fingerprint, its size in `bytes` and
  `archived_at`. To check a file, keep the fingerprint on the day it is published, download the file later and compute
  its SHA-256 (`sha256sum file.json` on Linux, `shasum -a 256 file.json` on macOS, `Get-FileHash file.json` in
  PowerShell): if both match, the file is exactly the one published. The service never overwrites an archived file.
- Values are stored at publication and served as published. A published week is only recomputed through an explicit
  correction, listed in `corrections` for the week and series served (empty when there is none). Each entry:
  `{ "family", "previous_value", "value", "previous_models", "models", "reason", "corrected_at" }`, where
  `previous_*` are the values before the correction and `value` / `models` those after it (also shown in
  `families`).

Both series have been published every Monday since 2026-10-05. An `at` date earlier than the first publication returns
**404**:

```json
{
  "error": "The InferIndex market index is published every Monday at 00:00 UTC, starting 2026-10-05",
  "first_publication": "2026-10-05T00:00:00.000Z"
}
```

An unknown `series` or a malformed `at` returns **400**.

## `GET /trending`

A ranking of models currently getting attention, up to 10 models. It is recomputed about every 6 hours.

```bash
curl "https://api.inferindex.dev/trending"
```

_Example response as of 2026-09-19 — the ranking changes._

```json
{
  "computed_at": "2026-09-19T07:09:10.156Z",
  "models": [
    { "rank": 1, "id": "deepseek/deepseek-v4.1-flash", "name": "DeepSeek: DeepSeek V4.1 Flash" }
  ]
}
```

| Field | Meaning |
|---|---|
| `computed_at` | When the ranking was computed |
| `models` | Up to 10 entries, in order: `rank` (1 first), `id` (usable as `model` in the other routes), `name` |

Returns **404** until a first ranking has been computed.

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

## GPU rental prices

Two routes list what it costs to rent a GPU by the hour, as the providers we track publish it. They are the prices behind [Self-host or API](#self-host-or-api). Each offer names its provider and links to the provider's public price page.

| Route | What it returns |
|---|---|
| `GET /gpus` | The GPU types we track, with the lowest price of each tier and how many providers offer it. |
| `GET /gpu-rentals?gpu=<id>` | The rental offers for one GPU, one block per tier, each with its own cheapest offer. |

```
GET https://api.inferindex.dev/gpus
GET https://api.inferindex.dev/gpu-rentals?gpu=h100-sxm-80gb
```

### How to read the prices

- Prices are in **USD per GPU per hour**, as each provider publishes them on its public price list, and **dated**: `price_since` (when we first saw this price) and `checked_at` (when we last confirmed it).
- **Tiers are never compared with each other.** `guaranteed` is on-demand capacity the provider presents as not interrupted (it is not a long-term reservation, and we do not verify availability); `community` is capacity from third-party hosts; `spot` is interruptible capacity the provider can reclaim. Each tier has its own cheapest offer; there is no winner across tiers, because they are different products.
- **The price per GPU depends on the configuration.** A single GPU and a multi-GPU node of the same provider can have different prices per GPU, so compare offers with the same `gpu_count`. Each tier gives its cheapest offer per configuration in `cheapest_by_gpu_count`.
- `billing` (`second`, `minute` or `hour`) and `region` are left out when the provider does not publish them; we never fill them in.
- An offer that was not re-checked for more than three times the reading interval of its source (at least six hours: about 18 hours today) is marked `"stale": true`. It stays listed, after the fresh offers, and is never the cheapest.
- Each provider's price list is read every six hours, and each offer carries `checked_at`. Answers are cached for up to 30 minutes (`Cache-Control: public, max-age=1800`); `fetched_at` says when the answer was built.
- The same per-IP limit applies as to the other routes (60 requests a minute without a key); cached answers are not counted.

### `GET /gpus`

No parameters. Response:

| Field | Meaning |
|---|---|
| `gpus` | One entry per GPU type: `id` (use it in `/gpu-rentals` and as `gpu` in `/self-host`), `name`, `family`, `form` (SXM, PCIe… or `null`), `vram_gb` (or `null`), and `cheapest`. |
| `gpus[].cheapest` | For each tier that has a fresh offer: `usd_per_gpu_hour`, `provider`, `gpu_count` (the configuration of that offer) and `providers` (how many distinct providers have a fresh offer for this GPU at this tier, whatever the configuration; not the number of providers at the lowest price). A tier without a fresh offer is absent. `gpu_count` is the configuration of the cheapest offer: it can be a multi-GPU node, which is not comparable with a single GPU (see `cheapest_by_gpu_count` in `/gpu-rentals`). At equal prices, the offer shown is the first by price, then by size of configuration, then by provider name. |
| `total` | Number of GPU types. |
| `note` | The rules above in one paragraph. |
| `fetched_at` | When the answer was built. |

### `GET /gpu-rentals`

| Parameter | Default | Meaning |
|---|---|---|
| `gpu` | required | A GPU id from `/gpus`, for example `h100-sxm-80gb`. |
| `tier` | all three | `guaranteed`, `community` or `spot`: only this tier. |
| `gpu_count` | all | `1`, `2`, `4`, `8` or `16`: only this published configuration. As of 6 October 2026 the largest published node has 8 GPUs, so `gpu_count=16` returns a 200 with empty tiers (`cheapest: null`, `offers: []`, `offers_total: 0`). |
| `region` | all | Only this region, as published by the provider. An unknown region returns a 400 that lists the regions we have. |
| `limit` | 10 | 1 to 50 offers per tier. |

Response:

| Field | Meaning |
|---|---|
| `gpu` | The GPU: `id`, `name`, `family`, `form`, `vram_gb`. |
| `tiers` | One block per tier (`guaranteed`, `community`, `spot`, or only the one you asked for). |
| `tiers.<tier>.cheapest` | The cheapest fresh offer of the tier, or `null`. |
| `tiers.<tier>.cheapest_by_gpu_count` | The cheapest fresh offer for each published configuration, keyed by `gpu_count`. |
| `tiers.<tier>.offers` | Up to `limit` offers, fresh ones first, then stale ones, each by increasing price. |
| `tiers.<tier>.offers_total` | How many offers the tier has before `limit`. |
| `note`, `fetched_at` | As above. |

An offer has: `provider`, `tier`, `usd_per_gpu_hour`, `gpu_count`, `billing` and `region` when published, `price_since`, `checked_at`, `source_url` (the provider's public price page, when there is one) and `stale` when it applies.

Errors: a missing or malformed `gpu` returns a 400; an invalid `tier`, `gpu_count`, `region` or `limit` returns a 400 that says the values allowed; a GPU we do not know, or one with no open offer, returns a 404 ("see /gpus"). `limit` applies to each tier: `offers_total` gives the number of offers the tier has.

### Examples

```bash
curl "https://api.inferindex.dev/gpus"
```

_Abridged example response as of 2026-10-06 — prices, providers and statuses change; `…` marks what was left out._

```json
{
  "gpus": [
    {
      "id": "b200-sxm-180gb",
      "name": "B200 SXM 180GB",
      "family": "B200",
      "form": "SXM",
      "vram_gb": 180,
      "cheapest": {
        "guaranteed": {
          "usd_per_gpu_hour": 6.69,
          "provider": "Lambda",
          "gpu_count": 8,
          "providers": 4
        },
        "spot": {
          "usd_per_gpu_hour": 3.56,
          "provider": "Verda",
          "gpu_count": 1,
          "providers": 2
        }
      }
    },
    {
      "id": "h100-sxm-80gb",
      "name": "H100 SXM 80GB",
      "family": "H100",
      "form": "SXM",
      "vram_gb": 80,
      "cheapest": {
        "guaranteed": {
          "usd_per_gpu_hour": 3.2,
          "provider": "Hyperstack",
          "gpu_count": 1,
          "providers": 9
        },
        "community": {
          "usd_per_gpu_hour": 2.69,
          "provider": "RunPod",
          "gpu_count": 1,
          "providers": 1
        },
        "spot": {
          "usd_per_gpu_hour": 1.89,
          "provider": "Verda",
          "gpu_count": 1,
          "providers": 2
        }
      }
    },
    "…"
  ],
  "total": 37,
  "note": "…",
  "fetched_at": "2026-10-06T12:37:31.000Z"
}
```

```bash
curl "https://api.inferindex.dev/gpu-rentals?gpu=h100-sxm-80gb&limit=3"
```

_Abridged example response as of 2026-10-06 — prices, providers and statuses change; `…` marks what was left out._

```json
{
  "gpu": {
    "id": "h100-sxm-80gb",
    "name": "H100 SXM 80GB",
    "family": "H100",
    "form": "SXM",
    "vram_gb": 80
  },
  "tiers": {
    "guaranteed": {
      "cheapest": {
        "provider": "Hyperstack",
        "tier": "guaranteed",
        "usd_per_gpu_hour": 3.2,
        "gpu_count": 1,
        "billing": "minute",
        "price_since": "2026-10-05T22:20:52.531Z",
        "checked_at": "2026-10-06T10:15:59.268Z",
        "source_url": "https://www.hyperstack.cloud/gpu-pricing"
      },
      "cheapest_by_gpu_count": {
        "1": {
          "provider": "Hyperstack",
          "tier": "guaranteed",
          "usd_per_gpu_hour": 3.2,
          "gpu_count": 1,
          "billing": "minute",
          "price_since": "2026-10-05T22:20:52.531Z",
          "checked_at": "2026-10-06T10:15:59.268Z",
          "source_url": "https://www.hyperstack.cloud/gpu-pricing"
        },
        "2": {
          "provider": "Lambda",
          "tier": "guaranteed",
          "usd_per_gpu_hour": 4.19,
          "gpu_count": 2,
          "price_since": "2026-10-05T22:20:51.232Z",
          "checked_at": "2026-10-06T10:15:58.151Z",
          "source_url": "https://lambda.ai/pricing"
        },
        "…": "…"
      },
      "offers": [
        {
          "provider": "Hyperstack",
          "tier": "guaranteed",
          "usd_per_gpu_hour": 3.2,
          "gpu_count": 1,
          "billing": "minute",
          "price_since": "2026-10-05T22:20:52.531Z",
          "checked_at": "2026-10-06T10:15:59.268Z",
          "source_url": "https://www.hyperstack.cloud/gpu-pricing"
        },
        {
          "provider": "Thunder Compute",
          "tier": "guaranteed",
          "usd_per_gpu_hour": 3.2,
          "gpu_count": 1,
          "billing": "minute",
          "price_since": "2026-10-05T16:41:21.815Z",
          "checked_at": "2026-10-06T10:36:03.247Z",
          "source_url": "https://www.thundercompute.com/pricing"
        },
        "…"
      ],
      "offers_total": 14
    },
    "community": {
      "cheapest": {
        "provider": "RunPod",
        "tier": "community",
        "usd_per_gpu_hour": 2.69,
        "gpu_count": 1,
        "price_since": "2026-10-05T22:20:50.100Z",
        "checked_at": "2026-10-06T10:15:56.970Z",
        "source_url": "https://www.runpod.io/pricing"
      },
      "cheapest_by_gpu_count": {
        "1": {
          "provider": "RunPod",
          "tier": "community",
          "usd_per_gpu_hour": 2.69,
          "gpu_count": 1,
          "price_since": "2026-10-05T22:20:50.100Z",
          "checked_at": "2026-10-06T10:15:56.970Z",
          "source_url": "https://www.runpod.io/pricing"
        }
      },
      "offers": [
        {
          "provider": "RunPod",
          "tier": "community",
          "usd_per_gpu_hour": 2.69,
          "gpu_count": 1,
          "price_since": "2026-10-05T22:20:50.100Z",
          "checked_at": "2026-10-06T10:15:56.970Z",
          "source_url": "https://www.runpod.io/pricing"
        }
      ],
      "offers_total": 1
    },
    "spot": {
      "cheapest": {
        "provider": "Verda",
        "tier": "spot",
        "usd_per_gpu_hour": 1.89,
        "gpu_count": 1,
        "price_since": "2026-10-05T16:41:20.528Z",
        "checked_at": "2026-10-06T10:36:02.227Z",
        "source_url": "https://verda.com/pricing"
      },
      "cheapest_by_gpu_count": {
        "1": {
          "provider": "Verda",
          "tier": "spot",
          "usd_per_gpu_hour": 1.89,
          "gpu_count": 1,
          "price_since": "2026-10-05T16:41:20.528Z",
          "checked_at": "2026-10-06T10:36:02.227Z",
          "source_url": "https://verda.com/pricing"
        },
        "8": {
          "provider": "CoreWeave",
          "tier": "spot",
          "usd_per_gpu_hour": 2.4388,
          "gpu_count": 8,
          "region": "europe",
          "price_since": "2026-10-05T22:20:53.841Z",
          "checked_at": "2026-10-06T10:16:00.300Z",
          "source_url": "https://www.coreweave.com/pricing"
        }
      },
      "offers": [
        {
          "provider": "Verda",
          "tier": "spot",
          "usd_per_gpu_hour": 1.89,
          "gpu_count": 1,
          "price_since": "2026-10-05T16:41:20.528Z",
          "checked_at": "2026-10-06T10:36:02.227Z",
          "source_url": "https://verda.com/pricing"
        },
        {
          "provider": "CoreWeave",
          "tier": "spot",
          "usd_per_gpu_hour": 2.4388,
          "gpu_count": 8,
          "region": "europe",
          "price_since": "2026-10-05T22:20:53.841Z",
          "checked_at": "2026-10-06T10:16:00.300Z",
          "source_url": "https://www.coreweave.com/pricing"
        },
        "…"
      ],
      "offers_total": 3
    }
  },
  "note": "…",
  "fetched_at": "2026-10-06T12:37:31.000Z"
}
```

### MCP

The MCP server has two tools for these routes: `list_gpus` (same as `/gpus`) and `gpu_rentals` (same parameters as `/gpu-rentals`, plus `detail`: `compact`, the default, returns the 3 cheapest offers of each tier; `full` returns up to 50).

## Self-host or API

`GET /self-host?model=<id>` answers: is it cheaper to host this open-weights model yourself on rented GPUs, or to use the cheapest API offer? The answer is an estimate, with hardware rental only (no deployment, monitoring, redundancy, storage or network costs). How it is built, level of evidence by level of evidence: [How the answer is built](self-host.md).

```
GET https://api.inferindex.dev/self-host?model=qwen/qwen3-32b&tokens_per_day=50000000
```

Parameters:

| Parameter | Default | Meaning |
|---|---|---|
| `model` | required | Id from `/models`. A model not in the catalogue returns a 404 with suggestions. |
| `tokens_per_day` | none | Your volume, input and output together. Or `requests_per_day` with `prompt_tokens` and `output_tokens` (not both). Without a volume you still get the verdict and the break-even volume. |
| `utilization` | 50 | 1 to 100: percent of the time the GPUs serve requests. |
| `tokens_per_second` | none | Your own measured throughput for the whole configuration, input and output tokens together, instead of our estimate. 1 to 1,000,000. |
| `weights_margin` | 70 | 50 to 95, whole percent of the GPU memory the model weights may fill; the rest is the engine's reserve and the context cache. |
| `gpu`, `gpu_count` | none | Impose a card (id from `/gpus`) and/or a number of GPUs (1, 2, 4, 8 or 16). |
| `quantization` | `fp8` | `fp16`, `bf16`, `fp8`, `int8`, `fp4`, `int4`. Must be published by at least one provider unless `fp16` or `bf16`. |
| `tier` | `guaranteed` | `guaranteed`, `community` or `spot`. Tiers are never compared with each other. |
| `detail` | `full` | `compact`: verdict, headline and key numbers. `full`: also the model, the three throughput scenarios and every assumption. (The MCP tool defaults to `compact`.) |

A parameter out of range returns a 400 with the name of the parameter.

### Response

Fields in **both** views (`detail=compact` and `detail=full`):

| Field | Meaning |
|---|---|
| `status` | `ok`, or `refused` (below). |
| `verdict` | `api_cheaper`, `self_host_cheaper_above`, `self_host_above_utilization` or `self_host_needs_throughput`. |
| `headline` | The answer in one sentence. When the evidence is not enough, it gives the break-even volume and the throughput required rather than "it depends". |
| `headline_kind` | Which sentence the headline is, as a stable value, so that a client can compose its own wording or a translation from fields instead of parsing the English sentence. One of the 16 kinds listed in [How the answer is built](self-host.md#the-headline-as-fields). Kinds may be added: a client that does not know a kind should show `headline`. |
| `headline_values` | Every value the headline quotes, rounded as the sentence quotes it; only the keys the sentence uses are present. Units: keys ending in `_tps` are tokens per second (`floor_output_tps` counts output tokens, the others input and output together); `volume_m`, `break_even_m` and `usable_capacity_m_low`/`_high` are million tokens a day; `api_usd_per_m` and `self_hosted_usd_per_m_low`/`_high` are USD per million tokens; keys ending in `_percent` are percent. Thousands and decimal separators are left to the client. Also: `prefix`, `at_break_even`, `mix` (`default_3_1` or `caller`), `request_length` (`not_given` or `within_benchmark`), `api_provider`, `gpu_name`, and the optional parts `floor_note` and `comparison` described in the page above. `gpu_name` and `api_provider` come from third-party pages: escape them before display. |
| `confidence` | How the throughput was obtained: `measured`, `derived`, `estimated`, `lower_bound`, `user_supplied`, `none` (unknown: no throughput is known for this configuration and the verdict would need one) or `not_needed` (the verdict does not rest on any throughput). See [How the answer is built](self-host.md#2-levels-of-evidence). |
| `threshold_m_tokens_per_day` | Break-even volume in million tokens a day: daily cost of the GPU configuration divided by the API price per million tokens. `null` when the API offer is free (`threshold_note` says so). |
| `required_tokens_per_second` | Minimum throughput, in input plus output tokens per second, that the whole configuration (all GPUs together) must deliver while serving for self-hosting to cost less than the API: `at_full_utilization` if it were busy all the time, `at_utilization` at `utilization_percent`. With a volume, `for_your_volume` gives the same two figures for serving that volume itself. `null` when the API offer is free. |
| `break_even_utilization_percent` | Per scenario (prudent, median, optimistic): the average utilization above which self-hosting costs less, whatever the number of GPUs. Above 100 means never at that throughput. `null` when there are no scenarios: no throughput is known at all (`confidence` `none`), the verdict needs none (`not_needed`), or only a published minimum exists. When the API offer is free and scenarios exist, it is an object whose three values are `null`: there is no break-even. Absent from a refusal. |
| `utilization_percent` | The utilization used. |
| `api` | The API offer compared: `provider`, `usd_per_m_tokens`, `checked_at`, `offers_compared`, `mix`, `basis` ("cheapest of N current standard offers…, promotions and unstable prices excluded"). |
| `configuration` | The GPU configuration. **Compact view:** `gpu`, `gpus`, `provider`, `tier`, `usd_per_hour`. **Full view:** `vendor`, `gpu_id`, `gpu_name`, `gpus_needed`, `gpus_billed`, `provider`, `region`, `tier`, hourly, per-GPU and daily prices, `vram_total_gb`, `weights_share`, `checked_at`, `source_url`, `throughput_evidence` (what is known about the throughput of this configuration; it can differ from `confidence`, which says what the verdict rests on), `engine_note` when we have no throughput for this model on this card, and, when we picked a configuration that is not the cheapest, `cheapest_alternative` and `selection_reason`. |
| `fetched_at` | When the answer was built. |
| `settled_alternative` | Present only when the configuration the answer is about has no verdict that published figures can decide (`self_host_needs_throughput`) and a dearer rented configuration has one: the cheapest such setup (card, number of GPUs, provider, tier, hourly and daily price with `checked_at` and `source_url`), its `verdict` (`self_host_cheaper_above`, or `self_host_above_utilization` with `busy_at_least_percent`, the share of the time it must be busy: for a measured value the break-even utilization of the prudent scenario, for a published minimum a utilization at which the floor is enough), the `confidence` it rests on (`measured` or `lower_bound`) and its break-even volume. It is another, dearer setup, shown for comparison; it is not a recommendation and it is never used in place of the configuration the answer is about. The verdict summarises the prudent case: asking for that configuration directly can return `self_host_needs_throughput`. An alternative resting on a published minimum holds for requests of up to 2,048 tokens in total. With a volume, it is shown only if that setup is cheaper at your volume and can serve it; the "busy at least" form exists only without a volume. Absent when you impose a card (`gpu`) or give `tokens_per_second`; with `gpu_count` alone, only configurations of that size are considered. |

Only in the **compact** view:

| Field | Meaning |
|---|---|
| `weights_margin_percent` | The weights margin used. |
| `note` | "Hardware rental only: the hourly price the provider publishes for the machine. It does not count the engineering to deploy and run the model, monitoring, redundancy, start-up time, or storage and network costs billed separately." |

Only in the **full** view:

| Field | Meaning |
|---|---|
| `model` | `id`, `name`, `params_b`, `kind` (`dense`, `moe` or `unknown`), `active_params_b`, `weights_gb`, `quantization`. |
| `weights_margin` | `used_percent`, `default_percent` and an `explanation`. |
| `throughput` | For a published minimum (`status: "lower_bound"`): `published_output_tokens_per_second`, `at_least_tokens_per_second`, `sufficient_utilization_percent`, `conversion`, `source` ("InferenceX (SemiAnalysis)"), `engine` (`vllm`, `sglang`, `trtllm`, `other` or `null`), `reference.source_url`, and `other_engine_ceiling` (`engine`, `output_tokens_per_second`) when another series on the same configuration (another engine, another run of the same engine, or an engine we cannot name) levelled off (our reading) at a lower figure: that is the ceiling observed for that series, not the capacity of the configuration, and it is not used; `engine` has the same possible values there. Otherwise the method and the reference behind the estimate, or `status: "unavailable"` with a `reason`. |
| `scenarios` | Prudent, median, optimistic: throughput, capacity, break-even utilization, cost per million tokens, and (with a volume) what happens at your volume. `null` without a throughput estimate. |
| `volume_m_tokens_per_day` | Your volume in million tokens a day, `null` without one. |
| `assumptions` | Every assumption in words, including the hardware-rental sentence above. |

### Refusals

We decline to give a number rather than an unreliable one. A refusal is a 200 response: `status: "refused"`, `verdict: null`, a `reason`, a `message` and a `detail`.
`reason` is one of `model_closed_or_unsized`, `quantization_not_published`, `unknown_gpu`, `gpu_not_rented`, `weights_exceed_margin`, `no_rental_configuration`, `gpu_memory_unknown`, `no_api_price`; the meaning of each is in [How the answer is built](self-host.md#5-when-we-decline-to-answer).

### Examples

No throughput is known for this model on the cheapest configuration: the answer gives the break-even volume and the throughput to compare with.

```bash
curl "https://api.inferindex.dev/self-host?model=qwen/qwen3.8-27b&detail=compact"
```

_Example response as of 2026-10-06 — prices, providers and statuses change._

```json
{
  "status": "ok",
  "verdict": "self_host_needs_throughput",
  "headline": "The API is cheaper below 57 million tokens a day. Above that, self-hosting wins only if the whole configuration (all GPUs together) delivers at least 1,319 tokens per second, input and output together, while serving at 50% utilization (660 if it were busy all the time).",
  "headline_kind": "break_even_and_required_throughput",
  "headline_values": {
    "prefix": null,
    "break_even_m": 57,
    "required_tps_at_utilization": 1319,
    "utilization_percent": 50,
    "required_tps_if_always_busy": 660
  },
  "confidence": "none",
  "threshold_m_tokens_per_day": 56.95,
  "required_tokens_per_second": {
    "at_full_utilization": 659.2,
    "at_utilization": 1318.3,
    "utilization_percent": 50
  },
  "break_even_utilization_percent": null,
  "utilization_percent": 50,
  "api": {
    "provider": "Consensusprotocol",
    "usd_per_m_tokens": 0.1475,
    "checked_at": "2026-10-06T11:21:25.933Z",
    "offers_compared": 56,
    "mix": {
      "source": "default_3_1",
      "input_share_percent": 75
    },
    "basis": "cheapest of 56 current standard offers for a 3:1 input:output mix (blended), promotions and unstable prices excluded"
  },
  "configuration": {
    "gpu": "RTX A6000 48GB",
    "gpus": 1,
    "provider": "Thunder Compute",
    "tier": "guaranteed",
    "usd_per_hour": 0.35
  },
  "weights_margin_percent": 70,
  "note": "Hardware rental only: the hourly price the provider publishes for the machine. It does not count the engineering to deploy and run the model, monitoring, redundancy, start-up time, or storage and network costs billed separately.",
  "fetched_at": "2026-10-06T12:37:31.000Z"
}
```

A published minimum settles it (a configuration is imposed here):

```bash
curl "https://api.inferindex.dev/self-host?model=minimax/minimax-m2.5&gpu=b200-sxm-180gb&gpu_count=4&detail=compact"
```

_Example response as of 2026-10-06 — prices, providers and statuses change._

```json
{
  "status": "ok",
  "verdict": "self_host_cheaper_above",
  "headline": "Self-hosting is cheaper above 1,671 million tokens a day at 50% utilization (API $0.39/M at a 3:1 input:output mix): the InferenceX benchmark (SemiAnalysis) gives at least 20,417 output tokens per second on this setup, which we convert to at least 40,835 tokens per second, input and output together, for a 3:1 input:output mix; this floor holds for requests of up to 2,048 tokens in total (the benchmark used 1,024 prompt tokens and 1,024 answer tokens).",
  "headline_kind": "published_minimum_settles",
  "headline_values": {
    "prefix": null,
    "utilization_percent": 50,
    "break_even_m": 1671,
    "api_usd_per_m": 0.39,
    "mix": "default_3_1",
    "floor_output_tps": 20417,
    "floor_total_tps": 40835,
    "request_length": "not_given"
  },
  "confidence": "lower_bound",
  "threshold_m_tokens_per_day": 1671.38,
  "required_tokens_per_second": {
    "at_full_utilization": 19344.8,
    "at_utilization": 38689.5,
    "utilization_percent": 50
  },
  "break_even_utilization_percent": null,
  "utilization_percent": 50,
  "api": {
    "provider": "Glama",
    "usd_per_m_tokens": 0.39,
    "checked_at": "2026-10-06T10:52:24.527Z",
    "offers_compared": 45,
    "mix": {
      "source": "default_3_1",
      "input_share_percent": 75
    },
    "basis": "cheapest of 45 current standard offers for a 3:1 input:output mix (blended), promotions and unstable prices excluded"
  },
  "configuration": {
    "gpu": "B200 SXM 180GB",
    "gpus": 4,
    "provider": "Lambda",
    "tier": "guaranteed",
    "usd_per_hour": 27.16
  },
  "weights_margin_percent": 70,
  "note": "Hardware rental only: the hourly price the provider publishes for the machine. It does not count the engineering to deploy and run the model, monitoring, redundancy, start-up time, or storage and network costs billed separately.",
  "fetched_at": "2026-10-06T12:37:31.000Z"
}
```

Without an imposed card, a dearer setup that published figures decide can be pointed out for comparison (`settled_alternative`, and the last sentence of the headline):

```bash
curl "https://api.inferindex.dev/self-host?model=minimax/minimax-m2.5&detail=compact"
```

_Example response as of 2026-10-06 — prices, providers and statuses change._

```json
{
  "status": "ok",
  "verdict": "self_host_needs_throughput",
  "headline": "The API is cheaper below 615 million tokens a day. Above that, self-hosting wins only if the whole configuration (all GPUs together) delivers at least 14,246 tokens per second, input and output together, while serving at 50% utilization (7,123 if it were busy all the time). For comparison, published figures decide it for a dearer setup: 4x B200 SXM 180GB at $652 a day, self-hosting cheaper above 1,671 million tokens a day (published minimum, for requests of up to 2,048 tokens in total).",
  "headline_kind": "break_even_and_required_throughput",
  "headline_values": {
    "prefix": null,
    "break_even_m": 615,
    "required_tps_at_utilization": 14246,
    "utilization_percent": 50,
    "required_tps_if_always_busy": 7123,
    "comparison": {
      "gpus": 4,
      "gpu_name": "B200 SXM 180GB",
      "usd_per_day": 652,
      "break_even_m": 1671,
      "busy_at_least_percent": null,
      "evidence": "published_minimum",
      "request_length": "not_given"
    }
  },
  "confidence": "none",
  "threshold_m_tokens_per_day": 615.38,
  "required_tokens_per_second": {
    "at_full_utilization": 7122.6,
    "at_utilization": 14245.1,
    "utilization_percent": 50
  },
  "settled_alternative": {
    "gpu_id": "b200-sxm-180gb",
    "gpu_name": "B200 SXM 180GB",
    "gpus": 4,
    "provider": "Lambda",
    "tier": "guaranteed",
    "usd_per_hour": 27.16,
    "usd_per_day": 651.84,
    "verdict": "self_host_cheaper_above",
    "confidence": "lower_bound",
    "threshold_m_tokens_per_day": 1671.38,
    "busy_at_least_percent": null,
    "checked_at": "2026-10-06T10:15:58.151Z",
    "source_url": "https://lambda.ai/pricing"
  },
  "break_even_utilization_percent": null,
  "utilization_percent": 50,
  "api": {
    "provider": "Glama",
    "usd_per_m_tokens": 0.39,
    "checked_at": "2026-10-06T10:52:24.527Z",
    "offers_compared": 45,
    "mix": {
      "source": "default_3_1",
      "input_share_percent": 75
    },
    "basis": "cheapest of 45 current standard offers for a 3:1 input:output mix (blended), promotions and unstable prices excluded"
  },
  "configuration": {
    "gpu": "L40 48GB",
    "gpus": 8,
    "provider": "CoreWeave",
    "tier": "guaranteed",
    "usd_per_hour": 10
  },
  "weights_margin_percent": 70,
  "note": "Hardware rental only: the hourly price the provider publishes for the machine. It does not count the engineering to deploy and run the model, monitoring, redundancy, start-up time, or storage and network costs billed separately.",
  "fetched_at": "2026-10-06T12:37:31.000Z"
}
```

A refusal:

```bash
curl "https://api.inferindex.dev/self-host?model=google/gemma-4-31b-it&gpu=rtx-4090-24gb&quantization=fp16"
```

_Example response as of 2026-10-06 — prices, providers and statuses change._

```json
{
  "status": "refused",
  "verdict": null,
  "reason": "weights_exceed_margin",
  "message": "The weights (62.6 GB in fp16) do not fit within 70% of the memory of the requested configuration.",
  "detail": {
    "weights_gb": 62.55,
    "quantization": "fp16",
    "memory_needed_gb": 89.4,
    "memory_available_gb": 24,
    "weights_margin_percent": 70,
    "smallest_configuration_that_fits": {
      "gpu_id": "rtx-pro-6000-96gb",
      "gpu_name": "RTX PRO 6000 96GB",
      "gpus": 1,
      "provider": "Nebius",
      "usd_per_hour": 1.8,
      "tier": "guaranteed"
    }
  },
  "fetched_at": "2026-10-06T12:37:31.000Z"
}
```

### MCP

The MCP server exposes the same answer as the tool `self_host_or_api` (same parameters, `detail` defaults to `compact`).

### Related routes

`/gpus` lists the GPU cards we know (the ids for the `gpu` parameter); `/gpu-rentals` lists the rental prices behind the configurations. Both are described in [GPU rental prices](#gpu-rental-prices).

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
