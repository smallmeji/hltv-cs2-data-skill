# hltv-cs2-data Skill

[中文说明](README.zh-CN.md)

`hltv-cs2-data` is a data-first skill for CS2 match and team analysis workflows. It turns HLTV-derived structured records into compact Markdown data packs for a user, another model, or a downstream strategy system.

It only defines the data layer: when the skill is triggered, the factual data pack must be produced first. Any analysis, judgment, or strategy after the data pack is outside this skill.

Betting advice, odds analysis, EV, Kelly, and stake sizing remain outside this skill.

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

Current default public static export version:

```text
static-raw-2026-05-06
```

Default public manifest URL:

```text
https://raw.githubusercontent.com/smallmeji/hltv-cs2-data-skill/main/public-data/manifest.json
```

This is the built-in public fallback for this repository. It is not the only possible source. If the user configures an API or provides another static JSON source, use the configured source first.

Forbidden as structured data sources:

- `smallmeji.github.io`
- GitHub Pages
- platform website pages
- the raw `https://raw.githubusercontent.com/.../public-data` directory itself
- search snippets, wiki pages, market pages, news snippets, or model memory

If a model reports a `404`, first check what it fetched:

| Fetched target | Meaning | Correct action |
|:--|:--|:--|
| `/public-data` directory | Directories are not JSON files | Fetch `/public-data/manifest.json` |
| `/manifest.json` | Required public entry point | If this fails, mark structured data unavailable |
| Exact JSON file | That record may be absent | Use manifest/indexes to find available paths |
| `smallmeji.github.io` / GitHub Pages / platform URL | Stale docs or cached model memory | Use the raw GitHub manifest; reinstall/update if it still uses the stale URL |

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
2. Fetch the selected structured source's capabilities / manifest.
3. Fetch exact records or endpoints by ID.
4. Use structured records for map pool, player ratings, CT/T, pistol, first-kill/first-death, Pick/Ban, H2H, recent rows, veto, score, and result fields.
5. If structured records are unavailable, stop at a partial data report and show `structured_database_not_queried`.

Default public static source record examples:

```text
matches/index.json
teams/<hltvTeamId>/summary.json
teams/<hltvTeamId>/players.json
teams/<hltvTeamId>/maps-overall.json
teams/<hltvTeamId>/map-details-overall.json
teams/<hltvTeamId>/map-details-lan.json
matches/<hltvMatchId>/data-pack.json
events/<eventId>/player-ratings.json
```

If the user gives a natural-language match request such as `PGL Aurora vs Heroic`, use the structured source's match search/index first. When exactly one row matches, fetch the corresponding match data pack. In the default public source, this means `matches/index.json` and that row's `data_pack_path`. Do not downgrade to a missing-field report just because direct HLTV page lookup failed.

Do not fill missing map/player/detail fields from search summaries, wiki pages, market pages, news snippets, or model memory.

Minimum valid flow for the default public raw GitHub source:

```text
manifest.json -> matches/index.json -> matches/<matchId>/data-pack.json
```

For another API/static source, use the equivalent capabilities/search/data-pack flow. If the resolved data pack contains a `markdown` field, use it as the factual skeleton. Do not rebuild an HLTV-only missing-field report.

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
- betting, EV, Kelly, or stake sizing

Anything after the data pack is outside this skill's scope. Betting-related content remains outside this skill.

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

If the model still says `smallmeji.github.io` returned 404, it is using stale instructions or stale conversation context. Reinstall/update the skill and start a fresh conversation.

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
