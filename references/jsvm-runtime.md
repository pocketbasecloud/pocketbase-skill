# The JSVM runtime

Everything in `pb_hooks/` runs in **goja**: an ES5.1+ interpreter embedded in the
Go binary, with PocketBase's Go API bound into globals. There is no event loop,
no `npm`, no `fetch`, no `setTimeout`, no Node standard library. `require()`
works only for local files (and a small set of shims).

## Isolation

PocketBase serializes each registered handler and executes it as its own goja
program. Nothing at file scope is in scope inside a handler — a top-level
function or const is a latent `ReferenceError`. A `.pb.js` file holds
registration calls and nothing else; shared code lives in a module, `require`d
inside each handler:

```js
// pb_hooks/helpers.js
module.exports = {
  audit(msg) { $app.logger().info(msg) },
}

// pb_hooks/main.pb.js
onRecordAfterCreateSuccess((e) => {
  const helpers = require(`${__hooks}/helpers.js`)
  helpers.audit("created")
  e.next()
}, "posts")
```

`__hooks` is the absolute path to the hooks directory. The module body runs once
per process and later `require`s return the cached exports — so a module-level
`let cached` persists and **is shared across handlers**, a convenient cache and
a silent coupling at once. Keep module top-level work cheap: it runs once, at
first require.

## Hook catalog

The `*Request` family fires only for API requests; the `*Model`/`*Record` family
fires for every write including your own `$app.save()` calls from a cron.

| Hook | When | Use it for |
| --- | --- | --- |
| `onRecordCreateRequest(fn, ...collections)` | API create, **after** the create rule passes | validation with a real message, defaulting, quota checks |
| `onRecordUpdateRequest` | API update, after the rule | guards that depend on the *stored* record |
| `onRecordDeleteRequest` | API delete, after the rule | referential guards |
| `onRecordAfterCreateSuccess` / `...UpdateSuccess` / `...DeleteSuccess` | after the transaction commits | side effects: mail, provisioning, counters |
| `onRecordCreate` / `onRecordUpdate` / `onRecordDelete` | every write, API or programmatic | invariants that must hold no matter the caller |
| `onRecordEnrich` | while serializing a record for a response | hiding or adding computed fields per viewer |
| `onRecordAuthRequest` / `onRecordAuthRefreshRequest` | successful auth / token refresh | login side effects, extra claims, re-check active status |
| `onRecordAuthWithPasswordRequest` | password sign-in, after the rule | login blocks, lockouts, MFA gating |
| `onRecordsListRequest` / `onRecordViewRequest` | list / view API requests | field masking and extra authorization collection rules can't express |
| `onBootstrap` | startup, after the app is initialized | one-time setup; **call `e.next()` first**, the app isn't ready before it |
| `onMailerSend`, `onRealtimeConnectRequest`, `onFileDownloadRequest` | as named | mail interception, connection policy, download policy |

Tags (the trailing collection names) match against collection id **or** name;
without them the hook fires for every collection.

**Read the stored record, not `e.record`, for any guard about current state.** By
the time an update hook runs, incoming values are already merged into `e.record`
— so a request that sets `status = "active"` looks un-locked, and anyone can lift
their own lock with a field write. Superuser and service contexts usually need a
bypass in these guards; check `e.auth` and let the superuser through explicitly.

## Custom routes

```js
routerAdd("POST", "/api/reports/{id}/publish", (e) => {
  const helpers = require(`${__hooks}/helpers.js`)
  const body = e.requestInfo().body            // the ONLY parsed body
  const id = e.request.pathValue("id")

  if (!e.auth) throw new UnauthorizedError("Sign in to publish.")
  const report = $app.findRecordById("reports", id)
  if (report.get("owner") !== e.auth.id) {
    throw new ForbiddenError("You can only publish your own reports.")
  }
  report.set("published", true)
  $app.save(report)
  return e.json(200, { published: true })
}, $apis.requireAuth())
```

- Middlewares are extra arguments: `$apis.requireAuth()`,
  `$apis.requireAuth("users")` to restrict the collection,
  `$apis.requireSuperuserAuth()`, `$apis.bodyLimit(bytes)`.
- Path params: `{id}` in the pattern, `e.request.pathValue("id")` to read.
- Query: `e.requestInfo().query`, headers: `e.requestInfo().headers` (lowercased).
- Respond with `e.json(status, obj)`, `e.string`, `e.html`, `e.redirect`,
  `e.noContent`. Return the result.
- Custom routes are **not** covered by collection API rules. Whatever the rule
  would have enforced, you enforce by hand — the most common authorization hole.

## Errors you can throw

`BadRequestError`, `UnauthorizedError`, `ForbiddenError`, `NotFoundError`, and
the generic `ApiError(status, message, data)`. Each takes a message and an
optional data map:

```js
throw new BadRequestError("Quota exceeded.", { plan: "free", limit: 5 })
```

Anything else that throws becomes a 500 with an opaque body, so wrap external
calls and convert their failures into an error whose message tells the user what
to do next.

## Records

```js
const collection = $app.findCollectionByNameOrId("posts")
const record = new Record(collection)
record.set("title", "Hello")
record.set("author", e.auth.id)
$app.save(record)

$app.findRecordById("posts", id)
$app.findFirstRecordByFilter("posts", "slug = {:slug}", { slug: input })
$app.findRecordsByFilter("posts", "author = {:a} && published = true", "-created", 20, 0, { a: e.auth.id })
$app.expandRecord(record, ["author"], null)
$app.delete(record)
```

