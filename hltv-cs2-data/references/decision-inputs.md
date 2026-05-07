# Decision Inputs

`decision_inputs` is the bridge between factual HLTV data and downstream user-owned analysis.

It does not decide who wins. It organizes factual, source-backed factors that a user-owned model can weight however it wants.

## Required Groups

Use these groups when available:

- `map_pool`: map history, pick/ban tendencies, raw win rate, weighted win rate, sample size, recent map rows.
- `head_to_head`: direct team-vs-team records, map-level head-to-head records, score history.
- `player_form`: annual rating, event rating, recent rating rows, missing player ratings.
- `roster_state`: starters, coach/stand-in flags, new players, roster changes, missing lineup.
- `side_profile`: CT-side and T-side win rates by team/map from HLTV team map stats pages; match-specific CT/T score splits only when available.
- `match_context`: event, tier, LAN/online, format, stage, schedule, travel/rest if available.
- `data_quality`: sample size issues, stale data, missing fields, direct HLTV vs API mode, reconstructed fields.

Do not add model-derived groups such as `winner_prediction`, `edge`, or `recommendation`.

## Item Shape

Each decision input item should use this shape where possible:

```json
{
  "group": "map_pool",
  "factor": "G2 Ancient raw win rate",
  "team": "G2",
  "team_hltv_id": 5995,
  "map": "Ancient",
  "value": "79%",
  "numeric_value": 0.79,
  "sample_size": 14,
  "source": "HLTV match page map stats",
  "source_url": "https://www.hltv.org/matches/2393346/g2-vs-faze-blast-rivals-2026-season-1",
  "trust_level": "observed",
  "as_of": "2026-05-01T12:00:00Z",
  "notes": [],
  "missing_reason": null
}
```

## Trust Levels

Use the same trust levels as the data pack:

- `observed`: directly visible from HLTV.
- `derived`: computed from observed HLTV rows.
- `reconstructed`: rebuilt for backtest because exact snapshot is unavailable.
- `missing`: expected but unavailable.

## Group Guidance

### map_pool

Include per-map factors such as:

- Team map win rate.
- Sample size.
- Pick percentage.
- Ban percentage.
- Recent results on the map.
- Map is unplayed or low sample.

When the user asks for a stronger/weaker or win-rate comparison and map detail data exists, `map_pool` should be organized per playable map rather than only as one summary table. For each map, include:

- overall and LAN sample size;
- W-L and raw win rate;
- rounds won / total rounds when collected;
- CT-side and T-side win rates;
- pistol round win rate;
- round win rate after first kill and after first death;
- pick percentage and ban percentage;
- data-quality note such as `small_sample`, `no_current_year_data`, `extreme_ban_rate`, or `likely_veto_variable`.

Maps with no current-year data for one side, extreme ban rate, or tiny sample should be flagged as `veto_variable` / `low_confidence_map` and kept separate from normal map-edge comparison. Do not average them together with regular played maps without a warning.

### head_to_head

Include:

- Direct match results.
- Direct map results.
- Scoreline history.
- Map-specific head-to-head rows when available.
- H2H unavailable marker when not found.

### player_form

Include:

- Player annual rating.
- Event rating.
- Maps played.
- Missing rating marker.
- Coach/stand-in rating unavailable marker.

### roster_state

Include:

- Confirmed lineup.
- Expected lineup.
- Stand-in.
- Coach playing.
- New player.
- Recent roster change if visible.

### side_profile

Include:

- Team CT-side win rate by map.
- Team T-side win rate by map.
- CT/T round sample counts when available.
- Source date window, defaulting to the current calendar year, e.g. 2026-01-01 to 2026-12-31.
- Match-specific CT/T score splits only when available.
- Side-stat missing marker when HLTV page/API does not expose it.

Use side-profile fields only as factual context. Do not infer tactical strength.

### match_context

Include:

- Event name and tier.
- LAN/online.
- BO format.
- Stage.
- Scheduled time.
- Match status.

### data_quality

Include:

- `direct_hltv_mode`.
- `central_warehouse_api_unavailable`.
- `small_sample`.
- `missing_event_rating`.
- `missing_lineup`.
- `veto_unavailable`.
- `map_side_stats_unavailable`.
- `exact_backtest_snapshot_unavailable`.

## Inference Boundary

Decision inputs are not predictions.

Allowed:

- "G2 Ancient raw win rate is 79% on 14 maps."
- "FaZe has missing event rating for player X."
- "Veto unavailable."

Not allowed inside `decision_inputs`:

- "G2 should win Ancient."
- "FaZe has 58% match win probability."
- "This is the recommended action."

Those belong outside `hltv-cs2-data`. This skill only returns facts and decision-input context.
