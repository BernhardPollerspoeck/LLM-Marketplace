# SproutDB C# Integration Reference

## Service Registration

```csharp
using SproutDB.Core.DependencyInjection;

// Option A: Builder callback
services.AddSproutDB(options =>
{
    options.DataDirectory = "/data/sproutdb";
    options.DefaultPageSize = 100;
    options.BulkLimit = 100;
    options.ChunkSize = 10_000;  // Default: 10000 (slots per allocation)
    options.AddMigrations<MyMigrations>("database_name");
});

// Option B: from IConfiguration ("SproutDB" section)
services.AddSproutDB(configuration);

// Option C: Configuration + code override
services.AddSproutDB(configuration, builder =>
{
    builder.DataDirectory = "/override/path";
});
```

### Durability (WAL group commit)

Every write is appended to the WAL (OS buffer) and acknowledged immediately; the fsync runs
in the background every `WalSyncInterval` (default 50 ms).

| Event | Lost |
|---|---|
| Process crash (exception, kill, OOM) | **Nothing** — WAL is in the OS cache, replayed on start |
| Power loss / OS crash | Acknowledged writes of the last ≤ `WalSyncInterval` |

```csharp
services.AddSproutDB(options =>
{
    options.WalSyncInterval = TimeSpan.Zero;  // fsync before every response: durable, much slower
});
// appsettings: "SproutDB": { "WalSyncIntervalMs": 0 }
```

### Remote client (same interfaces as embedded)

Many processes (e.g. several Orleans silos) share one SproutDB server over HTTP. Embedded
vs. remote is only a matter of registration — code against `ISproutServer` / `ISproutDatabase`
stays unchanged:

```csharp
// Embedded:  services.AddSproutDB(o => o.DataDirectory = "/data/sproutdb");
// Remote:
services.AddSproutDBClient(o =>
{
    o.BaseAddress = new Uri("https://db.example.com/");  // server with MapSproutDB() (+ MapSproutDBHub() for OnChange)
    o.ApiKey = "sdb_ak_...";                              // only if the server has auth
});

// Without DI:
using var client = new SproutDB.Core.Client.SproutClient(
    new SproutClientOptions { BaseAddress = new Uri("https://db.example.com/") });
ISproutDatabase db = client.GetOrCreateDatabase("shop");   // SelectDatabase throws if missing
```

- `Query`, `QueryAsync`, `@parameters`, typed LINQ API and `OnChange` (SignalR, one connection
  per subscription) all work remotely.
- `QueryAsync` cancellation applies only until the request is sent (then it waits), so an
  `OperationCanceledException` still means "nothing executed".
- Row values arrive as JSON primitives: `string`, `long`/`ulong`, `double`, `bool`, `null`,
  `List<object?>`; `_id` is always `ulong`. The typed API converts to property types.
- Server-only (throw `NotSupportedException` remotely): `GetDatabases()`, `Migrate(...)`
  (register migrations on the server), `SaveQuery(...)`.

## Auth (opt-in)

```csharp
services.AddSproutDBAuth(o => o.MasterKey = "sdb_ak_...");
```

## HTTP + Admin Endpoints

```csharp
var app = builder.Build();
app.MapSproutDB();        // POST /sproutdb/query
app.MapSproutDBHub();     // SignalR /sproutdb/changes
app.MapSproutDBAdmin();   // Blazor Admin UI /sproutdb/admin
```

## Direct Engine Access (without DI)

```csharp
var engine = new SproutEngine(new SproutEngineSettings
{
    DataDirectory = "/data/sproutdb"
});
var db = engine.GetOrCreateDatabase("mydb");

// Query() ALWAYS returns List<SproutResponse> (one entry per query/transaction).
List<SproutResponse> results = db.Query("get users");
SproutResponse result = results[0];

engine.Dispose();
```

---

## Interfaces

