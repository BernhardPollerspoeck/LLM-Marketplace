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

// Delete by predicate
users.Delete(u => u.Active == false);
users.Delete(u => u.Id == 42);
```

### Fluent schema builders (extension methods on `ISproutDatabase`)

```csharp
// create table users (name string 100, age ubyte, active bool default true)
db.CreateTable("users")
  .AddColumn<string>("name", size: 100)
  .AddColumn<byte>("age")
  .AddColumn<bool>("active", defaultValue: "true")
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
Content-Type: text/plain
```

| Header | Required | Description |
|---|---|---|
| `X-SproutDB-Database` | Yes | Active database |
| `X-SproutDB-ApiKey` | Yes when auth is enabled | API key |

The response is **always a JSON array** of `SproutResponse` objects (one per query; a transaction contributes its per-statement entries plus a final transaction marker).

| Status | When |
|---|---|
| 200 | Executed (per-query errors live inside the individual responses) |
| 400 | Empty body or missing database header |
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
