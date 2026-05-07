---
name: hltv-cs2-data
description: "Use for CS2 factual data packs from an HLTV-derived structured database/static JSON export: match/team resolution, map history, map detail, player ratings, lineup, veto, results, event ratings, decision-input context, or backtest context. When this skill is invoked, structured data hydration is mandatory and automatic; do not wait for the user to say database, data source, or static JSON. Default public source is the raw GitHub manifest at /public-data/manifest.json; never use smallmeji.github.io or GitHub Pages as the data source. HLTV pages are only identity/visible-fact sources; map/player/detail analysis must read configured API/warehouse or public static JSON records first. Data-layer only: this skill requires source-backed factual data-pack output first. Anything after the data pack is outside this skill."
metadata:
  short-description: Strategy-neutral HLTV CS2 data packs
---

# HLTV CS2 Data

This skill returns factual, source-bound CS2 data packs for downstream models or user-owned strategy.

Default skill behavior is data-pack-only. The skill only governs the data layer: when triggered, it must hydrate structured data and output the factual data pack first. Anything after the data pack is outside this skill.

Mandatory behavior: if the user invokes `hltv-cs2-data`, says "based on this skill", says "用/基于这个 skill", or asks a model that has this skill installed to look at a CS2 match/team, the model must automatically read the structured data source. The user does not need to say "go to the data source", "query the database", "read static JSON", or provide record paths. HLTV lookup alone is never a complete answer for this skill.

Current public-data freshness is defined by `manifest.generated_at`; do not rely on a hard-coded date in the skill text.

Required default manifest URL:

```text
https://raw.githubusercontent.com/smallmeji/hltv-cs2-data-skill/main/public-data/manifest.json
```

This is the only default public static database entry point. Do not derive another host. If a model reports a GitHub Pages / platform-site public-data URL, treat it as stale instructions and return to the raw GitHub manifest above.

## Fast Operating Contract

Use this compact contract before reading deeper references:

1. **Resolve identity briefly**: use HLTV/search only to identify the match/team/event, visible lineup, format, time, status, observed veto/score/result, or event ID.
2. **Hydrate structured data immediately**: read configured API/warehouse, user-provided static JSON, or the default raw GitHub manifest and exact records. Do this even when the user does not say "database".
3. **Use structured rows for stats**: maps, CT/T, pistol, first-kill/first-death, Pick/Ban, player ratings, recent rows, H2H, and decision inputs must come from exact structured records when available.
4. **Output a compact factual data pack first**: Chinese prompts use Chinese headers. Normal reports hide raw URLs, record paths, and JSON unless the user asks for debug/audit/machine-readable output.
5. **Then stop the skill boundary**: if the user asked for a winner, probability, or strategy, the host model may add a clearly separated section after the data pack, but that section is not part of `hltv-cs2-data`.

The skill is data-source flexible. It is not bound to this GitHub repository if another source provides equivalent manifest/search/data-pack/team/player/event-rating capabilities. The default raw GitHub export is only the public fallback.

## Hard Source Gate

Before outputting any map/player/detail section, the model must read a structured source and at least one exact structured record. This is automatic for every `hltv-cs2-data` or "this skill" request, including one-sentence prompts such as `基于这个 skill 看一下 5 月 9 号 PGL Aurora 和 Heroic 的比赛`.

Structured source can be:

- configured API / warehouse
- user-provided static JSON export
- default public raw GitHub static export

The default public source uses raw GitHub manifest/index/data-pack paths, but alternate sources only need to provide equivalent capabilities.

Forbidden as structured data sources:

- `smallmeji.github.io`
- GitHub Pages URLs
- platform website URLs
- the raw `/public-data` directory URL without `manifest.json`
- search snippets, wiki pages, market pages, or model memory

If the output would say the public source is `smallmeji.github.io` or GitHub Pages and returned `404`, that is stale-source leakage. Retry the raw manifest URL above. If the raw manifest cannot be read, stop at partial HLTV facts; do not write a complete report.

For the default public raw GitHub source, natural-language match lookup uses:

```text
manifest.json -> matches/index.json -> matches/<matchId>/data-pack.json
```

For an alternate API/static source, use the equivalent flow:

