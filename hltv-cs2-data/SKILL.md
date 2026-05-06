---
name: hltv-cs2-data
description: "Use when a user needs the hltv-cs2 product/data skill: CS2 factual data packs from an HLTV-derived structured database/static JSON export, including team resolution, match data, map pool history, player ratings, veto, lineup, event ratings, score history, decision-input context, or backtest/time-travel context. HLTV pages may be used for minimal identity lookup and visible match facts, but map/player/detail analysis MUST read the configured API/warehouse or public static JSON database records first. This skill prepares Markdown reports and optional JSON data outputs for downstream user-owned analysis. It is data-only: do not output predictions, probabilities, Veto hypotheses, score guesses, betting advice, EV, Kelly, or strategy conclusions."
metadata:
  short-description: Strategy-neutral HLTV CS2 data packs
---

# HLTV CS2 Data

## Purpose

`hltv-cs2-data` is the skill for the `hltv-cs2` product concept: an HLTV-derived CS2 multidimensional data guide that returns facts and model-ready decision inputs only.

Primary analysis data must come from the structured data layer: configured warehouse/API when available, otherwise the default public static JSON database export. Public HLTV pages are used to locate canonical match/team/event IDs and verify visible match facts such as schedule, lineup, format, stage, veto, map order, score, or result. Other public external sources may be used only to confirm match-background facts, not structured stats. Direct HLTV stats pages are only supplemental fallback after the structured source has been attempted and field-level missing data has been recorded.

Do not extend or reuse any private prediction, betting, or strategy framework. This skill has no built-in prediction model and must not produce final winner judgments. If the user asks "who is favored" or "who has higher win rate", interpret the request as "return the data needed for the user's own model to decide".

The skill must be usable by someone who only installs the skill and has no access to any private database, scraper, or API.

## Database-First Hydration Rule

For this skill, "find the data" means:

1. Use HLTV only as the identity layer: match ID, event ID, team IDs, team slugs, visible starters, veto, score, and status.
2. Immediately hydrate structured stats from the database/API/static JSON layer.
3. Only after structured records are read may the model write map-pool tables, player-rating tables, per-map detail analysis, and factual decision inputs.

Do not browse HLTV deep stats pages as the first source for maps, ratings, CT/T, pistol, first-kill, first-death, Pick/Ban, recent rows, or H2H when a static/API database source is available or may be available.

For team-name queries without a match URL, the model still must read the database export:

1. Resolve candidate teams through HLTV and/or `teams/index.json`.
2. Fetch `manifest.json`.
3. Fetch exact team records such as `teams/<id>/summary.json`, `teams/<id>/map-details-overall.json`, `teams/<id>/map-details-lan.json`, and `teams/<id>/players.json`.
4. If an event or match is resolved, fetch `events/<eventId>/player-ratings.json` or `matches/<matchId>/data-pack.json`.

If the model cannot read `manifest.json` and at least one exact static/API record, it must stop with `structured_database_not_queried`. Correct HLTV lineup, rankings, news, search snippets, or market pages are not enough.

## One-Sentence Query Mode

The user should be able to trigger the full workflow with one sentence. Do not require the user to provide raw JSON URLs, API paths, or a manual step list.

Examples:

```text
用 hltv-cs2-data 看 PGL Astana MOUZ vs Gentle Mates 谁胜率高
用 hltv-cs2-data 帮我看一下 PGL 上 Aurora 和 Heroic 谁更强
Use hltv-cs2-data to compare G2 vs FaZe for BLAST Rivals.
```

For one-sentence requests, automatically perform this sequence:

