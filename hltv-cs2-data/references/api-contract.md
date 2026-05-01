# API Contract

This is the product API shape for `hltv-cs2-data`. The skill may use this contract even before a backend exists.

## General Rules

- All API responses return JSON.
- Data-pack endpoints may also include `markdown`.
- All responses include `metadata`.
- All timestamps are ISO 8601 UTC.
- Public API terms should hide internal table names.
- Missing data is explicit, not silently omitted when material.

## Endpoints

### Resolve Team

```http
GET /teams/resolve?query=faze
```

Returns candidate teams by alias, name, slug, or HLTV ID.

### Match Data Pack

```http
GET /matches/{hltvMatchId}/data-pack?include=markdown,json
```

Returns match context, teams, lineups, ratings, map pool, veto, scores, warnings, and JSON schema fields.

### Compare Teams

```http
GET /compare?teamA=6667&teamB=8297&tier=S&asOf=2026-04-30T14:00:00Z
```

Returns factual team comparison data. Predicted probabilities must not be mixed into factual fields. If requested, return model-derived values only under a separate `model_inference` object.

Responses should include `decision_inputs` when enough factual data is available. These are model-ready factual factors, not predictions.

### Event Player Ratings

```http
GET /events/{eventId}/player-ratings
```

Returns player ratings for the event if available, plus missing-player diagnostics.

### Backtest Data Pack

```http
GET /backtest?matchId=2393335&asOf=2026-04-30T14:00:00Z
```

Returns the data state that should have been visible before the match.

## Error Shape

```json
{
  "error": {
    "code": "DATA_STALE",
    "message": "Match detail snapshot is older than the requested freshness window.",
    "retryable": true
  },
  "metadata": {
    "requested_at": "2026-05-01T12:00:00Z",
    "source_policy": "hltv_central_warehouse"
  }
}
```

Common codes:

- `TEAM_NOT_FOUND`
- `MATCH_NOT_FOUND`
- `DATA_STALE`
- `MISSING_LINEUP`
- `MISSING_EVENT_RATING`
- `BACKTEST_SNAPSHOT_UNAVAILABLE`
- `COLLECTOR_BLOCKED`
- `PARSER_FAILED`

## Freshness Defaults

- Rankings: 24 hours.
- Calendar: 6 hours.
- Match detail before match: 30 minutes for high-priority matches, 6 hours otherwise.
- Match result after match: 30 minutes after completion.
- Team map stats: 24 hours.
- Player ratings: 24 hours, unless event ratings are not published.

If freshness is outside tolerance, include warnings and return partial data when safe.

## Public Contract Boundary

The API may expose:

- HLTV IDs.
- Source URLs.
- Retrieval timestamps.
- Field-level source and trust level.

The API should not expose:

- Database credentials.
- Internal scraper browser mode.
- Private betting framework fields.
- Private correction numbers, private prediction logs, or hidden strategy-framework fields.
