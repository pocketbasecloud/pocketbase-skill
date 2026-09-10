---
name: pocketbase-backend
description: Use when building, changing, or debugging a PocketBase backend — anything in pb_hooks/ or pb_migrations/, JSVM hook files (*.pb.js), collection schemas, API rules, custom routes, cron jobs, migrations, or the SQLite queries behind them. Also use when a hook "does nothing", a create fails with a message you didn't write, a JSON or date field reads back wrong, a migration won't apply, or the database feels locked under load. Invoke it even for a one-line hook edit: PocketBase's JS runtime is goja, not Node, and most of its failure modes are silent — the code runs, returns 200, and writes the wrong thing.
---

# Building backends on PocketBase

PocketBase is one Go binary: SQLite + auth + REST + realtime + admin UI, extended
in JavaScript from `pb_hooks/`. That JS runs in **goja** — not Node, not V8.
Every hard bug comes from assuming otherwise, or from forgetting that one SQLite
file with one write lock serves every request.

**Versions.** Written against 0.39.x (this repo's pin), matching
`pb_data/types.d.ts`. Older 0.2x releases differ materially — pin the binary
before anything else.

## Orient before you edit

```bash
./pocketbase --version                    # pin assumptions to THIS version
pbc which                                  # which binary, and the pinned version
ls pb_hooks/ pb_migrations/ | tail -20     # what exists, migration timestamps
grep -rn "routerAdd\|onRecord\|cronAdd" pb_hooks/ | head -40
pbc use http://127.0.0.1:8090 && pbc login # point the CLI at the host you'll change
```

Establish three things the code won't tell you:

1. **Exact version.** JSVM APIs changed materially across 0.22 → 0.23 → 0.27
   (rules evaluate *before* the `*Request` hooks). What you remember is a
   hypothesis.
2. **Live API rules.** They drift from `pb_migrations/`; a rule edited in the
   admin UI on the running host is truth. Read them with
   `pbc rules get <collection>` before reasoning about access.
3. **Where logic lives.** If hooks are a thin auth-and-forward layer here,
   business logic in a hook lands in the one place with no test harness.

## Execution model

**Each handler runs as its own isolated program.** A function declared at file
scope in `main.pb.js` is invisible inside a handler (`ReferenceError`, stack
names a virtual `pb.js`). Shared code lives in a module, `require`d **inside**
the handler. A `.pb.js` file holds registration calls and nothing else.

Three silent-failure rules:
- **`e.next()` or the request dies.** Work before it runs pre-validation, after
  it runs post-write; omit it and the request stops — the usual "my hook did
  everything and swallowed the request".
- **The parsed body is only `e.requestInfo().body`.** `e.request.body` is an
  `io.ReadCloser`; a `typeof x === "object"` guard proves nothing.
- **Dates and JSON fields.** goja parses only the `T` date separator, and a
  `json` field round-trips JSON-encoded (`record.get("meta").value()` returns a
  quoted string). Store timestamps as **epoch milliseconds**; never parse a date
  string out of a JSON field.

Full binding surface — `$app`, `$http`, transactions, raw queries, worked
fixes: `references/jsvm-runtime.md`.

## Which layer is the gate

| | API rule | Hook |
| --- | --- | --- |
| Runs | before the `*Request` hook | after the rule passes |
| Survives a broken hooks file | yes | no |
| Explains itself | no — generic `Failed to create record.` | yes — throw `ForbiddenError("...")` |

Security belongs in the rule, the explanation in the hook. The hook's friendly
error is unreachable when the rule is the real gate, so the readable message has
to come from the client first. A deliberate pattern: coarse rule
(`plan = "pro"`) plus precise hook refusal (`role = "system_admin"`). To tell
which spoke: loosen the rule temporarily and re-send.

## Fire-and-forget: the record is the only channel

Slow work returns immediately; nothing hears what it returns. Every failure path
must write a terminal state onto the record (`status: "error"` + sub-status +
composed message), or clients polling it wait forever. A raw sub-status is not a
message — map the vocabulary once per runtime, return "unknown" for codes you
don't recognise.

## Schema and migrations

Migrations are append-only — **never edit one that has applied anywhere** (a fix
applied by hand in prod still needs a fresh no-op migration for new hosts and
staging). Change a field by reading it out by name and mutating it; never
rebuild + `fields.add()`, which matches **by id** and silently appends a
duplicate when ids have drifted. A partial-index `WHERE` must not contain
parentheses — `IN ('a','b')` fails loudly, balanced parens fold silently into
the column list. Full field-type and rule reference:
`references/schema-migrations.md`.

## One database, one write lock

- **Cron writing a row per entity is a lock hog** — batch it (`POST /api/batch`,
  enable it, note `maxRequests`); one bad row rolls the chunk back, which is why
  row-by-row retry is safe.
- **Never retry a write whose response never arrived** — a transport failure
  proves nothing, the write may have committed.
- **Index every column a cron or hook filters on**; scope subqueries by the
  caller (check `EXPLAIN QUERY PLAN` for `SCAN`).
- **A hook save fires its after-hooks** (possibly blocking HTTP) — compare
  before saving.
- **Store counters, never averages or percentiles**; downsample, never drop.

Batch mechanics, indexing, rollup schema design:
`references/data-and-scale.md`.

## Verify against the real binary

No unit harness exists for hooks — always drive a real binary; V8 accepting
your code proves nothing about production.

- **Install pinned**: `pbc install <version>` (or `pbc init <version>`). **Pass
  the version explicitly** — the `pocketbaseVersion` pin in `pbc.json` is not
  read by install.
- **Probe scripts**: scratch `pocketbase serve` + SDK calls answer "what does
  *this version* return". Do it before branching on the answer. **Default dirs
  resolve against the binary's directory (from `os.Args[0]`), not cwd** —
  `--dir` relocates only data, so a scratch-db probe still runs the binary's own
  hooks and migrations; a binary parked under the OS temp dir is assumed to be
  `go run` and silently flips to cwd-relative. Pass
  `--hooksDir`/`--migrationsDir`/`--publicDir` explicitly when the probe must
  exercise a specific project's hooks.
- **Drive with `pbc`** (`use` + `login`, then `records ls`, `rules get`,
  `settings get`, `cron run`, `collections export`) instead of throwaway curl.
- **Two logs**: `pbc logs -f` follows the app log; `pbc cloud logs pb --name <n>
  -f` tails the container, where a JSVM load failure shows up and the app log
  doesn't. A hook that "does nothing" is a log question first.
- **Extract aggregation SQL and run it standalone** on deliberately uneven
  fixtures; **test migrations** on a scratch `pb_data` asserting up/down/
  idempotence (`pocketbase migrate down N` prompts — feed it `y\n`).
- Claim a metric is broken only after **one real event through the pipe** —
  "everything is zero" is usually one misconfigured producer.

Report what you ran and what it printed; "tests written" is not "tests passed".
Runnable patterns and error-response reading:
`references/verification.md`.

## Working checklist

1. Establish version, live rules, where logic lives.
2. Decide the gate — rule, hook, or both — and where the readable message comes from.
3. Write the hook, everything `require`d inside the handler; no file scope.
4. Schema change? New migration, mutate fields by name, no parens in a partial index.
5. Check the write path against the lock: batched, indexed, not per-entity.
6. Verify against the real binary, drive it with `pbc`, read the logs, show the output.
7. Every failure path writes a terminal, explainable state.

## Relationship to the repo docs

Cross-cutting constraints — batch API, append-only migrations,
counters-not-averages, ENV fail-loud, composed messages — are authoritative in
root `CLAUDE.md` / `backend/CLAUDE.md` (tier 1/2). This skill is the deep-dive
home for their JSVM/SQLite mechanics. One home per fact; when one changes, keep
the others in sync (`docs/documentation-guide.md`).

## Reference index

| Read when | File |
| --- | --- |
| Writing any hook, route, or cron | `references/jsvm-runtime.md` |
| Touching collections, fields, rules, migrations | `references/schema-migrations.md` |
| Adding a query, cron, rollup, or read route | `references/data-and-scale.md` |
| Testing, probing, or debugging an error | `references/verification.md` |
