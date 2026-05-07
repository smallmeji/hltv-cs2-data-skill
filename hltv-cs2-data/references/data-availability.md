# Data Availability Matrix

This reference defines which data can be expected from each access mode. It prevents the skill from overpromising direct HLTV access.

## Access Modes

| Mode | User Setup | Intended Use |
|:---|:---|:---|
| Public standalone | None beyond the host model's normal web/page-reading/search tools | HLTV discovery plus static JSON database hydration |
| Direct HLTV partial fallback | None beyond the host model's normal web/page-reading/search tools | Partial visible facts only when structured data cannot be read |
| In-app/browser session | A user-visible browser session available to the model | Interactive investigation and manual verification |
| Internal collector | Maintained by the product operator with persistent browser/session handling | Scheduled collection into warehouse |
| Static JSON database export | Hosted or local exported JSON data packs. Default public manifest: `https://raw.githubusercontent.com/smallmeji/hltv-cs2-data-skill/main/public-data/manifest.json` | Required public structured-data source after HLTV identity resolution |
| API / warehouse | API key and hosted data service | Stable data packs, backtests, repeated use |

`hltv-cs2-data-platform` public-data paths and product website pages are not access modes for structured data. If a model uses one of those URLs and receives 404, that is stale-source usage, not evidence that the static database is unavailable. The accepted public source is the `hltv-cs2-data-skill` raw GitHub manifest or its documented compatibility alias.

## What Each Mode Can Provide

### Public Standalone Summary

| Data | Public Standalone Provides |
|:---|:---|
| Match information | Yes from HLTV discovery and/or exported match pack |
| Team IDs / lineup | Yes when visible/exported |
| Event ID | Yes when discoverable/exported |
| Event rating | From static JSON/API when exported; direct HLTV fallback may attempt and mark missing |
| Annual player rating | From static JSON/API when exported; direct HLTV fallback may attempt and mark missing |
| Current-year map summary | From static JSON/API when exported; direct HLTV fallback may attempt and mark fallback context |
| W/D/L, win rate, pick %, ban % | From static JSON/API exact rows when exported |
| CT/T side win rates | From static JSON/API when exported |
| Historical backtest snapshots | No; API / warehouse enhanced mode only |

Direct HLTV partial fallback can be useful but incomplete. If current-year map summary or player ratings cannot be loaded from a structured source, the output must not be presented as a complete report. This data skill never outputs numeric model inference.

For normal public users, direct HLTV should be attempted first only for match discovery and identity resolution. Static JSON database export should then be queried for structured map/player/side/history fields. Direct HLTV deep stats are last-resort supplemental fallback only when the database export is unavailable or missing a field.

### Detailed Matrix

| Data Type | Direct HLTV Partial | In-App / Browser Session | Internal Collector | Static JSON | API / Warehouse |
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

When direct fallback cannot retrieve a deep stats page, use field-level warnings instead of treating the data as absent:

- `default_static_source_unavailable`: the default public static JSON source could not be read.
- `static_record_not_found`: the default or configured static source is reachable, but the requested team/match/event record was not exported.
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
- Public standalone mode must not ask public users to run a local browser, CDP, Playwright, or scraper.
- Market prices, rankings, news snippets, and search summaries are not substitutes for missing current-year map/rating data and must not be used to produce probabilities inside this data skill.
- Liquipedia, Liquidpedia, wikis, news snippets, and search summaries are not substitutes for the structured database export.

## Recommended Resolution And Fallback Order

For a match URL:

1. Read the HLTV match page first when a match URL is provided.
2. If the user only provides event/team names, search/read HLTV match, event, results, and upcoming pages first to locate the relevant match page.
3. Resolve match ID, team IDs, slugs, event ID, format, time, status, lineup/veto/score when visible.
4. If API/warehouse is configured, use it to hydrate structured fields after identity resolution.
5. Otherwise read configured static JSON or the default public static manifest as the public database export.
6. For natural-language match lookup, use the selected source's match search/index before team-only records. In the default public source this is `matches/index.json`; when a clear match exists, read that row's match data-pack reference such as `data_pack_path`.
7. Hydrate team maps, map details, player ratings, event ratings, match packs, and compare-ready fields from the database export when present.
8. If the database export is unavailable or missing a field, then attempt direct HLTV current-year team map summary stats, current-year team player stats, and event player ratings as supplemental fallback.
9. Merge database fields into the data pack with `source=static_database` or `source=api_warehouse`; preserve direct HLTV fields with `source=direct_hltv`.
10. If full coverage is required and static/API is still missing fields, recommend API/warehouse mode.
