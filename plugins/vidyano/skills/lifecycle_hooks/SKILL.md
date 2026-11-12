---
name: lifecycle_hooks
description: "Customize an entity's create/load/validate/save/delete behavior via its PersistentObjectActions hooks. Covers OnConstruct, OnNew (defaults), OnLoad/OnPreviewLoad, the save pipeline (OnSave, PrePersistChanges, CheckRules), OnDelete (guarded/cascading), QueryExecuted grid shaping, the shared default-actions base, and value-object child persistence with parent recompute. PROACTIVE: invoke when the user overrides or adds a PersistentObjectActions class or any On* hook, sets defaults on new records, adds save/delete validation or side effects, or shapes a grid in QueryExecuted."
user-invocable: true
---

# Entity Lifecycle Hooks

Every editable entity has a server-side `PersistentObjectActions` class that customizes how it is created, loaded, validated, persisted, and deleted. Overriding its hooks is the most common way to add behavior to a module.

Not for: a toolbar button or bulk verb (`/vidyano:custom_actions`), declarative validation rules (`/vidyano:business_rules`), which rows a user may see (`/vidyano:row_security`), or the read-only `V*` grid behind the aggregate (`/vidyano:query_models`).

## When to use

The user wants to seed defaults on a new record, eager-load a referenced document, add cross-field validation, run a side effect on save, guard or cascade a delete, or color/shape a grid.

## The Actions class

For entity `Company`, the source generator emits `partial class CompanyActions : PersistentObjectActions<TContext, TEntity, TQueryEntity>` (your context, the entity, the query/list projection). You add a second `partial` file and override only the hooks the entity needs; registration is by naming convention (`<Entity>Actions`). With no query projection use the two-arg form `PersistentObjectActions<TContext, TEntity>`. Inject dependencies through the constructor.

## Hook order

| Phase | Hooks (in order) |
| --- | --- |
| Construct | `OnConstruct(PersistentObject)` / `OnConstruct(Query, parent)` |
| Create | `OnNew` |
| Open existing | `OnPreviewLoad` → `GetEntity` / `LoadDefaultIncludes` → `OnLoad` |
| Edit a `TriggersRefresh` field | `OnRefresh` |
| Before the PO goes to the client | `PreClient` — after `OnNew`/`OnLoad`/`OnRefresh`; last chance to toggle rules, visibility, read-only |
| Save | `OnSave` → `CheckRules` → `SaveNew`/`SaveExisting` → `UpdateEntity` → `AddToContext` → `PrePersistChanges` → `PersistToContext` → `PersistChanges` |
| Reference pickers | `OnSelectReference` (a picked row on a reference attribute) · `OnAddReference` / `OnRemoveReference` (built-in Add/Remove on a detail query, see `/vidyano:relationship_editors`) |
| Delete | `OnDelete` (all selected entities) → `MarkForDeletion(entity)` per entity → `CleanupRelatedBeforePersistChanges` → `PersistChanges` |
| List query | `Where` → `QueryExecuted` |

Always call `base.<Hook>(...)` (usually first; for `OnDelete` last, after your checks pass).

## OnConstruct — shape independent of a record's values

Bind option sources, disable actions, hide attributes by role:

```csharp
public override void OnConstruct(PersistentObject obj)
{
    base.OnConstruct(obj);
    obj[AttributeNames.Contract.Status].UseValueSource(Context.ContractStatuses);
}

public override void OnConstruct(Query query, PersistentObject? parent)
{
    base.OnConstruct(query, parent);
    query.DisableActions(QueryAction.New, QueryAction.Delete);   // system-managed list
}
```

## OnNew — seed defaults

`SetValue` for a user-visible default; `SetOriginalValue` for one that should not register as a pending change.

```csharp
public override void OnNew(PersistentObject obj, PersistentObject? parent, Query? query, Dictionary<string, string>? parameters)
{
    base.OnNew(obj, parent, query, parameters);
    obj[AttributeNames.Contract.StartDate].SetValue(Manager.Current.Today);
    obj[AttributeNames.Contract.Status].SetValue(nameof(ContractStatus.Draft));
    obj[AttributeNames.Contract.Owner].SetValue(Manager.Current.User!.Id);
}
```

Mint a **server-generated secret** here rather than letting a user pick one (secure-by-default), and keep the `NotEmpty` rule on the field so an empty value stays impossible even if the hook is bypassed:

```csharp
obj[nameof(Company.ApiKey)].SetValue(ObjectEx.GetSecureRandomString());
```

## OnLoad / OnPreviewLoad

`OnLoad` fires after the entity is fetched — fill display fields, lock fields, add notifications:

```csharp
public override void OnLoad(PersistentObject obj, PersistentObject? parent)
{
    base.OnLoad(obj, parent);
    obj[AttributeNames.Company.FullName].Visibility = AttributeVisibility.Query;
    obj[AttributeNames.Company.VatNumber].IsReadOnly = true;   // immutable natural key
}
```

