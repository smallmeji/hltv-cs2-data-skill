# Query Workflow

Use these recipes when preparing data packs.

## Required First Move

Do not begin the analytical body from HLTV text alone. HLTV lookup is only the identity step; database hydration is the first data step.

Compact rule for external models:

```text
HLTV identity -> structured manifest/capabilities -> exact match/team records -> factual data pack -> optional non-skill judgment
```

Do not invert this order. A correct HLTV lineup, event name, or bracket note does not prove the database was queried.

For every match/team comparison query:

1. Resolve match/team identity from HLTV or from the user's explicit IDs.
2. Hydrate from a structured source. Source priority:
   - configured API / warehouse
   - user-provided static JSON export
   - default public raw GitHub static export
3. For the default public raw GitHub export, fetch `https://raw.githubusercontent.com/smallmeji/hltv-cs2-data-skill/main/public-data/manifest.json`.
   - Do not call `https://raw.githubusercontent.com/smallmeji/hltv-cs2-data-skill/main/public-data` directly; raw GitHub directories can return `404`.
   - Do not call `hltv-cs2-data-platform` or platform-site public-data URLs; those are stale sources. The accepted public source is `hltv-cs2-data-skill` raw GitHub or its documented compatibility alias.
   - A directory `404` is not evidence that the database is unavailable. Try the manifest and exact files.
4. For alternate API/static sources, use the equivalent capabilities/search/data-pack endpoint or record. Do not require GitHub path names if the source exposes the same canonical fields.
5. If the prompt contains event/team names but no exact match ID, use match search/index before downgrading to team-only comparison. For the default public source this is `matches/index.json`.
6. Fetch exact structured records for the resolved entities. When the match index/search returns one clear match, fetch that match data pack first.
7. Only then write `地图池总览`, `逐图详细分析`, `队伍与选手 Rating 3.0`, or `给模型的决策输入`.

If the structured source capabilities/manifest or exact records cannot be fetched, output a short partial-facts response with `structured_database_not_queried`. Do not produce a full pre-match report.

Do not use direct HLTV deep stats pages as the primary replacement for this step. Direct HLTV map/player/stat pages are only supplemental fallback after the database/API/static records were attempted and logged.

HLTV match-page lineup is an identity fact. It may be correct even when no database query happened. Do not treat a correct lineup as proof that structured map/player data was loaded.

Structured stats require exact structured records. If the output contains map samples, CT/T, pistol, first-kill/first-death, Pick/Ban, annual ratings, or event ratings, each value must be traceable to an exact API/static record or explicitly marked as direct HLTV fallback. Player rating values are HLTV Rating 3.0 in human output. The default public JSON uses `rating2` as a compatibility field name; alternate sources may use another raw name, but human output must not call it Rating 2.0 unless the source explicitly says it is HLTV Rating 2.0.

Invalid-output guard:

- If the report says `公开数据源 smallmeji.github.io/hltv-cs2-data-platform 返回 404`, retry with an accepted `hltv-cs2-data-skill` manifest. Do not continue.
- If a structured match data pack exists, using only HLTV data is non-compliant.
- If the factual data pack contains `Veto 预测`, `禁图/选图预测框架`, possible map sequence, winner probability, or downstream recommendations, it is outside this skill.
- If the report labels player ratings as `Rating 2.0`, retry and label them `Rating 3.0`.
- If the report source line only says `HLTV CS2 数据平台` or another vague source label without `已读取 public static database export` or `已读取 API/warehouse`, retry the source execution log and structured fetch.
- If the report says CT/T, pistol, first-kill, first-death, Pick/Ban, or map details are missing while a fetched structured match data pack contains them, retry from the data-pack/record.

External public sources are allowed for match-background facts only. Official event pages, Liquipedia/wiki pages, news snippets, search summaries, and market pages may help confirm starters, stand-ins, coaches, format, stage, bracket context, schedule, LAN/online status, venue, or roster-change context. Label these fields as `external_context` when they are not from HLTV/database. Do not use them for map rows, win rates, CT/T, pistol, first-kill/first-death, Pick/Ban, player ratings, H2H, recent rows, veto, scores, or results.

## Query Types

Classify the request before fetching records:

| Query type | User wording | Required identity | Required records | Match-only fields |
|:--|:--|:--|:--|:--|
| `match_data_pack` | HLTV match URL or exact scheduled match | match ID + both teams | match data-pack when available, both team records | observed only |
| `team_comparison` | two teams, no confirmed match | both team IDs | both team records | `not_applicable` |
| `hypothetical_match` | "假设 X vs Y", "如果打 BO3", user-specified tier/format | both team IDs + assumptions | both team records | `not_applicable` unless provided |
| `single_team_profile` | one team only | one team ID | one team's records | `not_applicable` |

