# hltv-cs2-data Skill

[中文说明](README.zh-CN.md)

`hltv-cs2-data` is a skill for collecting and structuring CS2 data from HLTV into model-ready data packs.

The skill is intentionally strategy-neutral. It provides factual HLTV-derived data, organizes useful decision inputs, and keeps any model judgment separate from the data layer. It does not include a built-in betting model, private correction rules, EV logic, or stake sizing.

## What It Is For

Use this skill when you want an LLM to gather the data needed to analyze a CS2 match, team pair, event, roster, or historical matchup.

Typical use cases:

- Get a data pack for one HLTV match URL.
- Compare two teams across map pool, player form, roster status, and head-to-head history.
- Inspect map-specific win rates, CT/T side win rates, head-to-head records, and recent player form when available.
- Pull event player ratings when an event stats page is available.
- Build backtest-ready context with an `as_of` cutoff.
- Feed structured Markdown and JSON into a separate model or strategy layer.

The skill can support prediction workflows, but it does not make predictions by default. If the user explicitly asks the model to judge win rates or probabilities, the skill instructs the model to place that output under a separate `Model Inference` section.

Numeric `Model Inference` is gated by data completeness. If direct HLTV access is blocked by Cloudflare, or if current-year map summaries / player ratings cannot be loaded, the skill should return the factual partial pack and missing-field warnings, but it should not output exact win-rate percentages.

User-facing Markdown follows the user's language. For Chinese prompts, the skill should output Chinese headings, labels, warnings, and concise tables, while keeping JSON field names stable in English.

## What It Is Not

This repository does not provide:

- A private CS2 prediction model.
- Betting recommendations.
- EV, Kelly, max buy price, or stake sizing.
- A scraper service that bypasses access controls.
- A local database or private data dependency.
- Guaranteed complete historical snapshots in direct HLTV mode.

For normal public use, the skill first reads the default public static JSON manifest:

```text
https://raw.githubusercontent.com/smallmeji/hltv-cs2-data-skill/main/public-data/manifest.json
```

If that source is unreachable or does not contain the requested record, the skill falls back to public HLTV pages available in the current session. API/warehouse mode can provide richer and more reproducible snapshots when configured, but it is optional.

## Product Tiers

The skill is useful in three tiers:

| Tier | Best For | What To Expect |
|:--|:--|:--|
| Static JSON mode | Shared data packs for Claude/GPT/user models | Default public path. Avoids live HLTV/Cloudflare failures and gives stable team, match, event, and compare packs. |
| Lightweight mode | One-off public HLTV lookups | Fallback when static/API data is missing. Good for match basics, lineups, H2H, visible match-page map context, and best-effort stats-page lookups. Deep stats pages may fail and must be labeled as missing. |
| API mode | Repeatable analysis and production use | Recommended for complete current-year stats, CT/T side data, exact historical backtests, lineup/veto/result snapshots, batch usage, and stable freshness guarantees. |

Lightweight mode is enough for one-off public HLTV lookups. Static JSON or API mode is recommended for repeatable analysis, CT/T side data, historical backtests, and production use.

In lightweight mode, the host model's web reader may fail on HLTV stats pages. This is expected. The skill marks those fields as missing and uses warning code `core_data_insufficient_for_numeric_inference` when the missing fields are too important for numeric probability output.

## Usage Modes

`hltv-cs2-data` supports three operating modes.

### 1. Static JSON Mode

This is the default public mode. Users can ask natural questions without providing configuration:

```text
Use hltv-cs2-data to compare FaZe and G2. Who has the higher win rate?
```

The skill should try the default manifest first:

```text
https://raw.githubusercontent.com/smallmeji/hltv-cs2-data-skill/main/public-data/manifest.json
```

The base path prefix is:

