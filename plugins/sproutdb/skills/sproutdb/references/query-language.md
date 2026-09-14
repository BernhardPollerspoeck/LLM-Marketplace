# SproutDB Query Language Reference

## Literals

| Type | Syntax | Example |
|------|--------|---------|
| String | Single quotes | `'hello'`, `'it\'s'`, `'C:\\data\\'` |
| Integer | Digits | `42`, `-7` |
| Float | Digits with dot | `3.14`, `-0.5` |
| Boolean | Keyword | `true`, `false` |
| Null | Keyword | `null` — in WHERE (`is null`) and as an upsert value (`{payload: null}` clears the column) |
| Date | String format | `'2025-01-15'` |
| DateTime | String format | `'2025-01-15 14:30:00.0000'` |
| Duration | Number + unit | `7d`, `24h`, `30m`, `45s` |

### String escapes

Exactly two escapes exist inside string literals: `\'` → `'` and `\\` → `\`.
Any other backslash is kept literally (`'C:\temp'` is `C:\temp`).

```
upsert files {path: 'it\'s'}        ## it's
upsert files {path: 'C:\\data\\'}   ## C:\data\   ← a trailing backslash MUST be \\
upsert files {path: 'C:\temp'}      ## C:\temp    (lone backslash stays)
```

A value ending in `\` written as `'a\'` is a syntax error — the backslash escapes the
closing quote. **In code, don't build literals yourself — pass values as `@name`
parameters** (see below). If you must: double backslashes first, then escape quotes:
`value.Replace("\\", "\\\\").Replace("'", "\\'")`.

### Parameters (`@name`)

```
get users where email = @email
upsert gs {k: @k, etag: @next} on k when etag = @expected
get users where name in @names          ## a list parameter renders as [..]
```

Placeholders are bound from C# (`db.Query(q, new { email })`) or from the HTTP JSON body
`{"query": "...", "parameters": {...}}`. Each value becomes exactly one literal — never query
syntax. Only at value positions (not table/column names). Missing/unused parameter →
`PARAMETER_ERROR`. `@` inside a string literal is not a placeholder. Details:
csharp-integration.md.

## Comments

```
## This is a comment ##
get users ## inline comment ## where active = true
## Comment to end of line (closing ## optional)
```

---

## Multi-Query & Transactions

Multiple queries are separated by **semicolons**. Each query produces its own response,
returned positionally (`Query()` returns `List<SproutResponse>`).

```
## Batch: 3 queries -> 3 responses
get users; get orders; describe users
```

### Transactions (`atomic` ... `commit`)

Wrap mutations in `atomic; ... ; commit` for all-or-nothing execution with real
rollback (MMF-level). On any error inside the block, all changes are reverted.

```
atomic;
upsert accounts {_id: 1, balance: 50};
upsert accounts {_id: 2, balance: 150};
commit
```

Rules:
- `atomic` and `commit` each stand alone in their own segment (own semicolon).
- **Only `upsert`, `delete`, `get` and `describe` are allowed inside `atomic`.** Schema
  changes (`create`, `add column`, `purge`, …), backup/restore and auth commands are a
  `SYNTAX_ERROR` — a rollback cannot undo them. Create tables/indexes before the
  transaction (for table + unique index in one step use the `unique` column modifier).
- Any error (incl. `CONDITION_FAILED` / `EXPECTATION_FAILED`) rolls everything back; the
  result is then a single error response `transaction rolled back: …`.
- `get` / `describe` are allowed inside a transaction and see uncommitted writes
  (read-your-own-writes).
- The result list ends with a **transaction marker** response
  (`Operation = transaction`, `Affected` = total rows changed).
- Nested `atomic` blocks are not allowed.

---

## CREATE DATABASE

```
create database
create database with chunk_size N
```

Examples:
```
create database
create database with chunk_size 500
```

- `chunk_size` controls slot pre-allocation (100–1,000,000)

