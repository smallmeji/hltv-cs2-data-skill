# hltv-cs2-data Skill

[中文说明](README.zh-CN.md)

`hltv-cs2-data` is a skill for reading an HLTV-derived structured database / static JSON export and turning it into model-ready CS2 data packs.

The skill is intentionally data-only. It provides factual HLTV-derived data and organizes useful decision inputs, but it does not output winner predictions, probabilities, Veto hypotheses, score guesses, betting advice, EV logic, or stake sizing.

## What It Is For

Use this skill when you want an LLM to gather the data needed to analyze a CS2 match, team pair, event, roster, or historical matchup.

Typical use cases:

- Get a data pack for one HLTV match URL.
- Compare two teams across map pool, player form, roster status, and head-to-head history.
- Inspect map-specific win rates, CT/T side win rates, head-to-head records, and recent player form when available.
- Pull event player ratings when an event stats page is available.
- Build backtest-ready context with an `as_of` cutoff.
- Feed structured Markdown and JSON into a separate model or strategy layer.

The skill can support prediction workflows as a data provider, but it does not make predictions. If the user asks who is stronger or asks for win rates, the skill returns the factual data pack and decision inputs, then states that probability and strategy belong to the calling model or user-owned strategy.

User-facing Markdown follows the user's language. For Chinese prompts, the skill should output Chinese headings, labels, warnings, and concise tables, while keeping JSON field names stable in English.

## Critical Compliance Rule

### First Move: HLTV Locates Identity; Structured Data Feeds Analysis

The model must not read HLTV pages and then start the analytical body immediately. HLTV pages are the identity and visible-fact layer: match ID, team ID, event ID, visible lineup, format, stage, published veto, score, and status. Map pools, player ratings, CT/T, pistol rounds, first-kill/first-death conversion, Pick/Ban, and recent rows must come from structured database/API/static JSON records.

Public external sources may be used for match-background context, but not as a replacement for structured data. Allowed background fields include starters, stand-ins/coaches, format, stage, schedule, LAN/online status, venue/country, bracket/group context, and public roster-change notes. If these facts do not come from HLTV or the database, label them as `external_context`. These sources must not fill map win rates, samples, CT/T, pistol, first-kill/first-death, Pick/Ban, player ratings, H2H, recent map rows, veto, score, or result fields.

Every match/team comparison query must first complete:

1. Use HLTV to resolve match/team/event IDs.
2. Fetch the public static database manifest:
   `https://smallmeji.github.io/hltv-cs2-data-platform/public-data/latest/manifest.json`
   - Do not fetch `.../public-data` as if it were an API endpoint. GitHub raw directory URLs can return `404`; that does not mean the database export is unavailable.
   - Fetch `/manifest.json` or exact JSON files.
3. Fetch exact records by ID, for example:
   - `teams/<hltvTeamId>/summary.json`
   - `teams/<hltvTeamId>/players.json`
   - `teams/<hltvTeamId>/map-details-overall.json`
   - `teams/<hltvTeamId>/map-details-lan.json`
   - `matches/<hltvMatchId>/data-pack.json`
4. Only then write map pool, player rating, per-map detail, or decision inputs.

If step 2 or step 3 fails, stop and output:

```text
Only basic HLTV identity facts were located. The structured database/API/static JSON source was not read, so a complete hltv-cs2-data report cannot be generated.
warning: structured_database_not_queried
```

These outputs are non-compliant:

- `Data source: HLTV.org official data` without structured source status.
- Complete map pool / per-map detail with no static/API record.
- Full JSON in a normal report when the user did not ask for JSON.
- Search summaries, news, wiki, or market pages used as database fields.

A correct output must include a `Data Source Execution Log` / `数据源执行记录` near the top.

For external models, the compliant flow is:

```text
user request / HLTV URL
  -> minimal HLTV identity lookup: matchId, teamId, eventId, visible lineup/score
  -> fetch public-data/manifest.json
  -> fetch matches/<matchId>/data-pack.json or teams/<id>/*.json
  -> use only these structured records for map pool, ratings, per-map detail, and decision inputs
  -> do not output winner prediction, probabilities, Veto prediction, score guess, or betting advice
```

