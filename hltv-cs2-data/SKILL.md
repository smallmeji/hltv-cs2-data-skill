---
name: hltv-cs2-data
description: "Use when a user needs the hltv-cs2 product/data skill: CS2 data packs from HLTV-derived data, including team resolution, match data, map pool history, player ratings, veto, lineup, event ratings, score history, model-inference-ready context, or backtest/time-travel context. This skill prepares Markdown and JSON data outputs for downstream user-owned analysis. The HLTV data pack must keep facts separate from inference; if the user explicitly asks for prediction or probabilities, provide them only in a clearly labeled Model Inference section."
metadata:
  short-description: Strategy-neutral HLTV CS2 data packs
---

# HLTV CS2 Data

## Purpose

`hltv-cs2-data` is the skill for the `hltv-cs2` product concept: an HLTV CS2 multidimensional data guide that keeps facts, decision inputs, and inference separate.

It uses public HLTV pages first to locate and verify the match, teams, event, schedule, lineup, veto, and score context. After the match/team IDs are known, it must query the structured data layer: configured warehouse/API when available, otherwise the default public static JSON database export. Direct HLTV stats pages are only supplemental fallback when the structured source is unavailable or missing a field. If the user asks for judgment, the calling model may add a separate `Model Inference` section after the data pack.

Do not extend or reuse any private prediction, betting, or strategy framework. This skill has no built-in fixed prediction model; any judgment belongs to the calling model or user strategy.

The skill must be usable by someone who only installs the skill and has no access to any private database, scraper, or API.

## Source Policy

- Default public static base URL: `https://raw.githubusercontent.com/smallmeji/hltv-cs2-data-skill/main/public-data`. Default public manifest URL: `https://raw.githubusercontent.com/smallmeji/hltv-cs2-data-skill/main/public-data/manifest.json`.
- Default resolution order for normal users: direct HLTV match/event/team pages first for match discovery and identity resolution -> configured warehouse/API if available for structured stats -> default public static JSON database export for structured stats -> direct HLTV deep stats only as last-resort supplemental fallback.
- Default date window: current calendar year only, e.g. 2026-01-01 to 2026-12-31 for the current 2026 season, unless the user explicitly requests another window.
- Preferred structured source: HLTV-derived static JSON database export or central data warehouse/API. The default public static source is the public database export for external models.
- Original upstream source: HLTV pages only.
- Do not use private display websites as a data source. Display sites are presentation surfaces, not product data interfaces.
- Do not use Liquipedia, Liquidpedia, wikis, news snippets, or search summaries as the structured data source. They may only be mentioned as external context when HLTV and the database cannot locate a match, and they must never replace map stats, player ratings, CT/T, veto, or result fields.
- Do not require end users to run a local database, scraper, local browser, CDP session, or Playwright session.
- Public lightweight mode should rely only on the host model's normal public web/page-reading/search capabilities and mark inaccessible deep stats fields as missing.
- Retrieve only data visible from HLTV pages during the session and label unavailable deeper fields as missing.
- A central collector/API may improve completeness and backtesting later, but it is not required for normal skill use.
- If Cloudflare or access controls block collection, fail safely and report freshness/missing data. Do not fabricate data or bypass protections.
- Treat the default public static JSON source as the hosted HLTV-derived public database export for structured stats, not merely optional background context. When both direct HLTV and database/cache data are available, preserve field-level source labels.

## Mandatory Structured Data Gate

For any complete match data pack, team comparison, per-map detail analysis, or numeric `Model Inference`, the model must prove it queried a structured data source after resolving HLTV identity.

Required evidence in the output:

- `database_manifest_url` or API base URL.
- `database_manifest_status`: `success`, `failed`, or `not_configured`.
- At least one exact `database_record_path`, such as `matches/2394116/data-pack.json`, `teams/11861/map-details-overall.json`, or `events/8250/player-ratings.json`.
- Field-level source labels such as `direct_hltv`, `static_database`, `api_warehouse`, `direct_hltv_fallback`, or `missing`.

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

## Judgment Output Contract

