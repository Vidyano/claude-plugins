---
name: data_migrations
description: "Run one-shot data transforms against RavenDB at startup with Raven.Migrations: AddRavenDbMigrations() plus MigrationRunner.Run() after UseVidyano, a [Migration(yyyyMMddHHmm)] class overriding Up(), bulk backfill via PatchByQueryOperation, purging via DeleteByQueryOperation. PROACTIVE: invoke when the user seeds/backfills/normalizes/purges stored documents, needs a one-time data fix, moves a namespace (Raven-Clr-Type), or mentions Raven.Migrations/[Migration]/MigrationRunner/PatchByQueryOperation. Not for per-record defaults (see /vidyano:lifecycle_hooks) or recurring sweeps."
user-invocable: true
---

# RavenDB Data Migrations

Schema lives in code, but *data* drifts. A new non-nullable field needs a value on every existing document; a renamed collection leaves stale `Raven-Clr-Type` metadata behind; a removed feature leaves orphaned `lookupReferences/*` documents; a default catalog must exist on a fresh database. **Raven.Migrations** handles all of these: timestamped, one-shot `Migration` classes that run at startup, each exactly once per database, recorded so they never run again.

This is the data counterpart to the code-first model — the document-store analogue of EF schema migrations, except each migration is plain C# against a RavenDB session or a server-side query.

Not for: defaults applied as each record is created (`OnNew` in `/vidyano:lifecycle_hooks`), or recurring sweeps — those belong in a repeating background job, not a run-once migration.

## When to use

The user seeds/backfills/normalizes/purges stored documents, needs a one-time data fix that must complete before the app serves requests, renames a collection or moves a namespace (`Raven-Clr-Type`), or mentions `Raven.Migrations`/`[Migration]`/`MigrationRunner`/`AddRavenDbMigrations`/`PatchByQueryOperation`/`DeleteByQueryOperation`.

A migration is for a **one-time, idempotent transform of stored data**. Typical jobs:

- **Seed** default lookup-reference values, a singleton config document, or a catalog on a fresh database.
- **Backfill** a newly-added field onto every existing document.
- **Normalize / reshape** — split a legacy `"Key=Value"` string into child objects, recompute a derived total, rewrite `Raven-Clr-Type` after a namespace move.
- **Purge** documents for a removed feature or stale lookups.

It is the wrong tool for two adjacent jobs: recurring work belongs in a repeating background job (it runs on a clock, a migration runs once and is recorded done); per-record defaults applied as records are created belong in `OnNew` (`/vidyano:lifecycle_hooks`).

## Wiring the runner

Two lines. Register the services in `ConfigureServices`, then run the pending migrations in `Configure` **after** `UseVidyano` (so the store is ready):

```csharp
public void ConfigureServices(IServiceCollection services)
{
    // ... your IDocumentStore registration ...
    services.AddRavenDbMigrations();   // discovers [Migration] classes in this assembly
    // ...
}

public void Configure(IApplicationBuilder app)
{
    app.UseVidyano(Environment, Configuration);

    app.ApplicationServices.GetRequiredService<MigrationRunner>().Run();
    // ...
}
```

`AddRavenDbMigrations` resolves the `IDocumentStore` you already register for the app (in CronosCore repos that registration is the `AddCronosRavenDB()` helper). `Run()` finds every `[Migration]` class in the assembly, skips the ones already recorded in the database, and runs the rest in timestamp order. By convention the classes live under `Service/Migrations`.

## A migration class

A migration derives from `Raven.Migrations.Migration`, is tagged with a numeric `[Migration(...)]` version, and overrides `Up()`. The version is a sortable `yyyyMMddHHmm` timestamp — it both orders the migrations and uniquely identifies each one:

```csharp
using Raven.Migrations;

namespace Acme.Service.Migrations;

[Migration(202406091430)]
public class M_202406091430_SeedOrderStatuses : Migration
{
    public override void Up()
    {
        using var session = DocumentStore.OpenSession();
        // ... transform documents ...
        session.SaveChanges();
    }
}
```

`DocumentStore` is available on the base class — no injection. Name the file and class with the version and a short description (`M_202406091430_SeedOrderStatuses`) so the migration list reads like a changelog.

## Per-document work: seed and cleanup via a session

The most common shape opens a session, stores or deletes documents, and saves. **Guard seeds with an existence check** so re-running against a database that already has the data is a no-op — migrations should be safe to reason about even though the runner won't repeat them:

```csharp
[Migration(202406091430)]
public class M_202406091430_SeedOrderStatuses : Migration
{
    public override void Up()
    {
        using var session = DocumentStore.OpenSession();

        // Drop a superseded lookup document.
        if (session.Advanced.Exists("lookupReferences/OrderStatus"))
            session.Delete("lookupReferences/OrderStatus");

        // Seed the default catalog, skipping anything already present.
        foreach (var status in new[] { "Draft", "Submitted", "Approved", "Rejected" })
        {
            var id = OrderStatus.GetId(status);
            if (!session.Advanced.Exists(id))
                session.Store(new OrderStatus { Id = id, Code = status });
        }

        session.SaveChanges();
    }
}
```