If a model moves from HLTV pages straight into a full analysis, or only says `Data source: HLTV.org`, it did not use this skill correctly.

Visible starters from an HLTV match page may be correct and are allowed as lineup facts. That does not prove structured map/player data was loaded. Without a successful static/API database record fetch, the output is still only `direct_hltv_partial`; it must not produce full per-map analysis or numeric win probabilities.

Aliases must be resolved back to the canonical HLTV identity. For example, `M8` / `Gentle Mates` should resolve to HLTV team `13404` when confirmed by the match page. Do not resolve it to `M80` (`12376`) or another similarly named team.

The log must show:

- The HLTV match/team/event page used for identity resolution.
- The public database manifest or API base that was queried.
- At least one exact structured record path, for example `matches/2394116/data-pack.json`, `teams/11861/map-details-overall.json`, or `events/8250/player-ratings.json`.
- Field-level source labels such as `direct_hltv`, `static_database`, `api_warehouse`, `direct_hltv_fallback`, or `missing`.

If a model only reads HLTV, Liquipedia/wiki pages, news snippets, search summaries, or market pages, it has not complied with this skill. In that case it must mark the output as partial, add warning `structured_database_not_queried`, and must not output a full data pack, per-map detail analysis, veto prediction, or exact win-rate percentages.

Map-pool sections must also use only the current active map pool present in the structured record. The current 2026 public export uses these seven maps: `Ancient`, `Anubis`, `Dust2`, `Inferno`, `Mirage`, `Nuke`, and `Overpass`. Do not add `Vertigo`, `Cache`, `Train`, or other absent inactive maps to `Map Pool Overview`, `Per-Map Detail`, or `Special Veto Variables`.

When the user asks who is stronger, who has higher win rate, or who is favored, output is still data-only and must follow this fixed order:

1. `Data Source Execution Log`
2. `Data Status / Data Gaps`
3. `Match Info`
4. `Teams and Player Ratings`
5. `Map Pool Overview`
6. `Per-Map Detail`
7. `Special Veto Variables`
8. `Decision Inputs`
9. `JSON`

Veto predictions, exact win-rate percentages, winner leans, and score guesses must not appear in this skill. The factual `Veto / Score` section may contain only observed veto, map order, scores, or `unavailable`.

This rule protects the factual layer. A downstream model may use the data to create its own map weights, form weights, roster-impact notes, Veto hypotheses, win-rate estimates, or other strategy factors, but that output is outside this skill. Model-created metrics must not be presented as HLTV facts, and missing raw fields must not be filled with invented numbers.

Sections such as `recent 30-day form`, `event participation`, `opponent quality`, or `map-pool depth` are allowed as descriptive data summaries when they state the source rows, time window, and calculation method. If exact rows are unavailable, mark the metric as missing.

Normal human-facing reports do not need to print raw manifest URLs, GitHub raw URLs, database record paths, or full JSON. Show a compact source status instead, such as `structured data: public static database export loaded`. Print exact URLs, paths, and JSON only when the user asks for debug, audit, source details, JSON, or downstream model output.

Every map number must come from an exact team-map row. If a team has no row for a map, print `missing` / `no data`; do not infer it from the opponent, another map, search snippets, or overall team form. For example, if a data pack has no `Heroic + Anubis` row, Heroic Anubis must be `missing`.

Example: for `MOUZ vs Gentle Mates`, after HLTV identity resolution, a compliant model should attempt:

- `matches/<matchId>/data-pack.json`
- `teams/4494/summary.json`, `teams/4494/map-details-overall.json`, `teams/4494/map-details-lan.json`, `teams/4494/players.json`
- `teams/13404/summary.json`, `teams/13404/map-details-overall.json`, `teams/13404/map-details-lan.json`, `teams/13404/players.json`

If those records were not read, correct HLTV starters are not enough for a complete report.

This skill must not output betting advice, odds analysis, EV, Kelly, stake sizing, max buy price, winner prediction, probability, Veto prediction, or score prediction.

## What It Is Not

This repository does not provide:

- A private CS2 prediction model.
- Betting recommendations.
- EV, Kelly, max buy price, or stake sizing.
- A scraper service that bypasses access controls.
- A local database or private data dependency.
- Guaranteed complete historical snapshots in direct HLTV mode.

For normal public use, the skill first uses HLTV pages to locate and verify the match, teams, and event. After IDs are known, it must query the structured data layer. If no API is configured, the default structured source is the public static JSON database export:

```text
https://smallmeji.github.io/hltv-cs2-data-platform/public-data/latest/manifest.json
```

If the match page is found on HLTV, the database export is used to load structured fields such as map details, CT/T, player ratings, and exported match packs. If the match page cannot be found/read on HLTV, the skill may also use the manifest to reverse-search exported match/team records. API/warehouse mode can provide richer and more reproducible snapshots when configured, but it is optional.

## Product Tiers

The skill is useful in three tiers:

| Tier | Best For | What To Expect |
|:--|:--|:--|
| Public standalone mode | Match discovery plus hosted static database hydration | Default public path. HLTV is used to find the match/team IDs; the public static JSON database export is required for structured map/player/detail fields. |
| Static JSON database export | Shared data packs for Claude/GPT/user models | Required public structured-data path when no API is configured. Gives stable team, match, event, map-detail, and compare packs when exported. |
| API mode | Repeatable analysis and production use | Recommended for complete current-year stats, CT/T side data, exact historical backtests, lineup/veto/result snapshots, batch usage, and stable freshness guarantees. |

Direct HLTV-only output is partial fallback only. It is enough for visible match facts, but not enough for a complete report or per-map detail analysis. Static JSON or API mode is required for those sections.

In direct fallback, the host model's web reader may fail on HLTV stats pages. This is expected. The skill marks those fields as missing and does not convert missing data into predictions.

## Usage Modes

`hltv-cs2-data` supports three operating modes.

### 1. Minimal HLTV Lookup, Then Database Export

This is the default public mode. Users can ask natural questions without providing configuration:

```text
Use hltv-cs2-data to compare FaZe and G2. Who has the higher win rate?
```

The skill should first try to locate the match or teams on HLTV. For example, `PGL Aurora vs Heroic` should first search/read HLTV match, event, result, and upcoming pages to find the exact match page. After match/team IDs are known, it must stop deep HLTV stats browsing and query the database export:

```text
https://smallmeji.github.io/hltv-cs2-data-platform/public-data/latest/manifest.json
```

The base path prefix is:

```text
https://smallmeji.github.io/hltv-cs2-data-platform/public-data/latest
```

Do not fetch the base directory as JSON; append `/manifest.json` first.

Example layout:

```text
/manifest.json
/teams/index.json
/teams/6667/summary.json
/teams/6667/maps-overall.json
/teams/6667/map-details-overall.json
/matches/2393346/data-pack.json
/events/8250/player-ratings.json
```

To use another static source, configure:

```text
HLTV_CS2_STATIC_BASE_URL=https://your-static-data.example.com/latest
```

In this mode, the skill should use live HLTV for match discovery and visible facts, then use the static JSON database export for structured fields. Direct HLTV deep stats are only last-resort supplemental fallback after the database export was attempted and is unavailable or missing a field. Do not treat “HLTV page loaded” as completion.

Do not use Liquipedia, Liquidpedia, wikis, news snippets, or search summaries as substitute sources for map stats, player ratings, CT/T, veto, or result fields.

If the user asks "who is favored", "who has the higher win rate", or similar, the skill still starts with factual data. When map-detail data exists, the Markdown output must include:

- `Map Pool Overview`: seven-map summary with samples, W-D-L, win rate, pick %, and ban %.
- `Per-Map Detail`: one section per playable map, including overall/LAN sample, CT/T split, pistol win rate, first-kill / first-death conversion, rounds, and pick/ban tendency.
- `Special Veto Variables`: maps with no current-year data, extreme ban rate, tiny sample, or one side effectively not playing the map.
- `Decision Inputs`: factual model-ready inputs only; no winner conclusion.

### 2. Direct HLTV Partial Fallback

Use this fallback only when the structured static database/API cannot be read.

