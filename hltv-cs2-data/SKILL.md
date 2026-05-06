---
name: hltv-cs2-data
description: "Use for CS2 factual data packs from an HLTV-derived structured database/static JSON export: match/team resolution, map history, map detail, player ratings, lineup, veto, results, event ratings, decision-input context, or backtest context. HLTV pages are only identity/visible-fact sources; map/player/detail analysis must read configured API/warehouse or public static JSON records first. Data-only: no predictions, probabilities, Veto hypotheses, score guesses, betting, EV, Kelly, or strategy conclusions."
metadata:
  short-description: Strategy-neutral HLTV CS2 data packs
---

# HLTV CS2 Data

This skill returns factual, source-bound CS2 data packs for downstream models or user-owned strategy. It does not decide who wins.

Current public-data source version: `static-raw-2026-05-06`.

Required default manifest URL:

```text
https://raw.githubusercontent.com/smallmeji/hltv-cs2-data-skill/main/public-data/manifest.json
```

This is the only default public static database entry point. Do not derive another host. If a model reports a GitHub Pages / platform-site public-data URL, treat it as stale instructions and return to the raw GitHub manifest above.

## Non-Negotiable Rules

1. **Data-only boundary**: never output winner predictions, match/map probabilities, Veto predictions, score guesses, betting advice, odds analysis, EV, Kelly, stake sizing, or strategy recommendations.
2. **HLTV is identity first**: use HLTV pages to resolve match ID, team IDs, event ID, schedule, format, visible lineup, visible veto, score, or result.
3. **Structured data first for analysis**: map pools, map detail, CT/T, pistol, first kill/death, Pick/Ban, player ratings, recent rows, H2H, and decision inputs must come from exact API/warehouse/static JSON records when available.
4. **Manifest gate**: before writing map tables, player ratings, per-map detail, or decision inputs, fetch the manifest and at least one exact JSON/API record.
5. **Match index for natural language**: if the user gives event/team names without a match URL, fetch `matches/index.json` from the static export after the manifest and search it before falling back to team-only comparison. When a single matching row is found, fetch that row's `data_pack_path` first.
6. **Fail closed**: if manifest plus exact records cannot be read, output partial HLTV facts only with `structured_database_not_queried`.
7. **Exact rows only**: if a team-map/player row is absent, write `missing` / `无数据`. Do not infer it from opponent data, another map, rankings, snippets, or memory.
8. **Current active map pool only**: for 2026, use `Ancient`, `Anubis`, `Dust2`, `Inferno`, `Mirage`, `Nuke`, `Overpass` unless structured records explicitly say otherwise.
9. **Normal reports hide raw paths**: include compact source status. Show exact URLs, record paths, and JSON only for debug/audit/machine-readable requests.
10. **Rating field and label**: structured JSON currently stores player rating in `rating2` for compatibility. Treat `rating2` as the exported HLTV `Rating 3.0` value unless a record explicitly says otherwise. Do not look for or emit `rating_2_0`; human reports should label the column `Rating 3.0`.

Boundary sentence when the user asks for judgment:

```text
本 skill 只输出数据层；胜率判断由调用模型或用户策略完成。
```

## Query Routing

Classify the request first, then load only the relevant reference.

| Query type | Examples | Required reference |
|:--|:--|:--|
| `match_data_pack` | HLTV match URL; exact scheduled match | `references/query-workflow.md`, optionally `references/data-pack-contract.md` |
| `team_comparison` | `NAVI 和 G2`, `FaZe + FURIA` | `references/query-workflow.md` |
| `hypothetical_match` | `假设 NAVI 和 G2 打 S 级 LAN BO3` | `references/query-workflow.md` |
| `single_team_profile` | `看一下 Aurora 的数据` | `references/query-workflow.md` |
| `event_ratings` | `event=8250 player ratings` | `references/query-workflow.md` |
| `backtest` | `as_of_date`, historical cutoff | `references/backtest-mode.md` |
| product/API design | API, collector, schema, warehouse | `references/product-brief.md`, `references/api-contract.md`, `references/collector-contract.md` |
| data availability/debug | 404, CF, missing fields, source modes | `references/standalone-mode.md`, `references/data-availability.md` |

Do not bulk-load all references.

## Default Workflow

