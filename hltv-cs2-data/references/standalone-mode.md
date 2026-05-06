# Standalone Public Mode

Standalone public mode lets someone use `hltv-cs2-data` immediately after installing the skill, without any private database, scraper, local browser, CDP session, or Playwright session. It uses public HLTV pages only for match discovery and identity resolution, then must use the default public static JSON database export for structured stats.

## Goal

Given a natural-language request, a match URL, or two team names, first locate the relevant HLTV match/team/event page when possible. Then produce the best available strategy-neutral HLTV data pack by combining direct HLTV identity facts with configured API/warehouse data or the default public static JSON database export.

This is the default public mode. It must be useful, but honest about missing data. If the public database export is stale/missing, mark that field as unavailable instead of replacing it with Liquipedia/wiki/search snippets. If the structured source cannot be read at all, stop at partial facts.

Important: direct HLTV lookup is not the main data path. It is the ID and visible-fact path. The public static JSON database export is the required data path for map pools, player ratings, map details, CT/T, pistol, first-kill, first-death, Pick/Ban, and exported recent rows.

External public sources are allowed for match-background facts only. Use them to clarify starters, stand-ins, coaches, roster notes, format, stage, bracket context, schedule, LAN/online status, venue, or country. Label these as `external_context` when they are not from HLTV/database. Do not use them for map stats, player ratings, CT/T, pistol, first-kill/first-death, Pick/Ban, H2H, recent rows, veto, score, result, or any numeric analytical field.

## Default Public Database Export

Use this base URL unless the user provides another static source or API source:

```text
https://raw.githubusercontent.com/smallmeji/hltv-cs2-data-skill/main/public-data
```

The first fetch must be the manifest URL, not the directory URL:

```text
https://raw.githubusercontent.com/smallmeji/hltv-cs2-data-skill/main/public-data/manifest.json
```

The base URL is a path prefix, not a JSON document. Some runtimes will return 404 if they fetch the base directly. Do not treat that as data absence before trying `/manifest.json`.

If an external model reports "data source returned 404", first check whether it fetched the base directory instead of `/manifest.json` or an exact JSON file. Directory 404 is a client error, not a database miss.

Expected files include:

```text
/manifest.json
/teams/index.json
/teams/top.json
/teams/<hltvTeamId>/summary.json
/teams/<hltvTeamId>/maps-overall.json
/teams/<hltvTeamId>/maps-lan.json
/teams/<hltvTeamId>/map-details-overall.json
/teams/<hltvTeamId>/map-details-lan.json
/teams/<hltvTeamId>/players.json
/calendar/upcoming.json
/matches/<matchId>/data-pack.json
/events/<eventId>/player-ratings.json
```

If this source cannot be read, add `structured_database_unavailable` and continue with direct HLTV partial data only. Do not silently switch to Liquipedia or another wiki as a replacement data source. Do not produce a full report.

The public static JSON database export is usually enough for Top40 team map summaries, map-detail fields, player ratings, and exported match packs. Direct HLTV is the first match-discovery source and is often enough for match basics and simple context, but the database export is the required structured source for map/player/side/history fields when available.

External-model implementation note:

- Do not treat "I read HLTV successfully" as completion.
- Do not continue from HLTV match/team pages into a full report before reading `manifest.json` and exact JSON records.
- HLTV lineup/starter extraction is useful, but it only satisfies identity/lineup facts. It does not satisfy map, rating, CT/T, pistol, first-kill, first-death, Pick/Ban, or history fields.
- Do not use the host model's memory, search snippets, or summaries to fill the database step.
- Search snippets, wikis, official event pages, and market pages can only fill match-background context; they cannot satisfy the structured database step.
- If the model cannot fetch GitHub raw files, it must say the structured source is unavailable and stop before the analysis sections.
- Normal reports should not print the raw GitHub URLs; however, the model must still perform the fetch internally.

## Required Source Execution Log

Every standalone output must show whether the structured source was actually used. Put this near the top of Markdown, especially for Chinese output.

For normal human-facing reports, use compact status:

```text
数据源执行记录：
- HLTV 定位：成功
- 结构化数据：已读取 public static database export
- 读取记录：match data-pack + team map details + player ratings
- 字段来源：地图详情=static_database，赛事信息=direct_hltv，Veto=missing
```

