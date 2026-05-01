# Data Availability Matrix

This reference defines which data can be expected from each access mode. It prevents the skill from overpromising direct HLTV access.

## Access Modes

| Mode | User Setup | Intended Use |
|:---|:---|:---|
| Lightweight direct | None beyond the host model's normal web/page-reading/search tools | Best-effort public data pack |
| In-app/browser session | A user-visible browser session available to the model | Interactive investigation and manual verification |
| Internal collector | Maintained by the product operator with persistent browser/session handling | Scheduled collection into warehouse |
| Static JSON | Hosted or local exported JSON data packs | Stable public distribution without live HLTV reads |
| API / warehouse | API key and hosted data service | Stable data packs, backtests, repeated use |

## What Each Mode Can Provide

### Lightweight Summary

| Data | Lightweight Direct Provides |
|:---|:---|
| Match information | Yes |
| Team IDs / lineup | Yes |
| Event ID | Yes |
| Event rating | Attempt; mark missing if retrieval fails |
| Annual player rating | Attempt; mark missing if retrieval fails |
| Current-year map summary | Yes when the team map summary page is reachable; otherwise use match-page recent-core context with explicit fallback labeling |
| W/D/L, win rate, pick %, ban % | Yes from current-year map summary when reachable |
| CT/T side win rates | No; API / warehouse enhanced mode only |
| Historical backtest snapshots | No; API / warehouse enhanced mode only |

Lightweight direct data can be useful but incomplete. If current-year map summary or player ratings cannot be loaded, the data pack may still be emitted, but exact numeric model inference must be blocked by `references/inference-gate.md`.

### Detailed Matrix

| Data Type | Lightweight Direct | In-App / Browser Session | Internal Collector | Static JSON | API / Warehouse |
|:---|:---:|:---:|:---:|:---:|:---:|
| Match page basics: teams, event, time, format, status | Usually yes | Yes | Yes | Yes when exported | Yes |
| Match page visible lineup | Usually yes when visible | Yes | Yes | Yes when exported | Yes |
| Match page veto/result/scores | Yes when visible | Yes | Yes | Yes when exported | Yes |
| Team page current roster/rank/period rating | Usually yes | Yes | Yes | Yes when exported | Yes |
| Match page recent-core map stats | Usually yes | Yes | Yes | Yes when exported | Yes |
| Team map summary stats for current year | Attempt only; may fail with cache miss/CF | Usually yes after session is valid | Yes | Yes when exported | Yes |
| Team annual player stats for current year | Attempt only; may fail with cache miss/CF | Usually yes after session is valid | Yes | Yes when exported | Yes |
| Event player ratings by event ID | Attempt only; may fail with cache miss/CF | Usually yes after session is valid | Yes | Yes when exported | Yes |
| Per-map CT/T side win rates | No by default; requires map detail pages | Possible but slow | Yes | Yes when exported | Yes |
| Match-specific CT/T score splits | No guarantee | Possible when visible | Phase 2 | Yes when exported | Phase 2 / when collected |
| Exact historical as-of snapshots | No | No guarantee | Yes if collected then | Snapshot only if exported | Yes |
| Round-level/pistol/opening-side data | No | No guarantee | Phase 2+ | No unless exported | Phase 2+ |

## Direct Mode Failure Labels

When lightweight direct mode cannot retrieve a deep stats page, use field-level warnings instead of treating the data as absent:

- `fetch_failed_cache_miss`: the host model/page reader did not return a readable page snapshot.
- `fetch_failed_cf_challenge`: a direct HTTP request or page reader was blocked by Cloudflare/access challenge.
- `stats_page_unavailable_in_direct_mode`: the URL is known, but the direct mode cannot retrieve the table.
- `pro_api_required_for_full_coverage`: the field requires hosted collector/API data for reliable coverage.
- `core_data_insufficient_for_numeric_inference`: the data pack is too incomplete for specific win-rate percentages.

## Important Distinctions

- `Cache miss` is a page-reading-tool failure, not an HLTV fact. It means the host model could not retrieve a readable snapshot.
- `403` with `cf-mitigated: challenge` is an HLTV/Cloudflare access challenge.
- A known URL must not be reported as "data does not exist" unless HLTV itself clearly says no data exists.
- Match-page recent-core map stats must not be used as a replacement for current-year team map summary stats unless clearly labeled as fallback context.
- Lightweight direct mode must not ask public users to run a local browser, CDP, Playwright, or scraper.
- Market prices, rankings, news snippets, and search summaries are not substitutes for missing current-year map/rating data when producing numeric probabilities.

## Recommended Fallback Order

For a match URL:

1. If a static JSON manifest/API is configured, read it first.
2. Read match page facts only when no static/API pack exists.
3. Resolve team IDs, slugs, and event ID.
4. Attempt team page roster/rank/period rating.
5. Attempt current-year team map summary stats.
6. Attempt current-year team player stats.
7. Attempt event player ratings if event ID is known.
8. If any deep stats page fails, output the known canonical URL and a warning code.
9. If full coverage is required, recommend static JSON/API/warehouse mode.