Per-document iteration is the right tool when the transform is non-trivial — recomputing a hash, reprocessing a stored file, or reshaping nested objects in C#. For a simple field set across a whole collection, the server-side path below is far cheaper.

## Bulk backfill and reshape via patch-by-query

To touch many documents without pulling them into the process, send a `PatchByQueryOperation` — an RQL query with a JavaScript `update` block that runs on the server. This is the idiomatic way to backfill a new field or rewrite a value across an entire collection:

```csharp
using Raven.Client.Documents.Operations;
using Raven.Client.Documents.Queries;
using Raven.Migrations;

[Migration(202406091500)]
public class M_202406091500_BackfillOrderRegion : Migration
{
    public override void Up()
    {
        DocumentStore
            .Operations
            .Send(new PatchByQueryOperation(new IndexQuery
            {
                Query = """
                        from Orders as o
                        where not exists(o.Region)
                        update
                        {
                            o.Region = "Unassigned";
                        }
                        """,
            }))
            .WaitForCompletion();
    }
}
```

Two details matter here:

- **`where not exists(o.Region)`** — a newly-added field is *absent*, not null, so `not exists()` is the precise (and cheaper) predicate. Don't reach for `== null`.
- **`WaitForCompletion()`** blocks until the server finishes patching, so the migration isn't marked done until the data is actually transformed.

The `update` block is real JavaScript and can do non-trivial work — iterate a nested collection, derive one field from another:

```csharp
Query = """
        from Orders as o
        update
        {
            o.Lines.forEach(line => { line.Total = line.Quantity * line.UnitPrice; });
        }
        """,
```

Interpolate runtime values into the query string when you need them (an environment name, a cutoff date), but treat the query as server code — keep it simple and side-effect-free beyond the patch itself.

## Purging documents

Mirror the patch path with `DeleteByQueryOperation` (an RQL filter, no `update` block) to remove a dropped feature's documents server-side, or use `session.Delete` for a handful of known ids:

```csharp
// Many documents matching a filter — server-side.
DocumentStore
    .Operations
    .Send(new DeleteByQueryOperation(new IndexQuery
    {
        Query = "from LegacyAuditEntries where Archived == true",
    }))
    .WaitForCompletion();

// A few known documents — via a session.
using var session = DocumentStore.OpenSession();
session.Delete("lookupReferences/Statuses");
session.Delete("lookupReferences/PaymentStatuses");
session.SaveChanges();
```

## Making it reversible

Migrations are forward-only by default — the runner never calls anything to undo them. When one is risky enough to want a documented rollback, override `Down()`; it runs only on an explicit rollback, never during normal startup:

```csharp
[Migration(202406091500)]
public class M_202406091500_BackfillOrderRegion : Migration
{
    public override void Up() { /* set Region = "Unassigned" */ }

    public override void Down()
    {
        DocumentStore
            .Operations
            .Send(new PatchByQueryOperation(new IndexQuery
            {
                Query = """
                        from Orders as o
                        where o.Region == "Unassigned"
                        update
                        {
                            o.Region = null;
                        }
                        """,
            }))
            .WaitForCompletion();
    }
}
```

**`Down()` must also use `PatchByQueryOperation`, never a `session.Query<T>().Where(...)` loop** — the default 128-document page limit silently truncates such a loop, so a session-query rollback would leave most documents un-reverted with no error. Most migrations don't need a `Down()` at all; add it only where a mistake would be expensive to recover from by hand.

## Checklist

- [ ] Wired: `services.AddRavenDbMigrations()` in `ConfigureServices`; `MigrationRunner.Run()` in `Configure` **after** `UseVidyano`.
- [ ] Each migration is `[Migration(yyyyMMddHHmm)] class M_<timestamp>_<Desc> : Migration` under `Service/Migrations`, overriding `Up()`; `DocumentStore` taken from the base class.
- [ ] Per-document work (seed, cleanup, heavy reshape) → `DocumentStore.OpenSession()` + `Store`/`Delete` + one `SaveChanges()`; seeds guarded with `session.Advanced.Exists(id)`.
- [ ] Bulk field work (backfill, normalize, recompute) → `PatchByQueryOperation` with an RQL `update {}` block + `.WaitForCompletion()`; backfill filter uses `where not exists(field)`.
- [ ] Purge → `DeleteByQueryOperation` (+ `.WaitForCompletion()`) or `session.Delete(id)` for known ids.
- [ ] `Down()` only for risky migrations, and it too uses `PatchByQueryOperation` — never a session-query loop.

## Verify

1. Build and start the app against a copy of production-shaped data; the startup log shows the migration running once, and the app serves requests only after it finished.
2. In RavenDB Studio, the `MigrationRecords` collection (Raven.Migrations' bookkeeping) holds a document for the new version.
3. Spot-check the data: a backfilled document now carries the field (`from Orders where exists(Region)` matches everything); a purged collection is empty; a seeded document exists exactly once.
4. Restart the app: the migration does not run again and the log is silent about it.
5. Negative case: for a `PatchByQueryOperation`, check the operation count in Studio matches the expected document count — a filter that matched nothing means the predicate (often `not exists` vs `== null`) is wrong.

## Related skills

- `/vidyano:lifecycle_hooks` — `OnNew` for per-record creation defaults; the right place for the work a migration must *not* do.
