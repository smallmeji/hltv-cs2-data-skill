# Data Pack Contract

This contract defines the stable output shape for `hltv-cs2-data`. It keeps HLTV factual data, decision inputs, and optional downstream model inference separate.

## Markdown Structure

Use this order:

1. Query metadata.
2. Resolved teams.
3. Match context.
4. Lineups and roster notes.
5. Player ratings.
6. Map pool summary.
7. Recent map history.
8. Head-to-head history.
9. Veto and match detail.
10. Score/result history.
11. Decision Inputs.
12. Data warnings.
13. JSON block for factual data and decision inputs.
14. Optional Model Inference section, only if explicitly requested by the user.

Do not add prediction, probability, EV, strategy, or betting fields inside the factual HLTV data pack.

If the user explicitly asks for judgment, add a separate `Model Inference` section after the factual pack. The section must state that it is model-derived and not HLTV fact data.

## Phase 1 Fields

Phase 1 data packs must support these field groups when available:

- `teams`: identity, aliases, HLTV IDs, ranking snapshots.
- `match`: match ID, event, tier, LAN/online, format, schedule, status.
- `maps`: pick/ban, raw win rate, weighted win rate, sample size, tier/data filters, recent map rows.
- `players`: annual ratings, event ratings, missing rating status.
- `lineups`: starters, coaches, stand-ins, roster-change warnings.
- `head_to_head`: direct matchup records and map rows.
- `veto`: veto steps, map order, decider.
- `scores`: map scores, match score, result status.
- `decision_inputs`: factual model-ready factors grouped for downstream strategy.
- `warnings`: missing data, stale data, low sample, reconstruction.

## Phase 2 Fields

Phase 2 can add these fields after collector and warehouse support exists:

- `side_scores`: CT/T score split by map.
- `half_scores`: first-half and second-half scores.
- `starting_side`: starting CT/T side by team and map.
- `rounds`: round-level details if HLTV exposes reliable data.
- `roster_timeline`: historical lineup windows and transfer dates.

## JSON Shape

```json
{
  "schema_version": "hltv-cs2-data.v1",
  "metadata": {
    "query_type": "match_data_pack",
    "requested_at": "2026-05-01T12:00:00Z",
    "data_cutoff": "2026-05-01T11:55:00Z",
    "as_of_date": null,
    "source_policy": "hltv_central_warehouse",
    "missing_fields": [],
    "warnings": []
  },
  "match": {
    "hltv_match_id": 2393335,
    "hltv_url": "https://www.hltv.org/matches/2393335/faze-vs-furia-blast-rivals-2026-season-1",
    "event_id": 8250,
    "event_name": "BLAST Rivals 2026 Season 1",
    "tier": "S",
    "format": "bo3",
    "scheduled_at": "2026-04-30T15:00:00Z",
    "status": "scheduled",
    "team_a_hltv_id": 6667,
    "team_b_hltv_id": 8297
  },
  "teams": [
    {
      "side": "A",
      "hltv_id": 6667,
      "name": "FaZe",
      "slug": "faze-clan",
      "aliases": ["FaZe Clan", "faze"],
      "rank": {
        "rank": 18,
        "source": "hltv_rankings",
        "as_of": "2026-04-30T00:00:00Z"
      }
    }
  ],
  "lineups": [
    {
      "team_hltv_id": 6667,
      "source": "match_page",
      "status": "expected",
      "players": [
        {
          "name": "player",
          "hltv_player_id": null,
          "role_note": null,
          "is_coach": false,
          "is_stand_in": false,
          "rating_2026": null,
          "rating_event": null,
          "rating_status": "missing"
        }
      ],
      "warnings": []
    }
  ],
  "maps": [
    {
      "map": "Mirage",
      "team_a": {
        "team_hltv_id": 6667,
        "data_type": "S",
        "sample_maps": 8,
        "raw_win_rate": 0.5,
        "weighted_win_rate": 0.47,
        "pick_pct": 22.2,
        "ban_pct": 11.1,
        "recent_matches": []
      },
      "team_b": {
        "team_hltv_id": 8297,
        "data_type": "S",
        "sample_maps": 9,
        "raw_win_rate": 0.56,
        "weighted_win_rate": 0.53,
        "pick_pct": 30.0,
        "ban_pct": 5.0,
        "recent_matches": []
      }
    }
  ],
  "head_to_head": {
    "status": "available",
    "matches": []
  },
  "recent_matches": {
    "team_a": [],
    "team_b": []
  },
  "veto": {
    "source": "match_page",
    "status": "unavailable",
    "steps": []
  },
  "scores": {
    "status": "not_started",
    "maps": []
  },
  "decision_inputs": {
    "map_pool": [],
    "head_to_head": [],
    "player_form": [],
    "roster_state": [],
    "match_context": [],
    "data_quality": []
  },
  "not_included": [
    "map_win_probability",
    "match_win_probability",
    "winner_prediction",
    "strategy_recommendation",
    "betting_ev",
    "stake_sizing"
  ],
  "model_inference": null
}
```

## Required Metadata

Each response must expose:

- `schema_version`.
- `requested_at`.
- `data_cutoff`.
- `source_policy`.
- `missing_fields`.
- `warnings`.

If any field is reconstructed rather than directly observed, add a warning and mark the field source as `reconstructed`.

## Field Trust Levels

- `observed`: directly parsed from HLTV or stored from a direct HLTV parse.
- `derived`: computed from observed records.
- `reconstructed`: rebuilt from older records but not an exact historical snapshot.
- `missing`: unavailable and not inferred.

## Data Quality Warnings

Use explicit warnings for:

- Fewer than 5 map samples in the requested filter.
- Missing lineup.
- Missing player rating.
- Coach or stand-in detected.
- Team recently changed roster.
- Event rating unavailable.
- Historical snapshot unavailable for `as_of_date`.
- Collector parse error or stale data.

## Fact vs Inference Boundary

The data contract may include descriptive rates such as raw map win rate, weighted win rate, and sample count. These are historical data fields, not predictions.

Do not add model-derived fields inside factual objects such as `teams`, `match`, `maps`, `players`, `lineups`, `veto`, `scores`, or `metadata`.

`decision_inputs` are still factual fields. They may summarize and organize observed/derived facts for downstream models, but they must not contain predicted probabilities or recommended actions.

If requested, model-derived fields may appear only under `model_inference`, for example:

```json
"model_inference": {
  "requested": true,
  "disclaimer": "This section is model-derived inference, not HLTV fact data.",
  "map_win_probabilities": [],
  "match_win_probability": null,
  "winner_lean": null,
  "reasoning_summary": [],
  "uncertainty_notes": []
}
```

Never hide model-derived values in factual fields such as:

- `predicted_map_win_probability`
- `predicted_match_win_probability`
- `fair_odds`
- `ev`
- `stake`

Betting-specific fields remain outside this skill unless the user invokes a separate betting/strategy layer.