```csharp
// ISproutServer (Singleton)
ISproutDatabase GetOrCreateDatabase(string name);
ISproutDatabase SelectDatabase(string name);        // throws if not found
IReadOnlyList<ISproutDatabase> GetDatabases();
void Migrate(Assembly assembly, ISproutDatabase database);

// ISproutDatabase
string Name { get; }

// IMPORTANT: returns a LIST — one SproutResponse per semicolon-separated query
// (a transaction block collapses to its own marker entry). Index [0] for a single query.
List<SproutResponse> Query(string query);

// Async: doesn't block a thread while writes wait in the single-writer queue;
// pure reads complete synchronously. The token only cancels writes the writer has
// not started — OperationCanceledException means no write of this call ran
// (once a write ran, the rest of the batch runs too).
ValueTask<List<SproutResponse>> QueryAsync(string query, CancellationToken cancellationToken = default);

IDisposable OnChange(string table, Action<SproutResponse> callback);

// Saves a query to the _saved_queries system table (for the Admin UI).
// Creates the table automatically. Usable in migrations to seed saved queries.
void SaveQuery(string name, string query, bool pinned = false);
// db.SaveQuery("Low Stock", "get plants where stock < 20 order by stock");
// db.SaveQuery("Pending Orders", "get orders where status = 'pending'", pinned: true);
```

### Running a single query

```csharp
var response = db.Query("get users where active = true")[0];
if (response.Errors is { Count: > 0 })
{
    // handle response.Errors[0].Code / .Message
}
```

### Parameters (`@name`) — use these for every runtime value

Never interpolate values into the query string. Pass them as parameters: each value
becomes exactly one literal and can never turn into query syntax (no injection, no escaping).

```csharp
// Extension methods on ISproutDatabase (namespace SproutDB.Core)
db.Query("get users where email = @email", new { email });
await db.QueryAsync("get users where email = @email", new { email }, cancellationToken);
db.Query("upsert gs {k: @k, etag: @next} on k when etag = @expected",
    new { k = key, next = newEtag, expected = oldEtag });
db.Query("get users where name in @names", new { names = new[] { "a", "b" } });   // list -> [..]
db.Query("get users where name = @n", new Dictionary<string, object?> { ["n"] = "x" });
```

- Only at value positions (WHERE values, upsert values, `in @list`, `when`) — never table/column names.
- Names are case-insensitive; one placeholder may appear several times; shared by all statements of a batch.
- Types: string, char, numbers, bool, null, DateTime/DateTimeOffset(UTC)/DateOnly/TimeOnly,
  `byte[]` (base64 for blob columns), lists of these. Other types → `ToString()` as string.
- Missing/unused/unrenderable parameter → `PARAMETER_ERROR` (with position); nothing runs.
- `@` inside a string literal (`'mail@x.com'`) is not a placeholder.
- The typed LINQ API uses parameters internally.
- HTTP: `Content-Type: application/json` with `{"query": "...", "parameters": {...}}`.

### Multi-query batching

```csharp
// Three queries, three responses (positional, same order).
List<SproutResponse> responses = db.Query(
    "get users; get orders; describe users");
var users  = responses[0];
var orders = responses[1];
var schema = responses[2];
```

### Transactions

```csharp
// atomic ... commit wraps mutations; all-or-nothing (real MMF-level rollback).
// GET/DESCRIBE inside a transaction see uncommitted writes (read-your-own-writes).
var responses = db.Query(
    "atomic; " +
    "upsert accounts {_id: 1, balance: 50}; " +
    "upsert accounts {_id: 2, balance: 150}; " +
    "commit");
// The LAST entry is the transaction marker (Operation == SproutOperation.Transaction).
var marker = responses[^1];          // marker.Affected = total rows changed
```

---

## Typed LINQ API

Strongly-typed access via `db.Table<T>("name")`. The entity **must** implement `ISproutEntity`:

```csharp
public interface ISproutEntity   // built into SproutDB.Core
{
    ulong Id { get; set; }       // mapped to the _id column
}

public sealed class User : ISproutEntity
{
    public ulong Id { get; set; }          // <- required by ISproutEntity (maps to _id)
    public string Name { get; set; } = "";
    public string Email { get; set; } = "";
    public byte Age { get; set; }
    public bool Active { get; set; }
}
```

Constraint: `Table<T>` requires `T : class, ISproutEntity, new()`.

### Queries