Do not fail just because no match page exists. A missing match page only disables match-specific fields; it does not block team-level map/player data when team records exist.

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
2. If no exact match page is implied, resolve both teams from HLTV team pages and/or the structured source's team resolver/index.
3. After match/team IDs are known, immediately hydrate structured stats from the selected structured source. Source priority is configured API/warehouse, then user-provided static JSON, then default public raw GitHub export.
4. If the selected source exposes capabilities or a manifest, read it first. For the default public source, the manifest URL is `https://raw.githubusercontent.com/smallmeji/hltv-cs2-data-skill/main/public-data/manifest.json`.
5. If the prompt includes event context, use the source's match search/index and try to locate a single exact exported match. For the default public source this is `/matches/index.json`; if found, treat the query as `match_data_pack` and read the row's match data-pack reference, such as `data_pack_path`.
6. Use the source capabilities/manifest/index to fetch exact records or endpoints. Prefer a match data pack when an exact match is known. Otherwise fetch equivalent team summary, overall map summary, LAN map summary, overall map details, LAN map details, player ratings, and event player ratings when available. The default public source names these records `/matches/<matchId>/data-pack.json`, `/teams/<id>/summary.json`, `/teams/<id>/maps-overall.json`, `/teams/<id>/maps-lan.json`, `/teams/<id>/map-details-overall.json`, `/teams/<id>/map-details-lan.json`, `/teams/<id>/players.json`, and `/events/<eventId>/player-ratings.json`.
7. Add a `数据源执行记录` / `source_execution_log` section showing the HLTV identity step, structured source mode, record categories read, and field-level source labels. Exact paths/endpoints stay internal unless debug/audit output is requested.
8. If the source capabilities/manifest and at least one exact API/static/warehouse record were not read, stop before complete analysis. Add warning `structured_database_not_queried`; do not output map-pool detail, veto prediction, winner percentages, or a full pre-match report.
9. If the resolved match data pack contains a `markdown` field, use it as the canonical data skeleton. It is already rendered from the structured rows and should prevent field mapping mistakes. You may summarize or localize it, but do not drop rendered fields such as CT/T, pistol, first-kill, first-death, total rounds, Pick/Ban, or event-tier breakdown. If these fields appear in the data-pack/markdown, saying they are `未加载` is non-compliant.
10. Before writing map tables, run the mandatory field audit:
   - `比赛数 = sample_maps`
   - `W-L = wins-losses` or `W-D-L = wins-draws-losses`
   - `胜率 = raw_win_rate`, or `wins / sample_maps` only when `raw_win_rate` is missing
   - `Pick% = pick_pct`
   - `Ban% = ban_pct`
   - `CT/T = ct_side_win_rate / t_side_win_rate`
   - `手枪局 = pistol_round_win_rate`
   - `首杀后 = first_kill_round_win_rate`
   - `首死后 = first_death_round_win_rate`
   - `总回合 / 赢回合 = total_rounds_played / rounds_won`
   - `赛事等级分布 = event_tier_breakdown`
11. For match-background gaps only, public external context may be used and labeled `external_context`.
12. Only if the structured database source was attempted and is unavailable or missing a field, attempt direct HLTV current-year team map summary, annual player stats, and event rating pages as supplemental fallback.
13. Fetch map stats for the requested tier/filter if available.
14. Fetch player ratings and lineup if a match is specified and the source is reachable.
15. Return a Markdown factual report by default. Include JSON only when the user asks for machine-readable output, data-pack output, downstream LLM use, debug/audit output, or explicit JSON.
16. Build `Decision Inputs` from available facts.
17. If the user explicitly asks for judgment, this skill still only defines the data step: return the factual data pack first. A clearly separated downstream model judgment may follow after the skill boundary.
18. Match the user's language in Markdown. For Chinese prompts, use Chinese section titles and table labels, while preserving JSON keys in English.

Chinese source-log requirement:

The first normal section must be:

```text
## 数据源执行记录

- 技能触发：hltv-cs2-data
- 身份定位：HLTV / static match index / user input
- 结构化数据：已读取 public static database export
- 读取记录：match data-pack / team map details / player ratings / event ratings
- 输出边界：事实数据包优先；数据包之后不属于本 skill
```

Do not replace this with a single sentence such as `数据源：HLTV CS2 数据平台`.

