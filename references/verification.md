# Verifying and debugging

Hooks run inside the JSVM, so there is no unit-test harness for them. That is not
a licence to ship unverified — it means the verification has to be built out of
the real binary, extracted logic, and probes.

## Probe the real binary

When you need to know what *this version* does — a status code, an error shape, a
default — do not reason from source you half-remember. Spend a minute:

```bash
TMP=$(mktemp -d)
./pocketbase serve --dir "$TMP/pb_data" --http 127.0.0.1:8099 &
PB=$!
./pocketbase superuser upsert admin@example.com "probe-password" --dir "$TMP/pb_data"
curl -s -X POST 127.0.0.1:8099/api/collections/_superusers/auth-with-password \
  -H 'content-type: application/json' \
  -d '{"identity":"admin@example.com","password":"probe-password"}'
# ... the thing you actually want to know ...
kill $PB
```

Do this **before** writing the code that branches on the answer, not after
shipping it. Version differences that have burned people: a disabled batch
endpoint answers 403 not 400; oversized batches report the server ceiling in
`params.max`; API rules are evaluated ahead of the `*Request` hooks.

**The defaults do not follow cwd — they follow the binary.** `--dir` above
relocates only the data dir; `hooksDir`/`migrationsDir`/`publicDir` still
resolve against the directory of `os.Args[0]`, not the working directory. A
probe started with a project's binary loads that project's hooks and serves its
`pb_public` no matter where it runs — and a binary whose path starts with the OS
temp dir (or the Go cache) is assumed to be `go run` and flips to cwd-relative
instead. Fine for an API-shape probe; to exercise a specific project's hooks and
migrations against a scratch data dir, pass every dir explicitly:

```bash
./pocketbase serve \
  --dir "$TMP/pb_data" \
  --hooksDir /path/to/project/pb_hooks \
  --migrationsDir /path/to/project/pb_migrations \
  --publicDir /path/to/project/pb_public \
  --http 127.0.0.1:8099 &
```

## Drive the instance with `pbc`

`pbc` talks to any PocketBase instance, not only a cloud one. Use it instead of
hand-rolled curl: it authenticates once, speaks the same commands against a local
`./pocketbase serve` and a deployed instance, and `--json` makes every output
machine-readable.

```bash
pbc install 0.39.10          # pass the version explicitly — the pbc.json
                             # pocketbaseVersion pin is NOT read by install
./pocketbase serve &         # ...or point at a deployed URL instead

pbc use http://127.0.0.1:8090 --name local
pbc login --email admin@example.com --password "probe-password"
pbc whoami
```

| Question | Command |
| --- | --- |
| What did the hook actually write? | `pbc records ls posts --filter 'status="error"' --json` |
| What are the **live** API rules? | `pbc rules get posts` |
| Change a rule to find out which layer refused | `pbc rules set posts --list-rule ''` |
| What's in the schema right now? | `pbc collections get posts` / `pbc collections export --out schema.json` |
| Does the cron work? | `pbc cron ls`, then `pbc cron run <jobId>` |
| What are the instance settings? | `pbc settings get`, `pbc settings mail`, `pbc settings s3` |
| Change settings / test them | `pbc settings mail set ...`, `pbc settings mail test`, `pbc settings s3 test` |
| Snapshot before a risky migration | `pbc settings backup create` |

Three habits that make this pay off:

- **`collections export` before a migration, `rules get` after it.** The export
  is the diffable record of what the schema was; the rules read back is the only
  proof the migration's rules actually landed on the host (they drift).
- **`cron run <jobId>` instead of waiting.** A cron that fails inside the JSVM
  often logs nothing on schedule; running it on demand puts the failure in front
  of you immediately. `cron ls` also lists PocketBase's own jobs, which is a
  quick check that the JSVM loaded your `cronAdd` at all — a hook that failed to
  load has no job here.