1. Resolve the exact match when event context is present. Search/read HLTV match, event, upcoming, or results pages for the canonical `hltvMatchId`, `eventId`, teams, format, schedule, status, and visible lineup/veto/score.
2. Fetch the public static database manifest.
3. Prefer `matches/<hltvMatchId>/data-pack.json` if the match exists in the export.
4. Always fetch or attempt both teams' `summary.json`, `map-details-overall.json`, `map-details-lan.json`, and `players.json`.
5. If an `eventId` is known, fetch or attempt `events/<eventId>/player-ratings.json`.
6. Produce the compact factual data pack. If the user's wording asks for a judgment such as `谁胜率高`, `谁更强`, `favored`, or `probability`, do not answer with a probability or winner lean; instead output the factual comparison and say that prediction is outside this data-only skill.

If multiple plausible matches are found and the event/date cannot disambiguate them, ask one concise clarification question. Otherwise do not ask for extra URLs; use the database export and report missing records.

## Non-Negotiable Retrieval Checklist

Before writing any map pool, player rating, per-map detail, or decision-input section, the model must complete this checklist:

1. Locate/verify the match or teams on HLTV.
2. Fetch the public static database manifest: `https://smallmeji.github.io/hltv-cs2-data-platform/public-data/latest/manifest.json`.
   - Do not fetch `.../public-data` as if it were an API endpoint. Raw GitHub directory URLs can return `404`; that does not mean the database export is unavailable.
   - The manifest file is the required first structured-data fetch.
3. Fetch at least one exact static/API record after identity resolution, such as:
   - `teams/<hltvTeamId>/summary.json`
   - `teams/<hltvTeamId>/players.json`
   - `teams/<hltvTeamId>/map-details-overall.json`
   - `teams/<hltvTeamId>/map-details-lan.json`
   - `matches/<hltvMatchId>/data-pack.json`
4. In the answer, include a compact `数据源执行记录` saying whether structured data was read.

If step 2 or step 3 fails, stop. Do not continue into a full report. Output only:

```text
当前只能定位 HLTV 基础信息；结构化数据库/API/静态 JSON 未成功读取，不能生成完整 hltv-cs2-data 报告。
warning: structured_database_not_queried
```

Red flags for non-compliant output:

- `数据来源：HLTV.org 官方数据` with no structured source status.
- Source execution log appears only at the end of a long report instead of before analysis.
- Current-year map tables or player ratings with no `static_database` / `api_warehouse` field source.
- A normal report that includes a full JSON block without the user asking for JSON.
- A full pre-match analysis generated from search summaries, wiki/news snippets, or market pages.
- Sections such as `最近30天状态`, `赛事分布`, `对手质量`, `核心选手优势`, or `地图池深度` that contain numeric values but do not identify exact structured source rows.
- A report titled or structured as `深度分析报告` that presents conclusions before `数据源执行记录`.

## Identity Facts vs Structured Stats

HLTV match pages are allowed and preferred for identity facts:

- current match URL and `hltvMatchId`
- event, schedule, format, status
- match phase/stage, bracket context, LAN/online context when visible
- visible starters / lineup
- visible veto, map order, score, or result

Other public external sources, such as official event pages, Liquipedia/wiki pages, news snippets, search results, or market pages, may be used only as match-background context when clearly labeled `external_context`. Allowed background fields include:

- likely/announced starters, stand-ins, coaches, and roster notes
- event name, tournament phase, bracket position, playoff/group context
- match format, schedule/timezone, LAN/online status, venue/country
- public roster-change notes that explain lineup uncertainty

These sources must never be used to fill structured analytical fields such as map rows, sample counts, win rates, CT/T, pistol, first-kill, first-death, Pick/Ban, player ratings, H2H tables, or recent-map rows. If a background fact conflicts with HLTV or the database, label the conflict instead of silently choosing one.

However, a correct lineup does not satisfy the structured-data requirement. The following fields must come from exact static/API database records when available:

- current-year map rows, sample counts, W-D-L, raw win rate, weighted win rate, Pick %, Ban %
- CT/T side rates, pistol rate, first-kill and first-death round conversion
- annual player ratings and event ratings
- exported recent map rows, H2H rows, match data packs, and decision input tables

