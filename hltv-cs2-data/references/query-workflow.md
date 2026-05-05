# Query Workflow

Use these recipes when preparing data packs.

## Team-vs-Team Query

User input:

```text
faze + 黑豹
```

or:

```text
用 hltv-cs2-data 帮我看一下 FaZe 和 G2 谁胜率高
```

Workflow:

1. If the prompt includes an event context, such as `PGL 上 Aurora 和 Heroic`, search/read HLTV event, upcoming, result, and match pages first to locate the exact match page.
2. If no exact match page is implied, resolve both teams from HLTV team pages first.
3. After match/team IDs are known, hydrate structured stats from configured API/warehouse if available.
4. If no API/warehouse is configured, read the default public static JSON database export. The first required URL is `https://raw.githubusercontent.com/smallmeji/hltv-cs2-data-skill/main/public-data/manifest.json`.
5. Use the manifest to fetch exact record paths. Prefer `/matches/<matchId>/data-pack.json` when an exact match is known. Otherwise use `/teams/index.json`, `/teams/<id>/summary.json`, `/teams/<id>/maps-overall.json`, `/teams/<id>/maps-lan.json`, `/teams/<id>/map-details-overall.json`, `/teams/<id>/map-details-lan.json`, `/teams/<id>/players.json`, and `/events/<eventId>/player-ratings.json` when available.
6. Add a `数据源执行记录` / `source_execution_log` section showing the HLTV page read, manifest status, exact database paths read, and field-level source labels.
7. If the manifest or at least one exact API/static database record was not read, stop before complete analysis. Add warning `structured_database_not_queried`; do not output map-pool detail, veto prediction, winner percentages, or a full pre-match report.
8. Only if the structured database source is unavailable or missing a field, attempt direct HLTV current-year team map summary, annual player stats, and event rating pages as supplemental fallback.
9. Fetch map stats for the requested tier/filter if available.
10. Fetch player ratings and lineup if a match is specified and the source is reachable.
11. Return Markdown + JSON factual data pack.
12. Build `Decision Inputs` from available facts.
13. If the user explicitly asks for judgment, apply `references/inference-gate.md`; append numeric `Model Inference` only when the gate passes.
14. Match the user's language in Markdown. For Chinese prompts, use Chinese section titles and table labels, while preserving JSON keys in English.

## Match URL Query

User input:

```text
https://www.hltv.org/matches/2393335/faze-vs-furia-blast-rivals-2026-season-1
```

Workflow:

1. Extract `hltvMatchId`.
2. Read the HLTV match page first. Resolve event, teams, format, schedule, status, lineup/veto/score when visible, and `eventId` when possible.
3. Read API data-pack when configured to hydrate structured fields.
4. If no API is configured, read the configured or default public static JSON database export. Start with `/matches/<hltvMatchId>/data-pack.json`; if missing, use team/event JSON files by resolved IDs. Mark merged fields as `static_database`.
5. The static database export must start with `/manifest.json`, then exact record paths. Do not treat a 404 from the base directory as data absence.
6. Add `source_execution_log` with:
   - `hltv_match_page_status`
   - `database_manifest_url`
   - `database_manifest_status`
   - `database_record_paths`
   - `field_sources`
7. If `database_manifest_status` is not `success` and no API record was read, output only a partial HLTV pack with `structured_database_not_queried`. Do not produce `地图池总览`, `逐图详细分析`, veto prediction, numeric match probability, or a complete report.
8. Resolve both canonical team IDs and slugs from match-page team links, static team index, or team pages. Do not stop at match-page summary data.
9. Fetch lineups from the match page or static/API pack when visible/available.
10. Always attempt to fetch player ratings for visible starters:
   - Extract or resolve `eventId` from the match page/event link whenever possible.
   - Event rating must be fetched from `https://www.hltv.org/stats/players?event=<eventId>` when `eventId` is known.
   - Annual rating from HLTV team player stats for the current calendar year, e.g. `https://www.hltv.org/stats/teams/players/<teamId>/<slug>?startDate=2026-01-01&endDate=2026-12-31`. If `as_of_date` is provided, use that date's calendar year.
   - If a player is missing from event stats, keep annual rating if available and mark `rating_event` as `缺失`.
   - If a coach or stand-in has no rating, mark `rating_status` explicitly instead of guessing.
