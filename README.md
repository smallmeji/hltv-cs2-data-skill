# hltv-cs2-data Skill

`hltv-cs2-data` is a Codex skill for collecting and structuring CS2 data from HLTV into model-ready data packs.

The skill is intentionally strategy-neutral. It provides factual HLTV-derived data, organizes useful decision inputs, and keeps any model judgment separate from the data layer. It does not include a built-in betting model, private correction rules, EV logic, or stake sizing.

## What It Is For

Use this skill when you want an LLM to gather the data needed to analyze a CS2 match, team pair, event, roster, or historical matchup.

Typical use cases:

- Get a data pack for one HLTV match URL.
- Compare two teams across map pool, player form, roster status, and head-to-head history.
- Pull event player ratings when an event stats page is available.
- Build backtest-ready context with an `as_of` cutoff.
- Feed structured Markdown and JSON into a separate model or strategy layer.

The skill can support prediction workflows, but it does not make predictions by default. If the user explicitly asks the model to judge win rates or probabilities, the skill instructs the model to place that output under a separate `Model Inference` section.

## What It Is Not

This repository does not provide:

- A private CS2 prediction model.
- Betting recommendations.
- EV, Kelly, max buy price, or stake sizing.
- A scraper service that bypasses access controls.
- A local database or private data dependency.
- Guaranteed complete historical snapshots in direct HLTV mode.

The default mode works from public HLTV pages available in the current session. A future warehouse/API can provide richer and more reproducible snapshots, but it is optional.

## Installation

### Option 1: Install the folder

Copy the `hltv-cs2-data/` folder into your Codex skills directory:

```bash
mkdir -p ~/.codex/skills
cp -R hltv-cs2-data ~/.codex/skills/
```

Restart Codex or reload skills if your environment requires it.

### Option 2: Install the packaged skill

Use `hltv-cs2-data.skill` if your Codex environment supports installing `.skill` packages.

The package contains the same `hltv-cs2-data/` skill folder and references.

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

### Data + Model Inference

```text
Use hltv-cs2-data to collect G2 vs FaZe data, then estimate map and match win rates.
```

Expected behavior:

1. Output the factual HLTV data pack first.
2. Output `Decision Inputs`.
3. Append a clearly labeled `Model Inference` section.
4. State that inferred probabilities are model-derived, not HLTV facts.

### Backtest Context

```text
Use hltv-cs2-data to backtest match 2393335 as of 2026-04-30 22:30 Asia/Shanghai.
Only include data visible before that time.
```

Expected behavior:

- Apply cutoff discipline.
- Exclude final scores and post-match records if the requested time is pre-match.
- Mark exact historical snapshots as unavailable or reconstructed when a warehouse/API is not configured.

## Output Layers

The skill separates output into three layers.

### 1. HLTV Data Pack

This is the factual layer. It may include:

- `metadata`: query type, retrieval time, data cutoff, source policy, warnings.
- `teams`: names, aliases, HLTV IDs, rank snapshots when available.
- `match`: event, format, schedule, LAN/online context, status.
- `lineups`: starters, coaches, stand-ins, missing lineup warnings.
- `players`: annual ratings, event ratings, missing rating flags.
- `maps`: map pool data, sample sizes, raw win rate, weighted win rate if available.
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

It is designed for users who only install the skill and do not have a private API, local database, or scraper. The model reads the public HLTV pages available in the current session and returns the best available data pack.

Direct mode is useful, but it has limits:

- Some pages may be blocked or unavailable.
- Event player ratings may need a separate HLTV stats URL.
- Lineups may not be visible before match time.
- Veto may not exist until closer to match start.
- Historical `as_of` backtests are approximate unless a timestamped warehouse exists.
- Side-specific CT/T scores and round-level data are Phase 2 fields.

When data is missing, the skill should label it as missing rather than infer it.

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
