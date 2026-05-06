# Product Brief

`hltv-cs2-data` is the data skill for the `hltv-cs2` product concept.

## One-Line Positioning

Provide standardized HLTV CS2 data packs for downstream LLMs, models, analysts, and products, while keeping factual data separate from optional model inference.

The skill supplies data from HLTV. The user-owned model supplies judgment and strategy.

The skill must be usable in two practical forms:

- Public standalone skill: works immediately after installation by using public HLTV pages for match/team discovery, then the hosted static JSON database export for structured stats.
- Optional product API mode: uses a central warehouse/API for complete and repeatable data packs.

## What This Product Does

Given a match, team pair, event, map, or historical cutoff, return a multidimensional data pack and model-ready decision inputs covering:

- Team identity, HLTV IDs, aliases, and ranking snapshots.
- Match context: event, tier, format, LAN/online, time, status.
- Map data: pick rate, ban rate, raw win rate, weighted win rate, sample size, tier filter, recent results.
- Player data: annual ratings, event ratings, missing ratings.
- Lineup data: starters, coaches, stand-ins, roster changes, new-team warnings.
- Head-to-head data: direct matchup history when available.
- Veto and map order when published.
- Scores and historical results when available.
- Data quality warnings and freshness.
- Decision input groups for map pool, head-to-head, player form, roster state, match context, and data quality.
- Data availability labels showing whether a field is available from public static data, direct HLTV partial fallback, in-app/browser sessions, internal collector runs, or hosted API/warehouse data.

The same data pack can be used by different users with different strategies.

## Model Inference Boundary

The factual data pack must not include model-derived:

- Map win probability.
- Match win probability.
- Winner prediction.
- Score prediction.
- Betting EV.
- Stake sizing.
- Strategy recommendation.

If the user explicitly asks for probability or judgment, the calling model may append a separate `Model Inference` section after the data pack and decision inputs. That section must state that it is not HLTV fact data.

## Intended User Flow

Example:

```text
User asks: FaZe + FURIA, who has higher win rate?
```

`hltv-cs2-data` should first return the structured data and `Decision Inputs` needed for that judgment, including maps, ratings, recent form, roster notes, and warnings. If the user explicitly wants the model to judge, add a separate `Model Inference` section.

Example:

```text
User asks: Backtest NAVI vs GamerLegion before match time.
```

`hltv-cs2-data` should return only data visible before the cutoff and mark missing or reconstructed fields.

## Product Architecture

Default public standalone:

```text
HLTV public pages for identity -> hosted static JSON database export -> hltv-cs2-data skill -> user-owned model
```

Optional enhanced product API:

```text
HLTV -> collector -> central warehouse/API -> hltv-cs2-data skill -> user-owned model
```

End users should not run local scrapers, local databases, local browsers, CDP sessions, or Playwright sessions.

The display website is not a data source. It may show data, but the source of truth for shared skill use is HLTV itself. A central warehouse/API can later cache and enrich HLTV-derived data.

## Phase 1 Data Scope

Phase 1 must support:

- Teams.
- Matches.
- Maps.
- Players.
- Lineups.
- Head-to-head records.
- Per-map CT/T side win rates from team map stats pages when available.
- Veto.
- Scores and results.
- Warnings.
- Backtest cutoff discipline.

Default data window is the current calendar year. For the current season, use 2026-only data (`2026-01-01` to `2026-12-31`) unless the user explicitly requests another window.

## Phase 2 Data Scope

Phase 2 can add:

- Match-specific CT/T side score splits.
- Half scores.
- Starting side.
- Round-level data if HLTV exposes it reliably.
- More complete roster timelines.
- Exact historical snapshots for all aggregates.

Do not block Phase 1 on Phase 2 data.