```text
https://raw.githubusercontent.com/smallmeji/hltv-cs2-data-skill/main/public-data
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

In this mode, the skill should read static JSON before trying live HLTV pages.

If the user asks "who is favored", "who has the higher win rate", or similar, the skill still starts with factual data. When map-detail data exists, the Markdown output must include:

- `Map Pool Overview`: seven-map summary with samples, W-D-L, win rate, pick %, and ban %.
- `Per-Map Detail`: one section per playable map, including overall/LAN sample, CT/T split, pistol win rate, first-kill / first-death conversion, rounds, and pick/ban tendency.
- `Special Veto Variables`: maps with no current-year data, extreme ban rate, tiny sample, or one side effectively not playing the map.
- `Model Inference`: only when the user explicitly asks for judgment, clearly labeled as model-derived interpretation.

### 2. Lightweight / Direct HLTV Mode

Use this mode when you only want the skill instructions and public HLTV pages.

- No API key is required.
- No user-owned database is required.
- No local browser, CDP session, Playwright session, or local scraper is required from the user.
- The model gathers available data from HLTV pages through the host model's normal web/page-reading/search capability.
- Historical backtests are marked as `reconstructed` unless an exact snapshot is available.
- Missing fields must be reported explicitly instead of being guessed.

### 3. Pro / API Mode

Use this mode when a maintained HLTV data API is available.

Configure:

```text
HLTV_CS2_API_BASE_URL=https://your-api.example.com
HLTV_CS2_API_KEY=your_api_key
```

In API mode, the skill should query the data API first. The API is expected to return standardized Markdown + JSON data packs from a maintained collector and database.

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
   - Direct HLTV mode: `references/standalone-mode.md`
   - Backtest: `references/backtest-mode.md`
3. Ask the model to follow the fact-vs-inference boundary described in the skill.

## Quick Usage

### Match URL

```text
Use hltv-cs2-data for this match:
https://www.hltv.org/matches/2393346/g2-vs-faze-blast-rivals-2026-season-1
Output Markdown and JSON.
```

Expected behavior:

- Resolve match ID, teams, event, format, schedule, and status.
- Gather available map, lineup, player rating, veto, score, and recent result data.
- Mark missing data explicitly.
- Output a factual data pack and JSON block.
- If current-year map summaries or player ratings are blocked/missing, do not output exact win-rate percentages; show the missing fields and recommend API/warehouse mode for full coverage.

### Team Comparison

```text
Use hltv-cs2-data to compare NAVI and G2.
Focus on map pool, player form, roster changes, and head-to-head data.
```

Expected behavior:

- Resolve both teams to HLTV identities when possible.
- Collect public HLTV-derived team and map data.
- Organize factual factors into `decision_inputs`.
- Avoid declaring a winner unless the user explicitly asks for model inference.

### Team Comparison + Winner Lean

```text
Use hltv-cs2-data to compare PGL Aurora and Heroic. Who has the higher win rate?
Return factual data first, then per-map detail, then Model Inference.
```

Expected behavior:

1. Resolve the teams and any relevant match from static JSON / API / direct HLTV fallback.
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
5. Add `Model Inference` only at the end.

The skill is not a fixed prediction model. It organizes data; the final lean is the calling model's interpretation of that data.

### Data + Model Inference

```text
Use hltv-cs2-data to collect G2 vs FaZe data, then estimate map and match win rates.
```

Expected behavior:

1. Output the factual HLTV data pack first.
2. Output `Decision Inputs`.
3. If map-detail data exists, output `Per-Map Detail` and `Special Veto Variables` before inference.
4. Append a clearly labeled `Model Inference` section.
5. State that inferred probabilities are model-derived, not HLTV facts.

### Backtest Context

```text
Use hltv-cs2-data to backtest match 2393335 as of 2026-04-30 22:30 Asia/Shanghai.
Only include data visible before that time.
```

Expected behavior:

- Apply cutoff discipline.
- Exclude final scores and post-match records if the requested time is pre-match.
- Mark exact historical snapshots as unavailable or reconstructed when a warehouse/API is not configured.

## Worked Example: Lightweight Match Data Pack

This is the common path for a user who only installs the skill and does not have an API key.

### Step 1: Ask with a match URL

```text
Use hltv-cs2-data for this match:
https://www.hltv.org/matches/2393346/g2-vs-faze-blast-rivals-2026-season-1

