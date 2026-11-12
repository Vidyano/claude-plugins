---
name: row_security
description: "Filter which rows a user can see in Vidyano, the framework-general way: append a predicate in the Where(Source<T>, Query?) override, and repeat the check in GetEntity (a user can guess an ObjectId) and in ApplyDataSecurity for value objects; plus owner-scoped/self-service filtering. In CronosCore repos, the dcg plugin's `data_security` skill adds company-hierarchy scoping and `IWithDataSecurity` on top of this. PROACTIVE: invoke when the user restricts which rows/records a user can see, overrides Where/GetEntity for filtering, or implements owner/self-service visibility."
user-invocable: true
---

# Row-Level Data Security (general mechanism)

Vidyano's group/right model decides *which entities and actions* a user may touch. It does **not** decide *which rows* a user may see. Row-level security scopes the result of every query — and every direct load — to the subset a user is entitled to. This skill covers the framework-general mechanism; in CronosCore repos, the dcg plugin's `data_security` skill adds company-hierarchy scoping and `IWithDataSecurity` on top of it.

Not for: which *entities and actions* a user may touch at all (the group/right model), the order the hooks fire in (`/vidyano:lifecycle_hooks`), or a per-action record guard (`/vidyano:custom_actions`).

## When to use

The user wants to restrict which records appear in a grid, scope an entity to its owner ("my own rows"), or stop a user loading a record by guessing its id.

## The three read-path hooks

Every entity is served through a `PersistentObjectActions<TContext, TEntity, TQueryEntity>` class. Three of its virtuals — all declared by Vidyano.RavenDB (the NuGet drops a `PersistentObjectActionsReference.cs` in your `obj/` folder listing every override with its default body) — carry row security:

- **`Where(Source<TQueryEntity> source, Query? query)`** — filters the grid/list query. `Source<T>` wraps the RavenDB queryable; `source.Where(predicate)` adds a **server-side** filter (runs in RavenDB, not in memory). This is where the bulk of filtering happens.
- **`ApplyDataSecurity(TEntity entity, PersistentObject? obj, Query? query)`** — the per-row check on a loaded entity; return `null` to deny. Its default re-applies your `Where` predicate to that single entity (when the entity and its query type are the same class), and the framework also calls it for value-object rows that `Where` never sees.
- **`GetEntity(PersistentObject obj)`** — resolves a single record (opening a row, following a reference). The default loads the document and passes it through `ApplyDataSecurity`, so a guessed `ObjectId` comes back `null`.

**The golden rule: never rely on the grid filter alone.** A user who guesses an `ObjectId` can request it directly. With the defaults above, a `Where` override on an entity that lists as itself already covers that path; when the entity lists through a separate `V*` query model, the default cannot reuse a predicate written against another shape — override `ApplyDataSecurity` yourself. For the order these hooks fire in, see `/vidyano:lifecycle_hooks`.

## The recipe: compute the allowed set once, inject a predicate

Whatever the scope, the shape is identical — resolve the current principal, compute their allowed key set, append a predicate to the read path, and keep the bypass in one place:

```csharp
public partial class OrderActions
{
    // One central bypass — a role check, never a per-entity special case.
    private static bool IsPrivileged() => Manager.Current.User!.IsMemberOf(AppRoles.Administrators);

    // The allowed set, resolved from the current principal. Cache it per request if it is expensive.
    private string[] AllowedCompanyIds() =>
        Context.Memberships.Where(m => m.User == Manager.Current.User!.Id).Select(m => m.Company).ToArray();

    // 1. The grid. source.Where(...) runs inside RavenDB; In(...) becomes a server-side IN clause.
    protected override Source<Order> Where(Source<Order> source, Query? query = null)
    {
        if (IsPrivileged())
            return source;

        var allowed = AllowedCompanyIds();
        return source.Where(it => it.Company.In(allowed));
    }

    // 2. A direct load — a guessed ObjectId, a followed reference. The base already runs
    //    ApplyDataSecurity; override only to make the denial loud instead of a silent null.
    public override Order? GetEntity(PersistentObject obj)
        => base.GetEntity(obj) ?? throw new UnableToLoadEntityException(obj);

    // 3. The per-row check itself — also what the framework calls for rows Where never sees
    //    (value objects inside a parent document). Return null to deny.
    public override Order? ApplyDataSecurity(Order entity, PersistentObject? obj = null, Query? query = null)
        => IsPrivileged() || AllowedCompanyIds().Contains(entity.Company) ? entity : null;
}
```

