# Examples

Every command below is a plain `curl` against the public API, no key needed. Responses are abridged and dated; prices change.

## 1. What does a model cost right now, and who sells it?

```bash
curl "https://api.inferindex.dev/cheapest?model=meta-llama/llama-3.3-70b-instruct&limit=3"
```

_Response as of 2026-10-16, abridged._

```json
{
  "model": { "id": "meta-llama/llama-3.3-70b-instruct", "name": "Meta: Llama 3.3 70B Instruct" },
  "offers_total": 41,
  "cheapest": {
    "provider": "NanoGPT",
    "via": "nanogpt",
    "via_name": "NanoGPT",
    "input_per_1M": 0.05,
    "output_per_1M": 0.23,
    "blended_per_1M": 0.095,
    "promo": false,
    "checked_at": "2026-10-16T07:40:37.737Z"
  }
}
```

Read: `blended_per_1M` is the price at a 3:1 input:output mix, and `via_name` the seller when the offer is sold through an aggregator. `promo` flags a drop or a promotion (`false` here; `probable` is a drop we saw, not announced by the seller) and `checked_at` is when the price was last confirmed.

## 2. What would my workload cost per month?

```bash
curl "https://api.inferindex.dev/cheapest?model=openai/gpt-oss-120b&prompt_tokens=2000&output_tokens=500&requests_per_day=1000&limit=3"
```

_Response as of 2026-10-16, abridged._

```json
{
  "model": { "id": "openai/gpt-oss-120b", "name": "OpenAI: gpt-oss-120b" },
  "usage": { "prompt_tokens": 2000, "output_tokens": 500, "cached_ratio": 0, "requests_per_day": 1000 },
  "cheapest": {
    "provider": "Runware",
    "via_name": "LLM Gateway",
    "input_per_1M": 0.032,
    "output_per_1M": 0.14,
    "estimated_cost_per_request": 0.000134,
    "estimated_cost_per_month": 4.02,
    "estimate_approx": false,
    "estimate_notes": ["context_unknown", "max_output_unknown"]
  }
}
```

Read: `estimated_cost_per_month` covers 30 days at the stated workload, for the offer that is cheapest for that workload; `estimate_approx` is `true` when the source does not fully publish its tiers or cache price; `estimate_notes` lists caveats such as `context_unknown`.

## 3. Which of two models is cheaper for the same workload?

There is no route that compares two models. Call `/cheapest` once per model with the same workload and compare `estimated_cost_per_month`:

```bash
curl "https://api.inferindex.dev/cheapest?model=meta-llama/llama-3.3-70b-instruct&prompt_tokens=2000&output_tokens=500&requests_per_day=1000&limit=1"
curl "https://api.inferindex.dev/cheapest?model=openai/gpt-oss-120b&prompt_tokens=2000&output_tokens=500&requests_per_day=1000&limit=1"
```

_Responses as of 2026-10-16, abridged._

```json
{ "model": { "id": "meta-llama/llama-3.3-70b-instruct" }, "cheapest": { "provider": "NanoGPT", "via_name": "NanoGPT", "estimated_cost_per_month": 6.45 } }
{ "model": { "id": "openai/gpt-oss-120b" },    "cheapest": { "provider": "Runware",  "via_name": "LLM Gateway", "estimated_cost_per_month": 4.02 } }
```

Read: the two figures are comparable because the workload is the same; the website's comparison page shows them side by side. No route lists every offer of one provider either: the provider pages of the website do.

## 4. Has the price changed?

```bash
curl "https://api.inferindex.dev/history?model=deepseek/deepseek-v3.2&days=7&granularity=day&provider=DeepInfra"
```

_Response as of 2026-10-16, first two points of sixteen._

```json
{
  "model": { "id": "deepseek/deepseek-v3.2" },
  "days": 7,
  "granularity": "day",
  "history": [
    { "period": "2026-10-09", "provider": "DeepInfra", "quantization": "fp4", "variant": "", "input_min": 0.26, "input_max": 0.26, "input_last": 0.26, "output_min": 0.38, "output_max": 0.38, "output_last": 0.38 },
    { "period": "2026-10-09", "provider": "DeepInfra", "quantization": "fp4", "variant": "flex", "input_min": 0.208, "input_max": 0.208, "input_last": 0.208, "output_min": 0.304, "output_max": 0.304, "output_last": 0.304 }
  ]
}
```

Read: one point per day and per offer; equal `min`, `max` and `last` mean the price did not move that day. `variant: "flex"` is a lower-priority tier, not the standard price.

## 5. Who offers it in the EU, without training on my prompts?

```bash
curl "https://api.inferindex.dev/resellers?model=meta-llama/llama-3.3-70b-instruct&region=eu&no_training=true&strict=true&limit=3"
```

_Response as of 2026-10-16, abridged._

