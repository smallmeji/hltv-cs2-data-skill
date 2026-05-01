# Query Workflow

Use these recipes when preparing data packs.

## Team-vs-Team Query

User input:

```text
faze + 黑豹
```

Workflow:

1. Resolve aliases to HLTV IDs using HLTV pages/search by default; use API/warehouse only if explicitly configured.
2. Fetch current or `as_of_date` team snapshots if available.
3. Fetch map stats for the requested tier/filter if available.
4. Fetch player ratings and lineup if a match is specified and the source is reachable.
5. Return Markdown + JSON factual data pack.
6. Build `Decision Inputs` from available facts.
7. If the user explicitly asks for judgment, append a separate `Model Inference` section after the data pack.

## Match URL Query

User input:

```text
https://www.hltv.org/matches/2393335/faze-vs-furia-blast-rivals-2026-season-1
```

Workflow:

1. Extract `hltvMatchId`.
2. Read the HLTV match page directly by default; use API/warehouse only if explicitly configured.
3. Resolve event, teams, format, schedule.
4. Fetch lineups and ratings if visible/reachable.
5. Fetch map pool and map history if visible/reachable.
6. Include veto/scores only if available and visible for the requested mode.

## Event Ratings Query

User input:

```text
event=8250 player ratings
```

Workflow:

1. Fetch event player ratings snapshot.
2. Match players to lineups when a match is provided.
3. Mark missing players explicitly.
4. Do not average or interpret rating unless the user asks for a descriptive aggregate.

## Backtest Query

User input:

```text
Backtest match 2393335 as of 2026-04-30 22:30 Asia/Shanghai
```

Workflow:

1. Convert local time to UTC.
2. Fetch only records visible before cutoff.
3. Exclude final score and post-match records.
4. Mark reconstructed fields.
5. Return data pack with `backtest_context`.

## Missing Data Response

If the data warehouse/API is not available:

```text
当前无法生成完整 hltv-cs2-data 数据包，因为缺少中央数据源/API。
可输出的数据：...
缺失的数据：...
不能推断的内容：...
```

Do not fall back to private prediction reports, private notebooks, or hidden strategy documents as substitute data sources.

## Model Inference Request

User input:

```text
Use hltv-cs2-data to collect G2 vs FaZe data, then estimate map and match win rates.
```

Workflow:

1. Produce the HLTV factual data pack first.
2. Build `Decision Inputs` from facts such as map pool, H2H, player form, roster state, match context, and data quality.
3. Keep all facts and decision inputs out of inference fields.
4. Add `Model Inference` only after the data pack.
5. State clearly that inference is model-derived and not HLTV data.
6. Do not include betting EV, Kelly, stake, or private correction logic unless the user explicitly invokes a separate strategy or betting framework.
