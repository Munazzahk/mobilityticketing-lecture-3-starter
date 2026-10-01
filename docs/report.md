# Lecture 3 – Reporting Evidence

## Baseline

Before the experiments, the base query returned:

| Operator | Date | Captured amount | Captured payments |
| --- | --- | ---: | ---: |
| OP-BUS | 2026-04-29 | 36.00 | 1 |
| OP-METRO | 2026-04-29 | 36.00 | 1 |

The `payments` table is the authority for the reporting experiment.

## Experiment results

| Test | Base/function result | Trigger summary | What happened |
| --- | --- | --- | --- |
| Initial data | OP-METRO: 36 / 1 | 0 rows | Trigger did not process old payments |
| Insert Captured 20 DKK | OP-METRO: 56 / 2 | 20 / 1 | Trigger counted the new payment but still missed old data |
| Insert Failed 20 DKK | No revenue change | No change | Failed payment was correctly ignored |
| Failed → Captured | OP-METRO: 76 / 3 | Still 20 / 1 | Trigger does not react to UPDATE |
| Captured → Refunded | OP-METRO: 56 / 2 | Still 20 / 1 | Trigger does not react to UPDATE |
| Delete captured test payment | OP-METRO: 36 / 1 | Still 20 / 1 | Trigger does not react to DELETE |
| Duplicate external reference | Duplicate was accepted | Increased to 40 / 2 | Duplicate delivery can be counted twice |

## Materialized view staleness

After inserting the first captured 20 DKK payment:

- Base query: OP-METRO = 56 DKK / 2 payments
- Materialized view: OP-METRO = 36 DKK / 1 payment

The materialized view was stale because it stores a snapshot.

After:

sql
REFRESH MATERIALIZED VIEW daily_captured_revenue;

## Responsibility matrix

| Approach | Correct/current? | Write cost | Read cost | Hidden side effects | Rebuild |
| --- | --- | --- | --- | --- | --- |
| Direct SQL | Yes, reads current payments | Low | Higher | No | Always available from base tables |
| SQL function | Yes, reads current payments | Low | Higher | No | Always available from base tables |
| Materialized view | Only after refresh | Low | Low | No | Refresh from base tables |
| Trigger summary | Not always with current trigger | Higher | Low | Yes | Must be rebuilt from base tables |

**Authority:** The `payments` table is the source of truth.
**Freshness rule:** Direct SQL and the function see current data immediately. The materialized view is current only after `REFRESH`. The trigger summary is updated on captured inserts only.
**Rebuild path:** Stored reporting data should be rebuildable from the `payments` table and its related ticket, trip and route data.

## Decision record

### Decision

For the current MobilityTicketing case, use the SQL function for daily captured revenue, with `payments` as the authority.

### Why

The function reads the current payment data, so corrections, refunds and deletions are reflected automatically. It also keeps the reporting query in one reusable place.
The materialized view is useful if reporting becomes slow, but it can be stale and must be refreshed. The current trigger summary is not reliable because it misses old data, updates and deletes.

### Alternative

If the amount of data becomes large and the function becomes too slow, a materialized view could be added for faster reporting. It must have a clear refresh rule and can always be rebuilt from the base tables.

## Issue register

### Issue 1 – Trigger summary becomes incorrect

**Problem:** The trigger only runs after a new payment is inserted.
**Evidence:** When a payment changed from `Failed` to `Captured`, the function returned 76 DKK, but the trigger summary stayed at 20 DKK. It also did not react when a captured payment was refunded or deleted.
**Consequence:** The summary table can disagree with the real payment data.
**Possible improvement:** Handle updates and deletes as well, or rebuild the summary from the base tables. For the current case, the SQL function avoids this problem.