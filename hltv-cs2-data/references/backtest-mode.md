# Backtest Mode

Backtest mode returns only data that should have been visible before a given timestamp.

## Required Input

- `matchId` or both teams plus scheduled match time.
- `as_of_date` as ISO 8601 UTC.

If `as_of_date` is missing, do not call the result a backtest.

## Visibility Rules

Allowed before match:

- Rankings snapshots captured before `as_of_date`.
- Team map stats captured before `as_of_date`.
- Team match history with match dates before `as_of_date`.
- Player rating snapshots captured before `as_of_date`.
- Pre-match lineups if captured before `as_of_date`.
- Veto only if it was already published before `as_of_date`.

Not allowed:

- Final score after `as_of_date`.
- Post-match map history.
- Player ratings that include the match being tested.
- Veto published after `as_of_date`.
- Any manual hindsight notes.

## Reconstruction Policy

If exact historical snapshots do not exist:

- Reconstruct only from raw match rows whose `matchDate` is before `as_of_date`.
- Mark every reconstructed field with `trust_level: "reconstructed"`.
- Add a warning: `exact_snapshot_unavailable`.
- Do not reconstruct player rating unless the source query window and retrieval time are known.

## Output Requirements

Backtest output must include:

- `as_of_date`.
- `visibility_cutoff`.
- `included_snapshot_ids` or source references.
- `excluded_future_data_note`.
- `warnings`.

## Common Failure Cases

- Match result already stored but requested as pre-match: exclude result fields.
- Veto is known now but was not available at cutoff: mark veto unavailable.
- Roster changed after cutoff: use cutoff-visible roster only, or mark unavailable.
- Event ratings were published after the match started: do not include them in pre-match backtest unless timestamped proof exists.