When the user asks who is stronger, who has higher win rate, who is favored, or requests probabilities, the output must use a fixed fact-first structure. Do not produce a free-form "pre-match deep analysis report".

Mandatory Chinese section order for judgment requests:

1. `数据源执行记录`
2. `数据状态 / 数据缺口`
3. `比赛信息`
4. `队伍与选手 rating`
5. `地图池总览`
6. `逐图详细分析`
7. `特殊 Veto 变量`
8. `给模型的决策输入`
9. `JSON`
10. `模型推理`

Rules:

- `数据源执行记录` is mandatory and must list manifest/API status and exact record paths.
- `队伍与选手 rating` must show available annual ratings and event ratings. If event ratings, lineup, or confirmed starters are missing, show `缺失` and add warnings instead of omitting the section.
- `Veto / 比分` is factual only: visible veto steps, map order, scores, or `赛前不可见`. Veto prediction must not appear there.
- Any Veto prediction, winner lean, map-win probability, match-win percentage, or score guess belongs only under `模型推理`.
- `模型推理` must start with `以下为模型推理，不是 HLTV 事实数据。`
- `模型推理` must include `completeness_level`, `inference_permission`, and `missing_high_impact_fields` before any conclusion.
- If the inference gate blocks numeric probabilities, do not output exact percentages. A qualitative direction is allowed only when the user explicitly asks for judgment.
- `JSON` is mandatory for product/data outputs and must include `source_execution_log`, `metadata.completeness_level`, `warnings`, `decision_inputs`, and `model_inference` when inference is requested.

## Operating Modes

- **Lightweight / Direct HLTV mode**: default standalone mode for users without an API key. Accept match URLs or team names, read HLTV pages through the host model's normal public web/page-reading/search capability, output Markdown + JSON with missing-field warnings. No private database, scraper, local browser, or CDP required.
- **Static JSON database mode**: required structured-data mode for external users when no API is configured. After resolving match/team identity from HLTV, read `https://raw.githubusercontent.com/smallmeji/hltv-cs2-data-skill/main/public-data/manifest.json`, then read standardized JSON files under the same base, such as `/teams/<hltvTeamId>/summary.json`, `/teams/<hltvTeamId>/map-details-overall.json`, `/events/<eventId>/player-ratings.json`, or `/matches/<matchId>/data-pack.json`.
- **Pro / API mode**: enhanced mode for users with `HLTV_CS2_API_BASE_URL` and `HLTV_CS2_API_KEY`. After resolving match/team identity from HLTV or user-provided IDs, call the API for standardized data packs, exact snapshots, veto, lineup, result, and backtest support.
- **Internal collector mode**: backend maintenance mode only. It may use persistent browser profiles or CDP to collect HLTV stats into a central warehouse, but this is not part of the public lightweight skill user experience.
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
   - Inference gate: `references/inference-gate.md`.
   - Data pack schema: `references/data-pack-contract.md`.
   - Decision input schema: `references/decision-inputs.md`.
   - API contract: `references/api-contract.md`.
   - HLTV collector requirements: `references/collector-contract.md`.
   - Backtest rules: `references/backtest-mode.md`.
   - Query examples: `references/query-workflow.md`.
3. Resolve the match and teams before hydrating deep data:
   - If the user provides an HLTV match URL, open/read that HLTV match page first and extract `hltvMatchId`, teams, event, format, schedule, status, lineup/veto/score if visible, and `eventId` when possible.
   - If the user gives a natural-language query such as `PGL 上 Aurora 和 Heroic 谁胜率高`, search/read HLTV match/event/result/upcoming pages first to find the relevant match page. Do not start from static JSON unless HLTV cannot locate the match.
   - After a match page or canonical team IDs are resolved, hydrate structured stats from the configured warehouse/API when available.
   - If no API/warehouse is configured, the next required step is the default public static JSON database export: `https://raw.githubusercontent.com/smallmeji/hltv-cs2-data-skill/main/public-data/manifest.json`. Do not stop after HLTV page facts.
   - Read `/matches/<matchId>/data-pack.json`, `/teams/<hltvTeamId>/*.json`, and `/events/<eventId>/player-ratings.json` as available. Mark field-level source as `static_database`.
   - Record these attempts in `数据源执行记录` / `source_execution_log`. If the manifest or static/API records were not read, stop before complete analysis and add `structured_database_not_queried`.
   - If the match page cannot be found/read on HLTV, use the public static database export to reverse-search exported matches/teams.
   - If the structured database/API/static source is unavailable or missing a field, add `structured_database_unavailable` or `static_record_not_found`, then direct HLTV deep stats may be attempted as supplemental fallback with field-level warning labels.
   - If neither can retrieve enough data, output a partial pack with missing-source warnings.
