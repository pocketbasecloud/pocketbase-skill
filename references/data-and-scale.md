# Data, queries and scale

PocketBase is SQLite in WAL mode. Reads are concurrent and cheap; **writes are
serialized through one lock** shared by every HTTP request, every hook, every
cron, and the admin UI. Most PocketBase performance problems are one component
holding that lock, or one unindexed recurring query keeping the database busy for
nobody.

## The write lock

- **A cron that writes one row per entity does not scale** — at a few hundred
  entities it starves interactive traffic once a minute. Batch it.
- **Do fleet-wide reads, not per-entity loops.** One query returning 500 rows
  beats 500 queries, by more than the row count suggests, because each round trip
  also contends.
- **A hook write is never just a write.** Saving a record fires its after-hooks,
  which may make blocking `$http.send` calls, which may read the record back. A
  handler that writes on every call — a heartbeat, a version report — costs far
  more than the row. Compare first, save only on change.
- **Keep transactions short.** Everything else waits.
- Prefer an in-place update over delete-and-recreate; the latter is two writes
  plus two sets of hooks.

## Batch writes

`POST /api/batch` takes a list of sub-requests and executes them as **one
transaction**.

```json
{ "requests": [
  { "method": "POST", "url": "/api/collections/metrics/records", "body": { "k": "a", "v": 1 } },
  { "method": "POST", "url": "/api/collections/metrics/records", "body": { "k": "b", "v": 2 } }
]}
```

- **Disabled by default.** Enable it in Settings, or in a migration that sets
  `settings.batch`. A disabled endpoint answers **403** "Batch requests are not
  allowed." — not 400, so a classifier that only checks for 400 mislabels it.
- `maxRequests` caps the chunk size (default 50). Exceeding it answers 400
  "Invalid batch request data." with the server's ceiling in `params.max` — read
  it and adopt it rather than hard-coding a guess.
- **One transaction means one bad row rolls back the good ones.** That is what
  makes "retry the chunk row-by-row on rejection" correct rather than
  double-writing.
- Values are coerced where SQLite would coerce them (a numeric string into a
  number field is written as a number, silently). A non-parseable value fails its
  row and therefore the chunk.
- If you adopt a smaller ceiling after a rejection, **let it re-grow** — retry
  the configured size periodically. A lifetime-pinned ceiling means a limit
  raised on the server never takes effect.

## Retry rules

**Never retry a write whose response never arrived.** A batch can commit
server-side and lose the response in transport; re-sending it duplicates rows.

- HTTP error with a parsed body → the server rejected it → retrying is safe.
- Transport failure (`TypeError`, status 0, timeout) → unknown → skip and log.

A gap in a metrics series is cheaper than double-counted rows. Whichever side you
choose, say which in the log line, or the gap becomes a mystery.

**Idempotency markers must key on the same granularity as the action.** If an
operation runs once per (resource, domain), a marker keyed only by resource
reports "already done" for the second domain. Match the key to the call.

## Indexing

- **Anything a cron or hook filters on recurringly needs an index.** A timer-driven
  filter on an unindexed column is a full table scan on a schedule, forever, with
  no user waiting on the result.
- Index the columns you sort on together with the ones you filter on —
  `(user, created)` serves `WHERE user = ? ORDER BY created DESC` from the index;
  two single-column indexes do not.
- A unique index on a deterministic key is your idempotency arbiter (see
  `schema-migrations.md`).
- Verify with the plan, not by feel:

```sql
EXPLAIN QUERY PLAN SELECT ... ;
```

A `SCAN <table>` on anything that grows is the finding. `SEARCH ... USING INDEX`
is what you want.

## Query shapes to check

**Scope subqueries by the caller.** This shape is the classic:

```sql
-- the subquery groups the ENTIRE table, then the outer WHERE throws it away
SELECT m.* FROM metrics m
JOIN (SELECT resource_id, MAX(timestamp) t FROM metrics GROUP BY resource_id) newest
  ON newest.resource_id = m.resource_id AND newest.t = m.timestamp
WHERE m.user = {:user}
```

Push the caller's predicate *into* the subquery. On a shared database, a query
whose cost is independent of who asked is a cost everyone pays.

**"Newest row" is a fragile reading.** If a producer can write a successful row
with an empty value — a collector that restarted and hasn't measured yet — then
"the newest row" shows zero for a resource that has data. Read the newest row
*that has the field*, or carry the last known value forward explicitly.

**Empty and unread are different answers.** A read that swallows per-chunk errors
and returns `[]` tells its caller "nothing there" when the truth was "I couldn't
look". Return the failures alongside the data (`{ rows, failedIds }`) and let the
caller decide.

## The logs database

PocketBase's own request logs live in a **separate** SQLite file, reachable as
`$app.auxDB()`. Read them there. Folding them into your own tables from
`$app.db()` puts log analysis on the main write lock, which is precisely the
traffic you don't want to block.

A fold like that is at-least-once by nature: advance the watermark only after the
write succeeds, and make the destination idempotent with a unique index.

## Aggregation and rollups

**Store counters, never averages or percentiles.** An average cannot be
re-averaged and a percentile cannot be re-aggregated. If an hourly row holds
`avg_ms`, the daily figure derived from it is wrong by an amount that depends on
how uneven the traffic was — and it will look plausible.

The rollup row holds sums and counts:

| Store | Not |
| --- | --- |
| `response_ms_sum`, `response_ms_count` | `avg_response_ms` |
| latency histogram buckets | `p95_response_ms` |
| `sample_count` per averaged series | one shared count |

Keep a **separate count per averaged series**. If the raw query's three averages
each exclude different rows (nulls, unsampled, failed collections), one shared
`sample_count` silently mis-weights two of them.

Other rollup discipline:

- **Roll up only closed windows.** Never touch the hour still being written.
- Make it idempotent (unique index on the identity + window start, plus an
  insert-if-absent) and self-healing — let it re-check the last N hours so one
  failed run repairs itself instead of leaving a hole forever.
- **Downsample, never drop.** Skipping "boring" rows removes exactly the low
  samples, so a bucket with one 40% reading and five 0.2% readings charts 40%.
  The filter inflates the number it filters on. If you must store less, fold a
  window into one faithful row.
- **Before dropping or skipping any row, open every query that reads the table**
  and classify each column: `SUM` and `MAX` over zeroes survive; `AVG` does not,
  and neither does "the newest row", which a skip freezes.
- **Sparse series break charts silently.** Many chart libraries render a
  single-point line as nothing, and a `hasData = length > 0` guard calls that
  fine. Check any change that can shorten a series at 0, 1 and 2 points.

## Housekeeping

- **Prune on a schedule**, with a retention per table that matches how it's read
  (raw high-frequency rows for hours, rollups for days, daily summaries for
  years).
- **`VACUUM` is the only thing that returns pruned pages to the filesystem.**
  Deletes free pages inside the file. Run it rarely (weekly, off-peak), skip it
  under a small freelist, and know that it rewrites the database under an
  exclusive lock and needs free disk roughly equal to the database size.
- Back it up with PocketBase's own Settings → Backups (full `pb_data`, download
  and restore, optional S3 target) before building anything custom — it already
  does more than most hand-rolled scripts.
