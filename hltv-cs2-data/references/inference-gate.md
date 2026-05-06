# Inference Gate

This reference controls when `hltv-cs2-data` may append numeric model inference after a factual data pack.

The goal is not to stop downstream models from judging. The goal is to prevent exact percentages from being produced when the data pack is too incomplete, especially in direct HLTV partial fallback where HLTV stats pages may be blocked by Cloudflare or unavailable to the host model.

## Completeness Levels

Use one of these labels in `metadata.completeness_level`.

| Level | Meaning | Numeric inference |
|:---|:---|:---:|
| `complete` | Match basics, team IDs, lineup, current-year map summaries for both teams, annual player ratings for both teams, event ratings when event ID is known, and visible veto/result context when applicable. | Allowed |
| `usable` | Match basics, team IDs, current-year map summaries for both teams, plus either annual ratings or event ratings for most expected starters. Some non-critical fields may be missing. | Allowed with caveats |
| `partial` | Match basics and team IDs exist, but map summaries or player ratings are incomplete. | No exact percentages |
| `blocked` | HLTV pages are blocked, cache-missed, or only search snippets are available. | No exact percentages |

`complete` and `usable` require a successful structured data source check: API/warehouse response or public static JSON manifest plus at least one exact record path. Direct HLTV/search-only output cannot be marked `complete` or `usable` for numeric inference.

## High-Impact Fields

For a match or team-vs-team probability request, treat these as high-impact fields:

- Current-year team map summary for both teams.
- Annual player rating for expected starters or current roster.
- Event player rating when `eventId` is known.
- Confirmed or expected lineup, including coach/stand-in/new-roster notes.
- Veto/map-order context when the user asks for map-level or BO3/BO5 probability.
- Recent direct H2H or recent map rows, when requested by the user.

## Required Inference Metadata

Every `Model Inference` section must expose these fields before any conclusion:

```json
{
  "completeness_level": "complete | usable | partial | blocked",
  "inference_permission": "numeric_allowed | qualitative_only | blocked",
  "missing_high_impact_fields": [],
  "disclaimer": "This section is model-derived inference, not HLTV fact data."
}
```

Chinese Markdown must say:

```text
以下为模型推理，不是 HLTV 事实数据。
```

If `inference_permission` is not `numeric_allowed`, do not output exact match or map percentages.

## Blocking Rules

Block exact numeric inference if any of these is true:

1. The model did not successfully query a structured database/API/static JSON source after resolving HLTV identity.
2. Current-year map summary is missing for either team.
3. Player rating coverage is missing for both annual and event views.
4. Two or more high-impact fields are missing, blocked by Cloudflare, or available only from search snippets.
5. The only usable inputs are ranking, news snippets, market prices, or broad narrative context.
6. The source mode is direct HLTV partial fallback and marked `blocked`.

If blocked, add warning:

```json
{
  "code": "core_data_insufficient_for_numeric_inference",
  "severity": "high",
  "message": "Core map/rating/lineup data is incomplete; exact win-rate percentages are disabled for this data pack."
}
```

If the structured source was not queried, also add:

```json
{
  "code": "structured_database_not_queried",
  "severity": "high",
  "message": "The model resolved public HLTV/search facts but did not read the API/warehouse/static JSON database record required by hltv-cs2-data."
}
```

## What To Output When Blocked

Allowed:

- Factual data collected successfully.
- Field-level missing data table.
- Source URLs attempted.
- Qualitative direction if explicitly requested, clearly marked low confidence.

Not allowed:

- `65-72%`, `60/40`, or similar exact win-rate ranges.
- Map-level win probabilities.
- Implied betting or strategy recommendations.
- Filling missing fields from market prices or intuition.
- Veto prediction or full pre-match report when the structured database was not queried.
- Betting advice, odds analysis, EV, Kelly, stake sizing, or max buy price. These never belong to `hltv-cs2-data`, even when numeric model inference is allowed.

Chinese phrasing:

```text
本次数据完整度不足，不能给可靠的具体胜率百分比。
原因：双方年度地图池 / 选手 rating / event rating 中有关键字段未加载。
可以给方向性观察，但不能把它当成完整数据模型结论。

如果没有读取 API / 静态 JSON 数据库记录，还必须说明：
本次没有成功读取 hltv-cs2-data 的结构化数据库记录，因此只能输出部分 HLTV 事实，不能输出完整逐图分析或具体胜率。
```

## What To Output When Allowed

If the gate passes and the user explicitly asked for judgment:

- Put all probabilities under `Model Inference`.
- State that the inference is model-derived and not HLTV fact data.
- Include `completeness_level`, key missing caveats, and whether veto is pre-match unknown.
- Keep factual fields separate from inference fields.
- Put Veto prediction only inside `Model Inference` as `veto_hypothesis`; factual `Veto / 比分` may contain only observed veto, map order, scores, or `unavailable`.
- Do not add betting advice or odds commentary.

## API / Warehouse Override

API or warehouse mode may satisfy the gate even when direct HLTV is blocked, if the API response contains current-year map summary, player ratings, lineup context, and freshness metadata.

Do not treat API availability as automatic permission to infer. Apply the same completeness check to the returned data.