If the output has correct HLTV lineup but no successful static/API database fetch, it is still only `direct_hltv_partial`. It must not present itself as complete, and it must not produce full per-map analysis or numeric winner probabilities.

When a user uses aliases, resolve the canonical HLTV team identity before reading database records. Example: `M8` / `Gentle Mates` must resolve to HLTV team `13404` when that is the match-page team. Do not confuse it with `M80` (`12376`) or unrelated teams that merely contain similar strings.

## Source Policy

- Default public static base URL: `https://smallmeji.github.io/hltv-cs2-data-platform/public-data/latest`. Default public manifest URL: `https://smallmeji.github.io/hltv-cs2-data-platform/public-data/latest/manifest.json`.
- Default resolution order for normal users: minimal direct HLTV match/event/team lookup for identity and visible facts -> configured warehouse/API for structured stats if available -> default public static JSON database export for structured stats -> direct HLTV deep stats only as last-resort supplemental fallback for missing fields. The structured database/API/static step is mandatory before analysis sections.
- Default date window: current calendar year only, e.g. 2026-01-01 to 2026-12-31 for the current 2026 season, unless the user explicitly requests another window.
- Preferred structured source: HLTV-derived static JSON database export or central data warehouse/API. The default public static source is the public database export for external models.
- Original upstream source: HLTV pages only.
- Do not use private display websites as a data source. Display sites are presentation surfaces, not product data interfaces.
- Do not use Liquipedia, Liquidpedia, wikis, news snippets, search summaries, or market pages as the structured data source. They may be used as labeled `external_context` for match-background facts such as lineup notes, format, stage, schedule, venue, or bracket context, but they must never replace map stats, player ratings, CT/T, pistol, first-kill, first-death, Pick/Ban, H2H, recent rows, veto, score, or result fields when HLTV/database records are expected.
- Do not require end users to run a local database, scraper, local browser, CDP session, or Playwright session.
- Public standalone mode uses the host model's normal public web/page-reading/search capability only for HLTV discovery and identity resolution. It must then read the structured static JSON database export or configured API/warehouse for map/player/detail fields.
- Direct HLTV-only output is a degraded partial fallback. It can report visible match facts and missing fields, but it must not produce a complete report, per-map detail analysis, veto prediction, or numeric inference.
- Retrieve only data visible from HLTV pages during the session and label unavailable deeper fields as missing.
- A central collector/API may improve completeness and backtesting later, but it is not required for normal skill use.
- If Cloudflare or access controls block collection, fail safely and report freshness/missing data. Do not fabricate data or bypass protections.
- Treat the default public static JSON source as the hosted HLTV-derived public database export for structured stats, not merely optional background context. When both direct HLTV and database/cache data are available, preserve field-level source labels.

## Mandatory Structured Data Gate

For any complete match data pack, team comparison, per-map detail analysis, or decision-input section, the model must prove it queried a structured data source after resolving HLTV identity.

Required evidence in the output:

- `database_manifest_url` or API base URL.
- `database_manifest_status`: `success`, `failed`, or `not_configured`.
- At least one exact `database_record_path`, such as `matches/2394116/data-pack.json`, `teams/11861/map-details-overall.json`, or `events/8250/player-ratings.json`.
- Field-level source labels such as `direct_hltv`, `static_database`, `api_warehouse`, `direct_hltv_fallback`, or `missing`.

For normal human-facing reports, do not print raw manifest URLs, database paths, or full JSON unless the user asks for debug/audit/source details or machine-readable output. Instead show a compact source status such as `结构化数据：已读取 public static database export；记录：match data-pack + team map details` and keep exact paths in internal `source_execution_log` / optional JSON.

A report whose only source is `HLTV.org`, search summaries, news snippets, wiki pages, or market pages is `direct_hltv_partial`. It is not compliant as a full `hltv-cs2-data` report.

If this evidence is absent, the output is not compliant with this skill. In that case:

- Do not call the result a complete `hltv-cs2-data` data pack.
- Add warning code `structured_database_not_queried`.
- Output only partial HLTV facts and missing-source warnings.
- Do not output `地图池总览`, `逐图详细分析`, veto prediction, match winner percentages, map win percentages, or a full pre-match report.
- Do not use Liquipedia, wiki pages, news snippets, search summaries, or market prices to replace the missing structured database query.

## Active Map Pool Gate

All map-pool sections must be restricted to the active maps present in the structured data record. For the current 2026 export, the expected active pool is:

```text
Ancient, Anubis, Dust2, Inferno, Mirage, Nuke, Overpass
```

Rules:

- Build the active map list from the structured data pack when available, using unique map names from `maps`, `map_side_stats`, or an explicit `active_map_pool` field.
- Do not introduce inactive or absent maps such as `Vertigo`, `Cache`, or `Train` into `地图池总览`, `逐图详细分析`, or `特殊 Veto 变量` unless the user explicitly asks for historical inactive-map records.
- If an inactive historical map appears in a direct HLTV fallback page, label it as `inactive_historical_map` and exclude it from current BO3/BO5 map-pool and veto analysis.
- `特殊 Veto 变量` means unusual behavior inside the current active pool only. A map not present in the structured record must not be invented as a veto variable.

## Exact Row Data Gate

Map and player fields must be row-exact. A value exists only when the structured record contains that exact team/player + map/stat row.

Rules:

- For each team-map cell, require an exact row keyed by `team_hltv_id`, `map_name`, and `data_type`.
- If a team has no row for a map, output `无数据` / `missing`; do not infer values from the opponent, another map, old inactive maps, search snippets, or overall team form.
- Do not fill `sample_maps`, win rate, CT/T, pistol, first-kill, first-death, pick %, or ban % unless that exact row exists.
- If one side lacks a map row, move that map into `特殊 Veto 变量` or mark the cell as missing; do not call it `数据充分`.
- Player ratings require exact player rows. If event rating or annual rating is missing for a player, mark that field as `缺失`.

Example: if `Heroic + Anubis + overall` is absent from `matches/2394116/data-pack.json`, Heroic Anubis must be `无数据`, even if Aurora has Anubis data or Heroic has other map data.

## Data-Only Boundary

This skill is a data provider. It should not decide the match, create probabilities, predict Veto, guess scores, recommend strategy, or output betting content.

Use this boundary:

- **Factual data pack**: strict. Numeric cells must come from exact structured records or be marked missing.
- **Decision inputs**: still factual. They can group and summarize source-backed facts for downstream models, but they must not contain winner leans, map-win probabilities, match-win probabilities, Veto predictions, score guesses, or recommendations.
- **External match background**: allowed for context only and labeled `external_context`.
- **Prediction / strategy layer**: outside this skill. The user's own model may use the data elsewhere, but this skill must not write that layer.

Descriptive derived sections are allowed when all of these are true:

1. They appear after the factual data pack and are clearly labeled `derived_from_structured_data` or included under `给模型的决策输入`.
2. They state which factual inputs they used, such as map rows, player ratings, rank, lineup, CT/T, pistol, recent rows, or external background context.
3. They do not present model-created metrics as HLTV facts.
4. They do not fill missing raw fields with invented numbers.
5. They do not conclude who will win or estimate probability.

Allowed by default:

- `比赛信息`
- `队伍与阵容`
- `队伍与选手 rating`
- `地图池总览`
- `逐图详细分析`
- `特殊 Veto 变量`
- `近期记录 / H2H` only when exact recent/H2H rows exist
- `Veto / 比分` only for observed facts or `赛前不可见`
- `给模型的决策输入`

Allowed as derived analysis, but not as raw fact tables unless exact source rows exist:

- `最近30天战绩`
- `赛事参赛分布`
- `最近对手质量`
- `S-Tier 表现`
- `核心选手评分优势` beyond exact annual/event rating rows
- `地图池深度` as a scored conclusion before the factual map table