```csharp
var users = db.Table<User>("users");

// Run() -> SproutResponse (same object as HTTP / raw Query string)
SproutResponse response = users.Where(u => u.Age > 18).Run();

// Typed materialization
List<User> adults = users.Where(u => u.Age > 18).ToList();
List<User> top10  = users.Where(u => u.Active)
                         .OrderByDescending(u => u.Age)
                         .Take(10)
                         .ToList();
User? john = users.FirstOrDefault(u => u.Id == 42);
int activeCount = users.Where(u => u.Active).Count();

// Projection (Select shapes the generated `select` clause)
var names = users.Where(u => u.Active).Select(u => u.Name).Run();
```

Available builder methods: `Where`, `Select`, `OrderBy`, `OrderByDescending`, `Take`, `Run`, `ToList`, `FirstOrDefault`, `Count`, `Upsert`, `Delete`.

### Upsert / Delete

```csharp
// Insert (Id omitted -> auto-generated)
users.Upsert(new User { Name = "John", Email = "john@test.com", Age = 25 });

// Partial update via anonymous object (must include Id)
users.Upsert(new { Id = 1ul, Age = (byte)26 });

// Upsert on a match column instead of _id
users.Upsert(new User { Email = "john@test.com", Name = "John Doe" }, on: u => u.Email);

// Bulk upsert
users.Upsert(new[] { user1, user2, user3 }, on: u => u.Email);

// Conditional upsert (-> "... on email when version = 7")
var r = users.Upsert(user, on: u => u.Email, when: u => u.Version == expectedVersion);
if (r.Errors is { Count: > 0 } && r.Errors[0].Code == "CONDITION_FAILED")
{
    // r.Data holds the current stored row (empty if it does not exist)
}

// Insert only (-> "when not exists")
users.Upsert(user, on: u => u.Email, ifNotExists: true);

// Delete by predicate
users.Delete(u => u.Active == false);
users.Delete(u => u.Id == 42);

// Delete only if exactly N rows match, else EXPECTATION_FAILED and nothing deleted
users.Delete(u => u.Email == email && u.Version == v, expect: 1);
```

`Upsert` / `Delete` do not throw — they return the `SproutResponse`; check `Errors` for
`CONDITION_FAILED` / `EXPECTATION_FAILED`.

### Optimistic concurrency pattern (ETag)

```csharp
// Table:           create table gs (k string 64 strict unique, etag string 32, payload blob)
// First write:     upsert gs {k: @k, etag: @etag, payload: @p} on k when not exists
// Later writes:    upsert gs {k: @k, etag: @next, payload: @p} on k when etag = @expected
// Delete:          delete gs where k = @k and etag = @expected expect 1
// Multi-row:       atomic; <conditional upsert>; <conditional upsert>; commit
var res = db.Query("upsert gs {k: @k, etag: @next, payload: @p} on k when etag = @expected",
    new { k = key, next = newEtag, p = payloadBytes, expected = oldEtag })[0];
if (res.Errors?[0].Code == "CONDITION_FAILED")
{
    var storedEtag = res.Data is { Count: > 0 } ? res.Data[0]["etag"] as string : null; // null = row gone
    throw new InconsistentStateException(storedEtag, oldEtag);
}
```

### Fluent schema builders (extension methods on `ISproutDatabase`)

```csharp
// create table users (name string 100, age ubyte, active bool default true)
db.CreateTable("users")
  .AddColumn<string>("name", size: 100)
  .AddColumn<byte>("age")
  .AddColumn<bool>("active", defaultValue: "true")
  .Execute();

// .Unique() applies to the column added last -> "email string 320 strict unique"
db.CreateTable("accounts")
  .AddColumn<string>("email", size: 320, strict: true).Unique()
  .Execute();

db.AddColumn<byte>("users", "age");                 // add column users.age ubyte
db.AddColumn<string>("users", "bio", size: 500);    // string column requires size
db.AlterColumn("users", "name", size: 200);         // alter column users.name string 200
```

These throw `SproutQueryException` if the underlying query returns an error.

---

## SproutResponse