- No API key is required.
- No user-owned database is required.
- No local browser, CDP session, Playwright session, or local scraper is required from the user.
- The model reports only visible HLTV facts and missing-field warnings.
- It must not output a complete report, per-map detail analysis, Veto prediction, or numeric probabilities.
- Historical backtests are marked as `reconstructed` unless an exact snapshot is available.
- Missing fields must be reported explicitly instead of being guessed.

### 3. Pro / API Mode

Use this mode when a maintained HLTV data API is available.

Configure:

```text
HLTV_CS2_API_BASE_URL=https://your-api.example.com
HLTV_CS2_API_KEY=your_api_key
```

In API mode, the skill should query the data API first. The API is expected to return standardized data packs from a maintained collector and database. Human reports should be Markdown by default; JSON is included only when the user requests machine-readable or downstream-model output.

Benefits:

- Better historical reproducibility.
- Faster team and match lookup.
- Centralized lineup, veto, score, rating, and map-pool snapshots.
- Cleaner backtest support through `as_of` cutoffs.

If the API is unavailable, the skill can fall back to direct HLTV mode only when the user allows fallback or the task is not time-sensitive.

## Installation / Use

This repository is not tied to one specific runtime. At its core, it is a structured instruction folder:

- If your tool supports skill folders, install `hltv-cs2-data/` as a skill.
- If your tool supports `.skill` packages, install `hltv-cs2-data.skill`.
- If your tool does not support skills, load `hltv-cs2-data/SKILL.md` as the main instruction file and load files under `hltv-cs2-data/references/` only when needed.

### Option 1: Install the folder

Copy the `hltv-cs2-data/` folder into your skills directory:

```bash
SKILLS_HOME=/path/to/skills
mkdir -p "$SKILLS_HOME"
cp -R hltv-cs2-data "$SKILLS_HOME/"
```

Restart or reload skills if your environment requires it.

### Option 2: Install the packaged skill

Use `hltv-cs2-data.skill` if your environment supports installing `.skill` packages.

The package contains the same `hltv-cs2-data/` skill folder and references.

### Option 3: Use as plain instructions

For any model or workflow that does not have native skill support:

1. Use `hltv-cs2-data/SKILL.md` as the main system or project instruction.
2. Load only the relevant reference file for the task:
   - Match/team query: `references/query-workflow.md`
   - Output schema: `references/data-pack-contract.md`
   - Public standalone mode: `references/standalone-mode.md`
   - Backtest: `references/backtest-mode.md`
3. Ask the model to follow the fact-vs-inference boundary described in the skill.

## Quick Usage

### Match URL

```text
Use hltv-cs2-data for this match:
https://www.hltv.org/matches/2393346/g2-vs-faze-blast-rivals-2026-season-1
Output a normal Markdown report. Include JSON only if I ask for machine-readable output.
```

Expected behavior:

- Resolve match ID, teams, event, format, schedule, and status.
- Read the public static database manifest or configured API after resolving identity.
- Show compact `Data Source Execution Log` proving structured data was read.
- Gather available map, lineup, player rating, veto, score, and recent result data.
- Mark missing data explicitly.
- Output a factual report. Do not include a JSON block unless requested.
- If current-year map summaries or player ratings are blocked/missing, do not output exact win-rate percentages; show the missing fields and recommend API/warehouse mode for full coverage.

### Team Comparison

```text
Use hltv-cs2-data to compare NAVI and G2.
Focus on map pool, player form, roster changes, and head-to-head data.
```

Expected behavior:

- Resolve both teams to HLTV identities when possible.
- After team IDs are known, query the public static JSON database export or configured API for team/map/player data.
- Show the manifest/API status and exact database record paths used.
- Organize factual factors into `decision_inputs`.
- Avoid declaring a winner. This skill is data-only even when the user asks for judgment.

### Team Comparison + Decision Inputs

```text
Use hltv-cs2-data to compare PGL Aurora and Heroic. Who has the higher win rate?
Return factual data and decision inputs only.
```

Expected behavior:

