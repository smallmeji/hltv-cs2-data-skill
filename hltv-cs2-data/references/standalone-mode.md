# Lightweight / Direct HLTV Mode

Direct HLTV mode lets someone use `hltv-cs2-data` immediately after installing the skill, without any private database, scraper, or central API.

## Goal

Given a natural-language request, a match URL, or two team names, produce the best available strategy-neutral HLTV data pack from public HLTV pages.

This is the default mode. It must be useful, but honest about missing data.

If API credentials are configured, API mode should be tried first. Direct HLTV mode is still the public lightweight fallback.

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
4. If `as_of_date` is mentioned, enter backtest discipline, but mark exact snapshots unavailable unless an API/warehouse exists.

## Direct HLTV Data Sources

Use only public HLTV pages available in the session:

- Match page: match ID, teams, event, schedule, format, status, lineup if visible, veto if visible, scores if visible.
- Match page map stats: visible recent core win rates and sample counts.
- Team stats pages: map stats and recent map rows when accessible.
- Player/event stats pages: for match data packs, resolve `eventId` from the match page/event link, then fetch event ratings from `https://www.hltv.org/stats/players?event=<eventId>` for visible starters. Also attempt annual rating from player stats.
- Results/matches pages: results or upcoming schedules when needed.

Do not use private display websites as a source.

## Required Direct-Mode Warnings

If no central API/warehouse is configured, include:

```json
"warnings": [
  "direct_hltv_mode",
  "central_warehouse_api_unavailable"
]
```

Add missing-field warnings for unavailable data:

- `annual_rating_not_loaded`: annual player stats page was unreachable or the player could not be resolved.
- `event_rating_not_loaded`: event stats page was unreachable, event ID could not be resolved, or the player is not listed for the event.
- `lineup_unavailable`
- `veto_unavailable`
- `head_to_head_not_loaded`
- `exact_backtest_snapshot_unavailable`
- `side_scores_not_loaded`

## Output Behavior

Direct HLTV output must still follow the data pack contract:

- Markdown summary first.
- JSON summary or full JSON when requested.
- `Decision Inputs` summarizing factual factors for downstream models.
- Factual fields must not contain model-derived probability, prediction, strategy, EV, or stake fields.
- If explicitly requested, append a separate `Model Inference` section after the factual data pack.

If the user asks who is favored or who has higher win rate, first return the data pack. If the user wants this model to judge, append a clearly labeled `Model Inference` section.

## Limits

Direct HLTV mode cannot guarantee:

- Complete historical snapshots.
- Exact as-of backtests.
- Full head-to-head data.
- Complete annual/event rating coverage, but it must attempt those fields before marking them missing.
- CT/T side score coverage when the current HLTV page does not expose it.
- Round-level data.

When these are needed, recommend API/warehouse mode rather than inventing missing fields.
