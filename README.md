# hltv-cs2-data Skill

[中文说明](README.zh-CN.md)

`hltv-cs2-data` is a data-first skill for CS2 match and team analysis workflows. It turns HLTV-derived structured records into compact Markdown data packs for a user, another model, or a downstream strategy system.

It only defines the data layer: when the skill is triggered, the factual data pack must be produced first. What happens after the data pack is up to the user and host model.

## Structured Source Model

This skill is bound to a **structured data capability contract**, not to one fixed website or one fixed JSON path.

Structured source priority:

1. Configured API / warehouse.
2. User-provided static JSON export.
3. Default public raw GitHub static export.

Any source can replace the default source if it provides equivalent capabilities:

- team / match identity resolution or index
- match data pack
- team map summary
- team map details
- player ratings
- event ratings
- data gaps / warnings

Current default public static export freshness is recorded in `manifest.generated_at`. The skill file does not need to be updated every time the static database is refreshed.

Default public manifest URL:

```text
https://raw.githubusercontent.com/smallmeji/hltv-cs2-data-public/main/manifest.json
```

This is the preferred built-in public fallback for this repository. It is not the only possible source. If the user configures an API or provides another static JSON source, use the configured source first.

If the host model cannot read raw GitHub, these compatibility manifest aliases may be used:

```text
https://raw.githubusercontent.com/smallmeji/hltv-cs2-data-public/main/latest/manifest.json
https://smallmeji.github.io/hltv-cs2-data-public/latest/manifest.json
```

Important: the skill package and the public data export are now split. `hltv-cs2-data-skill` contains instructions only. `hltv-cs2-data-public` contains the static JSON data. A `404` from `hltv-cs2-data-platform/public-data` or the old `hltv-cs2-data-skill/public-data` path is stale-path usage, not database unavailability.

Forbidden as structured data sources:

- `smallmeji.github.io` URLs outside the accepted `hltv-cs2-data-public` compatibility alias
- GitHub Pages URLs outside the accepted `hltv-cs2-data-public` compatibility alias
- platform website pages
- `hltv-cs2-data-platform` public-data paths
- `hltv-cs2-data-skill` public-data paths after the repository split
- the raw `https://raw.githubusercontent.com/.../public-data` directory itself
- search snippets, wiki pages, market pages, news snippets, or model memory

If a model reports a `404`, first check what it fetched:

| Fetched target | Meaning | Correct action |
|:--|:--|:--|
| `/public-data` directory | Old repository layout or a directory URL, not a JSON file | Fetch the `hltv-cs2-data-public` manifest |
| `/manifest.json` | Required public entry point | If this fails, mark structured data unavailable |
| Exact JSON file | That record may be absent | Use manifest/indexes to find available paths |
| `smallmeji.github.io/hltv-cs2-data-platform/...` or `hltv-cs2-data-skill/public-data/...` | Stale docs or cached model memory | Use the `hltv-cs2-data-public` raw manifest or compatibility alias; reinstall/update if it still uses the stale URL |

## Repository Split

- `hltv-cs2-data-skill`: skill instructions, examples, and package only.
- `hltv-cs2-data-public`: generated static JSON data only.
- Updating match/team/event data updates the public data repository, not this skill repository.
- Users only need to update the skill when instructions or the data contract changes.

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
- Map summary: sample, W-D-L, win rate, Pick%, Ban%. If a current active map has no played sample for the selected team/filter, render it as a zero-sample row (`0 sample / 0-0 / 0%`), not as `missing`.
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
2. Fetch the selected structured source's capabilities / manifest.
3. Fetch exact records or endpoints by ID.
4. Use structured records for map pool, player ratings, CT/T, pistol, first-kill/first-death, Pick/Ban, H2H, recent rows, veto, score, and result fields.
5. If structured records are unavailable, stop at a partial data report and show `structured_database_not_queried`.

Default public static source record examples:

```text
matches/index.json
matches/by-date/<YYYY-MM-DD>.json
matches/by-date/index.json
events/index.json
teams/aliases.json
teams/<hltvTeamId>/summary.json
teams/<hltvTeamId>/players.json
teams/<hltvTeamId>/maps-overall.json
teams/<hltvTeamId>/map-details-overall.json
teams/<hltvTeamId>/map-details-lan.json
matches/<hltvMatchId>/data-pack.json
events/<eventId>/player-ratings.json
```

