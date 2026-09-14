---
name: sproutdb
description: SproutDB query language, schema design, and C# integration reference. ALWAYS use this skill when writing ANY code that touches SproutDB — queries, migrations, table design, upserts, conditional writes / optimistic concurrency (when, expect, ETags), @parameters, follows/joins (incl. semi/anti joins, exists / not exists), dedup by / top-N per group, TTL, indexes, auth, transactions, the typed LINQ API, or ISproutDatabase/ISproutServer usage (Query / QueryAsync). Also trigger when the user mentions SproutDB, sprout queries, or references SproutDB tables/namespaces. SproutDB has a custom query language that is NOT SQL — do not guess syntax, always consult this skill first. Even for simple queries, the syntax differs enough from SQL that checking is essential.
---

# SproutDB Reference Skill

SproutDB is a custom embedded/networked database with its own query language (NOT SQL).
**Never guess syntax — always check the reference files below.**

## When to read which file

| Task | Read |
|------|------|
| Writing queries (get, upsert, delete, create table, transactions, etc.) | [references/query-language.md](references/query-language.md) |
| Designing tables, choosing types, modeling relationships | [references/schema-design.md](references/schema-design.md) |
| C# integration (DI, migrations, typed LINQ API, `Query()`, responses) | [references/csharp-integration.md](references/csharp-integration.md) |

## Critical differences from SQL

- **UPSERT, not INSERT/UPDATE** — `upsert table {key: 'val'}` with a JSON-like body.
- **GET, not SELECT** — `get table where ...`, not `select * from table where ...`.
- **FOLLOW, not JOIN** — `follow source.col -> target.col as alias`.
- **No parentheses around the type size** — `create table t (name string 100)`, not `(name string(100))`.
- **`##` for comments** — not `--` or `/* */`. Single quotes for strings — `'hello'`, not `"hello"`.
- **`_id` is automatic** — ulong, auto-increment, never set manually on insert.
- **No ALTER TABLE** — use `add column`, `rename column`, `alter column` as separate statements.
- **`create index unique t.col`, not `create unique index`** — the `unique` keyword follows `index`. SQL's order is a parse error. Plain `create index` is non-unique. For a new table prefer the column modifier `create table t (k string 64 unique)` — table + index in one atomic statement.
- **No schema changes inside `atomic`** — only `upsert` / `delete` / `get` / `describe`; `create`/`add column`/`purge` inside a transaction is a `SYNTAX_ERROR`.
- **DELETE requires WHERE** — no statement deletes all rows at once.
- **A literal in the select needs an alias** — `select host, true as preserve_host`, never bare `select 1`. And `true`/`false`/`null` only count as literals before an `as`: `select true` means the *column* named `true`.
- **No duplicate output names once an alias is involved** — `select price * 2 as x, qty * 3 as x` is a parse error. Bare `select host, host` stays fine.
- **`select` comes BEFORE `where`** — `get users select name where active = true`. SQL-style `get users where active = true select name` is a parse error (`'select' must come directly after the table name`). Only after a `follow` may a second `select` appear at the end.
- **String escapes are `\'` and `\\` only** — a value ending in a backslash must be written `'C:\\data\\'`; `'a\'` does not close the literal. In code: double `\` first, then escape `'`.
- **`order by` columns must be in the select list** — SQL sorts by unselected columns, SproutDB rejects it: `select host order by port` is an error; select `port` too, or drop the select. (Exceptions: `order by _id ... limit N` and `after` cursor paging.)
- **No foreign keys** — relationships exist only at query time via `follow`.
- **EXISTS / NOT EXISTS are follow arrows, not subqueries** — `follow users._id -?> orders.user_id` (semi: rows with a match, once each), `follow users._id -!> orders.user_id` (anti: rows without a match). No target columns, `as` optional, no `select` on them. Don't confuse `-?>` (semi) with `->?` (left join). Emulating semi with `->` + `distinct` is wrong as soon as target columns are involved.
- **`group by` is aggregation only** — it returns the group columns plus `count` or the aggregate, nothing else (no other selected columns, no `page`). Don't use it to dedup rows; that's `dedup by`.
- **"Top 1 per group" is `dedup by`, not a window function** — `get orders order by created desc dedup by user_id` keeps the first row per key (after `order by`), with all columns. Columns must be in the result like `order by`.
- **Semicolons = multi-query / transaction delimiter** — `q1; q2; q3` runs three queries; `atomic; ...; commit` wraps them in a transaction. A single query needs no terminator.
- **Conditional writes use `when` / `expect`, not `IF`/`WHERE` on upsert** — `upsert t {…} on k when etag = 'E1'`, `… on k when not exists`, `delete t where … expect 1`. Failure codes: `CONDITION_FAILED` (Data = current row) / `EXPECTATION_FAILED`. `when` only on single-record upserts. Don't emulate CAS with read-compare-write in C# — it is not atomic.
- **Prefer junction tables for relationships** — an `array` column type exists, but M:N/1:N relations are modeled with junction tables, not arrays.

## Common LLM mistakes (read before writing C#)

- **Never interpolate runtime values into query strings** — use `@name` parameters: `db.Query("get users where email = @email", new { email })`. Values can't become syntax (no injection, no escaping). Placeholders only at value positions, not for table/column names.

- `ISproutDatabase.Query(string)` returns **`List<SproutResponse>`**, NOT a single `SproutResponse`. Index `[0]` for a single query. (This is a breaking change from older docs.)
- The typed `db.Table<T>("name")` API requires **`T : class, ISproutEntity, new()`** — `ISproutEntity` mandates a `ulong Id { get; set; }` mapped to `_id`.
- Transactions are written in the query string (`atomic; ...; commit`), there is no `BeginTransaction()` C# API.
- Several processes sharing one database → run one server (`AddSproutDB` + `MapSproutDB()`) and register **`services.AddSproutDBClient(o => o.BaseAddress = …)`** in the others — same `ISproutServer`/`ISproutDatabase` API, no hand-written HTTP code.
- In async code (ASP.NET Core, Orleans) use **`await db.QueryAsync(...)`** instead of wrapping `Query()` in `Task.Run` — it doesn't block a thread while writes queue. Same `List<SproutResponse>` result.