1. Use HLTV first to locate the relevant match/team identities, then query API or the public static JSON database export for structured map/player/side data.
2. Output `Map Pool Overview` with current-year overall/LAN samples, W-D-L, win rate, pick %, and ban %.
3. Output `Per-Map Detail` for each realistic playable map:
   - sample confidence;
   - overall and LAN results;
   - CT-side and T-side win rates;
   - pistol round win rate;
   - first-kill and first-death round win rates;
   - pick/ban tendency;
   - factual map-edge note.
4. Output `Special Veto Variables` for maps with no current-year data, extreme ban rate, or tiny samples.
5. Add `Decision Inputs` only at the end; do not output a winner lean.

The skill is not a prediction model. It organizes data; the final lean is the calling model's interpretation of that data outside this skill.

### Data-Only Output

```text
Use hltv-cs2-data to collect G2 vs FaZe data. Return factual data only.
```

Expected behavior:

1. Output the factual HLTV data pack first.
2. Output `Decision Inputs`.
3. If map-detail data exists, output `Per-Map Detail` and `Special Veto Variables`.
4. Do not append `Model Inference`.
5. If the user asks for prediction, state that prediction belongs to the calling model or user strategy.

### Backtest Context

```text
Use hltv-cs2-data to backtest match 2393335 as of 2026-04-30 22:30 Asia/Shanghai.
Only include data visible before that time.
```

Expected behavior:

- Apply cutoff discipline.
- Exclude final scores and post-match records if the requested time is pre-match.
- Mark exact historical snapshots as unavailable or reconstructed when a warehouse/API is not configured.

## Worked Example: Public Standalone Match Report

This is the common path for a user who only installs the skill and does not have an API key.

### Step 1: Ask with a match URL

```text
Use hltv-cs2-data for this match:
https://www.hltv.org/matches/2393346/g2-vs-faze-blast-rivals-2026-season-1

Output Markdown only.
Return factual data only, no win-rate prediction.
If a field cannot be retrieved, include a warning code. Do not show raw source URLs unless I ask for debug/source details.
```

### Step 2: What the skill should attempt

Public standalone mode should try these sources in order:

1. Match page: match ID, teams, event, format, schedule, status, visible lineup, veto, scores.
2. Team identity: HLTV team IDs, slugs, visible rank and roster context.
3. Static database manifest: `public-data/manifest.json`.
4. Exact static records: match data pack, team summaries, map details, players, event ratings when exported.
5. Direct HLTV current-year stats pages only as supplemental fallback for fields missing from the static database.
6. If the structured source cannot be read, stop at partial facts and warning `structured_database_not_queried`.

The default date window is the current calendar year. For 2026 matches, use `2026-01-01` to `2026-12-31` unless the user asks for another window.

### Step 3: Recommended Markdown sections

| Section | Contents |
|:--|:--|
| Data Status | Source mode, retrieved time, completeness, high-impact missing fields |
| Match Info | Match ID, event, schedule, format, status |
| Teams and Lineups | HLTV IDs, visible starters, coaches, stand-ins, lineup warnings |
| Player Data | Annual ratings, event ratings, missing rating flags |
| Map Pool Overview | Current-year map summary, samples, W/D/L, win rate, pick %, ban % |
| Per-Map Detail | Playable-map detail: overall/LAN, CT/T, pistol, first-kill, first-death, rounds, pick/ban, factual edge |
| Special Veto Variables | Maps with no current-year data, extreme ban rate, tiny samples, or one side effectively not playing them |
| Recent / H2H | Recent matches and direct matchup rows when available |
| Veto / Scores | Veto, map order, map scores when visible |
| Decision Inputs | Factual model-ready inputs only |
| Data Gaps | Missing fields and warning codes; raw source URLs only in debug/source mode |
| JSON | Only when requested; stable English keys for downstream systems |

### Step 4: If the user asks for judgment

If the user asks "which team has the higher win rate?", the output should remain:

1. Factual HLTV data pack first.
2. `decision_inputs` second.
3. Boundary note: this skill does not output probabilities or winner judgments.

### Step 5: How to read cache miss

`cache miss` does not mean HLTV has no data, and it does not mean the URL is wrong. It means the direct fallback page reader did not return a readable snapshot.