If exact recent/event/opponent-quality rows exist, the model may show derived tables with a clear source label and calculation window. If they do not exist, mark the metric `未加载`. Do not create new numeric tables from memory, snippets, or unlogged calculations.

## Data-Only Output Contract

When the user asks who is stronger, who has higher win rate, who is favored, or requests probabilities, the output must still be data-only. Translate that request into a structured factual comparison that another model or the user can judge from. Do not produce a free-form "pre-match deep analysis report".

Mandatory Chinese section order for comparison requests:

1. `数据源执行记录`
2. `数据状态 / 数据缺口`
3. `比赛信息`
4. `队伍与选手 rating`
5. `地图池总览`
6. `逐图详细分析`
7. `特殊 Veto 变量`
8. `给模型的决策输入`
9. `JSON` only when requested

Rules:

- `数据源执行记录` is mandatory and must list manifest/API status and exact record paths.
- `数据源执行记录` must be the first substantive section. Do not put it at the end after conclusions.
- The title should describe a data pack or comparison, not a free-form `深度分析报告`, unless the user explicitly requested a narrative report.
- In normal human-facing reports, `数据源执行记录` should be compact and should not expose raw URLs or database paths. It must still say whether structured data was read. Exact paths are for debug/audit mode or JSON output only.
- `队伍与选手 rating` must show available annual ratings and event ratings. If event ratings, lineup, or confirmed starters are missing, show `缺失` and add warnings instead of omitting the section.
- `Veto / 比分` is factual only: visible veto steps, map order, scores, or `赛前不可见`. Veto prediction must not appear there.
- Do not output Veto prediction, winner lean, map-win probability, match-win percentage, score guess, or qualitative "favored" conclusion.
- If the user asks directly for a prediction, answer with the data pack and one short boundary note: `本 skill 只输出数据层；胜率判断由调用模型或用户策略完成。`
- `JSON` is mandatory only when the user asks for product-ready output, downstream LLM use, API-like output, debug/audit output, or machine-readable data. For a normal human-readable report, omit the full JSON block by default and offer it as available on request.
- Do not output betting advice, odds analysis, EV, Kelly, stake sizing, or max buy price.

## Operating Modes

- **Public standalone mode**: default mode for users without an API key. Use HLTV pages only to locate/verify the match, teams, event, IDs, lineup/veto/score if visible; then read the default public static JSON database export for structured map/player/detail fields. Output normal Markdown by default. Include JSON only when the user asks for machine-readable or downstream-model output. No private database, scraper, local browser, or CDP required.
- **Static JSON database mode**: required structured-data mode for external users when no API is configured. After resolving match/team identity from HLTV, read `https://smallmeji.github.io/hltv-cs2-data-platform/public-data/latest/manifest.json`, then read standardized JSON files under the same base, such as `/teams/<hltvTeamId>/summary.json`, `/teams/<hltvTeamId>/map-details-overall.json`, `/events/<eventId>/player-ratings.json`, or `/matches/<matchId>/data-pack.json`.
- **Direct HLTV partial fallback**: degraded fallback when the structured source cannot be read or has no exact record. It may output visible match facts and missing-data warnings only. It must not output `地图池总览`, `逐图详细分析`, full prediction-style reports, veto predictions, or numeric probabilities.
- **Pro / API mode**: enhanced mode for users with `HLTV_CS2_API_BASE_URL` and `HLTV_CS2_API_KEY`. After resolving match/team identity from HLTV or user-provided IDs, call the API for standardized data packs, exact snapshots, veto, lineup, result, and backtest support.
- **Internal collector mode**: backend maintenance mode only. It may use persistent browser profiles or CDP to collect HLTV stats into a central warehouse, but this is not part of the public standalone skill user experience.
- **Design mode**: when asked to design collector/API/backend behavior, use product and collector references.

## Workflow

