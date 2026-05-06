# Query Workflow

Use these recipes when preparing data packs.

## Required First Move

Do not begin the analytical body from HLTV text alone. HLTV lookup is only the identity step; database hydration is the first data step.

For every match/team comparison query:

1. Resolve match/team identity from HLTV or from the user's explicit IDs.
2. Fetch `https://smallmeji.github.io/hltv-cs2-data-platform/public-data/latest/manifest.json`.
   - Do not call `https://smallmeji.github.io/hltv-cs2-data-platform/public-data/latest` directly; raw GitHub directories can return `404`.
   - A directory `404` is not evidence that the database is unavailable. Try the manifest and exact files.
3. Fetch exact static/API records for the resolved entities.
4. Only then write `地图池总览`, `逐图详细分析`, `队伍与选手 rating`, or `给模型的决策输入`.

If the manifest or exact records cannot be fetched, output a short partial-facts response with `structured_database_not_queried`. Do not produce a full pre-match report.

Do not use direct HLTV deep stats pages as the primary replacement for this step. Direct HLTV map/player/stat pages are only supplemental fallback after the database/API/static records were attempted and logged.

HLTV match-page lineup is an identity fact. It may be correct even when no database query happened. Do not treat a correct lineup as proof that structured map/player data was loaded.

Structured stats require database records. If the output contains map samples, CT/T, pistol, first-kill/first-death, Pick/Ban, annual ratings, or event ratings, each value must be traceable to an exact API/static record or explicitly marked as direct HLTV fallback.

External public sources are allowed for match-background facts only. Official event pages, Liquipedia/wiki pages, news snippets, search summaries, and market pages may help confirm starters, stand-ins, coaches, format, stage, bracket context, schedule, LAN/online status, venue, or roster-change context. Label these fields as `external_context` when they are not from HLTV/database. Do not use them for map rows, win rates, CT/T, pistol, first-kill/first-death, Pick/Ban, player ratings, H2H, recent rows, veto, scores, or results.

Alias discipline:

- Resolve aliases through the HLTV match page/team page or `teams/index.json`.
- If the prompt says `M8` and the match page says `Gentle Mates`, use HLTV team `13404`.
- Do not map `M8` to `M80` (`12376`) or to a similarly named opponent unless HLTV identity resolution confirms it.

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
2. If no exact match page is implied, resolve both teams from HLTV team pages and/or the public database `teams/index.json`.
3. After match/team IDs are known, immediately hydrate structured stats from configured API/warehouse if available.
4. If no API/warehouse is configured, read the default public static JSON database export. The first required URL is `https://smallmeji.github.io/hltv-cs2-data-platform/public-data/latest/manifest.json`.
5. Use the manifest to fetch exact record paths. Prefer `/matches/<matchId>/data-pack.json` when an exact match is known. Otherwise use `/teams/index.json`, `/teams/<id>/summary.json`, `/teams/<id>/maps-overall.json`, `/teams/<id>/maps-lan.json`, `/teams/<id>/map-details-overall.json`, `/teams/<id>/map-details-lan.json`, `/teams/<id>/players.json`, and `/events/<eventId>/player-ratings.json` when available.
6. Add a `数据源执行记录` / `source_execution_log` section showing the HLTV page read, manifest status, exact database paths read, and field-level source labels.
7. If the manifest or at least one exact API/static database record was not read, stop before complete analysis. Add warning `structured_database_not_queried`; do not output map-pool detail, veto prediction, winner percentages, or a full pre-match report.
8. For match-background gaps only, public external context may be used and labeled `external_context`.
9. Only if the structured database source was attempted and is unavailable or missing a field, attempt direct HLTV current-year team map summary, annual player stats, and event rating pages as supplemental fallback.
10. Fetch map stats for the requested tier/filter if available.
11. Fetch player ratings and lineup if a match is specified and the source is reachable.
12. Return a Markdown factual report by default. Include JSON only when the user asks for machine-readable output, data-pack output, downstream LLM use, debug/audit output, or explicit JSON.
13. Build `Decision Inputs` from available facts.
14. If the user explicitly asks for judgment, do not answer with a winner lean or probability. Return the factual data pack and boundary note: `本 skill 只输出数据层；胜率判断由调用模型或用户策略完成。`
15. Match the user's language in Markdown. For Chinese prompts, use Chinese section titles and table labels, while preserving JSON keys in English.