If the user gives a natural-language match request such as `May 10 PGL Aurora vs Heroic`, use the structured source's date index and match search/index first. Match date, event text, and both team aliases. When exactly one row matches, fetch the corresponding match data pack. In the default public source, this means `matches/by-date/<YYYY-MM-DD>.json` first, then `matches/index.json` and that row's `data_pack_path`. Do not ask the user for event ID or match ID first, and do not downgrade to a missing-field report just because direct HLTV page lookup failed.

Event rating IDs should also be resolved automatically: read `event_id` from the match data pack or match index, then try `events/<eventId>/player-ratings.json`. If a new event has no event ratings yet, output `event rating: not collected / not available yet`, and keep annual Rating 3.0 plus map data.

Do not fill missing map/player/detail fields from search summaries, wiki pages, market pages, news snippets, or model memory.

For Chinese output, use Chinese labels or Chinese plus standard acronyms: `范围`, `队伍`, `地图`, `样本`, `W-L`, `地图胜率`, `Pick%`, `Ban%`, `CT/T`, `手枪局`, `首杀后`, `首死后`, `回合`. Do not expose raw schema headers such as `Scope`, `sample`, `wr`, `pick_pct`, or `ban_pct` unless the user asks for raw JSON/schema/debug output.

Minimum valid flow for the default public raw GitHub source:

```text
manifest.json -> matches/by-date/<YYYY-MM-DD>.json when a date is present -> matches/index.json -> matches/<matchId>/data-pack.json -> events/<eventId>/player-ratings.json when event ID exists
```

For another API/static source, use the equivalent capabilities/search/data-pack flow. If the resolved data pack contains a `markdown` field, use it as the factual skeleton. Do not rebuild an HLTV-only missing-field report.

## Two-Phase Usage

The skill itself only owns phase 1:

1. Produce the factual data pack from structured records.
2. Stop the skill boundary.

If the user asks for a winner, probability, map lean, or strategy, the host model may continue after the data pack, but it should clearly label that section as model judgment outside this skill. The judgment must be based on the data pack and must not rewrite missing fields, map samples, ratings, or source status.

Recommended boundary:

```text
--- hltv-cs2-data data pack ends ---
The following is model judgment outside this skill:
```

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

The factual data pack must not contain:

- Veto prediction
- possible map sequence
- winner/win-probability conclusion
- Model Inference / model reasoning
- downstream recommendations or conclusions

Anything after the data pack is outside this skill's scope.

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

For install/debug testing, the model may mention the current version and the raw GitHub manifest URL before producing structured map/player sections. Normal user-facing reports should keep the source log compact and hide raw URLs.

Natural-language smoke test:

```text
Use hltv-cs2-data for the PGL Astana 2026 Aurora vs Heroic data pack.
```

Correct behavior:

- Internally reads `manifest.json`
- Internally reads `matches/index.json`
- Matches `matches/2394116/data-pack.json`
- Outputs Overall/LAN per-map detail, CT/T, pistol, first-kill, first-death, and Rating 3.0
- Does not show raw URLs/JSON unless debug is requested
- Does not put Veto prediction or winner judgment inside the factual data pack

If the model still says `smallmeji.github.io/hltv-cs2-data-platform` returned 404, it is using stale instructions or stale conversation context. Reinstall/update the skill and start a fresh conversation.

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
- `hltv-cs2-data/references/standalone-mode.md`: public static export/direct HLTV boundaries and smoke tests.
- `hltv-cs2-data/references/data-availability.md`: what each source mode can and cannot provide.
- `hltv-cs2-data/references/data-pack-contract.md`: output schema and field rules.
- `hltv-cs2-data/references/product-brief.md`: product scope.
- `hltv-cs2-data/references/collector-contract.md`: collector/API architecture.
- `hltv-cs2-data/references/api-config.md`: optional API mode.
- `hltv-cs2-data/references/examples.md`: output examples.

## License

MIT