Avoid N+1 on id-only references by eager-loading, then projecting in `OnLoad`:

```csharp
public override void LoadDefaultIncludes(IIncludeBuilder<Contract> builder)
{
    base.LoadDefaultIncludes(builder);
    builder.IncludeDocuments(x => x.Employee);
}
// ...in OnLoad: var employee = Context.Session.Load<Employee>(contract.Employee);
```

`GetEntity` can hydrate a transient collection so the populated entity is consistent everywhere. `OnPreviewLoad` fires before load with only the `ObjectId` known — use it to reroute (a "My Profile" menu to the user's record, or a synthetic composite-id row to its real entity):

```csharp
public override void OnPreviewLoad(PreviewLoadArgs e)
{
    if (string.IsNullOrWhiteSpace(e.ObjectId) || e.ObjectId is "Menu")
    {
        e.Reroute(Types.Person, Person.GetId(Manager.Current.User!.Id));
        return;
    }
    base.OnPreviewLoad(e);
}
```

## The save pipeline

Pick the hook that matches what you need to touch — the form (PO) or the populated entity.

- **`OnSave(obj)`** — runs first, before the entity is built. Audit stamping and side effects keyed on which attributes changed. Guard cross-document writes with `IsValueChanged` and call `Context.SaveChanges()` for the extra writes:

```csharp
public override void OnSave(PersistentObject obj)
{
    if (obj.IsNew) { obj[nameof(IAudit.CreatedOn)].SetValue(Manager.Current.NowOffset);
                     obj[nameof(IAudit.CreatedBy)].SetValue(Manager.Current.UserName); }
    else          { obj[nameof(IAudit.ModifiedOn)].SetValue(Manager.Current.NowOffset);
                     obj[nameof(IAudit.ModifiedBy)].SetValue(Manager.Current.UserName); }
    base.OnSave(obj);
}
```

- **`SaveNew` / `SaveExisting`** — split create/update. Override `SaveExisting` to capture what changed before the base persists it (useful for change-diffs); `SaveNew` for create-only side effects. Validation — including uniqueness — does not belong here; it goes in `CheckRules`.

- **`CheckRules`** — cross-field, uniqueness and parent-limit validation (per-attribute `Required`/`NotEmpty` belong in `model.json`). Report by setting `ValidationError`/an error notification (aborts the save) or throwing `FaultException`. Signature: `protected override void CheckRules(PersistentObject obj, Type entityType, TEntity? entity)`:

```csharp
protected override void CheckRules(PersistentObject obj, Type entityType, Contract? entity)
{
    base.CheckRules(obj, entityType, entity);

    var code = (string)obj[AttributeNames.Contract.Code]!;
    if (Context.Contracts.Any(c => c.Code == code && c.Id != obj.ObjectId))   // uniqueness
        obj[AttributeNames.Contract.Code].ValidationError = Messages.Contract_DuplicateCode;

    var start = (DateTime)obj[AttributeNames.Contract.StartDate]!;
    var end = (DateTime?)obj[AttributeNames.Contract.EndDate];
    if (end is not null && end < start)
        obj[AttributeNames.Contract.EndDate].ValidationError = Messages.Contract_EndBeforeStart;
}
```

- **`AddToContext`** — assign a deterministic/natural id before store (`entity.Id = Invoice.GetId(year, number)`).
- **`PrePersistChanges`** — recompute the entity's own derived fields (totals, sort keys); call `base` first.
- **`PersistToContext`** — wraps the store; guard the write or update an external counter.

## OnDelete — guard or cascade

`OnDelete` receives the selected entities and `QueryResultItem[]`; the base marks each entity for deletion and persists. Call `base.OnDelete(...)` last.

```csharp
// Guarded: refuse cleanly when dependents exist.
public override void OnDelete(PersistentObject? parent, IEnumerable<Company> entities, Query query, QueryResultItem[] selectedItems)
{
    var ids = selectedItems.Select(si => si.Id).ToArray();
    if (Context.Contracts.Any(c => c.Company.In(ids) && c.IsActive))
        throw new FaultException(Messages.CompanyActions_OnDelete_HasActiveContracts);
    base.OnDelete(parent, entities, query, selectedItems);
}
```

```csharp
// Cascading: delete owned children too. Batch-load with In(...) — one query, not one per project.
public override void OnDelete(PersistentObject? parent, IEnumerable<Project> entities, Query query, QueryResultItem[] selectedItems)
{
    var projectIds = entities.Select(p => p.Id).ToArray();
    Context.Tasks.Where(t => t.Project.In(projectIds))
           .Run(task => Context.MarkForDeletion(task));
    base.OnDelete(parent, entities, query, selectedItems);
}
```

For a single opened object the framework calls the protected `MarkForDeletion(TEntity entity)` per entity — override it to block a delete server-side (throw `SaveFailedException`) regardless of where the delete came from.

For a reversible delete, flip an `IsDeleted` flag, relabel the action "Archive", and filter the list query on `IsDeleted != true`.

## QueryExecuted — grid shaping

Fires after a list query; shape items before they reach the client. Call `base` first.

```csharp
public override void QueryExecuted(QueryExecutedArgs args)
{
    base.QueryExecuted(args);
    foreach (var item in args.Items)
    {
        var status = item.GetValue(AttributeNames.Invoice.Status).FromServiceString<InvoiceStatus>();
        item.DataTypeHints = "foreground=" + status switch
        {
            InvoiceStatus.Overdue => "var(--colors-red)",
            InvoiceStatus.Paid    => "var(--colors-green)",
            _                     => "var(--colors-grey)",
        };
    }
    if (args.Reason is ExecuteQueryReason.Excel)
        args.RemoveColumns(nameof(IWithUserId.UserId));   // strip a column from exports
}
```

## The shared DefaultPersistentObjectActions base

Rather than repeating cross-cutting behavior per entity, a module defines **one** generic base — `DefaultPersistentObjectActions<TContext, TEntity, TQueryEntity>` — that every source-generated `<Entity>Actions` inherits. Override the cross-cutting hooks once; each helper **no-ops when its marker interface is absent**, so an entity engages a concern just by implementing the interface (`IAudit` below):

```csharp
public partial class DefaultPersistentObjectActions<TContext, TEntity, TQueryEntity>
    : PersistentObjectActions<TContext, TEntity, TQueryEntity>
    where TContext : ITargetRavenDBContext
    where TEntity : class
    where TQueryEntity : class
{
    private static readonly bool isAudit = typeof(IAudit).IsAssignableFrom(typeof(TEntity));

    public DefaultPersistentObjectActions(TContext context) : base(context) { }

    public override void OnSave(PersistentObject obj)
    {
        if (isAudit)
        {
            var now = Manager.Current.NowOffset;
            var user = Manager.Current.UserName;
            if (obj.IsNew) { obj[nameof(IAudit.CreatedOn)].SetValue(now);  obj[nameof(IAudit.CreatedBy)].SetValue(user); }
            else           { obj[nameof(IAudit.ModifiedOn)].SetValue(now); obj[nameof(IAudit.ModifiedBy)].SetValue(user); }
        }
        base.OnSave(obj);
    }
}
```

Per-entity classes then stay thin. The same shape carries the other cross-cutting hooks — `Where`/`ApplyDataSecurity` for row scoping (`/vidyano:row_security`) and `AddToContext` for id assignment. In CronosCore repos the module base additionally wires company-hierarchy data security and natural ids (the dcg plugin's `data_security` and `natural_ids` skills).

## Value-object children

Nested collections (invoice lines) are value objects stored inside the root document, with their own `<Child>Actions` and a composite `{rootId};{collection};{childId}` `ObjectId`; every child edit or delete must recompute and persist the parent. The `OnRefresh`/`PrePersistChanges` recipe is in `${CLAUDE_PLUGIN_ROOT}/skills/lifecycle_hooks/references/value_objects.md`.

## Checklist

- [ ] Override only the hooks the entity needs; call `base.<Hook>` correctly (first, except `OnDelete` last).
- [ ] Per-attribute rules in `model.json`; cross-field validation in `CheckRules`.
- [ ] Cross-cutting concerns go in the shared base via marker interfaces, not per entity.
- [ ] Value-object edits/deletes recompute the parent and persist it.
- [ ] Side effects guarded by `IsValueChanged`/`IsNew` so they fire only when intended.

## Verify

1. Build; when a hook reads a new attribute, run a schema synchronize (in CronosCore repos, `/dcg:synchronize`) so `AttributeNames.*` resolve.
2. Click New: the `OnNew` defaults are pre-filled, and a `SetOriginalValue`d field does not mark the object dirty.
3. Save with a rule violation: the `CheckRules` message sits on the offending field and RavenDB Studio shows no new or changed document.
4. Save a valid object: the document in Studio carries the stamped and derived fields; the grid shows the `QueryExecuted` colouring.
5. Delete a guarded record: the `FaultException` text appears and the document is still in Studio; delete an unguarded one and it, plus its cascaded children, are gone.

## Related skills

- `/vidyano:custom_actions` — buttons and bulk operations; status-transition actions pair with these hooks.
- `/vidyano:exports_downloads` — `OnGetStream` for delivering a file.
- `/vidyano:file_storage` — storing an uploaded file as an attachment in `OnSave`, removing it in `OnDelete`.
- `/vidyano:imports` — `OnImport`, `SaveNew`, paste-a-list wizards.
- `/vidyano:row_security` — `Where`/`GetEntity` row-level scoping (the basis for the shared base's filtering).