## Hypothetical Match Query

User input:

```text
假设 NAVI 和 G2 打一场 S 级 LAN BO3，给我数据包
```

or:

```text
如果 Aurora vs Heroic 在 PGL 打，谁的数据更好？
```

Workflow:

1. Set `query_type=hypothetical_match` unless a real HLTV match is found quickly from the named event/date.
2. Resolve both teams through HLTV team pages and/or `teams/index.json`.
3. Fetch the selected structured source's capabilities/manifest and both teams' exact records/endpoints:
   - team summary
   - overall map summary
   - LAN map summary
   - overall map details
   - LAN map details
   - player ratings
   - default public source examples: `/teams/<id>/summary.json`, `/teams/<id>/maps-overall.json`, `/teams/<id>/maps-lan.json`, `/teams/<id>/map-details-overall.json`, `/teams/<id>/map-details-lan.json`, `/teams/<id>/players.json`
4. Record user-provided context as assumptions, for example `tier=S`, `format=BO3`, `match_environment=LAN`, `event=PGL`, `date=unknown`.
5. If a real event ID is explicitly known, attempt `/events/<eventId>/player-ratings.json`; otherwise mark event rating `not_applicable_without_event_id`.
6. Output `数据源执行记录`, `假设条件`, `数据状态 / 数据缺口`, `队伍与选手 Rating 3.0`, `地图池总览`, `逐图详细分析`, `特殊 Veto 变量`, and `给模型的决策输入`.
7. Mark Veto, map order, score, match status, and official lineup as `not_applicable` unless the user supplied them or a real match page was resolved.
8. Do not put winner lean, probability, Veto prediction, score guess, or downstream recommendations inside the factual data pack.

## Single-Team Query

User input:

```text
用 hltv-cs2-data 看一下 Aurora 这个队伍的数据
```

or:

```text
给我 FaZe 2026 的地图详细数据
```

Workflow:

1. Set `query_type=single_team_profile`.
2. Resolve the team through HLTV and/or `teams/index.json`.
3. Fetch the selected structured source's capabilities/manifest and the team's exact records/endpoints:
   - team summary
   - overall map summary
   - LAN map summary
   - overall map details
   - LAN map details
   - player ratings
   - default public source examples: `/teams/<id>/summary.json`, `/teams/<id>/maps-overall.json`, `/teams/<id>/maps-lan.json`, `/teams/<id>/map-details-overall.json`, `/teams/<id>/map-details-lan.json`, `/teams/<id>/players.json`
4. Output `数据源执行记录`, `队伍信息`, `数据状态 / 数据缺口`, `选手 rating`, `地图池总览`, `逐图详细数据`, `近期地图记录` if exported, and `给模型的决策输入`.
5. Mark opponent, event rating, Veto, score, and match status as `not_applicable` unless the user adds match/event context.
6. Do not infer team strength beyond source-backed facts.

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
3. Fetch the selected structured source's capabilities/manifest.
4. If HLTV search/page access did not produce a match ID, use the source's match search/index and search `search_text`, event name, team names, and date/status. In the default public source this is `matches/index.json`. If one clear exported match exists, use that row's `hltv_match_id` and match data-pack reference such as `data_pack_path`.
5. If a match data-pack record/endpoint exists, read it first. In the default public source this is `matches/<hltvMatchId>/data-pack.json` or the index row's `data_pack_path`.
6. Read or attempt both teams' equivalent records/endpoints for summary, overall map details, LAN map details, and players. Default public source examples are `teams/<id>/summary.json`, `teams/<id>/map-details-overall.json`, `teams/<id>/map-details-lan.json`, and `teams/<id>/players.json`.
7. If `eventId` is known, read or attempt event player ratings. In the default public source this is `events/<eventId>/player-ratings.json`.
8. If the match data-pack has a `markdown` field, use that pre-rendered Markdown as the starting skeleton and keep all factual fields it already contains. Do not mark CT/T, pistol, first-kill, first-death, rounds, Pick/Ban, or tier breakdown as missing when the skeleton contains them.
9. Output the compact Chinese structure:
   - `数据源执行记录`
   - `数据状态 / 数据缺口`
   - `比赛信息`
   - `队伍与选手 Rating 3.0`
   - `地图池总览`
   - `逐图详细分析`
   - `特殊 Veto 变量`
   - `给模型的决策输入`