Output Markdown and JSON.
Return factual data only, no win-rate prediction.
If a field cannot be retrieved, include the source URL and warning code.
```

### Step 2: What the skill should attempt

Lightweight mode should try these sources in order:

1. Match page: match ID, teams, event, format, schedule, status, visible lineup, veto, scores.
2. Team identity: HLTV team IDs, slugs, visible rank and roster context.
3. Event ID: parse from the match page event link when available.
4. Event player ratings: try `https://www.hltv.org/stats/players?event=<eventId>`.
5. Annual team player stats: try `https://www.hltv.org/stats/teams/players/<teamId>/<slug>?startDate=2026-01-01&endDate=2026-12-31`.
6. Annual team map summary: try `https://www.hltv.org/stats/teams/maps/<teamId>/<slug>?startDate=2026-01-01&endDate=2026-12-31`.
7. If a deep stats page fails, keep the canonical URL and mark the exact field as `missing`.

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
| Data Gaps | Missing fields, source URLs, warning codes |
| JSON | Stable English keys for downstream systems |

### Step 4: If the user asks for judgment

If the user asks "which team has the higher win rate?", the output should become:

1. Factual HLTV data pack first.
2. `decision_inputs` second.
3. A clearly labeled `Model Inference` section last.

Any probability or winner judgment belongs in `Model Inference`. It is model-derived interpretation, not HLTV source data.

### Step 5: How to read cache miss

`cache miss` does not mean HLTV has no data, and it does not mean the URL is wrong. It means the lightweight page reader did not return a readable snapshot.

Recommended field-level warning:

```json
{
  "field": "event_player_ratings",
  "source_url": "https://www.hltv.org/stats/players?event=8250",
  "status": "missing",
  "warning": "fetch_failed_cache_miss",
  "meaning": "The URL is known, but lightweight direct mode could not retrieve a readable table snapshot."
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

### 3. Optional Model Inference

`model_inference` appears only when the user explicitly asks for judgment, probability, or strategy.

It may include:

- Map win probabilities.
- Match win probability.
- Winner lean.
- Reasoning summary.
- Uncertainty notes.

It must not be mixed into factual fields.

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
    "private_strategy_rules"
  ],
  "model_inference": null
}
```

## Direct HLTV Mode

Direct HLTV mode is the default.

It is designed for users who only install the skill and do not have a private API, local database, scraper, local browser, CDP session, or Playwright session. The model reads the public HLTV pages available through its normal web/page-reading/search capability and returns the best available data pack.

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

### Lightweight Quick Matrix

| Data | Lightweight Direct Provides |
|:--|:--|
| Match information | Yes |
| Team IDs / lineup | Yes |
| Event ID | Yes |
| Event rating | Attempt; mark missing if retrieval fails |
| Annual player rating | Attempt; mark missing if retrieval fails |
| Current-year map summary | Yes when reachable; otherwise mark fallback context |
| W/D/L, win rate, pick %, ban % | Yes from current-year map summary when reachable |
| CT/T side win rates | Static JSON / API can provide them; lightweight direct mode does not guarantee them |
| Historical backtest snapshots | No; API / warehouse enhanced mode only |

### Mode-by-Mode Matrix

| Data | Lightweight Direct | In-app / Browser Session | Internal Collector | API / Warehouse |
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

Use these warnings when lightweight direct mode cannot retrieve a known stats URL:

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

Predicted probability, win-rate judgment, score forecast, or strategy recommendation must be placed only under `model_inference`, and only when the user asks for it.

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