---

## CREATE TABLE

```
create table NAME
create table NAME (col1 type [size] [strict] [default val] [unique], ...)
create table NAME (...) ttl DURATION
create table NAME (...) ttl DURATION with chunk_size N
create table NAME (...) with chunk_size N
```

Examples:
```
create table users (name string, email string 320 strict unique, age ubyte, active bool default true)
create table sessions (token string 64 strict) ttl 24h
create table products (name string 200, price double default 0)
```

- No parentheses around type size: `string 320` NOT `string(320)`
- `default VALUE` makes column non-nullable
- `strict` prevents type widening on the column
- `unique` creates the unique index **in the same statement** as the table (no window where
  the table exists without it). Same semantics as `create index unique` (NULLs allowed).
  Not for `blob`/`array`. Prefer it over a separate `create index unique` for new tables.
- Modifiers `strict` / `default` / `unique` may appear in any order
- `chunk_size` controls slot pre-allocation (100–1,000,000)
- Order: Columns → TTL → `with chunk_size`

---

## UPSERT (Insert & Update)

```
## Insert (new _id auto-generated)
upsert users {name: 'John', email: 'john@test.com', age: 25}

## Bulk Insert
upsert users [{name: 'John', age: 25}, {name: 'Jane', age: 30}]

## Update by _id (implicit on _id)
upsert users {_id: 1, name: 'John Updated'}

## Upsert by column (insert if not found, update if found)
upsert users {email: 'john@test.com', name: 'John'} on email

## Row-level TTL
upsert sessions {token: 'abc123', ttl: 24h}
```

Rules:
- No `_id` and no `on` → Insert (auto-generated ID)
- `_id` in body → implicit `on _id`, updates if exists
- `on COLUMN` → lookup by column value: update if found, insert if not. **With an index on
  COLUMN (e.g. `unique`) the lookup uses the B-Tree (O(log n)); without one it scans all rows
  (O(n)).** Always index upsert key columns.
- `_id` can NEVER be manually set on insert
- `{col: null}` clears a column (nullable columns only, else `NOT_NULLABLE`)
- TTL field `ttl: DURATION` sets row expiry (`0` = no row TTL, see TTL section)
- Bulk limit: default 100 records per upsert
- Response `Data` holds the written rows incl. `_id`. **Blob columns appear as their byte
  length (`long`)**, not as base64 — only `get` returns the content

### Conditional upsert (`when`) — compare-and-set

```
## Update only if the STORED row matches (optimistic concurrency / ETag)
upsert grain_state {grain_key: 'k', etag: 'E2', payload: '...'} on grain_key when etag = 'E1'

## Insert only — fails if the row already exists
upsert grain_state {grain_key: 'k', etag: 'E1'} on grain_key when not exists

## Update only — never inserts
upsert grain_state {grain_key: 'k', etag: 'E2'} on grain_key when exists

## Full WHERE grammar; implicit on _id works too
upsert grain_state {grain_key: 'k', etag: 'E2'} on grain_key when etag in ['E1', 'E1b'] and payload is not null
upsert grain_state {_id: 42, etag: 'E2'} when etag = 'E1'
```

| Clause | Row exists, condition true | Row exists, condition false | Row missing |
|---|---|---|---|
| `when <where-expr>` | update | `CONDITION_FAILED` | `CONDITION_FAILED` |
| `when exists` | update | – | `CONDITION_FAILED` |
| `when not exists` | `CONDITION_FAILED` | – | insert |
| no `when` | update | – | insert |

- The condition is evaluated against the **stored** row found via `on` / `_id`, not the new values.
- Atomic: check + write happen in one step on the single writer — also over HTTP with many
  clients/processes. Of two writers expecting the same ETag, exactly one wins.
- `CONDITION_FAILED` response: `Data` holds the **complete current row** (like `get`), so the
  caller learns the stored ETag without a second read. Empty `Data` = row missing.