Recommended field-level warning:

```json
{
  "field": "event_player_ratings",
  "source_url": "https://www.hltv.org/stats/players?event=8250",
  "status": "missing",
  "warning": "fetch_failed_cache_miss",
  "meaning": "The URL is known, but direct fallback could not retrieve a readable table snapshot."
}
```

## Output Layers

The skill separates output into three layers.

### 1. HLTV Data Pack

This is the factual layer. It may include:

- `metadata`: query type, retrieval time, data cutoff, source policy, warnings.
- `teams`: names, aliases, HLTV IDs, rank snapshots when available.
- `match`: event, format, schedule, LAN/online context, status.
- `lineups`: starters, coaches, stand-ins, missing lineup warnings.
- `players`: annual ratings, event ratings, missing rating flags.
- `maps`: map pool data, sample sizes, raw win rate, weighted win rate if available, pick/ban.
- `map_details`: per-map detail such as total rounds, rounds won, CT/T, pistol rounds, first-kill conversion, and first-death recovery.
- `head_to_head`: direct matchup history when found.
- `recent_matches`: recent match and map rows.
- `veto`: veto steps and map order when visible.
- `scores`: map scores and match result when visible.
- `warnings`: small samples, stale data, missing fields, parse limitations.
- `not_included`: fields intentionally excluded from factual data.

### 2. Decision Inputs

`decision_inputs` are factual, model-ready features. They do not decide who wins.

Common groups:

- `map_pool`
- `head_to_head`
- `player_form`
- `roster_state`
- `match_context`
- `data_quality`

These fields are intended for downstream user-owned models. Different users can weight the same inputs differently.

### 3. No Prediction Layer

`model_inference` is not part of this skill. If users need probabilities or strategy, they should feed the factual data pack and `decision_inputs` into their own model.

## Example JSON Skeleton

```json
{
  "schema_version": "hltv-cs2-data.v1",
  "metadata": {
    "query_type": "match_data_pack",
    "requested_at": "2026-05-01T12:00:00Z",
    "data_cutoff": "2026-05-01T12:00:00Z",
    "source_policy": "direct_hltv",
    "missing_fields": [],
    "warnings": []
  },
  "match": {
    "hltv_match_id": 2393346,
    "hltv_url": "https://www.hltv.org/matches/2393346/g2-vs-faze-blast-rivals-2026-season-1",
    "event_name": null,
    "format": "bo3",
    "status": "scheduled"
  },
  "teams": [],
  "lineups": [],
  "players": [],
  "maps": [],
  "head_to_head": {
    "status": "not_loaded",
    "matches": []
  },
  "recent_matches": {},
  "veto": {
    "status": "unavailable",
    "steps": []
  },
  "scores": {
    "status": "not_started",
    "maps": []
  },
  "decision_inputs": {
    "map_pool": [],
    "head_to_head": [],
    "player_form": [],
    "roster_state": [],
    "match_context": [],
    "data_quality": []
  },
  "not_included": [
    "betting_ev",
    "stake_sizing",
    "private_strategy_rules",
    "model_inference"
  ]
}
```

## Direct HLTV Mode

Direct HLTV-only mode is not the default complete-analysis path. It is a partial fallback.

It is designed for cases where the model can read public HLTV pages but cannot read the static JSON database export or API. The model may return visible facts and missing-field warnings only.

Direct mode is useful, but it has limits:

- Some pages may be blocked or unavailable.
- Event ratings, annual player stats, and annual team map summary pages may fail with `cache miss` or Cloudflare challenge. This does not mean the HLTV URL is wrong or the data does not exist.
- Lineups may not be visible before match time.
- Veto may not exist until closer to match start.
- Historical `as_of` backtests are approximate unless a timestamped warehouse exists.
- CT/T map side win rates require visiting each map detail page and should be provided by API / warehouse or internal collector mode.
- Side-specific CT/T scores and round-level data are Phase 2 fields.

When data is missing, the skill should label it as missing rather than infer it.

## Data Availability

### Public Standalone Quick Matrix