- **Know the two empties when setting a rule.** `--list-rule ''` writes `""`,
  which is *public*; `--list-rule 'null'` writes `null`, which is
  *superuser-only*. They are opposites, and passing the wrong one while
  "loosening a rule to see which layer refused" locks the collection down
  instead and confirms nothing.

Against a deployed resource the cloud subcommands cover the same ground from the
account side — `pbc cloud pb info`, `pbc cloud pb hooks ls`/`push`,
`pbc cloud env ls`/`set`, `pbc cloud data export`.

## Read the logs

**Debug by reading logs before reading code**, and know that there are two
different logs:

```bash
pbc logs -f                                        # app log (_logs) of the selected instance
pbc logs --filter 'data.status>=400' --per-page 50 # PocketBase filter syntax
pbc cloud logs pb --name my-instance -f            # deployed: tails the container
pbc cloud logs backend --name my-api --lines 200
```

- **`pbc logs`** reads PocketBase's **app log** — the `_logs` table in the aux
  database (`pb_data/auxiliary.db`). It shows what the HTTP layer saw: status,
  route, timing. It does **not** show a hook that failed to load, because that
  failure happens before any request.
  **The `_logs` columns are `id`, `level`, `message`, `data`, `created`** — the
  request context lives under the JSON `data` column (a request entry carries
  `data.status`, `data.type = "request"`, …), so the filter is
  `data.status>=400`, not `status>=400`. The wrong one is not an empty result,
  it is `Instance error (400): Something went wrong while processing your
  request.` There is no separate `_requests` table in 0.39 — it was folded into
  `_logs` — so the top-level `status`/`url`/`method` columns older docs mention
  are inside `data` now.
- **`pbc cloud logs`** tails the container's stdout on a deployed instance. This
  is where a JSVM load error, a `console.log` from a hook, and a cron's output
  appear. Without `--follow` it prints the last `--lines` (50 default, 1000 max)
  and exits; with `--follow` it streams until interrupted. Locally this stream is
  just your `./pocketbase serve` stdout — redirect it to a file rather than
  losing it in a terminal you will close.

So the sequence for "the hook does nothing": `pbc cloud logs` (did it even load?)
→ `pbc logs --filter 'data.status>=400'` (did the request arrive, and what did it
get?) → `pbc records ls` (what did it write?). Only then open the file.

## Extract the SQL and test it

Aggregation queries are where a mistake is both easy and invisible — the numbers
come out plausible. Pull the query out of the hook and test it standalone. Node
22+ has `node:sqlite` built in, so this needs no dependencies:

```js
import { DatabaseSync } from "node:sqlite"
import { readFileSync } from "node:fs"

const HOOKS = readFileSync("pb_hooks/main.pb.js", "utf8")

function extract(startMarker, endMarker) {
  const start = HOOKS.indexOf(startMarker)
  if (start < 0) throw new Error(`marker not found: ${startMarker}`)
  const end = HOOKS.indexOf(endMarker, start)
  if (end < 0) throw new Error(`end marker not found: ${endMarker}`)
  return HOOKS.slice(start, end)
}

const toNodeBinds = (sql) => sql.replace(/\{:(\w+)\}/g, "$$$1")

const rollupSql = toNodeBinds(extract("INSERT INTO metrics_hourly (", "`)\n      .bind({"))

const db = new DatabaseSync(":memory:")
db.exec("CREATE TABLE readings (bucket TEXT, pct REAL)")
const insert = db.prepare("INSERT INTO readings VALUES (?, ?)")
for (const [b, p] of [["h1", 40], ["h1", 0.2], ["h1", 0.2], ["h1", 0.2], ["h1", 0.2], ["h1", 0.2], ["h2", 5]])
  insert.run(b, p)

const rawSql = `SELECT bucket, AVG(pct) AS avg_pct FROM readings GROUP BY bucket`
const viaRollup = db.prepare(rollupSql).all()
const viaRaw = db.prepare(rawSql).all()
```

Two translations are needed and both are easy to forget: PocketBase binds
parameters as `{:name}` while `node:sqlite` wants `$name`, and any
`${templateExpression}` interpolated into the query has to be substituted with
the value the hook would have used.

Two things make this test earn its keep:

1. **Read the query out of the real file by marker**, so the test breaks when the
   query moves instead of silently testing a stale copy. Anchor the markers on
   distinctive SQL text (`"INSERT INTO metrics_hourly ("`), not on a comment —
   the marker has to be something nobody deletes while tidying up.
2. **Make the fixture uneven.** A uniform fixture makes a wrong re-averaging look
   right. Uneven data makes it fail loudly.

The assertion that catches the most is: *the rollup query and the raw query
return the same numbers*, for every bucket width the UI offers.

## Never test goja code under Node

V8 accepting a date format, a getter, or an object shape proves nothing about
production. Anything that touches `$app`, a record, a `json` field, or a date must
be exercised in the JSVM — via a probe instance, or via a route you hit with curl.
Pure functions (a formatter, a classifier, a window calculator) can be tested
outside, and are worth factoring out precisely so they can be.

If the test runner genuinely cannot run in your environment, verify what you can,
and **say what did and did not run**. "Tests written" is not "tests passed".

## Reading PocketBase errors

**The reason is almost never at the top level.** PocketBase returns a fixed
top-level message and puts the cause in a per-field map:

```json
{ "status": 400, "message": "Failed to create record.",
  "data": { "email": { "code": "validation_invalid_email", "message": "Must be a valid email address." } } }
```

So: read `data`, not `message`. When the request passed through your own service
layers, expect the cause nested two or three deep and re-wrapped at each hop.
Learn your stack's wrapper shapes and treat them as values to skip — otherwise
you read a `{data, message, status}` envelope *as* a field map and report three
fields called "data", "message" and "status".

**A rejection that never reached your code can only be identified by its status.**
A CDN or reverse proxy answering an oversized upload returns an HTML page; the
SDK can't parse it, so `data` is empty and `message` is contentless. Branch on
the status code (413, 502, 504) **before** branching on message text.

**Discriminate on structured details, not sentences.** Matching
`applied: [] && skipped: [...]` survives rewording; matching the sentence does not.

## Writing errors a user can act on

- **Compose the message; never echo an internal one.** Raw errors from an internal
  service leak host paths, ids and implementation.
- **A sub-status is not a message.** `agentUnreachable` is an identifier. Map the
  vocabulary to sentences once, and return "unknown" for anything unrecognised
  rather than inventing a cause.
- **Ask what the message tells the user to do next**, not just whether it's true.
  A truthful "upload failed, try again" is a defect when retrying is the one thing
  that cannot work.
- **Every failure path in a fire-and-forget flow must write a terminal state.**
  If the hook returns immediately, the record is the only channel; a path that
  throws without writing `status: "error"` leaves clients polling forever.
- If a check is relaxed to allow a "nothing to do" case, **say so out loud**. A run
  that packaged nothing printing the same success as a run that shipped everything
  is a bug moved, not fixed.

## Debugging discipline

- **Send one real event through the pipe before believing a zero.** "Everything is
  zero" is usually one misconfigured producer, not a system to audit.
- **One transient probe is not evidence.** Re-probe, and compare against a control
  before blaming infrastructure.
- **Check a failing test against the base branch before owning it.** Pre-existing
  failures absorbed into your diff make the change unreviewable.
- **A pipe or a flag changes what you see.** `cmd | tail` reports *tail's* exit
  status and hides the first line. Redirect to a file and read `$?` when the exit
  code or a header row matters.
- **Compare the launch timeline against the symptom timeline.** A feature that
  "regressed" often never worked, and the deploy history says so in one grep.
- **Logging is not free and silence is not proof.** A handler that fails inside a
  cron may produce no output at all if the failure is in the JSVM's own load path.
  Check the logs table and the process output; they are not the same place.