For debug/audit/JSON output, exact URLs and paths may be included:

```text
数据源执行记录：
- HLTV 定位：成功 / 失败，URL: ...
- 静态数据库 manifest：成功 / 失败，URL: https://raw.githubusercontent.com/smallmeji/hltv-cs2-data-skill/main/public-data/manifest.json
- 已读取数据库记录：matches/2394116/data-pack.json, teams/11861/map-details-overall.json, ...
- 字段来源：地图详情=static_database，赛事信息=direct_hltv，Veto=missing
```

If the manifest or at least one exact static/API database record was not read, add `structured_database_not_queried`. In that state the model must not output a complete data pack, per-map detail analysis, veto prediction, or numeric win-rate percentages. It may output only partial HLTV facts and missing-data notes.

Minimal fail-closed Chinese output:

```text
当前只能定位 HLTV 基础信息；结构化数据库/API/静态 JSON 未成功读取，不能生成完整 hltv-cs2-data 报告。
可输出：比赛 ID、队伍、赛事、时间、可见阵容/比分。
不可输出：完整地图池、逐图详细分析、Veto 预测、具体胜率百分比。
```

If API credentials are configured, API mode can be used after HLTV match/team identity is resolved. Otherwise use direct HLTV first for match discovery and the default public static JSON database export for structured fields.

Local browser/CDP access belongs only to internal collector maintenance. It must not be required from public standalone users.

If `HLTV_CS2_STATIC_BASE_URL` or a user-provided static data-pack URL exists, use that JSON source instead of the default public static source for structured-data hydration. Static JSON is the public database path for Claude/GPT-style environments.

## Accepted Inputs

Natural language examples:

```text
https://www.hltv.org/matches/2393346/g2-vs-faze-blast-rivals-2026-season-1
Please find the two teams' data for this match.
```

```text
Help me look at NAVI and G2 data.
```

```text
Compare NAVI and G2 on Mirage and Ancient, output JSON too.
```

```text
用 hltv-cs2-data 帮我看一下 FaZe 和 G2 谁胜率高
```

The user should not need to provide API query parameters.

## Query Resolution Order

1. If input contains an HLTV match URL, read that HLTV page first and extract `hltvMatchId`, team IDs/slugs, event, format, time, status, lineup/veto/score if visible, and `eventId` when possible.
2. If input is natural language with event/team names, search/read HLTV match, event, results, and upcoming pages first to locate the relevant match page. Example: `PGL 上 Aurora 和 Heroic 谁胜率高` should first find the PGL Aurora vs Heroic match on HLTV when possible.
3. Once the match page or canonical team IDs are known, stop identity lookup and hydrate structured stats from configured API/warehouse if available.
4. If no API/warehouse is configured, read the default or user-provided public database manifest and resolve `/matches/<matchId>/data-pack.json`, `/teams/index.json`, `/teams/<id>/*.json`, and `/events/<eventId>/player-ratings.json`.
5. Record the manifest status and exact database paths read in `source_execution_log`.
6. If the structured database was not queried successfully, fail closed: output partial facts only and add `structured_database_not_queried`.
7. Only if the structured database source was attempted and is unavailable or missing a field, attempt direct HLTV current-year stats pages as supplemental fallback for that field.
8. If a map is mentioned, restrict or highlight that map.
9. Use the current calendar year as the default data window, e.g. `2026-01-01` to `2026-12-31` in 2026.
10. If `as_of_date` is mentioned, enter backtest discipline and use the calendar year containing `as_of_date`, but mark exact snapshots unavailable unless an API/warehouse exists.

Alias resolution notes:

- Use HLTV match-page team links as the canonical identity when a match is known.
- For standalone team-name queries, use `/teams/index.json` after manifest fetch to confirm exported team IDs and slugs.
- `Gentle Mates` / `M8` should resolve to team `13404` when confirmed by HLTV. Do not resolve it to `M80` (`12376`) just because the alias text is similar.

## Direct HLTV Partial Fallback Sources

Use only public HLTV pages available in the session:

- Match page: match ID, teams, event, schedule, format, status, lineup if visible, veto if visible, scores if visible.
- Team pages: resolve canonical team IDs, slugs, current roster, and team links when the match page does not expose enough structure.
- Team map summary stats pages: primary source for map W/D/L, win rate, pick %, and ban % for the default year window. For example: `https://www.hltv.org/stats/teams/maps/<teamId>/<slug>?startDate=2026-01-01&endDate=2026-12-31`.
- Team player stats pages: primary source for annual player ratings filtered by team and current calendar-year window, for example `https://www.hltv.org/stats/teams/players/<teamId>/<slug>?startDate=2026-01-01&endDate=2026-12-31`.
- Event player stats pages: event rating source when `eventId` is known, for example `https://www.hltv.org/stats/players?event=8250`.
- Match page map stats: visible recent core win rates and sample counts only as a fallback or supplemental context. Do not treat these as the official `maps` section when the yearly team stats pages are reachable.
- Results/matches pages: results or upcoming schedules when needed.

Do not use private display websites as a source.

Do not use Liquipedia, Liquidpedia, wikis, news snippets, or search summaries as map/player/side/history data sources. They can help describe match-background context such as stage, format, bracket, venue, schedule, roster notes, stand-ins, or coaches; label that context as `external_context`.

For precise coverage expectations, follow `references/data-availability.md`.

## Required Direct-Mode Warnings

If no central API/warehouse is configured, include:

```json
"warnings": [
  "direct_hltv_mode",
  "central_warehouse_api_unavailable"
]
```

Add missing-field warnings for unavailable data:

- `structured_database_not_queried`: the model did not read an API/warehouse/static JSON database record after resolving HLTV identity.
- `core_data_insufficient_for_numeric_inference`: critical map/rating/lineup fields are missing, so exact win-rate percentages must not be produced.
- `annual_rating_not_loaded`: annual player stats page was unreachable or the player could not be resolved.
- `event_rating_not_loaded`: event stats page was unreachable, event ID could not be resolved, or the player is not listed for the event.
- `fetch_failed_cache_miss`: the host model/page reader could not return a readable snapshot for a known URL.
- `fetch_failed_cf_challenge`: the URL was blocked by Cloudflare/access challenge.
- `stats_page_unavailable_in_direct_mode`: a known deep stats page was not retrievable in direct fallback mode.
- `pro_api_required_for_full_coverage`: hosted collector/API data is required for reliable coverage.
- `lineup_unavailable`
- `veto_unavailable`
- `head_to_head_not_loaded`
- `exact_backtest_snapshot_unavailable`
- `map_side_stats_not_loaded`

## Output Behavior

Direct HLTV partial fallback output must still follow the data pack contract where applicable:

- Markdown summary first.
- `数据源执行记录` / `source_execution_log` before analysis.
- JSON summary or full JSON only when requested.
- `Decision Inputs` summarizing factual factors for downstream models.
- Factual fields must not contain model-derived probability, prediction, strategy, EV, or stake fields.
- If explicitly requested, append a separate `Model Inference` section after the factual data pack only after applying `references/inference-gate.md`.

If the user asks who is favored or who has higher win rate, first return the data pack. If the user wants this model to judge, append a clearly labeled `Model Inference` section only when the core data gate allows it. If the gate fails, provide a qualitative low-confidence direction at most and state that exact percentages are blocked.

Do not call an output a full `hltv-cs2-data` data pack if the database/API/static source was not queried. Use `partial HLTV facts` instead.

## Limits

Direct HLTV mode cannot guarantee:

- Complete historical snapshots.
- Exact as-of backtests.
- Full head-to-head data.
- Complete annual/event rating coverage, but it must attempt those fields before marking them missing.
- CT/T side win-rate coverage from live pages; direct fallback mode does not visit every map detail page. Use static database/API or collector mode for `map_side_stats`.
- Round-level data.

When these are needed, recommend API/warehouse mode rather than inventing missing fields.

## Public Standalone Inference Gate

Use this quick rule in standalone mode:

- If both teams have current-year map summary and at least one usable player-rating source, numeric inference may be allowed with caveats.
- If either team's current-year map summary is missing, do not output map or match win-rate percentages.
- If two or more high-impact fields are missing, do not output exact percentages.
- If the result relies on search snippets, rankings, or market prices only, mark `completeness_level=partial` or `blocked` and stop before numeric inference.