- An **expired TTL row counts as missing** (even before the cleanup removed it);
  `when not exists` then frees it (physical delete) and inserts a genuinely new row with a
  new `_id` — exactly as if the cleanup had already run.
- Inside `atomic`, `CONDITION_FAILED` rolls back the whole transaction; `Data` shows the row
  as stored **after** the rollback (not the transaction's intermediate state).
- **Single records only:** `when` on a bulk upsert is a `SYNTAX_ERROR` — use `atomic` with
  single conditional upserts instead.
- `when` needs `on` or an `_id` in the record (else `SYNTAX_ERROR`). `when not exists` needs
  `on` (a row cannot be inserted with an explicit `_id`).
- Clause order: `upsert T {…} on COL when …` — `when` comes last.

---

## GET

```
get TABLE
    [AGGREGATE column [as alias]]
    [select col1, col2 | LITERAL as alias | -select col1, col2]
    [distinct]
    [where WHERE]
    [count]
    [group by col1, col2]
    [order by col1 [desc], col2 [asc]]
    [dedup by col1, col2]
    [limit N]
    [page N size M]
    [after 'CURSOR']
    [follow FOLLOW]*
    [select col1, follow_alias.col2]
```

Examples:
```
get users
get users where active = true and age >= 18
get users select name, email where active = true
get users -select password_hash
get users order by name asc, created desc
get users page 2 size 20
get users where active = true limit 10
get users where active = true count

## Aggregation
get orders sum total as revenue where status = 'completed'
get orders avg total as average_order
get orders min created as first_order
get orders max total as biggest_order

## Group By
get orders sum total as revenue group by status

## Computed Columns
get products select name, price * quantity as line_total

## Literal Columns
get routes select host, true as preserve_host, 'auto' as protocol

## Distinct
get orders select status distinct

## Dedup by (first row per key, all columns)
get orders order by created desc dedup by user_id     ## newest order per user

## Semi / anti follow (exists / not exists)
get users follow users._id -?> orders.user_id         ## users with at least one order, once each
get users follow users._id -!> orders.user_id         ## users without any order
```

### Literal columns (`LITERAL as alias`)

Project constant values — useful to line up result shapes across queries (discriminator
fields, defaults for columns this table doesn't have).

```
get routes select host, true as preserve_host        ## bool
get routes select host, 'auto' as backend_protocol   ## string
get routes select host, 1 as version                 ## integer → long
get routes select host, 2.5 as factor                ## float → double
get routes select host, -1 as offset                 ## negative number
get routes select host, null as cert_path            ## null
get routes select 1 as x                             ## no real column at all
```

**Rules:**
- **The alias is mandatory** — `select host, 'auto'` → `SYNTAX_ERROR: literal in select requires an alias`
- The value is identical in **every** row, including rows a right/outer join builds without a source row
- Key order follows the select list: `select true as a, host` → `{ a, host }`
- Types: `1` → long, `2.5` → double, `'x'` → string, `true`/`false` → bool, `null` → null

**`true` / `false` / `null` are literals only before `as`.** They are identifier tokens, so without
a following `as` they stay column references:

```
get routes select true as x    ## literal true
get routes select true         ## the column named 'true' (UNKNOWN_COLUMN if absent)
```

**Not combinable with aggregates** — the aggregate sits before the select
(`get t sum port as total`) and the parser skips the select clause once it sees one.
`select`/`-select` and aggregates are mutually exclusive in the grammar.

**Not allowed in `-select`** — it removes columns by name, where a literal is meaningless →
`SYNTAX_ERROR: literals are not allowed in '-select'`.

**Post-follow:** literals work in a select after a `follow` too. A base literal survives an
explicit post-follow select only if listed there — exactly like a base computed column.

```
get routes select host, 1 as v follow routes._id -> backends.route_id as b select host, b.name
  → v is gone
get routes select host, 1 as v follow routes._id -> backends.route_id as b select host, b.name, v
  → v stays
```

There are no literals in the select attached **directly to a `follow`** (the target-table select):
a literal in the list marks the select as a post-follow select. Nothing is lost — a constant does
not depend on the joined row, so `follow ... select 1 as x` would equal the post-follow form.

### Order by needs the column in the result

Sorting runs on the projected rows. A sort column that exists in the table but not in the
select is rejected — it used to pass silently and simply not sort.

```
get routes select host, port order by port desc   ## ok — port is in the result
get routes order by port desc                     ## ok — no select, every column is there
get routes select host, port as p order by p      ## ok — the alias is the row key
get routes select host order by port desc         ## UNKNOWN_COLUMN: 'order by port' requires 'port' in the select list
get routes -select port order by port             ## error — port was excluded
get routes select host, port as p order by port   ## error — the key is now 'p'
```

Exceptions (work without the column selected):
- `order by _id [desc] limit N` — dedicated top-N path, sorts correctly
- `order by _id` with `after 'CURSOR'` — cursor paging sorts by `_id` by construction
- `count` — no rows come back, ordering is moot

> **Breaking:** queries like `select host order by port` used to run without error but came
> back unsorted. They are now an error.

**With `follow`** the sort runs **after** the joins, so followed columns (`alias.col`) really sort —
even when the post-follow select doesn't list them. Post-follow select aliases work too. The sort is
stable: rows with equal sort keys keep their join order.

```
get users follow users._id -> orders.user_id as o order by o.amount desc
get users follow users._id -> orders.user_id as o select name, o._id order by o.amount desc
get users follow users._id -> orders.user_id as o select name, o.amount as amt order by amt
```

> **Bugfix:** the sort used to run before the join — `order by o.amount` passed without error but
> did not sort at all.

The sort column must exist on the joined rows, otherwise `UNKNOWN_COLUMN`:

```
... as o order by o.amout                    ## column 'amout' does not exist on 'orders'
... as o order by x.amount                   ## 'x' is not the alias of a column-producing follow
... as o select amount order by o.status     ## add 'status' to the select of follow 'o'
get users select name follow ... order by email   ## column 'email' is not in the joined rows — add it to the select before 'follow'
... -?> orders.user_id as o order by o.amount     ## semi/anti follows have no columns
```

**With `group by`** the result only holds the group columns and `count` or the aggregate alias
(plus literals). Sorting by anything else → `UNKNOWN_COLUMN`:

```
get artikel group by gruppe order by count desc                ## ok
get artikel sum preis as summe group by gruppe order by summe  ## ok
get artikel select sku, name group by sku order by name        ## UNKNOWN_COLUMN: 'order by name' requires 'name' in the result — group by returns only the group columns and 'count'
```

> **Breaking:** these queries used to run without error and came back silently unsorted.
> `group by` is aggregation — for "one full row per key" use `dedup by`, not `group by`.

The **post-follow select** is checked the same way — a column that doesn't exist on the joined rows
(typo, wrong alias, `users.name` instead of `name`, base column not selected before `follow`) used
to be silently missing from the result and is now `UNKNOWN_COLUMN`. Also applies to `-select`.

### Dedup by (first row per key)

`dedup by col1, col2` keeps only the **first** result row per key combination — with all its
columns (unlike `distinct`, which dedups over the whole projected row). "First" means result
order, i.e. **after** `order by`. This is SQL's `DISTINCT ON` / "top 1 per group".

```
get orders order by created desc dedup by user_id           ## newest order per user
get orders order by amount desc dedup by user_id, status    ## biggest order per user+status
get users
    follow users._id -> orders.user_id as o
    order by o.amount desc
    dedup by _id                                            ## each user with their biggest order
get users
    follow users._id -> user_roles.user_id as ur
    follow ur.role_id -?> roles._id where roles.name = 'editor'
    dedup by _id                                            ## M:N semi → one row per user
```

**Rules:**
- Without `order by`, slot/join order decides — which row wins is then not guaranteed
- The columns must be **in the result** (same rule as `order by`): `select amount dedup by user_id` →
  `UNKNOWN_COLUMN: 'dedup by user_id' requires 'user_id' in the select list`. With a select alias, the alias counts.
- With `follow`, keys are row keys — base columns (`_id`, `name`), followed columns (`o.status`),
  post-follow select names. A semi/anti follow has no columns → `UNKNOWN_COLUMN`
- `null` is a key of its own (all `null` rows share one)
- Execution order: `where` → `distinct` → `follow` → `order by` → **`dedup by`** → `count` / `limit` / `page`
  (so `count` counts deduped rows, and so does `Paging.Total`)
- Position in the query text doesn't matter (like the other trailing clauses); allowed once
- **Cannot** combine with `group by`, aggregates or `after` → `SYNTAX_ERROR`

### Duplicate output names

An output name may appear only once as soon as at least one of the colliding entries carries an
explicit alias:

```
get routes select host, host                   ## ok — plain repetition
get routes select host, 1 as host              ## SYNTAX_ERROR: duplicate output name 'host'
get routes select host as x, port as x         ## SYNTAX_ERROR: duplicate output name 'x'
get orders select price * 2 as x, qty * 3 as x ## SYNTAX_ERROR: duplicate output name 'x'
get routes -select host, host                  ## ok — exclude is not checked
```

> **Breaking:** `select price * 2 as x, qty * 3 as x` used to be accepted (last writer silently
> won) and is now an error.

### Cursor paging (`after`)

Keyset paging over `_id` — use this instead of `page N size M` whenever you read
a table completely (hydration loops, exports). The client carries the last seen
`_id` as the cursor:

```
get orders after '0' limit 500      ## first page (ids > 0)
## Response.Paging: { next_cursor: '517', next: "get orders after '517' limit 500", ... }
get orders after '517' limit 500    ## next page
## ...loop until next_cursor = null → done
```

Rules:
- Returns rows with `_id > CURSOR`, ordered by `_id` ascending
- `limit N` = page size; without `limit` the server default page size applies
- Response `Paging`: `next_cursor` (last `_id` of the page, `null` on the final page),
  `next` (ready-to-run follow-up query), `total` (matching rows from the cursor onward), `page` = 0
- Combines with `where` and `select`; `order by _id` (asc) is allowed but redundant
- **Cannot** combine with `page`, `count`, `distinct`, `group by`, aggregates,
  `follow`, or any other `order by` → `SYNTAX_ERROR`
- Linear server cost for a full table walk (offset paging is quadratic), and
  correct under concurrent writes: `_id` is monotonic and never recycled, so
  deletes/inserts between pages never skip or duplicate rows
- Prefer `page N size M` only for UI grids with visible page numbers

`order by _id [desc] limit N` without `after` uses the same fast path
(top-N without full materialization) — identical results, just faster.

---

## DELETE

```
delete TABLE where WHERE
delete TABLE where WHERE expect N
```

**WHERE is mandatory** — no accidental full-table deletes.

```
delete users where active = false
delete sessions where created < '2024-01-01 00:00:00.0000'

## Conditional delete: only if exactly one row matches (ETag check)
delete grain_state where grain_key = 'k' and etag = 'E1' expect 1
```

- `expect N`: if a different number of rows matches, **nothing** is deleted and the result is
  `EXPECTATION_FAILED` (message states the count found). Inside `atomic` it rolls back.
- With `expect`, expired TTL rows count as missing: neither counted nor deleted (TTL cleanup
  removes them).
- Without `expect`, a delete signals "nothing matched" only via `affected = 0`.

---

## DESCRIBE

```
describe            ## List all tables
describe TABLE      ## Show table schema
```

Response includes `chunk_size` and `effective_chunk_size` properties.

---

## SHRINK

```
shrink table TABLE
shrink table TABLE chunk_size N
shrink database
shrink database chunk_size N
```

- `shrink table`: Compacts index + column files, closes gaps from deleted rows
- Not needed for steady delete + insert churn: freed slots are reused once ≥ 20 % of the
  slots are free (backfill), so such a table levels off at about rows / 0.8 slots. Use
  `shrink` only to physically shrink files after a large one-off delete.
- `shrink table chunk_size N`: Additionally sets a new chunk_size for the table
- `shrink database`: Sets DB-level chunk_size, shrinks all tables WITHOUT their own chunk_size
- Tables with their own chunk_size are **skipped** by `shrink database`
- Target slots: `max(chunk_size, ceil(rows / chunk_size) * chunk_size)`

---

## Column Operations

```
add column TABLE.COLUMN TYPE [SIZE] [strict] [default VALUE]
rename column TABLE.OLD_NAME to NEW_NAME
alter column TABLE.COLUMN string NEW_SIZE
```

---

## Index Operations

```
create index TABLE.COLUMN
create index unique TABLE.COLUMN
purge index TABLE.COLUMN
```

**`unique` comes AFTER `index`, not before it.** SQL writes `CREATE UNIQUE INDEX`; SproutDB writes `create index unique`. The SQL order is a parse error — `create` only accepts `database`, `table`, `index` or `apikey` as its next token, so `create unique index users.email` fails with `expected 'database', 'table', 'index' or 'apikey'`.

```
create index unique users.email     ✓
create unique index users.email     ✗ SYNTAX_ERROR
```

- Indexes are **not** unique by default. Plain `create index` builds a lookup structure and enforces no constraint. Only `create index unique` rejects duplicate values (`UNIQUE_VIOLATION`).
- Blob columns cannot be indexed

### NULL and unique indexes

A unique index constrains **non-null values only**. NULL is skipped entirely by the check, which has three consequences:

- **Any number of rows may hold NULL** in a unique column. Two NULLs are not "duplicates" — this differs from databases that treat NULL as a comparable value.
- **Omitting the column in an upsert skips the check** for that record, exactly as NULL does.
- `create index unique` on an existing column **fails** if that column already contains duplicate *non-null* values (`cannot create unique index on 'col': column contains duplicate values`). Pre-existing NULLs never block index creation.

```
create index unique users.email
upsert users {name: 'a', email: null}   ✓
upsert users {name: 'b', email: null}   ✓ second NULL is fine
upsert users {name: 'c'}                ✓ column omitted, no check
upsert users {name: 'd', email: 'x@y'}  ✓
upsert users {name: 'e', email: 'x@y'}  ✗ UNIQUE_VIOLATION
```

To make a column reject NULL, give it a `default` — a column is nullable **iff it has no default**. There is no `not null` keyword, and `strict` does not affect nullability (it only forbids type widening).

> **Caveat:** do not rely on `default` + `unique index` to force distinct values on every row. Defaults are materialized after the unique check runs, so two records that both omit the column receive the same default value without triggering `UNIQUE_VIOLATION`. Supply the value explicitly when it must be unique.

---

## Purge Operations

```
purge table NAME
purge database
purge column TABLE.COLUMN
purge index TABLE.COLUMN
purge ttl TABLE
```

---

## TTL

```
create table sessions (token string) ttl 24h
upsert sessions {token: 'abc', ttl: 1h}
purge ttl sessions
```

Units: `s` (seconds), `m` (minutes), `h` (hours), `d` (days).

Behavior you can rely on:
- Expired rows are **invisible to `get` immediately** — even before the background
  cleanup physically deletes them (cleanup runs periodically, default every 5 minutes).
- `upsert … on COL` treats an expired row as missing too: if `on` hits an expired,
  not-yet-cleaned row, that row is freed (physically deleted) and the record inserted as a
  **new row with a new `_id`** — an expired row is never revived with its old values.
- **Every upsert resets the expiry:** with `ttl: X` the row expires X from now; without a
  `ttl` field (or with `ttl: 0`) the table TTL applies again — or none if the table has
  none. So an update without `ttl` removes a row TTL.
- `purge ttl TABLE` removes the table-level TTL.

---

## Auth Commands

```
create apikey 'name'
purge apikey 'name'
rotate apikey 'name'
grant ROLE on DATABASE to 'apikey_name'
revoke DATABASE from 'apikey_name'
restrict TABLE to ROLE for 'apikey_name' on DATABASE
unrestrict TABLE for 'apikey_name' on DATABASE
```

Roles: `admin`, `writer`, `reader`. Restrict only `reader` or `none`.

---

## Backup & Restore

```
backup
restore 'path/to/backup'
```

---

## WHERE Clause

### Comparison

| Operator | Description |
|----------|-------------|
| `=` | Equal |
| `!=` | Not equal |
| `>`, `>=`, `<`, `<=` | Comparison |
| `contains` | Substring (string only) |
| `starts` | Prefix (string only) |
| `ends` | Suffix (string only) |
| `between X and Y` | Inclusive range |
| `not between X and Y` | Outside range |
| `in [v1, v2]` | Membership list |
| `not in [v1, v2]` | Exclusion list |

### Null checks

```
column is null
column is not null
```

### Membership

```
column in ['val1', 'val2', 'val3']
column not in ['val1', 'val2']
```

### Logic

| Operator | Precedence |
|----------|-----------|
| `or` | Lowest |
| `and` | Medium |
| `not` | Highest (prefix) |

String comparisons are **case-sensitive** (byte-level UTF-8).

---

## FOLLOW (Join)

### Arrow Types

| Arrow | Type | Behavior |
|-------|------|----------|
| `->` | Inner | Only rows with match in both tables |
| `->?` | Left | All source rows, NULL if no match |
| `?->` | Right | All target rows, NULL if no match |
| `?->?` | Outer | All rows from both tables |
| `-?>` | Semi | Each source row **once** if at least one target row matches — adds no target columns |
| `-!>` | Anti | Each source row that has **no** matching target row — adds no target columns |

> Don't mix them up: `->?` (left, `?` = that side may be missing) vs. `-?>` (semi, `?` **inside** the arrow).

### Syntax

```
follow SOURCE_TABLE.SOURCE_COL ARROW TARGET_TABLE.TARGET_COL as ALIAS
    [select col1, col2]
    [where CONDITION]

## Semi/anti: alias optional, no select
follow SOURCE_TABLE.SOURCE_COL (-?> | -!>) TARGET_TABLE.TARGET_COL [as ALIAS]
    [where CONDITION]
```

### Examples

```
## Inner join
get users
    follow users._id -> orders.user_id as orders

## Left join (all users, even without orders)
get users
    follow users._id ->? orders.user_id as orders

## Filter on followed table
get users
    follow users._id -> orders.user_id as orders
        where orders.total > 100

## Multiple conditions on follow filter
get users
    follow users._id -> orders.user_id as orders
        where orders.status in ['completed', 'shipped'] and orders.total > 50

## Follow filter with string operators
get users
    follow users._id -> orders.user_id as orders
        where orders.product starts 'Premium'

## Select on followed table
get users
    follow users._id -> orders.user_id as orders
        select product, total

## Chained follows
get users
    follow users._id -> orders.user_id as orders
    follow orders.product_id -> products._id as product

## Follow with filter + chained second follow
get users
    follow users._id -> orders.user_id as orders
        where orders.status = 'completed'
    follow orders.product_id -> products._id as product

## Post-follow select
get users
    follow users._id -> orders.user_id as orders
    select name, orders.total, orders.product

## Semi: users with at least one completed order (each user once)
get users
    follow users._id -?> orders.user_id where orders.status = 'completed'

## Anti: users without a completed order — the alias is optional, it only names the where prefix
get users
    follow users._id -!> orders.user_id as o where o.status = 'completed'

## Semi at the end of a chain: users with the admin role
get users
    follow users._id -> user_roles.user_id as ur
    follow ur.role_id -?> roles._id where roles.name = 'admin'

## Anti inside a chain: orders without line items
get users
    follow users._id -> orders.user_id as o
    follow o._id -!> order_items.order_id
    select name, o._id

## Several semi/anti follows = AND: users with a role but no order
get users
    follow users._id -?> user_roles.user_id
    follow users._id -!> orders.user_id
```

Follow expands rows: 1 user with 3 orders → 3 result rows.
Follow columns are prefixed with alias: `orders._id`, `orders.total`.
Follows run in sequence — each one works on the rows the previous one produced.
`order by` sorts after all follows, so `order by alias.col` sorts correctly.

### Semi / anti (`-?>` / `-!>`)

- **Filter only** — rows pass through unchanged (no expansion, no target columns, order kept)
- A `null` source key never matches: semi drops the row, anti keeps it
- **Works on the current rows of the chain**, not on the base table: after a `->` to a junction table,
  a semi filters every (user, junction) pair on its own. Duplicate junction rows → several rows per user
  → append `dedup by _id`.
- `as ALIAS` is optional; without it the where prefix is the table name (`orders.status`)
- `select` directly on a semi/anti follow → `SYNTAX_ERROR` (select base columns before `follow`)
- You cannot follow on from a semi/anti follow (`follow o._id -> ...` after `... -?> orders.user_id as o`)
  → `SYNTAX_ERROR: cannot follow from 'o'`
- Combines with every other follow (before/after `->`, `->?`, `?->`), `count`, `limit`, `page`, `order by`, `dedup by`

---

## Type Widening

Type widening works via `add column table.col newtype`. Allowed widenings:

- `ubyte` → `ushort` → `uint` → `ulong`
- `sbyte` → `sshort` → `sint` → `slong`
- `float` → `double`

Data is preserved, NULL values remain NULL.

---

## Error Codes

| Code | Description |
|------|-------------|
| `SYNTAX_ERROR` | Query parse failure |
| `UNKNOWN_TABLE` | Table doesn't exist |
| `UNKNOWN_COLUMN` | Column doesn't exist |
| `UNKNOWN_DATABASE` | Database doesn't exist |
| `TABLE_EXISTS` | Table already exists |
| `DATABASE_EXISTS` | Database already exists |
| `INDEX_EXISTS` | Index already exists |
| `INDEX_NOT_FOUND` | Index doesn't exist |
| `TYPE_MISMATCH` | Value doesn't match column type |
| `NOT_NULLABLE` | NULL for non-nullable column |
| `TYPE_NARROWING` | Type narrowing not allowed |
| `STRICT_VIOLATION` | Type widening on strict column |
| `BULK_LIMIT` | Too many records in bulk upsert |
| `WHERE_REQUIRED` | DELETE needs WHERE |
| `UNIQUE_VIOLATION` | Unique index violated |
| `ID_NOT_FOUND` | Upsert with an `_id` that does not exist |
| `CONDITION_FAILED` | `upsert … when …` condition not met — `Data` holds the current row (empty if none) |
| `EXPECTATION_FAILED` | `delete … expect N`: different number of matching rows, nothing deleted |
| `PARAMETER_ERROR` | `@name` parameter missing, unused or not renderable — nothing executed |
| `PROTECTED_NAME` | Name with `_` prefix (system-reserved) |
| `AUTH_REQUIRED` | No API key provided |
| `AUTH_INVALID` | API key invalid |
| `PERMISSION_DENIED` | Insufficient permissions |
| `KEY_EXISTS` | API key name already exists |
| `KEY_NOT_FOUND` | API key not found |