1. Classify the request:
   - Product/API brief or scope clarification.
   - Team identity resolution.
   - Match data pack.
   - Team-vs-team comparison.
   - Event player rating data.
   - Backtest/time-travel data pack.
   - Collector/API design or validation.
2. Load only the needed reference:
   - Product positioning: `references/product-brief.md`.
   - Standalone use: `references/standalone-mode.md`.
   - Data availability by access mode: `references/data-availability.md`.
   - Data pack schema: `references/data-pack-contract.md`.
   - Decision input schema: `references/decision-inputs.md`.
   - API contract: `references/api-contract.md`.
   - HLTV collector requirements: `references/collector-contract.md`.
   - Backtest rules: `references/backtest-mode.md`.
   - Query examples: `references/query-workflow.md`.
3. Resolve identity, then hydrate database records before any analysis:
   - If the user provides an HLTV match URL, open/read that HLTV match page first and extract `hltvMatchId`, teams, event, format, schedule, status, lineup/veto/score if visible, and `eventId` when possible.
   - If the user gives a natural-language query such as `PGL 上 Aurora 和 Heroic 谁胜率高`, search/read HLTV match/event/result/upcoming pages first to find the relevant match page. Do not start from static JSON unless HLTV cannot locate the match.
   - After a match page or canonical team IDs are resolved, stop browsing and hydrate structured stats from the configured warehouse/API when available.
   - If no API/warehouse is configured, the next required step is the default public static JSON database export: `https://smallmeji.github.io/hltv-cs2-data-platform/public-data/latest/manifest.json`. Do not stop after HLTV page facts and do not continue to deep HLTV stats before trying this manifest.
   - Read `/matches/<matchId>/data-pack.json`, `/teams/<hltvTeamId>/*.json`, and `/events/<eventId>/player-ratings.json` as available. Mark field-level source as `static_database`.
   - Record these attempts in `数据源执行记录` / `source_execution_log`. If the manifest or static/API records were not read, stop before complete analysis and add `structured_database_not_queried`.
   - If the match page cannot be found/read on HLTV, use the public static database export to reverse-search exported matches/teams.
   - If the structured database/API/static source is unavailable or missing a field, add `structured_database_unavailable` or `static_record_not_found`, then direct HLTV deep stats may be attempted as supplemental fallback with field-level warning labels.
   - If neither can retrieve enough data, output a partial pack with missing-source warnings.
4. Match the user's language for user-facing Markdown. If the user writes in Chinese, output Chinese headings, labels, and warnings by default. Keep JSON field names stable in English.
5. Emit normal Markdown by default. Emit JSON only when the user asks for product-ready output, downstream LLM use, API-like output, debug/audit output, or explicit JSON.
6. Include metadata, freshness, source URLs, cutoff time, sample sizes, and warnings.
7. Keep the data pack factual and organize `Decision Inputs` as factual features. If the user explicitly asks for probability, winner judgment, or strategy, do not provide that conclusion inside this skill; output the data and boundary note instead.
8. When a page or field cannot be read, use the data availability matrix to label whether the failure belongs to direct HLTV partial fallback, in-app/browser session mode, internal collector mode, or API/warehouse mode.
9. Do not output numeric model inference. Missing fields should be reported as data quality warnings, not converted into low-confidence predictions.

## Output Rules

Every complete structured data pack should include:

- `metadata`: query, source, retrieved time, data cutoff, version, missing fields.
- `source_execution_log`: HLTV lookup result, database/API manifest result, exact static/API record paths read, field-level source labels, and fallback status.
- `teams`: resolved team identity, aliases, HLTV IDs, rank snapshots.
- `match`: event, schedule, format, LAN/online context, status.
- `lineups`: expected or confirmed players, stand-ins, coaches, missing ratings.
- `players`: annual and event ratings when available.
- `maps`: map stats, match history, pick/ban data, sample sizes, tier filters.
- `map_side_stats`: per-team, per-map CT-side and T-side win rates from HLTV team map stats pages when available.
- `head_to_head`: direct team-vs-team history when available.
- `recent_matches`: recent map and match rows used for downstream modeling.
- `match_detail`: format, event, date, veto, map order, scores when available.
- `side_scores`: match-specific CT/T score splits when visible or available through API/warehouse.
- `decision_inputs`: model-ready factual factors such as map pool, head-to-head, player form, roster state, match context, and data quality.
- `warnings`: small sample, stale data, roster changes, missing event data, low confidence parsing.
- `not_included`: explicit note that prediction, probability, strategy, and betting fields are not part of the HLTV fact data pack.