```text
capabilities/manifest -> match search/index -> match data pack
```

If the resolved match data pack contains a rendered Markdown field, use that Markdown as the canonical factual skeleton. Do not rebuild the report from HLTV alone.

Do not ask the user to confirm database usage. If structured data cannot be read, fail closed with `structured_database_not_queried`.

## Data Completeness Gate

When a structured match data pack exists, do not summarize it into a thinner report that drops available fields. Preserve all rendered map-detail columns unless the user explicitly asks for a short summary.

Required map-detail fields when present:

```text
sample_maps, W-L, raw_win_rate, pick_pct, ban_pct, event_tier_breakdown,
CT/T, pistol, first_kill, first_death, total_rounds_played / rounds_won
```

These are canonical output fields, not a hard storage schema. If another API/source uses different raw names, map its equivalent fields to these labels before output.

Scope discipline:

- Do not merge `Overall` and `LAN` into one unlabeled "2026" table.
- Do not mix Overall values and LAN values in the same row.
- If both Overall and LAN are present, show both sections or explicitly label the chosen scope.
- Do not say a field is missing until checking the exact selected `overall` / `lan` row in the structured source.

The first section of every normal Chinese output must be exactly `数据源执行记录`. A vague label such as `数据源: HLTV CS2 数据平台` is not enough.

Minimum source log:

```text
## 数据源执行记录

- 技能触发：hltv-cs2-data
- 身份定位：HLTV / static match index / user input
- 结构化数据：已读取 API/warehouse 或 public static database export
- 读取记录：match data-pack / team map details / player ratings / event ratings
- 输出边界：事实数据包优先；数据包之后不属于本 skill
```

If the model cannot truthfully write `结构化数据：已读取 API/warehouse` or `结构化数据：已读取 public static database export`, it must stop before `地图池总览` and `逐图详细分析`.

## Non-Negotiable Rules

1. **Data-layer boundary**: factual sections must contain only source-backed data. The skill's job is to produce the data pack first. If the user asks for judgment, the host model can continue only after a clear boundary such as `以下为非本 skill 的模型判断`.
2. **HLTV is identity first**: use HLTV pages to resolve match ID, team IDs, event ID, schedule, format, visible lineup, visible veto, score, or result.
3. **Structured data first for analysis**: map pools, map detail, CT/T, pistol, first kill/death, Pick/Ban, player ratings, recent rows, H2H, and decision inputs must come from exact API/warehouse/static JSON records when available.
4. **Manifest gate**: before writing map tables, player ratings, per-map detail, or decision inputs, fetch the manifest and at least one exact JSON/API record.
5. **No user cue required**: never wait for the user to explicitly request the database or data source. A request that names this skill, says "this skill", or asks for a CS2 match/team while the skill is active is already permission and instruction to hydrate structured data.
6. **Match index for natural language**: if the user gives event/team names without a match URL, use the selected structured source's match search/index after the capabilities/manifest step and search it before falling back to team-only comparison. In the default public source this is `matches/index.json`. When a single matching row is found, fetch that row's match data-pack reference first, such as `data_pack_path`.
7. **Fail closed**: if manifest plus exact records cannot be read, output partial HLTV facts only with `structured_database_not_queried`.
8. **Exact rows only**: if a player/rating row is absent, write `missing` / `无数据`. For a current active map that a team has not played in the selected year/filter, output a zero-sample row (`样本=0`, `W-L=0-0`, rates/Pick/Ban/rounds=`0`) and mark `状态=0样本`; do not infer it from opponent data, another map, rankings, snippets, or memory.
9. **Current active map pool only**: for 2026, use `Ancient`, `Anubis`, `Dust2`, `Inferno`, `Mirage`, `Nuke`, `Overpass` unless structured records explicitly say otherwise.
10. **Normal reports hide raw paths**: include compact source status. Show exact URLs, record paths/endpoints, and JSON only for debug/audit/machine-readable requests.
11. **Rating field and label**: structured JSON currently stores player rating in `rating2` for compatibility. Treat `rating2` as the exported HLTV `Rating 3.0` value unless a record explicitly says otherwise. Do not look for or emit `rating_2_0`; human reports should label the column `Rating 3.0`.
12. **Chinese-facing labels**: for Chinese reports, use Chinese table headers or Chinese+standard acronyms. Prefer `范围`, `队伍`, `地图`, `样本`, `W-L`, `地图胜率`, `Pick%`, `Ban%`, `CT/T`, `手枪局`, `首杀后`, `首死后`, `回合`; do not expose raw-only headers such as `Scope`, `sample`, `wr`, `pick_pct`, or `ban_pct` unless the user asks for raw JSON/schema.
13. **Invalid-output triggers**: if a normal data-pack section includes `smallmeji.github.io`, GitHub Pages as the data source, vague source-only text such as `HLTV CS2 数据平台` without `已读取 public static database export` or `已读取 API/warehouse`, `Rating 2.0`, merged/mixed Overall-LAN map rows, active current-map rows shown as `missing` instead of zero-sample records, raw English-only table headers in a Chinese report, a claim that CT/T/pistol/first-kill/first-death are missing while a structured match data pack was available, raw JSON/URLs in a normal human report, or model-derived judgment/probability/Veto hypotheses mixed into factual sections, the data-pack output is non-compliant and must be retried from the structured source.

