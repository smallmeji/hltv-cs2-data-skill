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

## Language and Readability

User-facing Markdown should follow the user's language. If the user writes in Chinese, use Chinese section titles, table labels, warnings, and short explanatory notes. Keep JSON field names in English so the output remains stable for programs and downstream models.

Default Chinese Markdown structure:

1. `数据状态`
   - One compact table: source mode, retrieval time, data cutoff, completeness, high-impact missing fields.
   - Start with a short sentence such as: `这是事实数据包，不包含预测、投注建议、EV 或仓位。`
2. `比赛信息`
   - Match ID, event, tier, LAN/online, BO format, schedule, status.
3. `队伍与阵容`
   - Team names, HLTV IDs, rank snapshots, starters, coach/stand-in flags.
4. `选手数据`
   - Annual rating and event rating when available.
   - Mark missing values as `缺失`, not as guessed values.
   - For match URL queries with visible lineups, annual rating and event rating are required fetch attempts. If either cannot be retrieved, show a per-player `rating_status` and add a warning.
5. `地图池`
   - Per-map comparison table.
   - Include sample size next to every win-rate field.
   - Show both raw win rate and weighted win rate when available.
6. `近期记录 / H2H`
   - Recent match rows and direct matchup rows when available.
   - If unavailable, say `未加载` or `HLTV 当前页面未提供`.
7. `警匪胜率`
   - CT-side and T-side win rates by team and map from HLTV team map stats pages, e.g. `/stats/teams/maps/<teamId>/<slug>?startDate=YYYY-01-01&endDate=YYYY-12-31` and detailed map pages.
   - If unavailable, say `未加载` and explain whether the source page did not expose it or the collector does not support it yet.
8. `Veto / 比分`
   - Veto steps, map order, score state, result state.
9. `给模型的决策输入`
   - Factual inputs grouped into `地图池`, `对位数据`, `选手状态`, `阵容状态`, `警匪胜率`, `比赛环境`, `数据质量`.
   - Do not put predicted win rates or recommendations here.
10. `数据缺口`
   - Explicitly list missing data and what cannot be inferred.
11. `JSON`
   - Factual data and decision inputs with stable English keys.
12. Optional `模型推理`
   - Only when the user asks for judgment.

Readability rules:

- Prefer compact tables over long paragraphs.
- Use `缺失`, `未加载`, `不可见`, `样本不足`, and `重建数据` consistently.
- Keep the top-level Markdown concise; put full machine-readable detail in JSON.
- Do not output English headings for a Chinese prompt unless the user asks for English.

## Phase 1 Fields

Phase 1 data packs must support these field groups when available:

- `teams`: identity, aliases, HLTV IDs, ranking snapshots.
- `match`: match ID, event, tier, LAN/online, format, schedule, status.
- `maps`: pick/ban, raw win rate, weighted win rate, sample size, tier/data filters, recent map rows.
- `map_side_stats`: CT-side and T-side win rates by team/map from HLTV team map stats pages.
- `players`: annual ratings, event ratings, missing rating status.
- `lineups`: starters, coaches, stand-ins, roster-change warnings.
- `head_to_head`: direct matchup records and map rows.
- `side_scores`: match-specific CT/T score splits by map when visible or available from API/warehouse.
- `veto`: veto steps, map order, decider.
- `scores`: map scores, match score, result status.
- `decision_inputs`: factual model-ready factors grouped for downstream strategy.
- `warnings`: missing data, stale data, low sample, reconstruction.

## Phase 2 Fields

Phase 2 can add these fields after collector and warehouse support exists:

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
  "map_side_stats": [
    {
      "team_hltv_id": 6667,
      "team_name": "FaZe",
      "map": "Mirage",
      "data_type": "year_2026",
      "ct_side_win_rate": null,
      "t_side_win_rate": null,
      "ct_rounds": null,
      "t_rounds": null,
      "source": "https://www.hltv.org/stats/teams/maps/6667/faze?startDate=2026-01-01&endDate=2026-12-31"
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
  "side_scores": [
    {
      "match_map_id": null,
      "map_order": 1,
      "map": "Mirage",
      "team_hltv_id": 6667,
      "team_name": "FaZe",
      "ct_score": null,
      "t_score": null,
      "starting_side": null,
      "source": "missing"
    }
  ],
  "decision_inputs": {
    "map_pool": [],
    "head_to_head": [],
    "player_form": [],
    "roster_state": [],
    "side_profile": [],
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
