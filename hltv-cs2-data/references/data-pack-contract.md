# Data Pack Contract

This contract defines the stable output shape for `hltv-cs2-data`. It keeps HLTV factual data and model-ready decision inputs separate from any downstream prediction layer.

## Markdown Structure

Use this order:

1. Source execution log.
2. Query metadata.
3. Resolved teams.
4. Match context.
5. Lineups and roster notes.
6. Player ratings.
7. Map pool summary.
8. Recent map history.
9. Head-to-head history.
10. Veto and match detail.
11. Score/result history.
12. Decision Inputs.
13. Data warnings.
14. Optional JSON block for factual data and decision inputs, only when requested.

Do not add prediction, probability, Veto hypothesis, score guess, EV, strategy, or betting fields inside the HLTV data pack.

If the user's overall request asks for judgment, this skill still only defines the factual data-pack step: the data pack must be output first. A host model may continue after a clear boundary, but that later judgment is outside this skill and must not alter source-backed facts.

A complete data pack must include proof that the model queried the structured data layer. If no API/warehouse/static JSON record was read after HLTV identity resolution, the output must be marked as partial and must include warning `structured_database_not_queried`.

## Language and Readability

User-facing Markdown should follow the user's language. If the user writes in Chinese, use Chinese section titles, table labels, warnings, and short explanatory notes. Keep JSON field names in English so the output remains stable for programs and downstream models.

Default Chinese Markdown structure:

1. `数据源执行记录`
   - HLTV 定位状态、结构化源状态、读取的结构化记录类别、字段来源。
   - A compliant output must internally record at least one exact structured record reference, such as an API endpoint, warehouse record ID, or static JSON path. Default public source examples include `matches/2394116/data-pack.json` and `teams/11861/map-details-overall.json`.
   - For normal human-facing reports, do not show raw URLs or exact database paths/endpoints. Show compact source status and freshness only. Exact references are reserved for JSON/debug/audit output.
2. `数据状态 / 数据缺口`
   - One compact table: source mode, retrieval time, data cutoff, completeness, high-impact missing fields.
   - Include date window. Default is current calendar year, e.g. `2026-01-01 至 2026-12-31`.
   - Start with a short sentence such as: `这是事实数据包，不包含预测、投注建议、EV 或仓位。`
3. `比赛信息`
   - Match ID, event, tier, LAN/online, BO format, schedule, status.
4. `队伍与阵容`
   - Team names, HLTV IDs, rank snapshots, starters, coach/stand-in flags.
5. `队伍与选手 Rating 3.0`
   - Annual HLTV Rating 3.0 and event HLTV Rating 3.0 when available.
   - JSON may keep the compatibility field name `rating2`; do not use `rating_2_0`.
   - Mark missing values as `缺失`, not as guessed values.
   - For match URL queries with visible lineups, annual rating and event rating are required fetch attempts. If either cannot be retrieved, show a per-player `rating_status` and add a warning.
6. `地图池`
   - Per-map comparison table.
   - Include sample size next to every win-rate field.
   - Show both raw win rate and weighted win rate when available.
7. `近期记录 / H2H`
   - Recent match rows and direct matchup rows when available.
   - If unavailable, say `未加载` or `HLTV 当前页面未提供`.
8. `警匪胜率`
   - CT-side and T-side win rates by team and map from HLTV team map stats pages, using the current calendar-year window by default, e.g. `/stats/teams/maps/<teamId>/<slug>?startDate=2026-01-01&endDate=2026-12-31` and detailed map pages.
   - If unavailable, say `未加载` and explain whether the source page did not expose it or the collector does not support it yet.
9. `Veto / 比分`
   - Observed veto steps, map order, score state, result state, or `赛前不可见`.
   - Do not include Veto predictions in this factual section.
10. `给模型的决策输入`
   - Factual inputs grouped into `地图池`, `对位数据`, `选手状态`, `阵容状态`, `警匪胜率`, `比赛环境`, `数据质量`.
   - Do not put predicted win rates or recommendations here.
11. `数据缺口`
   - Explicitly list missing data and what cannot be inferred.
