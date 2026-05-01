# Query Workflow

Use these recipes when preparing data packs.

## Team-vs-Team Query

User input:

```text
faze + 黑豹
```

Workflow:

1. If static JSON/API is configured, resolve aliases from that source first; otherwise resolve aliases using HLTV pages/search.
2. Fetch current or `as_of_date` team snapshots if available.
3. Fetch map stats for the requested tier/filter if available.
4. Fetch player ratings and lineup if a match is specified and the source is reachable.
5. Return Markdown + JSON factual data pack.
6. Build `Decision Inputs` from available facts.
7. If the user explicitly asks for judgment, apply `references/inference-gate.md`; append numeric `Model Inference` only when the gate passes.
8. Match the user's language in Markdown. For Chinese prompts, use Chinese section titles and table labels, while preserving JSON keys in English.

## Match URL Query

User input:

```text
https://www.hltv.org/matches/2393335/faze-vs-furia-blast-rivals-2026-season-1
```

Workflow:

1. Extract `hltvMatchId`.
2. Read static JSON/API data-pack first when available; otherwise read the HLTV match page directly.
3. Resolve event, teams, format, schedule.
4. Resolve both canonical team IDs and slugs from match-page team links or team pages. Do not stop at match-page summary data.
5. Fetch lineups from the match page when visible.
6. Always attempt to fetch player ratings for visible starters:
   - Extract or resolve `eventId` from the match page/event link whenever possible.
   - Event rating must be fetched from `https://www.hltv.org/stats/players?event=<eventId>` when `eventId` is known.
   - Annual rating from HLTV team player stats for the current calendar year, e.g. `https://www.hltv.org/stats/teams/players/<teamId>/<slug>?startDate=2026-01-01&endDate=2026-12-31`. If `as_of_date` is provided, use that date's calendar year.
   - If a player is missing from event stats, keep annual rating if available and mark `rating_event` as `缺失`.
   - If a coach or stand-in has no rating, mark `rating_status` explicitly instead of guessing.
7. Fetch yearly team map stats from HLTV team stats pages using the current calendar-year window, e.g. `https://www.hltv.org/stats/teams/maps/<teamId>/<slug>?startDate=2026-01-01&endDate=2026-12-31`. This is the primary source for the `地图池` section.
8. In lightweight mode, stop at the team map summary page and mark `map_side_stats` as `Pro/API only` or `未加载`. CT/T side win rates require visiting each map detail page and should be collected asynchronously in Pro/API mode.
9. Use match-page map stats only as `recent_core_context` or fallback if the yearly stats pages cannot be reached. Label its time window explicitly.
10. Fetch head-to-head map rows when reachable; if not reachable, mark `head_to_head` as `未加载`.
11. Include veto/scores only if available and visible for the requested mode.
12. For Chinese prompts, output the compact Chinese structure: `数据状态`, `比赛信息`, `队伍与阵容`, `选手数据`, `地图池`, `近期记录 / H2H`, `警匪胜率`, `Veto / 比分`, `给模型的决策输入`, `数据缺口`, `JSON`.
13. If the user asked for probability or winner judgment, apply `references/inference-gate.md`. If the gate fails, add `core_data_insufficient_for_numeric_inference` and do not give exact percentages.

## Event Ratings Query

User input:

```text
event=8250 player ratings
```

Workflow:

1. Fetch event player ratings snapshot.
2. Use the canonical HLTV URL format `https://www.hltv.org/stats/players?event=<eventId>`.
3. Match players to lineups when a match is provided.
4. Mark missing players explicitly.
5. If a lineup player is missing from the event page, attempt annual rating fallback before marking the player fully missing.
6. Do not average or interpret rating unless the user asks for a descriptive aggregate.

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

For Chinese output, missing data should be explicit and brief:

```text
数据缺口：
- 赛事 rating：未加载
- Veto：赛前不可见
- 警匪胜率：HLTV team map stats 页面未加载或 collector 暂未采集
- 精确 as_of 快照：不可用，本次只能标记为重建数据

不能推断：
- 不能把缺失 rating 补成估计值
- 不能把历史胜率当成预测胜率
```

## Model Inference Request

User input:

```text
Use hltv-cs2-data to collect G2 vs FaZe data, then estimate map and match win rates.
```

Workflow:

1. Produce the HLTV factual data pack first.
2. Build `Decision Inputs` from facts such as map pool, H2H, player form, roster state, match context, and data quality.
3. Keep all facts and decision inputs out of inference fields.
4. Apply `references/inference-gate.md`.
5. Add numeric `Model Inference` only when the gate passes.
6. If the gate fails, output a short note such as `核心地图/rating 数据不足，不能给可靠具体胜率百分比`; qualitative direction is allowed only if explicitly requested and must be marked low confidence.
7. State clearly that inference is model-derived and not HLTV data.
8. Do not include betting EV, Kelly, stake, or private correction logic unless the user explicitly invokes a separate strategy or betting framework.