## One-Sentence Judgment Query

User input:

```text
用 hltv-cs2-data 看 PGL Astana MOUZ vs Gentle Mates 谁胜率高
```

or:

```text
用 hltv-cs2-data 帮我看一下 PGL 上 Aurora 和 Heroic 谁更强
```

Workflow:

1. Treat this as a full data-pack request. Do not ask the user for JSON paths, API paths, or a match URL unless multiple plausible matches remain after lookup.
2. Find the exact HLTV match when an event context is present. Extract `hltvMatchId`, `eventId` if visible, team IDs/slugs, format, schedule, stage/status, and visible lineup/veto/score.
3. Fetch the public database manifest.
4. If `matches/<hltvMatchId>/data-pack.json` exists, read it first.
5. Read or attempt both teams' records:
   - `teams/<id>/summary.json`
   - `teams/<id>/map-details-overall.json`
   - `teams/<id>/map-details-lan.json`
   - `teams/<id>/players.json`
6. If `eventId` is known, read or attempt `events/<eventId>/player-ratings.json`.
7. Output the compact Chinese structure:
   - `数据源执行记录`
   - `数据状态 / 数据缺口`
   - `比赛信息`
   - `队伍与选手 rating`
   - `地图池总览`
   - `逐图详细分析`
   - `特殊 Veto 变量`
   - `给模型的决策输入`
8. Keep the factual data pack compact and source-bound. Custom descriptive sections such as `赛事参赛分布`, `最近30天状态`, `最近对手质量`, or `地图池深度` are allowed only as derived data summaries after the factual sections, with source labels and calculation windows. If exact rows are missing, mark the metric `未加载`.
9. Keep the source log at the top. A source log at the end is non-compliant.

## Match URL Query

User input:

```text
https://www.hltv.org/matches/2393335/faze-vs-furia-blast-rivals-2026-season-1
```

Workflow:

1. Extract `hltvMatchId`.
2. Read the HLTV match page first. Resolve event, teams, format, schedule, status, lineup/veto/score when visible, and `eventId` when possible.
3. Stop HLTV browsing after identity/visible facts are resolved. Read API data-pack when configured to hydrate structured fields.
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
9. Fetch lineups from the match page or static/API pack when visible/available. If lineup is unclear, public external context may be used for starters/stand-in/coaches/roster notes and labeled `external_context`.
10. Always attempt to fetch player ratings from structured records first. If static/API ratings are missing, then use direct HLTV fallback for visible starters:
   - Extract or resolve `eventId` from the match page/event link whenever possible.
   - Event rating must be fetched from `https://www.hltv.org/stats/players?event=<eventId>` when `eventId` is known.
   - Annual rating from HLTV team player stats for the current calendar year, e.g. `https://www.hltv.org/stats/teams/players/<teamId>/<slug>?startDate=2026-01-01&endDate=2026-12-31`. If `as_of_date` is provided, use that date's calendar year.
   - If a player is missing from event stats, keep annual rating if available and mark `rating_event` as `缺失`.
   - If a coach or stand-in has no rating, mark `rating_status` explicitly instead of guessing.