12. `JSON`
   - Factual data and decision inputs with stable English keys.
   - Include this section only when the user asks for machine-readable output, data pack output, downstream LLM use, debug/audit output, or explicit JSON.
13. No `模型推理` inside the data pack
   - This skill defines only the data layer.
   - Do not put Veto prediction, winner lean, match percentage, map percentage, score guess, strategy, or betting content inside factual sections or JSON facts.
   - Anything after the data pack is outside this skill. If a model continues, use a visible separator such as `--- hltv-cs2-data 数据包结束 ---`.

Readability rules:

- Prefer compact tables over long paragraphs.
- Use `缺失`, `未加载`, `不可见`, `样本不足`, and `重建数据` consistently.
- Keep the top-level Markdown concise; put full machine-readable detail in JSON.
- Do not output English headings for a Chinese prompt unless the user asks for English.
- Do not show raw URLs, static JSON paths, endpoints, or full JSON in normal human-facing reports. Keep them only in debug/audit/JSON output.

## Phase 1 Fields

Phase 1 data packs must support these field groups when available:

- `teams`: identity, aliases, HLTV IDs, ranking snapshots.
- `match`: match ID, event, tier, LAN/online, format, schedule, status.
- `active_map_pool`: active map names used by the structured record. If omitted, consumers must derive it from unique map names in `maps` and `map_side_stats`.
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

## Exact Row Requirement

All factual numeric cells must be backed by exact rows from the structured record.

- Team-map stats require exact `team_hltv_id + map_name + data_type`.
- Player ratings require exact player rows.
- Event ratings require exact event/player rows.
- If a player/rating exact row is absent, set the field to `missing` / `缺失` / `无数据`.
- If a current active-map team row is absent because the team has no current-year sample in the selected filter, normalize it as a zero-sample active-map row: `sample_maps=0`, `wins=0`, `losses=0`, `raw_win_rate=0`, `pick_pct=0`, `ban_pct=0`, `total_rounds_played=0`, `rounds_won=0`, side/pistol/first-kill/first-death rates `0`, and `detail_status=zero_sample`.
- Do not infer missing team-map values from the opponent, another map, prior roster history, search snippets, or broad team form.
- A zero-sample active-map row is valid factual data, but it is not `数据充分`. A map cannot be marked `数据充分` unless both teams have non-zero usable exact rows for that map.

## Mandatory Map Field Mapping

When a static/API map-detail row exists, the human-facing output must use these exact field mappings. Do not invent alternate field names and then mark values as `0` or missing.

| 中文字段 | Required source field |
|:---|:---|
| 地图样本 / 比赛数 | `sample_maps` |
| 胜场 | `wins` |
| 平局 | `draws` when present, otherwise `0` / omitted |
| 负场 | `losses` |
| W-D-L / W-L | `wins`, `draws`, `losses` |
| 胜率 | `raw_win_rate`; if it is missing but `wins` and `sample_maps` exist, compute `wins / sample_maps` and label it as derived from exact row fields |
| Pick% | `pick_pct` |
| Ban% | `ban_pct` |
| CT 胜率 | `ct_side_win_rate` |
| T 胜率 | `t_side_win_rate` |
| 总回合 | `total_rounds_played` |
| 赢回合 | `rounds_won` |
| 手枪局样本 | `pistol_rounds` |
| 手枪局胜场 | `pistol_rounds_won` |
| 手枪局胜率 | `pistol_round_win_rate` |
| 首杀后回合胜率 | `first_kill_round_win_rate` |
| 首死后回合胜率 | `first_death_round_win_rate` |
| 赛事等级分布 | `event_tier_breakdown` |
| 数据状态 | `detail_status` and `side_stats_status` |

Common wrong mappings are forbidden:

- Do not use `matches_played`, `maps_played`, `played`, `win_percent`, or `map_win_rate` unless those exact fields are present in the fetched JSON.
- Do not convert an absent alias field into `0`. If the alias field is absent, look for the required source field above.
- Do not show `比赛数 0` when `sample_maps` is non-zero.
- Do not show `胜率 0.0%` when `raw_win_rate` is present or can be computed from `wins / sample_maps`.
- Do not omit CT/T, pistol, first-kill, first-death, total rounds, or event-tier breakdown when those fields exist in `map-details` or `matches/<id>/data-pack.json`.