```json
{
  "model": { "id": "meta-llama/llama-3.3-70b-instruct" },
  "total_offers": 4,
  "filters_unknown": { "region": 53, "no_training": 26, "quantization": 9 },
  "excluded": { "region_unknown_strict": 54, "region_mismatch": 8, "training_unknown_strict": 26, "training_not_excluded": 6 },
  "offers": [
    { "provider": "CloudFerro", "blended_per_1M": 0.4763, "conditions": { "provider": { "regions": { "value": "EU (Poland)" }, "training_on_prompts": { "value": "no" } } } },
    { "provider": "IONOS",      "blended_per_1M": 0.7285, "conditions": { "provider": { "regions": { "value": "Germany (EU)" }, "training_on_prompts": { "value": "no" } } } }
  ]
}
```

Read: without `strict=true`, offers whose region or policy is not published are kept and counted in `filters_unknown`; "not published" is not "no". With `strict=true` they are excluded, which is why the list is short; `excluded` says how many fell out and why.

## 6. Rent a GPU or pay the API?

```bash
curl "https://api.inferindex.dev/self-host?model=qwen/qwen3-32b&tokens_per_day=50000000"
```

_Response as of 2026-10-16, abridged._

```json
{
  "status": "ok",
  "verdict": "api_cheaper",
  "headline": "At 50 million tokens a day, the API is cheaper whatever the throughput ($0.123/M at a 3:1 input:output mix; self-hosting breaks even at 68.6 million tokens a day).",
  "threshold_m_tokens_per_day": 68.57,
  "required_tokens_per_second": { "at_utilization": 1587.4, "utilization_percent": 50 },
  "api": { "provider": "Nscale", "usd_per_m_tokens": 0.1225, "offers_compared": 21 },
  "assumptions": ["Hardware rental only: the hourly price the provider publishes for the machine. It does not count the engineering to deploy and run the model, monitoring, …", "…"],
  "configuration": { "gpu_name": "RTX A6000 48GB", "gpus_needed": 1, "provider": "Thunder Compute", "usd_per_day": 8.4, "throughput_evidence": "none" }
}
```

Read: `threshold_m_tokens_per_day` is the volume at which a rented configuration costs the same as the cheapest API; above it, renting wins only if the setup sustains `required_tokens_per_second`, a condition and not a promise. The figures cover hardware rental only; `assumptions` lists what is left out.

## 7. Where does the market stand, by model family?

```bash
curl "https://api.inferindex.dev/index?series=market"
```

_Response for the publication of 2026-10-12, abridged._

```json
{
  "index": "InferIndex market index",
  "series": "market",
  "published_at": "2026-10-12T00:00:00.000Z",
  "method_version": "1.2",
  "unit": "USD per 1M tokens, blended 3:1 (input:output)",
  "compared_to": "2026-10-05T00:00:00.000Z",
  "families": [
    { "family": "open_small", "value": 0.125, "models": 54, "change_on_same_models_pct": 0, "same_models": 54 },
    { "family": "open_medium", "value": 0.2953, "models": 30, "change_on_same_models_pct": 2.16, "same_models": 30 }
  ],
  "corrections": []
}
```

Read: each family's `value` is the median blended price of the models counted; `change_on_same_models_pct` compares only the models counted in both publications, so a model entering or leaving does not move it. `corrections` lists any value corrected after publication, with the reason (empty here).

## 8. Where does a model stand in the public leaderboards?

```bash
curl "https://api.inferindex.dev/rankings?model=deepseek/deepseek-v3.2"
```

_Response as of 2026-10-16, abridged._

```json
{
  "model": { "id": "deepseek/deepseek-v3.2" },
  "rankings": [
    { "source": "Arena", "bench": "text_style_control", "listed_as": "deepseek-v3.2", "rank": 123, "out_of": 415, "score": 1424.67, "votes": 48134, "published_on": "2026-10-13",
      "scale": { "score_name": "Arena Score", "count_name": "votes" } }
  ],
  "attribution": { "license": "CC BY 4.0", "license_url": "https://creativecommons.org/licenses/by/4.0/" }
}
```

Read: the rank and score are the source's, never computed by us; `scale` says what the score is called and what the count is. Keep the `attribution` object when you reuse the data: it is licensed under CC BY 4.0.

## With an assistant (MCP)

Once the server is connected (`https://mcp.inferindex.dev/mcp`), ask in plain words; the assistant picks the tool.

- _"What is the cheapest way to call DeepSeek V3.2 right now?"_ → `cheapest`: the price, the seller, the date it was checked, and a flag when the price is a promotion.
- _"I send about 2,000 tokens in and get 500 out, 1,000 times a day, on gpt-oss-120b. What will it cost per month?"_ → `estimate_cost`: cost per request and per month, with the assumptions repeated.
- _"Is it cheaper to self-host Qwen3 32B than to use an API?"_ → `self_host_or_api`: the break-even volume and the throughput needed as a condition; hardware rental only.
