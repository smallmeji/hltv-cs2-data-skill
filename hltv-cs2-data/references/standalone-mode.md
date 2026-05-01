# Lightweight / Direct HLTV Mode

Direct HLTV mode lets someone use `hltv-cs2-data` immediately after installing the skill, without any private database, scraper, central API, local browser, CDP session, or Playwright session.

## Goal

Given a natural-language request, a match URL, or two team names, produce the best available strategy-neutral HLTV data pack from public HLTV pages using only the host model's normal web/page-reading/search capabilities.

This is the default mode. It must be useful, but honest about missing data. If the host model cannot access a deep HLTV stats page, mark that field as unavailable instead of asking the user to run a local browser.

Direct mode is often enough for match basics and simple context. It is not enough for confident numeric probabilities when HLTV blocks the yearly stats pages. In that case, output the partial data pack and missing fields, but do not produce exact win-rate percentages.

If API credentials are configured, API mode should be tried first. Direct HLTV mode is still the public lightweight fallback.

Local browser/CDP access belongs only to internal collector maintenance. It must not be required from public lightweight users.

## Accepted Inputs

Natural language examples:

```text
https://www.hltv.org/matches/2393346/g2-vs-faze-blast-rivals-2026-season-1
Please find the two teams' data for this match.
```

```text
Help me look at NAVI and G2 data.
```

```text
Compare NAVI and G2 on Mirage and Ancient, output JSON too.
```

The user should not need to provide API query parameters.

## Query Resolution Order

1. If input contains an HLTV match URL, extract `hltvMatchId` and parse team names from the URL/page.
2. If input contains two team names, resolve them by HLTV search, ranking page, or team pages.
3. If a map is mentioned, restrict or highlight that map.
4. Use the current calendar year as the default data window, e.g. `2026-01-01` to `2026-12-31` in 2026.
5. If `as_of_date` is mentioned, enter backtest discipline and use the calendar year containing `as_of_date`, but mark exact snapshots unavailable unless an API/warehouse exists.

## Direct HLTV Data Sources

Use only public HLTV pages available in the session:

- Match page: match ID, teams, event, schedule, format, status, lineup if visible, veto if visible, scores if visible.
- Team pages: resolve canonical team IDs, slugs, current roster, and team links when the match page does not expose enough structure.
- Team map summary stats pages: primary source for map W/D/L, win rate, pick %, and ban % for the default year window. For example: `https://www.hltv.org/stats/teams/maps/<teamId>/<slug>?startDate=2026-01-01&endDate=2026-12-31`.
- Team player stats pages: primary source for annual player ratings filtered by team and current calendar-year window, for example `https://www.hltv.org/stats/teams/players/<teamId>/<slug>?startDate=2026-01-01&endDate=2026-12-31`.
- Event player stats pages: event rating source when `eventId` is known, for example `https://www.hltv.org/stats/players?event=8250`.
- Match page map stats: visible recent core win rates and sample counts only as a fallback or supplemental context. Do not treat these as the official `maps` section when the yearly team stats pages are reachable.
- Results/matches pages: results or upcoming schedules when needed.

Do not use private display websites as a source.

For precise coverage expectations, follow `references/data-availability.md`.

## Required Direct-Mode Warnings

If no central API/warehouse is configured, include:

```json
"warnings": [
  "direct_hltv_mode",
  "central_warehouse_api_unavailable"
]
```

Add missing-field warnings for unavailable data:

- `core_data_insufficient_for_numeric_inference`: critical map/rating/lineup fields are missing, so exact win-rate percentages must not be produced.
- `annual_rating_not_loaded`: annual player stats page was unreachable or the player could not be resolved.
- `event_rating_not_loaded`: event stats page was unreachable, event ID could not be resolved, or the player is not listed for the event.
- `fetch_failed_cache_miss`: the host model/page reader could not return a readable snapshot for a known URL.
- `fetch_failed_cf_challenge`: the URL was blocked by Cloudflare/access challenge.
- `stats_page_unavailable_in_direct_mode`: a known deep stats page was not retrievable in lightweight mode.
- `pro_api_required_for_full_coverage`: hosted collector/API data is required for reliable coverage.
- `lineup_unavailable`
- `veto_unavailable`
- `head_to_head_not_loaded`
- `exact_backtest_snapshot_unavailable`
- `map_side_stats_not_loaded`

## Output Behavior

Direct HLTV output must still follow the data pack contract:

- Markdown summary first.
- JSON summary or full JSON when requested.
- `Decision Inputs` summarizing factual factors for downstream models.
- Factual fields must not contain model-derived probability, prediction, strategy, EV, or stake fields.
- If explicitly requested, append a separate `Model Inference` section after the factual data pack only after applying `references/inference-gate.md`.

If the user asks who is favored or who has higher win rate, first return the data pack. If the user wants this model to judge, append a clearly labeled `Model Inference` section only when the core data gate allows it. If the gate fails, provide a qualitative low-confidence direction at most and state that exact percentages are blocked.

## Limits

Direct HLTV mode cannot guarantee:

- Complete historical snapshots.
- Exact as-of backtests.
- Full head-to-head data.
- Complete annual/event rating coverage, but it must attempt those fields before marking them missing.
- CT/T side win-rate coverage; lightweight mode does not visit every map detail page. Use Pro/API or collector mode for `map_side_stats`.
- Round-level data.

When these are needed, recommend API/warehouse mode rather than inventing missing fields.

## Lightweight Inference Gate

Use this quick rule in standalone mode:

- If both teams have current-year map summary and at least one usable player-rating source, numeric inference may be allowed with caveats.
- If either team's current-year map summary is missing, do not output map or match win-rate percentages.
- If two or more high-impact fields are missing, do not output exact percentages.
- If the result relies on search snippets, rankings, or market prices only, mark `completeness_level=partial` or `blocked` and stop before numeric inference.
