# Changelog

Changes to the public data and documentation that API users should know about. Dates and times are UTC.

## Data corrections

### 2026-09-26 — qwen/qwen3.8-omni-flash, Alibaba Cloud list price (output)

- **What was wrong:** the output price of Alibaba Cloud's own offer was published as 0.016 USD per 1M tokens
  (standard deployment) and 0.014 (Global deployment). The correct prices are 0.47 and 0.382. Input prices
  (0.15 and 0.113) were correct.
- **Cause:** on Alibaba Cloud's pricing page, the cached-input price column was read as the output price.
- **Period:** from 2026-09-22 20:11 to 2026-09-25 around 10:30, when the offer was flagged stale.
- **Impact:** during that period, `/cheapest` returned the Global offer (0.113 input / 0.014 output) as the
  cheapest for this model, and the daily series of `/history?series=cheapest` selected it on 2026-09-23 and
  2026-09-24.
- **Fix:** the page is read correctly since 2026-09-25 23:50. The price history, including those two days, was
  corrected on 2026-09-26 00:05 and now shows 0.47 and 0.382.
- **Not affected:** the market index (no value was computed or published from these prices).
