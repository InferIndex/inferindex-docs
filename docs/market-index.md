# InferIndex market index — methodology

*Version 1.1, adopted 2026-09-22, fixed before the first publication on Monday 2026-10-05.*
[Version française](market-index.fr.md)

The InferIndex market index tracks what one million LLM tokens cost, week after week, for each family of models.
This page describes exactly how it is computed, so that every published value can be understood and checked.

## What is measured

- **Unit**: the price of one million tokens, in US dollars, **blended** 3:1 — three input tokens for one output
  token: `(3 × input + output) / 4`.
- **Grouping**: one value per **model family** (see below).
- **Frequency**: published every **Monday**, at the value of **00:00 UTC** that day. First publication:
  **2026-10-05**.
- **Data**: InferIndex's own price records only, collected since tracking started on **2026-09-14**. No data from
  before that date, and no data from any other index or dataset, goes into it. Every value can be reproduced from
  our records.

## Two series, never mixed

1. **Market series** (`series=market`): built from the offers we collect from providers and resellers — what you
   actually pay for a model.
2. **Official series** (`series=official`): built from the **list prices** we record on the labs' own pricing pages.
   - An announced discount is not a list price: the undiscounted price is used.
   - Only the standard service level counts.
   - When a lab publishes prices for several regions, the cheapest region is used.

Both series start on 2026-09-14 and use the same families and the same minimum of 3 models per family. They answer different
questions — "what does the market charge?" and "what do the labs list?" — and are never combined.

## Reference price of a model (market series)

We take the offers **in force at the moment of calculation** (the same rule as `/history?at=`), then **exclude**:

- offers on promotion, announced or probable, including promotions that have ended;
- prices converted from points or credits;
- a gateway's single price for several unnamed backends;
- offers that are stale at that moment (not re-checked for more than three times their source's collection
  interval, six hours at least);
- offers flagged as resold below the lab's own list price (`below_official_list`);
- non-standard service levels (flex, batch, priority);
- model ids that the lab has redirected to another model;
- unofficial resellers of access to closed models, which InferIndex never lists;
- newly added models still under review at the time of calculation;
- prices whose currency has no exchange rate.

For a closed model, every reseller counts, not only the lab.

We then keep **one price per provider** (its lowest), and the model's reference price is the **median of the three
cheapest providers**. **A model with fewer than 3 providers has no reference price.**

Prices in another currency are converted to USD at the latest ECB reference rate published before the measured
instant (dated no later than the day before; for a Monday 00:00 UTC publication, the previous Friday's rate).

In the official series, a model's reference is its lab list price (see *Two series*): undiscounted, standard
service level, cheapest region. If the lab's pricing page can no longer be read, its last list price counts for 14
days at most; after that, the model leaves the series until the page is read again. Market-series exclusions and the 3-provider minimum don't apply. Models under review
are excluded in both series.

## Families

Each model is placed in one family by a written rule, applied in this order:

1. **Special use** (moderation, embeddings, reranking, speech, OCR): outside the index.
2. **Code** (`code`): the model's name designates a code model (coder, codestral, devstral, codex, "-code"), open or
   closed.
3. **Closed models** — models whose weights the lab does not publish, from a written list (for example Claude, GPT,
   Gemini, Grok, Qwen Max and Plus, Amazon Nova, Mistral Medium 3). Each entry is backed by the model's official page
   (the model is served through an API) and by the model's absence from the lab's official Hugging Face
   organization. A model whose status is uncertain stays unclassified:
   - **Compact closed** (`compact_closed`): a closed model is compact when its name carries one of the lab's
     compact tier words, or when the lab itself presents it as lightweight or compares it with compact models — never
     based on its price. See [Compact closed models](#compact-closed-models);
   - **Frontier closed** (`frontier_closed`): all other closed models.
4. **Open-weight models**, by the total parameter count shown on the model's Hugging Face page (recorded, dated
   and sourced by us):
   - **Large** (`open_large`): 200 billion parameters or more;
   - **Medium** (`open_medium`): 40 to 200 billion;
   - **Small** (`open_small`): under 40 billion.
5. **Otherwise, unclassified**: outside the index and listed as such — we never guess a family. This covers, for
   example, an open model whose size isn't recorded yet, or a model whose weights status isn't established.

### Compact closed models

**(A) Tier words in the name**: mini, nano, micro, lite, flash (including flash-lite and flashx), haiku, luna, air,
small, tiny. "turbo" is not one of them: GPT-4 Turbo was a top-of-range model.

**(B) The lab's own written positioning**, for models whose name carries none of these words:

| Model | What the lab writes | Source, read on |
|---|---|---|
| Perplexity Sonar | "Lightweight, cost-effective search model with grounding" | docs.perplexity.ai, 2026-09-22 |
| Inception Mercury 2.5 | "Comparable to cost-optimized frontier models like GPT-5.6 Luna (Low), Gemini 3.5 Flash-Lite, and Claude Haiku 4.5" (launch post) | inceptionlabs.ai, 2026-09-22 |

Reasoning is not a family: most recent models are hybrid, so a separate family would be neither stable nor
exclusive.

Parameter counts of new open models are recorded at each Monday calculation.

## Value of a family

A family's value is the **unweighted median of its models' reference prices**. **A family with fewer than 3 models
has no value that week** (`value: null`).

**Why no weighting**: we don't observe the volumes actually consumed, so any weighting would be an assumption. The
median is robust to extreme cases and easy to check. Every publication lists the models counted in each family, so
anyone can redo the calculation.

## Limits

- **Not a constant-basket index.** A family's composition changes when a model arrives, disappears, or doesn't
  have enough providers. Each publication gives the number of models counted, and a week-to-week change can come
  from a change in composition as much as from a change in prices.
- The index reflects list and offer prices as published, not negotiated or volume discounts.
- Prices come from public information and can contain errors; see the
  [terms of use](https://api.inferindex.dev/terms).

## Versions

The method is fixed before the first publication. Any later change gets a new version number, with its date and
reason, in the history below. Each publication states the `method_version` it was computed with. **Published values
are never silently recomputed.**

**Corrections.** A published week is never silently recomputed. If a data error is fixed after publication, the week
is recomputed through an explicit correction with a mandatory reason: every family whose value or composition changes
keeps its previous value, and `GET /index` lists the corrections for that date (previous value, new value, reason,
date of the correction).

| Version | Date | Change |
|---|---|---|
| 1.0 | 2026-09-18 | First version |
| 1.1 | 2026-09-22 | Written definition of "compact" for closed models (lab tier name or the lab's own written positioning, never price); three models move to compact: GLM-5.3 FlashX, Perplexity Sonar, Inception Mercury 2.5. In the official series, a list price that can no longer be read counts for 14 days at most. Applies from the first publication |

## Access

`GET https://api.inferindex.dev/index?series=market` (or `official`), optionally `&at=YYYY-MM-DD` for a past
Monday. See [`GET /index`](api.md#get-index) in the API reference.