For normal Chinese output, the minimum per-map detail table should include:

`队伍`, `地图`, `类型(overall/LAN)`, `样本`, `W-L`, `胜率`, `Pick%`, `Ban%`, `S/A/B/C 分布`, `CT/T`, `手枪局`, `首杀后`, `首死后`, `回合`.

For Chinese reports, use Chinese labels or Chinese+standard acronyms. Do not expose raw schema headers such as `Scope`, `sample`, `wr`, `pick_pct`, `ban_pct`, `first_kill`, or `first_death` unless the user asks for JSON/schema/debug output.

If the fetched `matches/<id>/data-pack.json` contains a `markdown` field, treat that Markdown as the canonical pre-rendered data skeleton. You may reformat it for readability, but you must not drop fields that are already rendered there.

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
    "date_window": {
      "start": "2026-01-01",
      "end": "2026-12-31",
      "policy": "current_calendar_year"
    },
    "as_of_date": null,
    "source_policy": "hltv_central_warehouse",
    "missing_fields": [],
    "warnings": []
  },
  "source_execution_log": {
    "hltv_match_page": {
      "url": "https://www.hltv.org/matches/2393335/faze-vs-furia-blast-rivals-2026-season-1",
      "status": "success",
      "source": "direct_hltv"
    },
    "structured_source": {
      "mode": "static_database",
      "capabilities_url": "https://raw.githubusercontent.com/smallmeji/hltv-cs2-data-skill/main/public-data/manifest.json",
      "capabilities_status": "success",
      "record_references": [
        "matches/2393335/data-pack.json",
        "teams/6667/map-details-overall.json",
        "teams/8297/map-details-overall.json",
        "events/8250/player-ratings.json"
      ]
    },
    "field_sources": {
      "match": "direct_hltv+static_database",
      "maps": "static_database",
      "map_side_stats": "static_database",
      "players": "static_database",
      "veto": "missing"
    }
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
  "active_map_pool": [
    "Ancient",
    "Anubis",
    "Dust2",
    "Inferno",
    "Mirage",
    "Nuke",
    "Overpass"
  ],
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
    "veto_prediction",
    "score_prediction",
    "strategy_recommendation",
    "betting_ev",
    "stake_sizing"
  ]
}
```

## Required Metadata

Each response must expose:

- `schema_version`.
- `requested_at`.
- `data_cutoff`.
- `source_policy`.
- `source_execution_log`.
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

- Inactive or absent maps appearing in direct fallback data.
- Fewer than 5 map samples in the requested filter.
- Missing lineup.
- Missing player rating.
- Coach or stand-in detected.
- Team recently changed roster.
- Event rating unavailable.
- Historical snapshot unavailable for `as_of_date`.
- Collector parse error or stale data.

## Data-Only Boundary

The data contract may include descriptive rates such as raw map win rate, weighted win rate, and sample count. These are historical data fields, not predictions.

Map-pool sections must use only the active map pool exposed by the structured record or derived from the record. Do not invent inactive maps such as `Vertigo`, `Cache`, or `Train` as current Veto variables. If an inactive map is shown for historical context, mark it `inactive_historical_map` and keep it outside current map-pool averages and Veto context.

Do not add model-derived fields inside factual objects such as `teams`, `match`, `maps`, `players`, `lineups`, `veto`, `scores`, or `metadata`.

`decision_inputs` are still factual fields. They may summarize and organize observed/derived facts for downstream models, but they must not contain predicted probabilities or recommended actions.

Do not output model-derived values anywhere in this skill, including:

- `predicted_map_win_probability`
- `predicted_match_win_probability`
- `winner_prediction`
- `winner_lean`
- `veto_hypothesis`
- `score_prediction`
- `fair_odds`
- `ev`
- `stake`

Betting-specific fields remain outside this skill.