## Query Routing

Classify the request first, then load only the relevant reference.

| Query type | Examples | Required reference |
|:--|:--|:--|
| `match_data_pack` | HLTV match URL; exact scheduled match | `references/query-workflow.md`, optionally `references/data-pack-contract.md` |
| `team_comparison` | `NAVI 和 G2`, `FaZe + FURIA` | `references/query-workflow.md` |
| `hypothetical_match` | `假设 NAVI 和 G2 打 S 级 LAN BO3` | `references/query-workflow.md` |
| `single_team_profile` | `看一下 Aurora 的数据` | `references/query-workflow.md` |
| `event_ratings` | `event=8250 player ratings` | `references/query-workflow.md` |
| `backtest` | `as_of_date`, historical cutoff | `references/backtest-mode.md` |
| product/API design | API, collector, schema, warehouse | `references/product-brief.md`, `references/api-contract.md`, `references/collector-contract.md` |
| data availability/debug | 404, CF, missing fields, source modes | `references/standalone-mode.md`, `references/data-availability.md` |

Do not bulk-load all references.

## Default Workflow

1. Resolve identity:
   - If a match URL exists, read the HLTV match page and extract `hltvMatchId`, teams, event, format, time, status, visible lineup/veto/score, and event ID when visible.
   - If event/team names are given, search/read HLTV match/event/upcoming/results/team pages only long enough to resolve canonical IDs, then use `matches/index.json` to find an exported exact match before treating the request as team-only.
   - If no real match exists, use `teams/index.json` after manifest fetch and mark the request as `team_comparison`, `hypothetical_match`, or `single_team_profile`.
2. Fetch structured data:
   - Use configured API/warehouse first when available.
   - Otherwise use a user-provided static JSON source when available.
   - Otherwise use the default raw GitHub public export. Fetch its manifest exactly. Do not fetch `smallmeji.github.io`, GitHub Pages, platform-site URLs, or the raw directory URL as the structured source.
   - For natural-language match lookup, use the source's match search/index. In the default public export this is `matches/index.json`; if exactly one row matches the event/team context, fetch its `data_pack_path`.
   - Prefer the resolved match data pack for known matches. In the default public export this is `matches/<matchId>/data-pack.json`.
   - Fetch exact team records as needed:
     - `teams/<id>/summary.json`
     - `teams/<id>/maps-overall.json`
     - `teams/<id>/maps-lan.json`
     - `teams/<id>/map-details-overall.json`
     - `teams/<id>/map-details-lan.json`
     - `teams/<id>/players.json`
   - If event ID is known, attempt `events/<eventId>/player-ratings.json`.
3. Write a compact `数据源执行记录` before analysis.
4. Use database/API/static rows for factual tables.
5. Mark missing or not-applicable fields explicitly.
6. Output Markdown by default. Include JSON only if requested for machine-readable, downstream-model, debug, or audit use.

If a match data-pack contains a `markdown` field, use that Markdown as the canonical factual skeleton. Do not state that CT/T, pistol, first-kill, first-death, rounds, Pick/Ban, or tier breakdown are missing when those values are present in the data-pack.

