# Collections, rules and migrations

## Collection types

| Type | What it is |
| --- | --- |
| `base` | ordinary records |
| `auth` | records that can log in — adds `email`, `password`, `verified`, `tokenKey`, and OAuth2/OTP/MFA options |
| `view` | a read-only collection backed by a SQL `SELECT`; no writes, no hooks on write, rules still apply |

`view` collections are the cheapest way to expose a join or an aggregate to
clients without writing a custom route — but their SQL runs on every list
request, so check its plan.

Superusers live in the system `_superusers` collection. They are not "a user with
a role"; they bypass all API rules. Application roles belong in a field on your
own `users` collection.

## Fields

`text`, `editor`, `number`, `bool`, `email`, `url`, `date`, `autodate`, `select`,
`file`, `relation`, `json`, `geoPoint`, plus the auth-only system fields.

Things that bite:

- **`json` is a string in disguise.** It round-trips JSON-encoded through the
  JSVM (see `jsvm-runtime.md`), it can't be indexed usefully, and filtering into
  it is a scan. Use it for opaque payloads, not for anything you query or sort on.
- **`select` values are a fixed list.** Widening it is a migration; code that
  writes a value not in the list fails validation at save time, which in a
  fire-and-forget hook means a record stuck non-terminal.
- **`file` fields** carry `maxSize` per file and `maxSelect` per record. An
  upload limit is never only this number — see [Count every layer](#count-every-layer-before-writing-a-limit-into-ui-copy).
- **`relation`** has `maxSelect` (1 = single, >1 = multi) and `cascadeDelete`.
  A relation's **target collection cannot be edited in place** — changing it
  means dropping the field and adding a new one with a new id, which discards the
  data in it. Plan the target before you ship the field.
- **`autodate`** with `onCreate`/`onUpdate` is the built-in `created`/`updated`.
  Don't hand-roll them.

## API rules

Five per collection: `listRule`, `viewRule`, `createRule`, `updateRule`,
`deleteRule`. Each is a filter expression, or `null`.

**`null` means superuser-only — not "open".** An empty string `""` means open to
everyone. That distinction is the single most common accidental data leak in a
PocketBase deployment; check which one you wrote.

```
@request.auth.id != ""                       // any signed-in user
user = @request.auth.id                      // owner-only
@request.auth.role = "admin"                 // field on the auth record
@collection.org_members.user ?= @request.auth.id   // membership via another collection
@request.body.status = "draft"               // constrain what a write may contain
```

Rules run **before** the corresponding `*Request` hook, so a strict rule plus a
friendly hook error yields the rule's generic message and never the hook's.
Decide the gate deliberately — see the table in `SKILL.md`.

Rules are enforced on the collection API only. Custom routes, hooks and the admin
SDK bypass them entirely.

**`@request` belongs in the rule, never in a client-sent filter.** A regular
(non-superuser) request that itself supplies `filter=... && createdBy =
@request.auth.id` as a query param is rejected outright — `403 Only superusers
can filter by @request.` PocketBase reserves `@request`/`@collection` macros
inside a client-submitted filter string to superuser tokens, even though the
value the macro resolves to (the caller's own auth id) is harmless. This is a
different failure from a rule being too strict — it fires before any rule
evaluation, on the filter string itself. Fix it by resolving the id client-side
and interpolating the literal instead: `createdBy = "${userId}"`. If the goal
was scoping to the caller for security, put `@request.auth.id` in the
collection's own `listRule` instead — the client-sent filter should never need
`@request` at all.

**Live rules drift.** Anything edited in the admin UI on a running host is not in
`pb_migrations/`. Read the rules off the instance before you reason about access,
and when you change one for real, write the migration too.

## Migrations

Files in `pb_migrations/`, named `<unix-timestamp>_<description>.js`, applied in
filename order and recorded in `_migrations`.

```js
/// <reference path="../pb_data/types.d.ts" />

migrate((app) => {
  const collection = new Collection({
    name: "invoices",
    type: "base",
    listRule: "user = @request.auth.id",
    viewRule: "user = @request.auth.id",
    createRule: null,
    updateRule: null,
    deleteRule: null,
    fields: [
      { name: "user", type: "relation", required: true, collectionId: "_pb_users_auth_", maxSelect: 1, cascadeDelete: true },
      { name: "amountCents", type: "number", onlyInt: true, required: true },
      { name: "status", type: "select", maxSelect: 1, required: true, values: ["draft", "sent", "paid"] },
      { name: "created", type: "autodate", onCreate: true },
      { name: "updated", type: "autodate", onCreate: true, onUpdate: true },
    ],
    indexes: ["CREATE INDEX idx_invoices_user_created ON invoices (user, created)"],
  })
  return app.save(collection)
}, (app) => {
  return app.delete(app.findCollectionByNameOrId("invoices"))
})
```

The second function is the **down** migration and must actually reverse the
first. `pocketbase migrate down N` **prompts for confirmation**, so a scripted
call with piped stdin gets "The command has been cancelled" and reverts nothing —
feed it `y\n`. `migrate up` needs no input.

The `/// <reference>` line is not a comment to strip: it gives the editor and the
type checker the PocketBase globals.

**Never modify a migration that has applied anywhere.** Add a new one. The only
exception is a migration that failed to apply *everywhere*, so no host recorded
it — then fixing it in place is the honest repair.

A long migration chain often cannot replay from zero (one old migration
references something a later one deleted). Test new migrations against a fixture,
not a from-scratch replay. See [Testing a migration](#testing-a-migration).

## Changing an existing field

Read it out by name and mutate it:

```js
migrate((app) => {
  const collection = app.findCollectionByNameOrId("invoices")
  collection.fields.getByName("status").values = ["draft", "sent", "paid", "void"]
  return app.save(collection)
}, (app) => {
  const collection = app.findCollectionByNameOrId("invoices")
  collection.fields.getByName("status").values = ["draft", "sent", "paid"]
  return app.save(collection)
})
```

**Do not rebuild the field and `fields.add()` it.** `add()` matches by field
**id**, not name. Against a field whose id has drifted from what your migration
assumes — which happens the moment anyone edits the collection in the admin UI —
it appends a *second* field with the same name and the save fails with
`Duplicated or invalid field name <name>`. Admin-UI-generated migrations use the
rebuild form because they authored the id they match; a hand-written migration
has not.

Widening a `select` becomes two-step when you also need to *narrow* it later: the
down migration must first rewrite rows holding the removed value, or the save
fails validation.

## Indexes

Declared as raw `CREATE INDEX` statements on the collection.

```js
collection.indexes = [
  "CREATE UNIQUE INDEX idx_events_dedupe ON events (dedupe_key)",
  "CREATE INDEX idx_enforcements_open ON enforcements (user, kind) WHERE stage != 'resolved'",
]
```

**A `WHERE` clause must not contain parentheses.** PocketBase's index parser
greedily binds the column group to the *last* `)` in the expression:

- `WHERE stage IN ('a','b')` → `Invalid CREATE INDEX expression`, loudly.
- `WHERE (a = 1 AND b = 2)` → silently mangled: the parens fold into the column
  list and you get an index over the wrong columns.

Encode set membership without parens, or drop the partial index and filter in the
query.

A **unique index is the best idempotency primitive PocketBase gives you.** A cron
that may run twice, an at-least-once webhook, a retried batch — give the row a
deterministic `dedupe_key` and let the index arbitrate, rather than checking
existence first (which races).

## Count every layer before writing a limit into UI copy

A file upload passes through your CDN's body limit, your reverse proxy's
`request_body max_size`, and the field's `maxSize`. The smallest silently wins,
and the user sees a failure from a layer that never reaches your error handling.
Grep every cap on the path and reconcile them in one change.

## Testing a migration

Against the real binary, on a scratch directory — fast enough to keep as a
permanent test:

```bash
TMP=$(mktemp -d)
mkdir -p "$TMP/pb_migrations"
cp pb_migrations/<baseline collections>.js "$TMP/pb_migrations/"   # real field ids
cp pb_migrations/<new migration>.js        "$TMP/pb_migrations/"

./pocketbase migrate up   --dir "$TMP/pb_data" --migrationsDir "$TMP/pb_migrations"
./pocketbase migrate up   --dir "$TMP/pb_data" --migrationsDir "$TMP/pb_migrations"   # idempotent?
printf 'y\n' | ./pocketbase migrate down 1 --dir "$TMP/pb_data" --migrationsDir "$TMP/pb_migrations"
./pocketbase migrate up   --dir "$TMP/pb_data" --migrationsDir "$TMP/pb_migrations"
```

Then assert on what the database actually holds, not on what the migration
intended:

```sql
SELECT fields FROM _collections WHERE name = 'invoices';
SELECT sql    FROM sqlite_master WHERE type = 'index' AND tbl_name = 'invoices';
```

The schema lives in `_collections.fields` as a **JSON string** — not a JSON
column, and not under `options` in current versions. Run `pragma_table_info` on
`_collections` before writing an assertion, because that layout has moved
between versions.

The baseline fixture matters: copy the collections with the **real field ids**
from production migrations. A fixture built from freshly-generated ids cannot
reproduce the `fields.add()` duplicate-name failure, which is the one this test
exists to catch.