1. Resolve identity:
   - If a match URL exists, read the HLTV match page and extract `hltvMatchId`, teams, event, format, time, status, visible lineup/veto/score, and event ID when visible.
   - If event/team names are given, search/read HLTV match/event/upcoming/results/team pages only long enough to resolve canonical IDs, then use `matches/index.json` to find an exported exact match before treating the request as team-only.
   - If no real match exists, use `teams/index.json` after manifest fetch and mark the request as `team_comparison`, `hypothetical_match`, or `single_team_profile`.
2. Fetch structured data:
   - Fetch the required manifest.
   - For natural-language match lookup, fetch `matches/index.json`; if exactly one row matches the event/team context, fetch its `data_pack_path`.
   - Prefer `matches/<matchId>/data-pack.json` for known matches.
   - Fetch exact team records as needed:
     - `teams/<id>/summary.json`
     - `teams/<id>/maps-overall.json`
     - `teams/<id>/maps-lan.json`
     - `teams/<id>/map-details-overall.json`
     - `teams/<id>/map-details-lan.json`
     - `teams/<id>/players.json`
   - If event ID is known, attempt `events/<eventId>/player-ratings.json`.
3. Write a compact `数据源执行记录` before analysis.
4. Use database/API/static rows for factual tables.
5. Mark missing or not-applicable fields explicitly.
6. Output Markdown by default. Include JSON only if requested for machine-readable, downstream-model, debug, or audit use.

If a match data-pack contains a `markdown` field, use that Markdown as the canonical factual skeleton. Do not state that CT/T, pistol, first-kill, first-death, rounds, Pick/Ban, or tier breakdown are missing when those values are present in the data-pack.

## 404 Handling

When a public-data request returns `404`:

1. If the URL is the raw GitHub `/public-data` directory, retry exact `/public-data/manifest.json`.
2. If the URL is not the required raw GitHub manifest host/path, treat it as stale instructions and do not use it.
3. If `manifest.json` works but an exact record path 404s, use manifest/index files to discover available records.
4. If `manifest.json` cannot be read, stop with `structured_database_unavailable` / `structured_database_not_queried`.

Do not turn a 404 into a full HLTV-only report.

## Output Shape

For Chinese comparison or match requests, use this compact order:

1. `数据源执行记录`
2. `数据状态 / 数据缺口`
3. `比赛信息` or `假设条件` / `队伍信息`
4. `队伍与选手 Rating 3.0`
5. `地图池总览`
6. `逐图详细分析`
7. `特殊 Veto 变量`
8. `近期记录 / H2H` when exact rows exist
9. `Veto / 比分` only for observed facts or `not_applicable`
10. `给模型的决策输入`
11. `JSON` only when requested

For English prompts, mirror the same structure in English.

Normal source log example:

```text
数据源执行记录：
- HLTV 定位：成功
- 结构化数据：已读取 public static database export
- 读取记录：match data-pack + team map details + player ratings
- 字段来源：地图详情=static_database，赛事信息=direct_hltv，Veto=missing
```

If structured records were not read:

```text
当前只能定位 HLTV 基础信息；结构化数据库/API/静态 JSON 未成功读取，不能生成完整 hltv-cs2-data 报告。
warning: structured_database_not_queried
```

## Special Modes

### Single-Team Profile

Use when the user asks for one team only. Resolve one team ID and read the team records. Do not require a match ID. Opponent, event rating, Veto, score, and match status are `not_applicable` unless the user provides match/event context.

### Hypothetical Match

Use when the user asks an assumed matchup. Resolve both team IDs and read both teams' records. Treat user-provided tier, format, LAN/online, date, event, or map subset as `assumptions`, not observed HLTV facts. Do not invent lineup, Veto, map order, score, event ratings, or H2H rows.

### Direct HLTV Partial Fallback

Use only after structured data was attempted and failed or lacks a field. Direct fallback can report visible facts and missing warnings only; it must not produce full map tables, per-map detail, numeric inference, or a prediction-style report.

## Reference Map

- `references/query-workflow.md`: detailed recipes for match URL, team comparison, single-team, hypothetical match, event ratings, and exact record paths.
- `references/data-pack-contract.md`: stable Markdown/JSON output contract and field mapping.
- `references/decision-inputs.md`: how to organize factual decision inputs.
- `references/standalone-mode.md`: public standalone mode and external-model failure handling.
- `references/data-availability.md`: what each access mode can and cannot provide.
- `references/backtest-mode.md`: cutoff and time-travel rules.
- `references/api-contract.md`: hosted API shape.
- `references/collector-contract.md`: collector/warehouse requirements.
- `references/product-brief.md`: product positioning.
- `references/inference-gate.md`: compatibility note for older no-prediction references.
