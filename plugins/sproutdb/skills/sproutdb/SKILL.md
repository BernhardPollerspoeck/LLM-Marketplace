---
name: sproutdb
description: SproutDB query language, schema design, and C# integration reference. ALWAYS use this skill when writing ANY code that touches SproutDB — queries, migrations, table design, upserts, follows/joins, TTL, indexes, auth, transactions, the typed LINQ API, or ISproutDatabase/ISproutServer usage. Also trigger when the user mentions SproutDB, sprout queries, or references SproutDB tables/namespaces. SproutDB has a custom query language that is NOT SQL — do not guess syntax, always consult this skill first. Even for simple queries, the syntax differs enough from SQL that checking is essential.
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
- **`create index unique t.col`, not `create unique index`** — the `unique` keyword follows `index`. SQL's order is a parse error. Plain `create index` is non-unique.
- **DELETE requires WHERE** — no statement deletes all rows at once.
- **A literal in the select needs an alias** — `select host, true as preserve_host`, never bare `select 1`. And `true`/`false`/`null` only count as literals before an `as`: `select true` means the *column* named `true`.
- **No duplicate output names once an alias is involved** — `select price * 2 as x, qty * 3 as x` is a parse error. Bare `select host, host` stays fine.
- **`order by` columns must be in the select list** — SQL sorts by unselected columns, SproutDB rejects it: `select host order by port` is an error; select `port` too, or drop the select. (Exceptions: `order by _id ... limit N` and `after` cursor paging.)
- **No foreign keys** — relationships exist only at query time via `follow`.
- **Semicolons = multi-query / transaction delimiter** — `q1; q2; q3` runs three queries; `atomic; ...; commit` wraps them in a transaction. A single query needs no terminator.
- **Prefer junction tables for relationships** — an `array` column type exists, but M:N/1:N relations are modeled with junction tables, not arrays.

## Common LLM mistakes (read before writing C#)

- `ISproutDatabase.Query(string)` returns **`List<SproutResponse>`**, NOT a single `SproutResponse`. Index `[0]` for a single query. (This is a breaking change from older docs.)
- The typed `db.Table<T>("name")` API requires **`T : class, ISproutEntity, new()`** — `ISproutEntity` mandates a `ulong Id { get; set; }` mapped to `_id`.
- Transactions are written in the query string (`atomic; ...; commit`), there is no `BeginTransaction()` C# API.