10. Keep the factual data pack compact and source-bound. Custom descriptive sections such as `赛事参赛分布`, `最近30天状态`, `最近对手质量`, or `地图池深度` are allowed only as derived data summaries after the factual sections, with source labels and calculation windows. If exact rows are missing, mark the metric `未加载`.
11. Keep the source log at the top. A source log at the end is non-compliant.
12. End the skill data pack after the factual sections. Anything after the data pack is outside this skill. If the host model continues, add a visible boundary before judgment.

## Match URL Query

User input:

```text
https://www.hltv.org/matches/2393335/faze-vs-furia-blast-rivals-2026-season-1
```

Workflow:

1. Extract `hltvMatchId`.
2. Read the HLTV match page first. Resolve event, teams, format, schedule, status, lineup/veto/score when visible, and `eventId` when possible.
3. Stop HLTV browsing after identity/visible facts are resolved. Read API data-pack when configured to hydrate structured fields.
4. If no API is configured, read the configured static JSON source or default public static JSON database export. Start with the source's match data-pack endpoint/record; for the default public source this is `/matches/<hltvMatchId>/data-pack.json`. If missing, use team/event records by resolved IDs. Mark merged fields as `static_database`.
5. A static database export should start with a manifest/capabilities file, then exact records. For the default public source this is `/manifest.json`, then exact record paths. Do not treat a 404 from a base directory as data absence.
6. Add `source_execution_log` with:
   - `hltv_match_page_status`
   - `structured_source_mode`
   - `structured_source_status`
   - `record_categories_read`
   - `field_sources`
7. If the structured source was not successfully read and no API/static/warehouse record was read, output only a partial HLTV pack with `structured_database_not_queried`. Do not produce `地图池总览`, `逐图详细分析`, veto prediction, numeric match probability, or a complete report.
8. Resolve both canonical team IDs and slugs from match-page team links, static team index, or team pages. Do not stop at match-page summary data.
9. Fetch lineups from the match page or static/API pack when visible/available. If lineup is unclear, public external context may be used for starters/stand-in/coaches/roster notes and labeled `external_context`.
10. Always attempt to fetch player ratings from structured records first. If static/API ratings are missing, then use direct HLTV fallback for visible starters:
   - Extract or resolve `eventId` from the match page/event link whenever possible.
   - Event rating must be fetched from `https://www.hltv.org/stats/players?event=<eventId>` when `eventId` is known.
   - Annual rating from HLTV team player stats for the current calendar year, e.g. `https://www.hltv.org/stats/teams/players/<teamId>/<slug>?startDate=2026-01-01&endDate=2026-12-31`. If `as_of_date` is provided, use that date's calendar year.
   - Display both as HLTV `Rating 3.0`; keep the structured `rating2` key for compatibility.
   - If a player is missing from event stats, keep annual rating if available and mark `rating_event` as `缺失`.
   - If a coach or stand-in has no rating, mark `rating_status` explicitly instead of guessing.
11. Fetch yearly team map stats from API/static database first. If unavailable or missing, then attempt HLTV team stats pages with the current calendar-year window, e.g. `https://www.hltv.org/stats/teams/maps/<teamId>/<slug>?startDate=2026-01-01&endDate=2026-12-31`.
12. In static/API mode, use exported `map-details` when present for CT/T, pistol, first kill, and first death fields. In direct HLTV partial fallback, stop at the team map summary page and mark `map_side_stats` as `Pro/API only` or `未加载` unless the map detail page was explicitly loaded.
13. Use match-page map stats only as `recent_core_context` or fallback if the yearly stats pages cannot be reached. Label its time window explicitly.
14. Fetch head-to-head map rows when reachable; if not reachable, mark `head_to_head` as `未加载`.
15. Include veto/scores only if available and visible for the requested mode.
16. For Chinese prompts, output the compact Chinese structure: `数据源执行记录`, `数据状态`, `比赛信息`, `队伍与阵容`, `选手数据`, `地图池总览`, `逐图详细分析`, `特殊 Veto 变量`, `近期记录 / H2H`, `警匪胜率`, `Veto / 比分`, `给模型的决策输入`, `数据缺口`, and optional `JSON` only when requested.
17. If the user asked for probability or winner judgment, finish this skill's data pack first. Anything after the data pack is outside this skill.

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

For stronger/weaker or win-rate comparison requests, replace the single `地图池` section with:

1. `地图池总览`: one compact table across all maps.
2. `逐图详细分析`: one subsection per playable map. Use overall/LAN sample, W-L, raw win rate, CT/T, pistol, first-kill/first-death, rounds, pick/ban, and data-quality notes.
3. `特殊 Veto 变量`: maps with one side at zero sample, extreme ban rate, or tiny sample. These should not be treated as normal map edges.