| Data | Public Standalone Provides |
|:--|:--|
| Match information | Yes from HLTV discovery and/or exported match pack |
| Team IDs / lineup | Yes when visible/exported |
| Event ID | Yes when discoverable/exported |
| Event rating | Static JSON/API when exported; direct fallback may attempt and mark missing |
| Annual player rating | Static JSON/API when exported; direct fallback may attempt and mark missing |
| Current-year map summary | Static JSON/API when exported; direct fallback may attempt and mark fallback context |
| W/D/L, win rate, pick %, ban % | Static JSON/API exact rows when exported |
| CT/T side win rates | Static JSON/API when exported |
| Historical backtest snapshots | No; API / warehouse enhanced mode only |

### Mode-by-Mode Matrix

| Data | Direct HLTV Partial | In-app / Browser Session | Internal Collector | API / Warehouse |
|:--|:--:|:--:|:--:|:--:|
| Match basics | Usually yes | Yes | Yes | Yes |
| Lineup / veto / score | When visible | Yes | Yes | Yes |
| Team page roster / rank / period rating | Usually yes | Yes | Yes | Yes |
| Match-page recent-core map data | Usually yes | Yes | Yes | Yes |
| Current-year team map summary | Attempt, may fail | Usually yes | Yes | Yes |
| Current-year team player stats | Attempt, may fail | Usually yes | Yes | Yes |
| Event ratings | Attempt, may fail | Usually yes | Yes | Yes |
| CT/T map side win rates | No by default | Possible but slow | Yes | Yes |
| Exact as-of snapshots | No guarantee | No guarantee | Yes | Yes |

Use these warnings when direct fallback cannot retrieve a known stats URL:

- `fetch_failed_cache_miss`
- `fetch_failed_cf_challenge`
- `stats_page_unavailable_in_direct_mode`
- `pro_api_required_for_full_coverage`

## API / Warehouse Mode

The skill also documents an optional product architecture:

```text
HLTV -> collector -> central warehouse/API -> hltv-cs2-data skill -> downstream model
```

This mode is useful for:

- Repeatable historical backtests.
- Raw snapshot retention.
- Stronger freshness guarantees.
- Match detail, veto, lineup, and result snapshots.
- Product APIs shared across users.

The skill does not require this architecture to be useful, but the references define how it should work if built.

## Reference Files

The skill uses progressive disclosure. `SKILL.md` stays compact and loads references only when needed.

- `references/product-brief.md`: product positioning and scope.
- `references/standalone-mode.md`: direct HLTV usage.
- `references/data-availability.md`: what each access mode can and cannot provide.
- `references/data-pack-contract.md`: Markdown and JSON output contract.
- `references/decision-inputs.md`: model-ready factual feature schema.
- `references/query-workflow.md`: common query recipes.
- `references/backtest-mode.md`: cutoff and historical replay discipline.
- `references/api-contract.md`: optional API shape.
- `references/collector-contract.md`: optional collector and warehouse requirements.

## Data Boundary

The factual data pack may include descriptive historical aggregates, such as:

- Raw map win rate.
- Weighted map win rate if available from the configured data source.
- Pick and ban percentage.
- Sample size.
- Player rating.
- Recent result rows.

These are historical facts or derived descriptive fields. They are not predictions.

Predicted probability, win-rate judgment, score forecast, or strategy recommendation is outside this skill. Feed the data pack and `decision_inputs` into a downstream model if that layer is needed.

## Safety and Access Policy

- Use public HLTV pages or a configured HLTV-derived API.
- Do not bypass access controls.
- If Cloudflare or another access control blocks data collection, return a partial pack with warnings.
- Do not fabricate missing data.
- Do not expose private credentials, internal database tables, or hidden strategy rules.

## Development

Validate the skill before publishing:

```bash
python3 /path/to/skill-creator/scripts/quick_validate.py hltv-cs2-data
```

Recommended privacy checks before sharing:

```bash
find . -name .DS_Store -print
rg -n "credential|local path|private display URL|hidden strategy name" .
```

Boundary documentation may mention private systems in generic terms, but it should not point to a real private system, URL, credential, or local path.

## License

MIT