```csharp
public sealed class SproutResponse
{
    public SproutOperation Operation { get; init; }
    public List<Dictionary<string, object?>>? Data { get; init; }
    public int Affected { get; init; }
    public SchemaInfo? Schema { get; init; }   // ChunkSize + EffectiveChunkSize (describe)
    public PagingInfo? Paging { get; init; }
    public List<SproutError>? Errors { get; init; }
    public string? AnnotatedQuery { get; init; }
}

public sealed class SchemaInfo
{
    public int ChunkSize { get; init; }
    public int EffectiveChunkSize { get; init; }
    // ... plus column definitions, populated by `describe`
}

public sealed class PagingInfo
{
    public int Total { get; init; }
    public int PageSize { get; init; }
    public int Page { get; init; }        // 0 for cursor paging
    public string? Next { get; init; }    // ready-to-run follow-up query (offset AND cursor)
    public string? NextCursor { get; init; } // only for 'after' queries; null = last page
}
```

## SproutOperation enum (byte)

```csharp
Error = 0, Get = 1, Upsert = 2, Delete = 3, Describe = 4,
CreateTable = 5, CreateDatabase = 6, PurgeTable = 7, PurgeDatabase = 8,
PurgeColumn = 9, AddColumn = 10, RenameColumn = 11, AlterColumn = 12,
CreateIndex = 13, PurgeIndex = 14, Backup = 15, Restore = 16,
CreateApiKey = 17, PurgeApiKey = 18, RotateApiKey = 19,
Grant = 20, Revoke = 21, Restrict = 22, Unrestrict = 23,
PurgeTtl = 24, ShrinkTable = 25, ShrinkDatabase = 26, Transaction = 27
```

---

## Migrations

### Interface

```csharp
public interface IMigration
{
    int Order { get; }
    MigrationMode Mode => MigrationMode.Once;
    void Up(ISproutDatabase db);
}
```

### Modes

| Mode | Behavior |
|------|----------|
| `Once` | Runs once, tracked in `_migrations` table, skipped on restart |
| `OnStartup` | Runs every start, NOT tracked. For cleanup tasks |

### Example

```csharp
public sealed class CreateUsers : IMigration
{
    public int Order => 1;
    public void Up(ISproutDatabase db)
    {
        db.Query("create table users (name string 100, email string 320 strict, active bool default true)");
        db.Query("create index unique users.email");  // note: "index unique", not SQL's "unique index"
    }
}

public sealed class AddAge : IMigration
{
    public int Order => 2;
    public void Up(ISproutDatabase db)
    {
        db.Query("add column users.age ubyte");
    }
}

public sealed class CleanupOnStart : IMigration
{
    public int Order => 99;
    public MigrationMode Mode => MigrationMode.OnStartup;
    public void Up(ISproutDatabase db)
    {
        db.Query("delete sessions where active = false");
    }
}
```

### Registration

```csharp
services.AddSproutDB(options =>
{
    options.DataDirectory = "/data";
    options.AddMigrations<MyMigrations.Marker>("shop");
});
```

`SproutMigrationHostedService` runs migrations BEFORE Kestrel starts. On failure → server won't start.

---

## HTTP API

```
POST /sproutdb/query
Content-Type: text/plain           <- body is the query
Content-Type: application/json     <- body is {"query": "...", "parameters": {...}}
```

```json
{"query": "upsert gs {k: @k, etag: @next} on k when etag = @expected",
 "parameters": {"k": "user/42", "next": "E2", "expected": "E1"}}
```
Parameter values: string, number, true/false, null or an array of those (objects → 400).

| Header | Required | Description |
|---|---|---|
| `X-SproutDB-Database` | Yes | Active database |
| `X-SproutDB-ApiKey` | Yes when auth is enabled | API key |

The response is **always a JSON array** of `SproutResponse` objects (one per query; a transaction contributes its per-statement entries plus a final transaction marker).

| Status | When |
|---|---|
| 200 | Executed (per-query errors live inside the individual responses) |
| 400 | Empty body, missing database header or invalid JSON body |
| 401 | Auth missing/invalid (auth enabled) |
| 403 | Insufficient permission (auth enabled) |

## SignalR change notifications

```csharp
// In-process
var sub = db.OnChange("users", response =>
{
    // response.Operation, response.Data, response.Affected ...
});
sub.Dispose(); // unsubscribe
```

Hub endpoint `/sproutdb/changes`. Client methods: `Subscribe(database, table)` / `Unsubscribe(database, table)`. Server pushes `OnChange(SproutResponse)`. Groups: `{database}.{table}` for data, `{database}._schema` for schema changes.
