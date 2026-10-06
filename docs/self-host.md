# Self-host or API? How the answer is built

InferIndex can answer one question for an open-weights model in our catalogue: **is it cheaper to rent GPUs and run the model yourself, or to use the cheapest API offer?**
The answer is an estimate built from rules we set ourselves. This page says what the answer always contains, what each level of evidence means, which settings we use by default and when we decline to answer. Every default below is our choice, not a technical requirement, and you can change most of them.

> **Hardware rental only: the hourly price the provider publishes for the machine.** It does not count the engineering to deploy and run the model, monitoring, redundancy, start-up time, or storage and network costs billed separately.

Endpoint reference: [API reference](api.md#self-host-or-api). MCP tool: `self_host_or_api`.

## 1. What the answer always contains

| What | Field | Meaning |
|---|---|---|
| A one-sentence verdict | `headline`, `verdict` | One of `api_cheaper`, `self_host_cheaper_above`, `self_host_above_utilization`, `self_host_needs_throughput`. |
| The break-even volume | `threshold_m_tokens_per_day` | Daily cost of the GPU configuration divided by the API price per million tokens: the volume, in million tokens a day, above which self-hosting costs less (provided the setup can serve it). |
| The throughput self-hosting needs | `required_tokens_per_second` | The minimum throughput, in input plus output tokens per second, that the whole configuration (all GPUs together) must deliver while serving for self-hosting to cost less than the API. `at_full_utilization` if it were busy all the time, `at_utilization` at the utilization you gave. When you give a volume, `for_your_volume` gives the same two figures for serving that volume itself rather than the break-even volume. |
| What published measurements show | `throughput`, `confidence` | How the throughput was obtained (see section 2) and, for a published minimum, the figures behind it. |
| The price used | `api`, `configuration` | The API offer compared (provider, price per million tokens at a 3:1 input:output mix unless you describe yours, date checked, number of offers compared, "cheapest of N current standard offers, promotions and unstable prices excluded") and the GPU configuration (card, number of GPUs, provider, tier, hourly price, date checked). |

The four verdicts:

- `api_cheaper`: the API is cheaper at any volume, or, when you give a volume under the break-even volume, at that volume whatever the throughput. It is also the verdict when the cheapest API offer is free.
- `self_host_cheaper_above`: self-hosting is cheaper above the break-even volume, at your utilization.
- `self_host_above_utilization`: at your utilization the API is cheaper, but self-hosting wins above the break-even utilization.
- `self_host_needs_throughput`: the throughput scenarios disagree, no throughput is known for this model on this card, or only a published minimum exists and it is not enough to decide. The headline then states the break-even volume and the throughput required, so you can compare it with what your setup delivers.

We never answer "it depends" without saying on what: when the evidence is not enough, the headline gives the number to compare against.

### The headline as fields

The headline exists only in English. To let a client write its own sentence (or a translation) without parsing ours, the response also gives `headline_kind`, which names the sentence, and `headline_values`, every value the sentence quotes, rounded as the sentence quotes it. Only the keys the sentence uses are present, and thousands and decimal separators are left to the client. Kinds may be added: a client that does not know a kind should show `headline`.

| `headline_kind` | When |
|---|---|
| `api_free` | The cheapest API offer is free. |
| `volume_at_or_under_break_even` | A volume at or under the break-even volume: the API is cheaper (or self-hosting is no cheaper) whatever the throughput. |
| `volume_needs_throughput` | A volume above the break-even volume, and no throughput is known: the sentence gives the throughput required. |
| `volume_api_cheaper` | A volume, scenarios exist, and the API wins. |
| `volume_self_host_cheaper` | A volume, scenarios exist, and self-hosting wins. |
| `volume_exceeds_capacity` | A volume that no scenario can serve on one configuration. |
| `volume_only_at_higher_estimates` | A volume that self-hosting serves cheaper only at our higher throughput estimates. |
| `api_cheaper_any_volume` | No volume: the API wins at any volume. |
| `self_host_cheaper_above` | No volume: self-hosting is cheaper above the break-even volume. |
| `self_host_above_utilization` | No volume: self-hosting wins above a utilization level. |
| `break_even_and_required_throughput` | No volume, and no usable throughput: the sentence gives the break-even volume and the throughput required. |
| `busy_at_least_higher_estimates_only` | The scenarios disagree, and at our lowest estimate even constant use would not pay off. |
| `busy_at_least_all_estimates` | The scenarios disagree, but the occupancy needed is the same once rounded. |
| `busy_at_least_between_estimates` | The scenarios disagree: two occupancy levels, one per end of our range. |
| `published_minimum_settles` | No volume: a published minimum decides. |
| `published_minimum_settles_at_volume` | A volume: a published minimum decides. |

When the API price and the lowest self-hosted cost round to the same figure under `api_cheaper_any_volume`, the sentence quotes one more decimal for both, up to four, so that the two never look equal under a verdict of inequality (for example "$1.5/M … vs $1.504/M self-hosted").

Three parts can be added to any kind: `prefix` (`wide_estimate`, `estimate`, `your_throughput` or `null`: the words that open the sentence), `floor_note` (`does_not_settle` or `does_not_settle_longer_requests`, a published minimum that does not decide) and `comparison` (the settled alternative, as quoted). For a refusal the sentence is identified by `reason` and its values are in `detail`. The only texts that come from third-party pages are `gpu_name` and `api_provider`: escape them before display.

## 2. Levels of evidence

`confidence` says how the throughput of the chosen configuration was obtained. We use the words below in the response.

### Measured

A published figure exists for **this model, on this card, with this number of GPUs, at this precision**. We take the highest figure published for that configuration (best engine), unless another series (another engine, or another run of the same engine) has a published minimum above it on the same configuration: then the lower figure is only the ceiling observed for one series, it is not used, and the answer is a published minimum (below).
For figures read from InferenceX (SemiAnalysis) we keep one point per configuration: the highest output throughput of a series of measurements at different concurrency levels, taken from a single run, and only when throughput had stopped rising with concurrency. That is **our reading of the sweep**, not a label from the source. Our rule: a curve is one run with at least three concurrency levels; throughput is read as levelled off if the highest point is not the last of the series, or if the last step (concurrency multiplied by at least 1.5) brings less than 15% more throughput. Otherwise the figure is a published minimum (below).
A "measured" figure is the highest throughput published for this setup with the engines tested, a peak under benchmark conditions: another engine or tuning may do better, and an interactive service with latency targets sustains less. The response says so in `assumptions`.
Around the median we show a range of our own: the figure divided and multiplied by 1.25 for a measured value.

### Derived

An estimate scaled from a published figure: the same model on another card, another number of GPUs or another precision (scaled by memory bandwidth, GPU count and the ratio of bytes per parameter), or a similar model of the same type (scaled by memory bandwidth and active weights). Our range around the median is wider than for a measured figure: divided and multiplied by 2 (by 3 for `estimated`, below). The method sentence says what was scaled.
A card for which the rental listing does not give the form factor (SXM or PCIe) while the figure was measured on a specific form is also reported as derived, and the method names the form that was measured.
"Same model at a lighter precision" assumes the lighter weights are read faster, which is the optimistic case; the range is widened to include the published figure unscaled, that is, no change of speed.

### Published minimum (`lower_bound`)

When the only published figure comes from a series that was **still rising at its last point**, that figure is a **floor**, not the capacity: the machine can do at least that, probably more. We store and show it as "at least", with the confidence `lower_bound`.

A floor can only settle the question in one direction:

- If "at least this much" is already enough for self-hosting to cost less (at your utilization, or for the volume you give), the verdict is `self_host_cheaper_above` and the headline says so, naming the source.
- If it is not enough, we conclude nothing: the real capacity may be higher. The headline keeps its own sentence and adds that the floor does not settle it. A floor never produces `api_cheaper`, and when the volume is under the break-even volume the verdict is `api_cheaper` without relying on it.

If another series on the same configuration has a published minimum above the measured figure, the answer gives both: `throughput.engine` is the engine of the minimum, `throughput.other_engine_ceiling` (engine and output tokens per second) is the lower figure, and a line in `assumptions` names the two series: with two different known engines, "saw <engine A> level off at X output tokens per second (our reading), while <engine B> reached at least Y with throughput still rising"; when an engine is unknown, or both are the same, "has two series" without saying that they differ. It ends: "Neither figure is the capacity of the configuration, which is at least the higher one: we use only that figure, as a floor." The headline, which relies only on the floor, does not mention the other figure.

One more rule of ours applies to an estimate we keep: if the low end of its range would fall below the published minimum for the same setup, we raise the low end to that minimum, because the capacity is at least the minimum. The method sentence of the estimate then ends: "…; we raise the low end of our range to the published minimum for this setup, since the capacity is at least that".

If one of our own figures on the same configuration (an estimate, or a measured value without a recorded precision) comes out below the published floor, it is set aside, and the response says so in the same line of `assumptions`: "A figure of ours, estimated or measured without a recorded precision, that came out below this published floor is set aside: a published floor outranks it."

The words used, as of this writing (the exact sentence varies with your mix and with the request sizes; see the test condition below):

- Settles: "…: the InferenceX benchmark (SemiAnalysis) gives at least B output tokens per second on this setup, which we convert to at least C tokens per second, input and output together, for your mix."
- Does not settle: "A published benchmark (InferenceX, SemiAnalysis) gives a floor of C tokens per second, input and output together, for your mix; the real capacity may be higher, so this does not settle it."

Fields of the `throughput` block for a floor:

- `published_output_tokens_per_second` (B): what the benchmark measured, in output tokens per second for the whole configuration.
- `at_least_tokens_per_second` (C): that figure converted to input plus output tokens per second for your mix (below).
- `sufficient_utilization_percent`: a utilization at which the floor alone makes self-hosting cheaper above the break-even volume. It is a sufficient condition, not a necessary one: below it the real capacity may still be enough. `null` if the floor never is enough, or if your requests exceed 2,048 tokens in total.
- `conversion`: the conversion in words. `source`: "InferenceX (SemiAnalysis)". `reference.source_url`: the public run the figure comes from.
- `engine`: the inference engine of that run, one of `vllm`, `sglang`, `trtllm`, `other`, or `null` when unknown. `other_engine_ceiling`: present only in the case described above.

### Other values of `confidence`

- `estimated`: a wide-range estimate (our range: divided and multiplied by 3), from a model of a different type of architecture, or from a model of the same type whose active weights differ from the reference by more than a factor of 4. We never prefer a configuration for an estimate of this kind (see section 4).
- `user_supplied`: you gave `tokens_per_second`; it replaces our estimate. Give input and output tokens together, for the whole configuration.
- `none`: unknown. No throughput is known for this model on this configuration, and the verdict would need one.
- `not_needed`: the verdict does not rest on any throughput: at or under the break-even volume, or when the API offer is free, the API is cheaper whatever the throughput.

`configuration.throughput_evidence` says what is known about the throughput of the rented configuration, `confidence` says what the verdict rests on: the two can differ, for example `lower_bound` and `not_needed` for a volume under the break-even.

## 3. Source and conversion

### Source

Published figures come from public sources. As of 6 October 2026, all the published figures we use come from InferenceX (SemiAnalysis). The structured figures (a point per model, card, number of GPUs, engine and precision) are read from **InferenceX (SemiAnalysis)**, which publishes its data through a public API and links each point to the public run that produced it. We read:

- only models that are in our catalogue, with an exact match on the source's model key, never by inference;
- only single-node runs, with prompt reading and generation on the same GPUs, without speculative decoding and without offloading part of the model out of the GPUs;
- only single-turn benchmarks, the 1,024-token prompt / 1,024-token answer test, and at most one curve per configuration and engine (model, card, number of GPUs, engine and precision): the most recent single run with at least three concurrency levels;
- only cards we can name in our reference (currently H100, H200, B200 and B300, read as SXM cards; MI300X, MI325X, MI355X; RTX PRO 6000), in 1, 2, 4 or 8 GPU configurations; full racks are not read. At most 12 points are kept per model, the cards our tracked providers rent first; a configuration can carry two points (a levelled-off value and a higher minimum from another engine).

We do not copy the source's tables: one point per configuration is kept, with the run link of the run that was read. When the source stops publishing a figure, we stop using it, within a few days at most. We do not read MLPerf Inference for now, because we have not yet established what its tokens-per-second figure counts.

### From output tokens to input plus output tokens

The source counts output tokens per second. Our comparison needs input plus output tokens, at your mix. Two cases, both chosen to be the **least favourable to self-hosting**:

- **Your mix has at least as much input as output** (by default 3:1 input to output, so 75% input): at the benchmark's 1,024-in / 1,024-out test, the configuration served B output tokens per second and as many input tokens. We assume reading a prompt token costs no more than generating one, and count it as equal. So the configuration delivers **at least twice B** in input plus output tokens per second. Example, as arithmetic: 5,000 output tokens per second → at least 10,000 tokens per second, input and output together.
- **Your mix has more output than input**: we count only generation: B divided by the share of output tokens in your mix. Example: 5,000 output tokens per second with 25% input → 5,000 / 0.75 ≈ 6,666 tokens per second.

For measured, derived and estimated figures published in generated tokens only, we convert them to your input:output mix (3:1 by default) with a prefill-to-generation speed ratio of 3 (prudent scenario), 5 (median) and 8 (optimistic). The three scenarios in `scenarios` combine those three conversions with the ends of our range (divided by, the median, multiplied by 1.25, 2 or 3 depending on the level of evidence).

Two more rules of ours apply to these scenarios, and `assumptions` says so:

- **A published output figure is taken as if the GPUs only generated tokens**, ignoring that the benchmark also read prompts: "We treat a published output figure as if the GPUs only generated tokens, ignoring that the benchmark also read prompts: this understates the throughput of the whole configuration, which is the cautious side for self-hosting." The sentence appears only when the scenarios come from a figure published in output tokens.
- **No scenario falls below the published minimum for the same setup**, converted to the same mix, because the capacity is at least that. When a scenario was raised, the response says: "We do not let any scenario fall below the published minimum for this setup (converted to the same mix), since the capacity is at least that", followed, when you did not give your request sizes, by "; this minimum holds for requests of up to 2,048 tokens in total."

### The test condition

A floor holds for **requests of up to 2,048 tokens in total (the benchmark used 1,024 prompt tokens and 1,024 answer tokens)**. Longer requests cost more per token (attention, memory for the cache). That is the one rule, applied the same way everywhere:

- If you give your request sizes (`prompt_tokens` and `output_tokens`) and prompt plus answer exceed 2,048 tokens in total, the floor is quoted but settles nothing. A request of 2,000 prompt tokens and 48 answer tokens is within the rule.
- If you do not give them (a volume in tokens only), the sentence that settles ends with: "this floor holds for requests of up to 2,048 tokens in total (the benchmark used 1,024 prompt tokens and 1,024 answer tokens)." If you give sizes within that rule, it ends with: "your requests are within the benchmark's 2,048 tokens in total."

The sentence quoted for a floor varies with your mix (the default 3:1 mix is named in it) and with whether you gave your request sizes.

### A settled alternative

When the configuration the answer is about has no verdict that published figures can decide (`self_host_needs_throughput`), but a **dearer** rented configuration has one, the response points it out in `settled_alternative` and at the end of the headline. It is another setup, shown for comparison: it costs more than the one the answer is about, it is not a recommendation, and it is never used in place of the configuration the answer is about. The rest of the response does not change.

- We show one setup: the cheapest dearer one whose verdict a measured value or a published minimum decides. A setup for which the only established verdict would be "the API is cheaper" is not shown.
- `settled_alternative` gives the card, the number of GPUs, the provider, the tier, the hourly and daily price with `checked_at` and `source_url`, the verdict, the evidence it rests on (`measured` or `lower_bound`) and its break-even volume.
- The verdict is `self_host_cheaper_above` (self-hosting is cheaper above the break-even volume at your utilization) or `self_host_above_utilization` with `busy_at_least_percent`: the share of the time that setup must be busy. For a measured value it is the break-even utilization of the prudent scenario, which holds in all three scenarios; for a published minimum it is a utilization at which the floor is enough (sufficient, not necessary: less may also be enough). The verdict summarises the prudent case: asking for that configuration directly (`gpu`, `gpu_count`) can return `self_host_needs_throughput` with both ends of the range.
- An alternative that rests on a published minimum holds for requests of up to 2,048 tokens in total; none is shown when you describe longer requests.
- It is absent when you impose a card (`gpu`) or give `tokens_per_second`; with `gpu_count` alone, only configurations of that size are considered.
- With a volume, it is shown only if that setup is cheaper at your volume and can serve it; the "if that setup is busy at least N% of the time" form exists only without a volume.
- The sentence added to the headline reads, for example (as of 6 October 2026): "For comparison, published figures decide it for a dearer setup: 4x B200 SXM 180GB at $652 a day, self-hosting cheaper above 1,671 million tokens a day (published minimum, for requests of up to 2,048 tokens in total)." With a utilization: "…self-hosting cheaper above 882 million tokens a day if that setup is busy at least 69% of the time (measured)." The last words name the evidence; the length condition appears only for a published minimum and only when you did not describe your requests.

## 4. Settings and defaults

| Setting | Default | Range | Meaning |
|---|---|---|---|
| `utilization` | 50 | 1 to 100 | Percent of the time the GPUs serve requests. A machine rented by the hour costs the same when idle. |
| `weights_margin` | 70 | 50 to 95 | Percent of the GPU memory the model weights may fill. The rest is the engine's reserve and the context cache. |
| Mix | 3:1 input:output (75% input) | — | Used for the API price and for the throughput conversion, unless you give `requests_per_day`, `prompt_tokens` and `output_tokens`: then your own mix is used. |
| `quantization` | `fp8` | `fp16`, `bf16`, `fp8`, `int8`, `fp4`, `int4` | Precision of the weights. Below `fp16`/`bf16`, at least one provider must publish the model in that precision: we do not assume weights exist in a precision nobody publishes. |
| `tier` | `guaranteed` | `guaranteed`, `community`, `spot` | Rental tier. Tiers are never compared with each other. |
| `gpu`, `gpu_count` | none | id from `/gpus`; 1, 2, 4, 8 or 16 | Impose a card and/or a number of GPUs. Without them we pick the configuration (below). |
| `tokens_per_second` | none | 1 to 1,000,000 | Your own measured throughput for the whole configuration, input and output together. |
| Volume | none | | `tokens_per_day`, or `requests_per_day` with `prompt_tokens` and `output_tokens`. Without a volume you still get the verdict and the break-even volume. |

**The API price** is the cheapest current standard offer for the mix, among the providers we publish, with promotions and prices we flag as unstable excluded. `api.basis` says how many offers were compared.

**How we pick the configuration.** This is our rule, not a technical requirement. We pick the cheapest configuration whose weights fit. If a configuration costing up to 25% more has a measured throughput, we pick it instead; failing that, one with a derived throughput; failing that, one with a published minimum that settles the verdict; among equals, the cheaper. A configuration whose throughput is only a wide estimate is never preferred. When we pick a configuration that is not the cheapest, `cheapest_alternative` shows the cheapest and `selection_reason` says why, for example: "Selected over the cheapest configuration (<card>) because its throughput is measured; it costs 20% more per hour, within our 25% price tolerance."

## 5. When we decline to answer

We decline to give a number rather than an unreliable one. A refusal is a normal response (`status: "refused"`, `verdict: null`) with a machine-readable `reason`, a `message` and a `detail`.

| `reason` | Meaning |
|---|---|
| `model_closed_or_unsized` | The model has no published weights, or no known size: it cannot be self-hosted or priced. |
| `quantization_not_published` | No provider publishes this model in the precision you asked for. `detail` lists the precisions that are published. |
| `unknown_gpu` | The `gpu` you gave is not in our reference: see `/gpus`. |
| `gpu_not_rented` | No provider we track rents this card at this tier right now, so there is no price. |
| `gpu_memory_unknown` | We do not know the memory of this card, so we cannot say whether the weights fit. |
| `no_rental_configuration` | No provider publishes a configuration with the number of GPUs you asked for. |
| `weights_exceed_margin` | The weights do not fit within the margin on the configuration you asked for. `detail` gives the memory needed and available and the smallest configuration that fits. |
| `no_api_price` | There is no current API offer to compare with (none is publishable, current, standard and stable): see `/cheapest`. |

Other responses: a parameter out of range returns a 400 that names the parameter; a model that is not in our catalogue returns a 404 with suggestions.

## 6. What this does not tell you

- It is a comparison of **prices**, not of quality: it says nothing about whether the open model matches the API model you would otherwise use.
- Hardware rental only (see the note at the top).
- Peak benchmark conditions, not a service with latency targets: an interactive service sustains less.
- Prices are those the providers published when we last checked: `checked_at` is in the response.