**Always bind user input with `{:param}` + a params object.** String-concatenating
into a filter is injectable.

Typed reads: `record.getString(f)`, `getBool`, `getInt`, `getFloat`,
`getDateTime`, `getStringSlice`. For a `json` field, `record.get(f)` returns a
Go value whose `.value()` is the **JSON encoding** — a quoted string for a
string, so `JSON.parse` it.

Hidden fields (`hidden: true`, e.g. secrets) are stripped from responses and from
user-token writes; only the admin/superuser context can set them.

## Transactions

```js
$app.runInTransaction((txApp) => {
  const a = txApp.findRecordById("accounts", from)
  a.set("balance", a.getInt("balance") - amount)
  txApp.save(a)
  const b = txApp.findRecordById("accounts", to)
  b.set("balance", b.getInt("balance") + amount)
  txApp.save(b)
})
```

Use `txApp` inside — using the outer `$app` escapes the transaction. Throwing
rolls back. Keep transactions short: they hold the write lock.

## Bound globals

| Global | Notes |
| --- | --- |
| `$app` | records, collections, `db()`, `auxDB()`, `logger()`, `settings()`, `runInTransaction()` |
| `$http.send({url, method, body, headers, timeout})` | **blocking** HTTP. Returns `{statusCode, body, raw, headers}`. No async — a slow call in a request hook is a slow request |
| `$os.getenv(k)`, `$os.exec` | environment and processes |
| `$security` | `randomString`, `md5`, `sha256`, `hs256`, `createJWT`, `parseUnverifiedJWT` |
| `$filesystem.fileFromPath / fileFromBytes / fileFromURL` | build file field values |
| `$mails.sendRecordVerification` etc. / `new MailerMessage({...})` + `$app.newMailClient().send(m)` | mail |
| `$apis` | route middlewares and API error helpers |
| `arrayOf`, `DynamicModel`, `Record`, `Collection` | constructors for query targets and models |
| `$dbx` | expression helpers for the query builder |

Raw SQL, when the record API isn't enough:

```js
const rows = arrayOf(new DynamicModel({ day: "", total: 0 }))
$app.db()
  .newQuery("SELECT date(created) AS day, COUNT(*) AS total FROM posts WHERE author = {:a} GROUP BY day")
  .bind({ a: e.auth.id })
  .all(rows)
```

`DynamicModel` needs a zero value per column of the right *type* — an `int` field
declared as `""` comes back wrong.

## Cron

```js
cronAdd("pruneMetrics", "0 2 * * *", () => {
  const helpers = require(`${__hooks}/helpers.js`)
  helpers.prune()
})
```

Same isolation rule: `require` inside. Crons run on the instance's clock in UTC
and are not distributed — two instances of the same database run the job twice.
Make every job idempotent (a unique index plus an insert-if-absent is the usual
shape) and advance any watermark **only after** the work succeeded, so a failure
retries rather than skips.

## Boundary gotchas

**`typeof` proves nothing about a Go value.** goja reports Go structs, readers
and slices all as `"object"`. A check like `typeof body === "object"` can match
an `io.ReadCloser` and hand you a stream whose every field reads `undefined`.
Check for a field you actually need, or parse and validate.

**`e.request.body` is not the body.** `e.request` is Go's `*http.Request`;
`e.request.body` is a reader you get exactly one pass over — reserve it for
cases that need the raw bytes as sent (webhook signature verification).
Everything else uses `e.requestInfo().body`.

**Dates.** goja's `Date` parser accepts the `T` separator only; PocketBase's own
datetime format (`2026-08-17 04:16:00.000Z`, space-separated) is *not* reliably
parseable by `new Date()`. Use `record.getDateTime(f)` for datetime fields, and
for anything you store yourself, store **epoch milliseconds** and compare
integers.

**JSON fields round-trip encoded.** `record.get("meta").value()` returns the JSON
text, so a stored timestamp inside one has quotes around it and then hits the
date trap above. When reading a value that might be legacy, accept only the shape
you now write (a number, or a digit string) and treat anything else as absent so
one fallback run heals the row.

**A fallback that fires on every request is invisible.** When a field has a
default, something must distinguish absent from defaulted, or a parse that never
works looks like a feature nobody uses. Log the fallback, or count it.

**No async.** There are no promises worth using and no timers. "Fire and forget"
means handing the work to something outside the JSVM (an HTTP call to a service,
a row another process picks up), not scheduling it here.

## Configuration

Read settings from a file with an environment-variable fallback, and **fail
loudly on an unrecognised environment** rather than defaulting — defaulting is
how a staging host starts sending real email:

```js
const ENVIRONMENTS = ["DEV", "TEST", "QA", "PROD"]

function getConfig() {
  let file = {}
  try { file = require(`${__hooks}/config.json`) || {} } catch (_) {}
  const env = file.ENV || $os.getenv("ENV")
  if (ENVIRONMENTS.indexOf(env) === -1) {
    throw new Error(`ENV must be one of ${ENVIRONMENTS.join(", ")} (received "${env}")`)
  }
  return { ENV: env, /* ... */ }
}
```

Precedence file-then-environment, and treat an **empty** file value as absent so
a half-filled config falls through instead of silently configuring nothing.