For an entity that *is* the scoping key rather than *referencing* it, the predicate becomes `it.Id.In(allowed)`.

## Owner-scoped / self-service filtering

When an entity belongs to a single user — timesheets, personal tickets, a self-service portal — the allowed set collapses to one key and the recipe is unchanged: the predicate is `it.Owner == me`, where `me` is whatever the owner field stores (the user id, the login, a mapped person id, or the audit `CreatedBy`), and the bypass is a management/auditor role. Compare the owner *field* rather than the `ObjectId` — document ids carry a `collection/` prefix, so an `ObjectId.StartsWith(me)` check is fragile and only happens to work when the id embeds the owner key. This is the most common place an Insecure-Direct-Object-Reference slips in, so keep step 2 (throw, don't return `null`) on a denied direct load.

## Per-row read-only and delete gating

Visibility is only half the story — a user may see a row but not edit or delete it. Gate these in the lifecycle hooks, not in `Where`, reusing the same bypass:

```csharp
public override void OnLoad(PersistentObject obj, PersistentObject? parent)
{
    base.OnLoad(obj, parent);
    if (!IsPrivileged())
        obj.IsReadOnly = true;               // visible, not editable
}

protected override void MarkForDeletion(Order entity)
{
    if (!IsPrivileged())
        throw new SaveFailedException();      // block delete server-side, wherever the delete came from
    base.MarkForDeletion(entity);
}
```

## Push the join into RavenDB

When the allowed set is large or shared across many rows, do not load-then-filter in memory. Either store an allow-list on each indexed row (predicate `row.AllowFor.Any(x => x.In(mine))`) or build a reduce index of allowed keys per user and join against it. The join that answers "may this user see this row?" should happen **inside RavenDB**. Project the security fields when you build the `*_Overview` index.

## Three rules for every approach

1. **Filter every read path** — `Where`, `GetEntity`, and any direct `OnLoad`. The grid filter alone is not a security boundary.
2. **Put the bypass decision in one place** — a single role guard, never scattered per entity.
3. **Push the join into RavenDB** — project security fields onto the index and filter there.

Row-level security is the layer most likely to fail silently. Pair it with the group/right model (which gates entities/actions) for defense in depth.

## Checklist

- [ ] `Where` appends a server-side predicate (`source.Where(...)`, `In(...)`).
- [ ] Direct loads covered: the default `GetEntity` → `ApplyDataSecurity` chain when the entity lists as itself, your own `ApplyDataSecurity` when it lists through a `V*`; throw, don't return null.
- [ ] Value objects checked in `ApplyDataSecurity`.
- [ ] Edit/delete gated in `OnLoad`/`MarkForDeletion(TEntity)` if needed.
- [ ] One central bypass guard; security fields projected onto the index for server-side filtering.

## Verify

1. Build; sign in as a scoped user: the grid shows only rows whose scoping field is in that user's allowed set, while RavenDB Studio shows rows outside it exist.
2. Copy the URL of an out-of-scope record as an administrator and open it as the scoped user: the unable-to-load error appears, not the record.
3. Sign in as the bypass role: every row lists and every record opens.
4. Value objects: open a parent whose embedded rows span scopes; only the permitted rows render in the detail grid.
5. With gating on: an in-scope, non-privileged user sees the record read-only, and a delete attempt fails server-side with the document still in Studio.

## Related skills

- `/vidyano:lifecycle_hooks` — the order and firing of `Where`/`GetEntity`/`OnLoad`/`MarkForDeletion`.
- `/vidyano:custom_actions` — gating *which records* an action may touch.
- In CronosCore repos, the dcg plugin's `data_security` skill adds company-hierarchy scoping and `IWithDataSecurity` on top of this mechanism.