This is required when static/API map-detail fields are available. In direct HLTV mode, if map-detail fields are unavailable, state that the per-map detail section is limited to summary fields.

Active map-pool rule:

- Derive the active map list from the structured record. If no explicit `active_map_pool` field exists, use the unique map names found in `maps` and `map_side_stats`.
- For the current 2026 public export, the expected active maps are `Ancient`, `Anubis`, `Dust2`, `Inferno`, `Mirage`, `Nuke`, and `Overpass`.
- Do not add `Vertigo`, `Cache`, `Train`, or any other inactive/absent map to `地图池总览`, `逐图详细分析`, or `特殊 Veto 变量`.
- `特殊 Veto 变量` is only for active maps that are in the structured record but have unusual sample / ban / zero-sample behavior.

Exact row rule:

- Each map/team value must come from an exact structured row keyed by `team_hltv_id`, `map_name`, and `data_type`.
- If the exact row is absent because the team has no current-year sample on an active map, render a zero-sample row instead of `missing`: `样本=0`, `W-L=0-0`, `胜率=0%`, `Pick%=0%`, `Ban%=0%`, `CT/T=0%/0%`, `手枪局=0%`, `首杀后=0%`, `首死后=0%`, `回合=0/0`, `状态=0样本`.
- Do not infer a missing team-map row from the opponent's row, another map, HLTV snippets, rankings, or team form.
- Do not call a map `数据充分` unless both teams have usable exact rows for that map.
- If one team is zero-sample on an active map, move the map into `特殊 Veto 变量` or clearly mark the zero-sample side in `地图池总览`.

Chinese label rule:

- Use Chinese headers or Chinese+standard acronyms in normal Chinese reports: `范围`, `队伍`, `地图`, `样本`, `W-L`, `地图胜率`, `Pick%`, `Ban%`, `CT/T`, `手枪局`, `首杀后`, `首死后`, `回合`.
- Avoid raw schema headers such as `Scope`, `sample`, `wr`, `pick_pct`, `ban_pct`, `first_kill`, or `first_death` unless the user explicitly requests raw JSON/schema/debug output.

Data-pack-first comparison contract:

For prompts like `谁胜率高`, `who is favored`, `estimate win rate`, or `判断谁更强`, the skill output must still begin with data and use this order:

1. `数据源执行记录`
2. `数据状态 / 数据缺口`
3. `比赛信息`
4. `队伍与选手 Rating 3.0`
5. `地图池总览`
6. `逐图详细分析`
7. `特殊 Veto 变量`
8. `给模型的决策输入`
9. `JSON` only when the user requests machine-readable output, data pack output, downstream LLM use, debug/audit output, or explicit JSON.

Do not skip factual sections. If a section's source is missing, output the section with `缺失` / `未加载` and warning codes.

If the host model answers the judgment part, it must add it after a boundary:

```text
--- hltv-cs2-data 数据包结束 ---
以下为非本 skill 的模型判断：
```

That second section may compare strength, but it must not be presented as HLTV fact or overwrite the data pack.

Source display:

- For normal human-facing reports, keep `数据源执行记录` compact: source type, status, freshness, and record categories read.
- `数据源执行记录` must appear before map analysis and decision inputs.
- Do not print raw manifest URLs, GitHub raw URLs, database record paths/endpoints, or full JSON in a normal report unless the user asks for debug/audit/source details.
- Exact source paths/endpoints still belong in internal `source_execution_log` and optional JSON.
- If the output source says only `HLTV.org`, search, wiki, or news snippets and no structured source was read, stop and emit partial facts only. Do not continue into full report sections.

Factual-section boundary:

- Do not put `模型推理` inside the factual data pack.
- Do not put Veto prediction, winner lean, map-win probability, match-win probability, score guess, or downstream recommendations inside factual tables, `给模型的决策输入`, or JSON facts.
- Anything after this data pack is outside `hltv-cs2-data`.

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

## Prediction / Judgment Request

User input:

```text
Use hltv-cs2-data to collect G2 vs FaZe data, then estimate map and match win rates.
```

Workflow:

1. Produce the HLTV factual data pack.
2. Build `Decision Inputs` from facts such as map pool, H2H, player form, roster state, match context, and data quality.
3. If map-detail data is available, include `逐图详细分析` and `特殊 Veto 变量`.
4. End the skill data pack. Do not place probability, winner lean, Veto prediction, score guess, or downstream recommendations inside factual sections or JSON facts.
5. Anything after the factual data pack is outside `hltv-cs2-data`.