11. Fetch yearly team map stats from API/static database first. If unavailable or missing, then attempt HLTV team stats pages with the current calendar-year window, e.g. `https://www.hltv.org/stats/teams/maps/<teamId>/<slug>?startDate=2026-01-01&endDate=2026-12-31`.
12. In static/API mode, use exported `map-details` when present for CT/T, pistol, first kill, and first death fields. In direct lightweight fallback mode, stop at the team map summary page and mark `map_side_stats` as `Pro/API only` or `未加载` unless the map detail page was explicitly loaded.
13. Use match-page map stats only as `recent_core_context` or fallback if the yearly stats pages cannot be reached. Label its time window explicitly.
14. Fetch head-to-head map rows when reachable; if not reachable, mark `head_to_head` as `未加载`.
15. Include veto/scores only if available and visible for the requested mode.
16. For Chinese prompts, output the compact Chinese structure: `数据源执行记录`, `数据状态`, `比赛信息`, `队伍与阵容`, `选手数据`, `地图池总览`, `逐图详细分析`, `特殊 Veto 变量`, `近期记录 / H2H`, `警匪胜率`, `Veto / 比分`, `给模型的决策输入`, `数据缺口`, `JSON`.
17. If the user asked for probability or winner judgment, apply `references/inference-gate.md`. If the gate fails, add `core_data_insufficient_for_numeric_inference` and do not give exact percentages.

For stronger/weaker or win-rate judgment requests, replace the single `地图池` section with:

1. `地图池总览`: one compact table across all maps.
2. `逐图详细分析`: one subsection per playable map. Use overall/LAN sample, W-L, raw win rate, CT/T, pistol, first-kill/first-death, rounds, pick/ban, and data-quality notes.
3. `特殊 Veto 变量`: maps with one side missing current-year data, extreme ban rate, or tiny sample. These should not be treated as normal map edges.

This is required when static/API map-detail fields are available. In direct HLTV mode, if map-detail fields are unavailable, state that the per-map detail section is limited to summary fields.

Active map-pool rule:

- Derive the active map list from the structured record. If no explicit `active_map_pool` field exists, use the unique map names found in `maps` and `map_side_stats`.
- For the current 2026 public export, the expected active maps are `Ancient`, `Anubis`, `Dust2`, `Inferno`, `Mirage`, `Nuke`, and `Overpass`.
- Do not add `Vertigo`, `Cache`, `Train`, or any other inactive/absent map to `地图池总览`, `逐图详细分析`, or `特殊 Veto 变量`.
- `特殊 Veto 变量` is only for active maps that are in the structured record but have unusual sample / ban / no-data behavior.

## Event Ratings Query

User input:

```text
event=8250 player ratings
```

Workflow:

1. Fetch event player ratings snapshot.
2. Use the canonical HLTV URL format `https://www.hltv.org/stats/players?event=<eventId>`.
3. Match players to lineups when a match is provided.
4. Mark missing players explicitly.
5. If a lineup player is missing from the event page, attempt annual rating fallback before marking the player fully missing.
6. Do not average or interpret rating unless the user asks for a descriptive aggregate.

## Backtest Query

User input:

```text
Backtest match 2393335 as of 2026-04-30 22:30 Asia/Shanghai
```

Workflow:

1. Convert local time to UTC.
2. Fetch only records visible before cutoff.
3. Exclude final score and post-match records.
4. Mark reconstructed fields.
5. Return data pack with `backtest_context`.

## Missing Data Response

If the data warehouse/API is not available:

```text
当前无法生成完整 hltv-cs2-data 数据包，因为没有成功读取结构化数据库/API/静态 JSON 记录。
可输出的数据：...
缺失的数据：...
不能推断的内容：...
```

Do not fall back to private prediction reports, private notebooks, or hidden strategy documents as substitute data sources.

Do not use Liquipedia, Liquidpedia, wikis, news snippets, or search summaries as substitute structured data sources. They can be external context only, not the source for map stats, player ratings, CT/T, veto, or result fields.

For Chinese output, missing data should be explicit and brief:

```text
数据缺口：
- 赛事 rating：未加载
- Veto：赛前不可见
- 警匪胜率：HLTV team map stats 页面未加载或 collector 暂未采集
- 精确 as_of 快照：不可用，本次只能标记为重建数据
- 结构化数据库记录：未读取时必须标 `structured_database_not_queried`

不能推断：
- 不能把缺失 rating 补成估计值
- 不能把历史胜率当成预测胜率
- 不能在没有数据库读取记录时输出完整逐图分析、Veto 预测或具体胜率
```

## Model Inference Request

User input:

```text
Use hltv-cs2-data to collect G2 vs FaZe data, then estimate map and match win rates.
```

Workflow:

1. Produce the HLTV factual data pack first.
2. Build `Decision Inputs` from facts such as map pool, H2H, player form, roster state, match context, and data quality.
3. If map-detail data is available, include `逐图详细分析` and `特殊 Veto 变量` before inference.
4. Keep all facts and decision inputs out of inference fields.
5. Apply `references/inference-gate.md`.
6. Add numeric `Model Inference` only when the gate passes.
7. If the gate fails, output a short note such as `核心地图/rating 数据不足，不能给可靠具体胜率百分比`; qualitative direction is allowed only if explicitly requested and must be marked low confidence.
8. State clearly that inference is model-derived and not HLTV data.
9. Do not include betting EV, Kelly, stake, or private correction logic unless the user explicitly invokes a separate strategy or betting framework.