11. Fetch yearly team map stats from API/static database first. If unavailable or missing, then attempt HLTV team stats pages with the current calendar-year window, e.g. `https://www.hltv.org/stats/teams/maps/<teamId>/<slug>?startDate=2026-01-01&endDate=2026-12-31`.
12. In static/API mode, use exported `map-details` when present for CT/T, pistol, first kill, and first death fields. In direct HLTV partial fallback, stop at the team map summary page and mark `map_side_stats` as `Pro/API only` or `未加载` unless the map detail page was explicitly loaded.
13. Use match-page map stats only as `recent_core_context` or fallback if the yearly stats pages cannot be reached. Label its time window explicitly.
14. Fetch head-to-head map rows when reachable; if not reachable, mark `head_to_head` as `未加载`.
15. Include veto/scores only if available and visible for the requested mode.
16. For Chinese prompts, output the compact Chinese structure: `数据源执行记录`, `数据状态`, `比赛信息`, `队伍与阵容`, `选手数据`, `地图池总览`, `逐图详细分析`, `特殊 Veto 变量`, `近期记录 / H2H`, `警匪胜率`, `Veto / 比分`, `给模型的决策输入`, `数据缺口`, and optional `JSON` only when requested.
17. If the user asked for probability or winner judgment, do not provide a probability or winner conclusion. Return the data pack and state that judgment is outside this data-only skill.

Concrete record example:

For `https://www.hltv.org/matches/2394118/mouz-vs-gentle-mates-pgl-astana-2026`, after HLTV identity resolution, the model should attempt at minimum:

- `/matches/2394118/data-pack.json`
- `/teams/4494/summary.json`
- `/teams/4494/map-details-overall.json`
- `/teams/4494/map-details-lan.json`
- `/teams/4494/players.json`
- `/teams/13404/summary.json`
- `/teams/13404/map-details-overall.json`
- `/teams/13404/map-details-lan.json`
- `/teams/13404/players.json`

If those records are not read, do not output a full MOUZ vs Gentle Mates report. Correct HLTV starters alone are insufficient.

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

Exact row rule:

- Each map/team value must come from an exact structured row keyed by `team_hltv_id`, `map_name`, and `data_type`.
- If the exact row is absent, write `无数据` / `missing` for that team-map cell.
- Do not infer a missing team-map row from the opponent's row, another map, HLTV snippets, rankings, or team form.
- Do not call a map `数据充分` unless both teams have usable exact rows for that map.
- If one team lacks an exact row on an active map, move the map into `特殊 Veto 变量` or clearly mark the missing side in `地图池总览`.

Data-only comparison contract:

For prompts like `谁胜率高`, `who is favored`, `estimate win rate`, or `判断谁更强`, the output must still be data-only and use this order:

1. `数据源执行记录`
2. `数据状态 / 数据缺口`
3. `比赛信息`
4. `队伍与选手 rating`
5. `地图池总览`
6. `逐图详细分析`
7. `特殊 Veto 变量`
8. `给模型的决策输入`
9. `JSON` only when the user requests machine-readable output, data pack output, downstream LLM use, debug/audit output, or explicit JSON.

Do not skip factual sections. If a section's source is missing, output the section with `缺失` / `未加载` and warning codes.

Source display:

- For normal human-facing reports, keep `数据源执行记录` compact: source type, status, freshness, and record categories read.
- `数据源执行记录` must appear before map analysis and decision inputs.
- Do not print raw manifest URLs, GitHub raw URLs, database record paths, or full JSON in a normal report unless the user asks for debug/audit/source details.
- Exact source paths still belong in internal `source_execution_log` and optional JSON.
- If the output source says only `HLTV.org`, search, wiki, or news snippets and no structured source was read, stop and emit partial facts only. Do not continue into full report sections.

No-prediction rules:

- Do not output `模型推理`.
- Do not output Veto prediction, winner lean, map-win probability, match-win probability, score guess, betting advice, odds analysis, EV, Kelly, stake sizing, or max buy price.
- If the user asks for judgment, add only: `本 skill 只输出数据层；胜率判断由调用模型或用户策略完成。`

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

## Prediction Request

User input:

```text
Use hltv-cs2-data to collect G2 vs FaZe data, then estimate map and match win rates.
```

Workflow:

1. Produce the HLTV factual data pack.
2. Build `Decision Inputs` from facts such as map pool, H2H, player form, roster state, match context, and data quality.
3. If map-detail data is available, include `逐图详细分析` and `特殊 Veto 变量`.
4. Do not output probability, winner lean, Veto prediction, score guess, strategy, or betting content.
5. Add the boundary note: `本 skill 只输出数据层；胜率判断由调用模型或用户策略完成。`
