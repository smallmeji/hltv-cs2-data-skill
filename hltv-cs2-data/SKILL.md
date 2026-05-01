---
name: hltv-cs2-data
description: "Use when a user needs the hltv-cs2 product/data skill: CS2 data packs from HLTV-derived data, including team resolution, match data, map pool history, player ratings, veto, lineup, event ratings, score history, model-inference-ready context, or backtest/time-travel context. This skill prepares Markdown and JSON data outputs for downstream user-owned analysis. The HLTV data pack must keep facts separate from inference; if the user explicitly asks for prediction or probabilities, provide them only in a clearly labeled Model Inference section."
metadata:
  short-description: Strategy-neutral HLTV CS2 data packs
---

# HLTV CS2 Data

## Purpose

`hltv-cs2-data` is the skill for the `hltv-cs2` product concept: an HLTV CS2 multidimensional data guide that keeps facts, decision inputs, and inference separate.

It prepares factual data packs directly from HLTV pages by default, then organizes model-ready `Decision Inputs`. If the user asks for judgment, the calling model may add a separate `Model Inference` section after the data pack.

Do not extend or reuse any private prediction, betting, or strategy framework. This skill has no built-in fixed prediction model; any judgment belongs to the calling model or user strategy.

The skill must be usable by someone who only installs the skill and has no access to any private database, scraper, or API.

## Source Policy

- Default source: direct HLTV pages and public HLTV stats pages.
- Default date window: current calendar year only, e.g. 2026-01-01 to 2026-12-31 for the current 2026 season, unless the user explicitly requests another window.
- Optional enhanced source: HLTV-derived central data warehouse or API, only when explicitly configured or requested.
- Original upstream source: HLTV pages only.
- Do not use private display websites as a data source. Display sites are presentation surfaces, not product data interfaces.
- Do not require end users to run a local database or scraper.
- Retrieve only data visible from HLTV pages during the session and label unavailable deeper fields as missing.
- A central collector/API may improve completeness and backtesting later, but it is not required for normal skill use.
- If Cloudflare or access controls block collection, fail safely and report freshness/missing data. Do not fabricate data or bypass protections.

## Operating Modes

- **Lightweight / Direct HLTV mode**: standalone mode for users without an API key. Accept match URLs or team names, read HLTV pages directly, output Markdown + JSON with missing-field warnings. No private database required.
- **Pro / API mode**: enhanced mode for users with `HLTV_CS2_API_BASE_URL` and `HLTV_CS2_API_KEY`. Call the API first for standardized data packs, exact snapshots, veto, lineup, result, and backtest support.
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
   - Data pack schema: `references/data-pack-contract.md`.
   - Decision input schema: `references/decision-inputs.md`.
   - API contract: `references/api-contract.md`.
   - HLTV collector requirements: `references/collector-contract.md`.
   - Backtest rules: `references/backtest-mode.md`.
   - Query examples: `references/query-workflow.md`.
3. Gather data from the best available source:
   - If API base URL and key are configured, call API mode first.
   - If API is unavailable or unconfigured, use lightweight Direct HLTV mode and add `direct_hltv_fallback`.
   - If neither can retrieve enough data, output a partial pack with missing-source warnings.
4. Match the user's language for user-facing Markdown. If the user writes in Chinese, output Chinese headings, labels, and warnings by default. Keep JSON field names stable in English.
5. Emit both Markdown and JSON when the user asks for product-ready output or downstream LLM use.
6. Include metadata, freshness, source URLs, cutoff time, sample sizes, and warnings.
7. Keep the data pack factual and organize `Decision Inputs` as factual features. If the user explicitly asks for probability, winner judgment, or strategy, add a separate `Model Inference` section after the data pack and label it as non-HLTV inference.

## Output Rules

Every data pack should include:

- `metadata`: query, source, retrieved time, data cutoff, version, missing fields.
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

1. `数据状态`: source, freshness, completeness, missing high-impact fields.
2. `比赛信息`: match ID, event, format, time, status.
3. `队伍与阵容`: teams, ranks, starters, coach/stand-in notes.
4. `选手数据`: annual/event ratings and missing rating flags.
5. `地图池`: per-map samples, raw win rate, weighted win rate, pick/ban when available.
6. `近期记录 / H2H`: recent records and direct matchup map rows when available.
7. `警匪胜率`: each team's CT-side and T-side win rate by map when available from HLTV team map stats pages.
8. `Veto / 比分`: veto, map order, scores, result when visible.
9. `给模型的决策输入`: factual factors grouped by map pool, head-to-head, player form, roster state, side profile, match context, and data quality.
10. `数据缺口`: what is missing and what must not be inferred.
11. `JSON`: stable English-key JSON for downstream use.

Use tables for dense comparative data. Avoid long prose unless explaining a data warning.

For match URL data packs, visible starters require rating lookup attempts:

- Resolve `eventId` from the match page/event link whenever possible.
- Fetch event rating from `https://www.hltv.org/stats/players?event=<eventId>` when `eventId` is known.
- Try annual rating for the current calendar year or requested `as_of_date` year.
- For team map stats and map side stats, use the current calendar-year window by default, e.g. `startDate=2026-01-01&endDate=2026-12-31`.
- If either lookup fails, mark the exact player/field as `缺失` / `missing`; do not leave the whole rating section unloaded without attempting lookup.
- If a coach, stand-in, or new player has no listed rating, report that as a data quality warning rather than inferring a value.

If the user requested judgment, append:

- `model_inference`: clearly labeled model-derived interpretation, separated from facts.

## Query Examples

Use this skill for prompts like:

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

## Boundaries

Allowed:

- Summarize and structure facts.
- Compute descriptive aggregates already defined in the data contract, such as sample counts, raw win rate, weighted win rate from the data warehouse, and freshness.
- Organize factual features into `decision_inputs` for downstream user-owned models.
- Identify data quality issues.
- Produce Markdown and JSON.
- Provide model inference only when explicitly requested, and only after a separate factual HLTV data pack.

Not allowed:

- Mix model inference into factual HLTV fields.
- Treat `decision_inputs` as predictions.
- Present model-derived probability as HLTV data.
- Use private correction logs, private prediction frameworks, or hidden strategy rules as a model.
- Recommend bets.
- Compute Kelly, EV, or max buy price.
- Fill missing data by intuition.

## Backtest Discipline

When `as_of_date` is provided, only include data visible before that timestamp. See `references/backtest-mode.md`.

If exact historical snapshots are unavailable, mark the field as reconstructed or unavailable. Never leak post-match results into a pre-match data pack.
