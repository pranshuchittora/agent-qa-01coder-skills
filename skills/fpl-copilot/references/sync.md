# Data synchronization contract

Use available HTTP, JSON processing, database, and time capabilities. Select suitable native tools, installed commands, or scripting libraries from the environment. This contract is the authoritative refresh workflow; executable helpers are optional implementations and their behavior must satisfy it.

## Required datasets and freshness

Inspect the `sync_metadata` singleton and whether required tables contain a usable snapshot. Timestamps represent successful persistence, in UTC.

| Dataset | Fetch when | Success metadata |
|---------|------------|------------------|
| Bootstrap | Missing, incomplete, or older than 6 hours | `last_bootstrap_sync` |
| Fixtures | Missing, incomplete, older than 2 hours on a match day, or older than 6 hours otherwise | `last_fixtures_sync` |
| Player history | Needed for requested metrics and missing or not known to reflect the current bootstrap snapshot | Per-player receipt described below |

A match day has a fixture kicking off on the current UTC date. If fixture data cannot establish this, use the 2-hour threshold. An explicit user request to refresh overrides freshness reuse. Initialize bootstrap before fixtures, and fixtures before player histories, to satisfy referenced IDs.

The existing schema lacks per-player freshness fields. Persist a companion `player-history-sync.json` document in the resolved data root, keyed by player ID, with `fetched_at` and the bootstrap timestamp used for that refresh. Verify these receipts when deciding whether to reuse a player's history. Missing receipts mean freshness is unknown; an empty verified history still gets a receipt. The global `last_player_stats_sync` and `last_synced_gameweek` certify only a completed all-player batch, never a single-player refresh.

## Fetch and validate

1. Request the relevant JSON endpoint from the base URL in `api.md`: bootstrap, fixtures, or the resolved player's element summary. Check transport success and JSON validity before processing.
2. Validate the expected arrays (`teams`, `events`, `elements`, fixtures array, or `history`) and required identifiers. A missing array is an error; an empty player history can be valid. Treat counts as observations, not fixed season constants.
3. Use the mapping tables in `api.md`. Preserve fields with matching names; apply its explicit renamed fields. Cast numeric strings to numbers and booleans to 0/1. Preserve null for unknown scores, kickoff times, gameweek assignments, and playing chances. Apply schema defaults only to absent fields where a default is defined; reject invalid required values.
4. Map element types 1–4 to GKP/DEF/MID/FWD; report unknown codes as validation failures. For player-history rows, use the requested player ID when `element` is absent; reject a conflicting ID. Set `updated_at` to the successful snapshot's UTC timestamp for tables that define it.

**Complete when:** the response is validated and mapped records satisfy the schema, with unexpected fields/types reported rather than silently guessed.

## Persist and verify

- Initialize an absent store from `schema.sql` using available database operations. Bind values as parameters or use equivalent structured writes.
- Persist each dataset atomically. Bootstrap includes teams, gameweeks, and players in one transaction; fixtures form another; each player's history forms another. Upsert by schema keys, preserving multiple fixtures in a double gameweek. Replaying the same snapshot must not duplicate records.
- Commit successful-refresh metadata with the corresponding dataset. A failed validation or transaction leaves the prior successful snapshot and timestamp intact.
- Write per-player receipts only after the history transaction succeeds. If interrupted before the receipt write, a repeat refresh is safe. For an explicitly requested full-player batch, record progress and mark global completion only after every requested player succeeds.
- Verify persisted keys, counts against the validated response, and success metadata before reporting completion. Report a failed or partial refresh and the last usable snapshot's age. Use cached data only with that limitation stated; if no usable data exists, identify the blocked analysis.
- Space player-history requests by at least 100ms, respect server retry guidance, and bound retries. Keep bulk payloads in local/runtime storage where supported; return only the records needed for analysis to model context.

**Complete when:** required records are queryable, verification matches the imported snapshot, and success metadata corresponds to committed data.