4. Match the user's language for user-facing Markdown. If the user writes in Chinese, output Chinese headings, labels, and warnings by default. Keep JSON field names stable in English.
5. Emit both Markdown and JSON when the user asks for product-ready output or downstream LLM use.
6. Include metadata, freshness, source URLs, cutoff time, sample sizes, and warnings.
7. Keep the data pack factual and organize `Decision Inputs` as factual features. If the user explicitly asks for probability, winner judgment, or strategy, add a separate `Model Inference` section after the data pack and label it as non-HLTV inference.
8. When a page or field cannot be read, use the data availability matrix to label whether the failure belongs to lightweight direct mode, in-app/browser session mode, internal collector mode, or API/warehouse mode.
9. Before any numeric model inference, apply `references/inference-gate.md`. If the core data threshold is not met, do not output specific win-rate percentages; output only factual data, missing fields, and a qualitative low-confidence direction if explicitly requested.

## Output Rules

Every data pack should include:

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
- `not_included`: explicit note that model inference fields are not part of the HLTV fact data pack.

For Chinese user-facing output, prefer this compact Markdown order:

1. `数据源执行记录`: HLTV 定位是否成功、manifest/API 是否成功、读取了哪些数据库路径、哪些字段来自 `static_database` / `api_warehouse` / `direct_hltv` / `missing`。
2. `数据状态`: source, freshness, completeness, missing high-impact fields.
3. `比赛信息`: match ID, event, format, time, status.
4. `队伍与阵容`: teams, ranks, starters, coach/stand-in notes.
5. `选手数据`: annual/event ratings and missing rating flags.
6. `地图池总览`: per-map samples, raw win rate, weighted win rate, pick/ban when available.
7. `逐图详细分析`: when map detail data exists and the user asks for stronger/weaker/win-rate judgment, analyze each playable map separately using sample size, overall/LAN win rate, CT/T, pistol, first-kill/first-death, rounds won, pick/ban, and data quality.
8. `特殊 Veto 变量`: maps that one team rarely plays, permanently bans, has no current-year data on, or has extreme low sample. Do not mix these maps into the normal map average.
9. `近期记录 / H2H`: recent records and direct matchup map rows when available.
10. `警匪胜率`: each team's CT-side and T-side win rate by map when available from HLTV team map stats pages.
11. `Veto / 比分`: veto, map order, scores, result when visible.
12. `给模型的决策输入`: factual factors grouped by map pool, head-to-head, player form, roster state, side profile, match context, and data quality.
13. `数据缺口`: what is missing and what must not be inferred.
14. `JSON`: stable English-key JSON for downstream use.

Use tables for dense comparative data. Avoid long prose unless explaining a data warning.

## Per-Map Detail Analysis Requirement

When the user asks `who is favored`, `who has higher win rate`, `which team is stronger`, or any similar judgment request, and map detail data is available, the Markdown output must include a per-map detail section before `Model Inference`.

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

This requirement is still strategy-neutral: it organizes factual map-level evidence. Any final winner direction or percentage belongs only in `Model Inference`.

For match URL data packs, visible starters require rating lookup attempts:

- Resolve `eventId` from the match page/event link whenever possible.
- Fetch event rating from `https://www.hltv.org/stats/players?event=<eventId>` when `eventId` is known.
- Try annual rating for the current calendar year or requested `as_of_date` year.
- After resolving team IDs from the match page or team links, always visit each team's HLTV stats pages for the current calendar-year window before finalizing `maps`, `map_side_stats`, or `players`. Match-page map stats are only a recent-core fallback, not the primary map-pool source.
- For team map summary stats, use the current calendar-year window by default, e.g. `https://www.hltv.org/stats/teams/maps/<teamId>/<slug>?startDate=2026-01-01&endDate=2026-12-31`.
- Lightweight mode stops at the team map summary page. Per-map CT/T side win rates require visiting each map detail page and belong to Pro/API or collector mode.
- For annual team player ratings, use current-year HLTV team player stats when available, e.g. `https://www.hltv.org/stats/teams/players/<teamId>/<slug>?startDate=2026-01-01&endDate=2026-12-31`.
- If either lookup fails, mark the exact player/field as `缺失` / `missing`; do not leave the whole rating section unloaded without attempting lookup.
- If a coach, stand-in, or new player has no listed rating, report that as a data quality warning rather than inferring a value.

If the user requested judgment, append:

- `model_inference`: clearly labeled model-derived interpretation, separated from facts, only after applying the inference gate.
- `veto_hypothesis`: optional model-derived Veto prediction, only inside `model_inference`; never under factual `Veto / 比分`.

## Model Inference Gate

Numeric probabilities are allowed only when the collected data is strong enough for the requested judgment.

For match-level or map-level probability requests, do **not** output exact percentages if either condition is true:

- The structured database/API/static export was not queried after HLTV identity resolution.
- Current-year map summary is missing for either team.
- Two or more high-impact fields are missing or blocked, including current-year map summary, annual player ratings, event player ratings when event ID is known, confirmed/expected lineup, or relevant veto/map-order context.

When the gate blocks numeric inference:

- Say clearly that the factual data pack is incomplete.
- List the exact missing fields and source URLs that failed.
- Use warning code `core_data_insufficient_for_numeric_inference`.
- If the user asked who is favored, allow only a qualitative statement such as `方向性偏 G2，但不能给可靠百分比`.
- Recommend API/warehouse data for full coverage.

Lightweight direct mode under Cloudflare failure is usually `partial` or `blocked` completeness. It must not produce confident numeric ranges such as `65-72%` from rankings, search snippets, or market prices alone.

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
Use hltv-cs2-data to fetch a FaZe + FURIA data pack. Output Markdown and JSON.
```

```text
Use hltv-cs2-data for this match URL:
https://www.hltv.org/matches/2393335/faze-vs-furia-blast-rivals-2026-season-1
```

```text
Backtest match 2393335 as of 2026-04-30 22:30 Asia/Shanghai. Only return data visible at that time.
```

If the user asks which team has the higher win rate, provide the relevant data pack first. If the user explicitly asks this model to judge after collecting data, append `Model Inference`.

Default user-facing input can be simple natural language. Do not require users to know API parameters.

For normal public users, do not ask for a Static Base URL unless the default public static source is unreachable or the user wants a different dataset.

## Boundaries

Allowed:

- Summarize and structure facts.
- Compute descriptive aggregates already defined in the data contract, such as sample counts, raw win rate, weighted win rate from the data warehouse, and freshness.
- Organize factual features into `decision_inputs` for downstream user-owned models.
- Identify data quality issues.
- Produce Markdown and JSON.
- Provide model inference only when explicitly requested, and only after a separate factual HLTV data pack.
- Apply the inference gate before giving numeric probabilities.

Not allowed:

- Mix model inference into factual HLTV fields.
- Treat `decision_inputs` as predictions.
- Present model-derived probability as HLTV data.
- Use private correction logs, private prediction frameworks, or hidden strategy rules as a model.
- Recommend bets.
- Compute Kelly, EV, or max buy price.
- Fill missing data by intuition.
- Give specific win-rate percentages when the core data gate is not satisfied.

## Backtest Discipline

When `as_of_date` is provided, only include data visible before that timestamp. See `references/backtest-mode.md`.

If exact historical snapshots are unavailable, mark the field as reconstructed or unavailable. Never leak post-match results into a pre-match data pack.