If the structured database/API/static source was not read, do not emit a complete data pack. Emit a partial-facts response with warning `structured_database_not_queried`.

For Chinese user-facing output, prefer this compact Markdown order:

1. `数据源执行记录`: HLTV 定位是否成功、manifest/API 是否成功、是否读取了结构化数据。普通报告只显示紧凑状态，不显示原始 URL 或数据库路径；debug/JSON 模式才显示 exact paths。
2. `数据状态`: source, freshness, completeness, missing high-impact fields.
3. `比赛信息`: match ID, event, format, time, status.
4. `队伍与阵容`: teams, ranks, starters, coach/stand-in notes.
5. `选手数据`: annual/event ratings and missing rating flags.
6. `地图池总览`: per-map samples, raw win rate, weighted win rate, pick/ban when available.
7. `逐图详细分析`: when map detail data exists, summarize each playable map separately using sample size, overall/LAN win rate, CT/T, pistol, first-kill/first-death, rounds won, pick/ban, and data quality.
8. `特殊 Veto 变量`: maps that one team rarely plays, permanently bans, has no current-year data on, or has extreme low sample. Do not mix these maps into the normal map average.
9. `近期记录 / H2H`: recent records and direct matchup map rows when available.
10. `警匪胜率`: each team's CT-side and T-side win rate by map when available from HLTV team map stats pages.
11. `Veto / 比分`: veto, map order, scores, result when visible.
12. `给模型的决策输入`: factual factors grouped by map pool, head-to-head, player form, roster state, side profile, match context, and data quality.
13. `数据缺口`: what is missing and what must not be inferred.
14. `JSON`: only when requested; stable English-key JSON for downstream use.

Use tables for dense comparative data. Avoid long prose unless explaining a data warning.

## Per-Map Detail Analysis Requirement

When the user asks `who is favored`, `who has higher win rate`, `which team is stronger`, or any similar judgment request, and map detail data is available, the Markdown output must include a per-map detail section. The section is factual; it is not a winner prediction.

For each playable map, include:

- `样本可信度`: overall and LAN sample sizes, small-sample warning when relevant.
- `结果表现`: W-L, raw win rate, rounds won / total rounds when available.
- `CT/T 结构`: CT-side and T-side win rates.
- `手枪局`: pistol round win rate.
- `首杀后 / 首死后`: round win rate after first kill and after first death.
- `Pick/Ban 倾向`: pick percentage and ban percentage.
- `数据对位`: which team has the factual edge on that map, or whether the map is too close / too incomplete.
- `Veto 影响`: whether the map is likely a real playable map, a likely ban, or a high-sensitivity decider variable.

If both teams effectively do not play a map, one team has no current-year data, or one team has an extreme ban rate / tiny sample, move that map into `特殊 Veto 变量` instead of treating it as a normal map edge. Example: in a BO3 where both sides each remove one map, the normal detailed analysis may focus on roughly five or six realistic maps, while high-ban/no-data maps are documented separately.

This requirement is strategy-neutral: it organizes factual map-level evidence for a downstream model. Do not add final winner direction or percentage.

For match URL data packs, visible starters require rating lookup attempts:

- Resolve `eventId` from the match page/event link whenever possible.
- Fetch event rating from `https://www.hltv.org/stats/players?event=<eventId>` when `eventId` is known.
- Try annual rating for the current calendar year or requested `as_of_date` year.
- After resolving team IDs from the match page or team links, always visit each team's HLTV stats pages for the current calendar-year window before finalizing `maps`, `map_side_stats`, or `players`. Match-page map stats are only a recent-core fallback, not the primary map-pool source.
- For team map summary stats, use the current calendar-year window by default, e.g. `https://www.hltv.org/stats/teams/maps/<teamId>/<slug>?startDate=2026-01-01&endDate=2026-12-31`.
- Direct HLTV partial fallback stops at the team map summary page. Per-map CT/T side win rates require visiting each map detail page and belong to static database/API or collector mode.
- For annual team player ratings, use current-year HLTV team player stats when available, e.g. `https://www.hltv.org/stats/teams/players/<teamId>/<slug>?startDate=2026-01-01&endDate=2026-12-31`.
- If either lookup fails, mark the exact player/field as `缺失` / `missing`; do not leave the whole rating section unloaded without attempting lookup.
- If a coach, stand-in, or new player has no listed rating, report that as a data quality warning rather than inferring a value.

## No Prediction Policy

This skill must not output model inference.

Never output:

- winner lean or "favored" conclusion;
- map-win or match-win probabilities;
- score prediction;
- Veto hypothesis or predicted map pool;
- betting advice, odds analysis, EV, Kelly, stake sizing, or max buy price;
- strategy recommendation.

If the user asks for any of the above, return the factual data pack and add:

```text
本 skill 只输出 HLTV 衍生数据和给模型的事实输入；胜率、Veto 预测和策略判断由调用模型或用户自己的策略完成。
```

Direct HLTV partial fallback under Cloudflare failure is usually `partial` or `blocked` completeness. It must not produce confident numeric ranges or qualitative winner conclusions from rankings, search snippets, or market prices alone.

## Query Examples

Use this skill for prompts like:

```text
用 hltv-cs2-data 帮我看一下 FaZe 和 G2 谁胜率高
```

```text
用 hltv-cs2-data 查这场比赛的数据：
https://www.hltv.org/matches/2393346/g2-vs-faze-blast-rivals-2026-season-1
```

```text
Use hltv-cs2-data to fetch a FaZe + FURIA machine-readable data pack with Markdown and JSON.
```

```text
Use hltv-cs2-data for this match URL:
https://www.hltv.org/matches/2393335/faze-vs-furia-blast-rivals-2026-season-1
```

```text
Backtest match 2393335 as of 2026-04-30 22:30 Asia/Shanghai. Only return data visible at that time.
```

If the user asks which team has the higher win rate, provide the relevant data pack and decision inputs only. Do not append `Model Inference`.

Default user-facing input can be simple natural language. Do not require users to know API parameters.

For normal public users, do not ask for a Static Base URL unless the default public static source is unreachable or the user wants a different dataset.

## Boundaries

Allowed:

- Summarize and structure facts.
- Compute descriptive aggregates already defined in the data contract, such as sample counts, raw win rate, weighted win rate from the data warehouse, and freshness.
- Organize factual features into `decision_inputs` for downstream user-owned models.
- Identify data quality issues.
- Produce Markdown and JSON.
- Keep all outputs data-only, including when the user asks who is stronger.

Not allowed:

- Mix model inference into factual HLTV fields.
- Treat `decision_inputs` as predictions.
- Present model-derived probability as HLTV data.
- Output model-derived probability at all.
- Use private correction logs, private prediction frameworks, or hidden strategy rules as a model.
- Recommend bets.
- Compute Kelly, EV, or max buy price.
- Fill missing data by intuition.
- Give specific win-rate percentages when the core data gate is not satisfied.

## Backtest Discipline

When `as_of_date` is provided, only include data visible before that timestamp. See `references/backtest-mode.md`.

If exact historical snapshots are unavailable, mark the field as reconstructed or unavailable. Never leak post-match results into a pre-match data pack.
