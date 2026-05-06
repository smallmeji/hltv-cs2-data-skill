# hltv-cs2-data Skill

[中文说明](README.zh-CN.md)

`hltv-cs2-data` is a data-only skill for CS2 match and team analysis workflows. It turns HLTV-derived structured records into compact Markdown data packs for a user, another model, or a downstream strategy system.

It does not output predictions, win probabilities, Veto hypotheses, score guesses, betting advice, EV, Kelly, or stake sizing.

## Current Data Source

Public static export version:

```text
static-raw-2026-05-06
```

Default manifest URL:

```text
https://raw.githubusercontent.com/smallmeji/hltv-cs2-data-skill/main/public-data/manifest.json
```

This is the only default public static database entry point. Do not use a GitHub Pages URL, platform-site URL, or raw directory URL as the data source.

If a model reports a `404`, first check what it fetched:

| Fetched target | Meaning | Correct action |
|:--|:--|:--|
| `/public-data` directory | Directories are not JSON files | Fetch `/public-data/manifest.json` |
| `/manifest.json` | Required public entry point | If this fails, mark structured data unavailable |
| Exact JSON file | That record may be absent | Use manifest/indexes to find available paths |
| Any GitHub Pages/platform URL | Stale docs or cached model memory | Reinstall/update the skill |

## What It Does

The skill can produce factual data packs for:

- One HLTV match URL.
- Two-team comparison, such as `FaZe vs G2`.
- Single-team profile, such as `Aurora`.
- Hypothetical comparison, such as `assume NAVI vs G2 in an S-tier LAN BO3`.
- Event player rating context when the event ID is known.
- Backtest context with an `as_of` cutoff when records are available.

Typical fields:

- Match identity: match ID, event, format, schedule, status.
- Team identity: HLTV team IDs, names, rankings.
- Lineup context when visible or exported.
- Current active-map pool.
- Map summary: sample, W-D-L, win rate, Pick%, Ban%.
- Map detail when exported: CT/T round win rate, pistol rate, first-kill and first-death conversion, rounds played/won.
- Player ratings: annual and event HLTV Rating 3.0 when exported. JSON uses `rating2` as the compatibility field name; do not use `rating_2_0`.
- Observed veto, map order, score, and result when available.
- Data gaps and warnings.
- `decision_inputs`: factual signals for a downstream model.

Current active maps in the public export are:

```text
Ancient, Anubis, Dust2, Inferno, Mirage, Nuke, Overpass
```

Do not add inactive maps such as Vertigo, Cache, or Train unless they exist in a structured record.

## Required Workflow

For a match or team query:

1. Use HLTV only to resolve visible identity facts: match ID, team ID, event ID, visible lineup, format, schedule, status, published veto, and result.
2. Fetch the static database manifest.
3. Fetch exact JSON records by ID.
4. Use structured records for map pool, player ratings, CT/T, pistol, first-kill/first-death, Pick/Ban, H2H, recent rows, veto, score, and result fields.
5. If structured records are unavailable, stop at a partial data report and show `structured_database_not_queried`.

Expected record examples:

```text
teams/<hltvTeamId>/summary.json
teams/<hltvTeamId>/players.json
teams/<hltvTeamId>/maps-overall.json
teams/<hltvTeamId>/map-details-overall.json
teams/<hltvTeamId>/map-details-lan.json
matches/<hltvMatchId>/data-pack.json
events/<eventId>/player-ratings.json
```

Do not fill missing map/player/detail fields from search summaries, wiki pages, market pages, news snippets, or model memory.

## Output Boundary

Normal user-facing output should be concise and should not print raw URLs, record paths, or full JSON unless the user asks for debug, audit, source details, JSON, or downstream-machine output.

Recommended human report sections:

1. Data Source Execution Log
2. Data Status / Data Gaps
3. Match Info
4. Teams and Player Rating 3.0
5. Map Pool Overview
6. Per-Map Detail
7. Special Veto Variables
8. Decision Inputs

When the user asks "who has the higher win rate" or "who is favored", the skill still provides data only. The calling model or user's strategy layer may make the final judgment outside this skill.

## Install

Install the packaged skill file supported by your host environment:

```text
hltv-cs2-data.skill
```

After installation, test with:

```text
Use hltv-cs2-data to get the data pack for:
https://www.hltv.org/matches/2393346/g2-vs-faze-blast-rivals-2026-season-1
```

The model should mention the current version and the raw GitHub manifest URL before producing structured map/player sections.

## Example Prompts

```text
Use hltv-cs2-data to get the factual data pack for this match:
https://www.hltv.org/matches/2393346/g2-vs-faze-blast-rivals-2026-season-1
```

```text
Use hltv-cs2-data to compare NAVI and G2 by map pool, player ratings, roster context, and available per-map detail.
```

```text
Use hltv-cs2-data to build a single-team data profile for Aurora.
```

```text
Use hltv-cs2-data to build a hypothetical S-tier LAN BO3 comparison for Aurora vs Heroic.
```

## Reference Files

Detailed instructions live in the skill package:

- `hltv-cs2-data/SKILL.md`: core runtime rules.
- `hltv-cs2-data/references/data-source-modes.md`: public static export/API/direct HLTV boundaries.
- `hltv-cs2-data/references/data-pack-contract.md`: output schema and field rules.
- `hltv-cs2-data/references/product-brief.md`: product scope.
- `hltv-cs2-data/references/collector-contract.md`: collector/API architecture.
- `hltv-cs2-data/references/api-config.md`: optional API mode.
- `hltv-cs2-data/references/examples.md`: output examples.

## License

MIT