If the user asks for a natural-language judgment such as "谁胜率高", this skill still only governs the data-pack step: output the data pack first and keep factual tables source-backed. A separated downstream judgment section is allowed only after the data pack and must not rewrite factual fields.

## 404 Handling

When a public-data request returns `404`:

1. If the URL is the raw GitHub `/public-data` directory, retry exact `/public-data/manifest.json`.
2. If the URL contains `smallmeji.github.io`, GitHub Pages, or a platform-site public-data path, treat it as stale instructions, not as database failure. Replace it with the required raw GitHub manifest URL.
3. If the URL is not the required raw GitHub manifest host/path, treat it as stale instructions and do not use it.
4. If `manifest.json` works but an exact default-source record path 404s, use manifest/index files to discover available records. For alternate sources, use the equivalent source capabilities/index.
5. If `manifest.json` cannot be read, stop with `structured_database_unavailable` / `structured_database_not_queried`.

Do not turn a 404 into a full HLTV-only report.

## Output Shape

For Chinese comparison or match requests, use this compact order:

1. `数据源执行记录`
2. `数据状态 / 数据缺口`
3. `比赛信息` or `假设条件` / `队伍信息`
4. `队伍与选手 Rating 3.0`
5. `地图池总览`
6. `逐图详细分析`
7. `特殊 Veto 变量`
8. `近期记录 / H2H` when exact rows exist
9. `Veto / 比分` only for observed facts or `not_applicable`
10. `给模型的决策输入`
11. `JSON` only when requested

Forbidden data-pack content:

- `推荐投注` / `EV` / `Kelly`
- model-derived winner lean or probability inside `数据源执行记录`, `地图池总览`, `逐图详细分析`, `Veto / 比分`, `给模型的决策输入`, or JSON facts

Allowed after the data pack when the user asks for it:

- clearly separated downstream model judgment
- user-defined strategy interpretation
- non-skill reasoning based on the data pack

Use a boundary such as:

```text
--- hltv-cs2-data 数据包结束 ---
以下为非本 skill 的模型判断：
```

For English prompts, mirror the same structure in English.

Normal source log example:

```text
数据源执行记录：
- 技能触发：hltv-cs2-data
- HLTV 定位：成功
- 结构化数据：已读取 public static database export
- 读取记录：match data-pack + team map details + player ratings
- 输出边界：事实数据包优先；数据包之后不属于本 skill
- 字段来源：地图详情=static_database，赛事信息=direct_hltv，Veto=missing
```

If structured records were not read:

```text
当前只能定位 HLTV 基础信息；结构化数据库/API/静态 JSON 未成功读取，不能生成完整 hltv-cs2-data 报告。
warning: structured_database_not_queried
```

## Special Modes

### Single-Team Profile

Use when the user asks for one team only. Resolve one team ID and read the team records. Do not require a match ID. Opponent, event rating, Veto, score, and match status are `not_applicable` unless the user provides match/event context.

### Hypothetical Match

Use when the user asks an assumed matchup. Resolve both team IDs and read both teams' records. Treat user-provided tier, format, LAN/online, date, event, or map subset as `assumptions`, not observed HLTV facts. Do not invent lineup, Veto, map order, score, event ratings, or H2H rows.

### Direct HLTV Partial Fallback

Use only after structured data was attempted and failed or lacks a field. Direct fallback can report visible facts and missing warnings only; it must not produce full map tables, per-map detail, numeric inference, or a prediction-style report.

## Reference Map

- `references/query-workflow.md`: detailed recipes for match URL, team comparison, single-team, hypothetical match, event ratings, and exact record references.
- `references/data-pack-contract.md`: stable Markdown/JSON output contract and field mapping.
- `references/decision-inputs.md`: how to organize factual decision inputs.
- `references/standalone-mode.md`: public standalone mode and external-model failure handling.
- `references/data-availability.md`: what each access mode can and cannot provide.
- `references/backtest-mode.md`: cutoff and time-travel rules.
- `references/api-contract.md`: hosted API shape.
- `references/collector-contract.md`: collector/warehouse requirements.
- `references/product-brief.md`: product positioning.
- `references/inference-gate.md`: compatibility note for older no-prediction references.
