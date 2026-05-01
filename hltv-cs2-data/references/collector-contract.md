# HLTV Collector Contract

This document defines the collection layer required for productized `hltv-cs2-data`. The collector reads HLTV and writes a central warehouse/API source. End users do not run this collector.

## Collection Principles

- HLTV is the original upstream source.
- The display website is not a product data source; it is only a presentation layer.
- End users should not need a local database or local scraper.
- A central collector writes to a central warehouse/API consumed by the skill and product clients.
- Store raw snapshots and normalized records.
- Use parser versions for every extraction path.
- Write atomically: no partial match detail or half-updated team records.
- On Cloudflare or access challenge, fail the job safely and mark the source stale.
- Do not bypass access controls.

## Source Types

Collect these source types:

- `rankings`: team rankings and rank date.
- `calendar`: upcoming matches and event metadata.
- `team_map_stats`: pick/ban and map summary stats.
- `team_map_side_stats`: CT-side and T-side win rates by team/map from HLTV team map stats pages and detailed map pages.
- `team_match_history`: per-team map match rows.
- `team_player_ratings`: team player annual ratings.
- `event_player_ratings`: event-specific player ratings.
- `match_detail`: match page facts.
- `match_result`: completed score and per-map result.
- `side_scores`: match-specific CT/T side score splits by map when HLTV exposes them reliably.
- `veto`: veto steps and map order.
- `lineup`: confirmed or expected players.

## Raw Snapshot Requirement

Every collection job should persist a raw snapshot record:

```json
{
  "source_url": "https://www.hltv.org/matches/2393335/faze-vs-furia-blast-rivals-2026-season-1",
  "source_type": "match_detail",
  "retrieved_at": "2026-05-01T12:00:00Z",
  "parser_version": "match_detail.v1",
  "http_status": 200,
  "content_hash": "sha256:...",
  "storage_ref": "raw/hltv/match_detail/2393335/2026-05-01T120000Z.html",
  "parse_status": "success",
  "cf_detected": false
}
```

Storing full HTML can be replaced by object storage or compressed snapshots. The normalized record must keep `raw_snapshot_id`.

## Normalized Records

Minimum product tables or API entities:

- `teams`: HLTV ID, name, slug, aliases, current rank.
- `ranking_snapshots`: rank, date, source.
- `matches`: match ID, URL, teams, event, format, scheduled time, status.
- `match_maps`: map order, map name, score, winner, final/ongoing status.
- `match_side_scores`: CT/T side score splits by match map and team.
- `match_scores`: match score, per-map score, result status.
- `veto_steps`: step number, actor team, action, map, source.
- `lineups`: match ID, team ID, player ID/name, coach flag, stand-in flag, source.
- `head_to_head_matches`: direct team-vs-team match and map records.
- `player_rating_snapshots`: player/team/event/year rating rows.
- `team_map_snapshots`: map stats by team, tier/data type, date window.
- `team_map_side_stats`: CT/T win-rate fields can be stored in `team_map_snapshots` or a separate normalized view, but must be exposed as `map_side_stats` in the API.
- `team_map_matches`: per-map match history rows.
- `collection_jobs`: job status, started time, finished time, failures.

Existing local scraper tables can be mapped into this model, but product APIs should not expose local implementation table names as public contract.

## Match Detail Collector

For each HLTV match URL, parse:

- HLTV match ID.
- Event ID and event name.
- Scheduled time.
- Team IDs, names, slugs.
- Match format.
- Match status.
- Veto steps when visible.
- Map order.
- Per-map scores when visible.
- CT/T side score splits when visible.
- Confirmed lineups when visible.
- Coach/stand-in indicators when visible.

If a field is not visible pre-match, mark it `missing` or `unavailable_pre_match`; do not infer.

## Lineup and Rating Fallback

Lineup resolution order:

1. Match page confirmed lineup.
2. Event page roster if available.
3. Team page current roster.
4. Existing database roster snapshot.

Rating resolution order:

1. Event rating for the same event.
2. 2026 annual rating from HLTV stats.
3. Missing rating warning.

Only use fallback ratings when the response marks the source and reason.

## Team Map Side Stats Collector

For each relevant team/date window, parse:

- Team map summary URL for the current calendar-year window by default, for example `https://www.hltv.org/stats/teams/maps/6667/faze?startDate=2026-01-01&endDate=2026-12-31`.
- Detailed map pages linked from the summary when required.
- Per-map CT-side win rate.
- Per-map T-side win rate.
- CT/T round or side sample counts when visible.
- Date window and source URL.

Expose these records as `map_side_stats` in the data pack. This is different from `side_scores`, which is a match-specific score split.

## Backtest Collection Requirement

To support exact time travel, retain timestamped snapshots:

- Rankings by date.
- Player rating snapshots by query window.
- Team map stat snapshots by query window.
- Match detail snapshots before and after veto/result.

If exact snapshots are absent, the API must mark data as reconstructed or unavailable.

## Phase Boundary

Phase 1 required:

- Central warehouse/API as the product source of truth.
- Match detail.
- Veto.
- Lineup.
- Scores and results.
- Side score schema and API field; collector should fill it when visible, otherwise mark it missing.
- Map side win-rate fields from team map stats pages.
- Head-to-head records.
- Raw snapshots.
- Central data-pack API shape.

Phase 2:

- Half scores.
- Opening side.
- Pistol/advanced round details if HLTV exposes them reliably.

Do not block Phase 1 lightweight usage on CT/T data, but the product API contract should reserve and expose `side_scores` from the start.
