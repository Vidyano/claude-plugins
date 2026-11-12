---
name: detail_queries
description: "Show only the opened record's children in a detail tab: a public method taking CustomQueryArgs on the child's PersistentObjectActions, resolved from Source: \"Custom.<Parent>_<Child>\"; guard with args.EnsureParent(Types.X) or return Empty<T>.Queryable. PROACTIVE: invoke when the user wants a detail tab/sub-grid listing one parent's children, filters a query by the opened record, builds a master-detail screen or parent-scoped picker, or mentions CustomQueryArgs/EnsureParent/args.Parent/Custom.<source>. Not for built-in attach/detach buttons (see /vidyano:relationship_editors)."
user-invocable: true
---

# Parent-Scoped Detail Queries (`CustomQueryArgs`)

Open a `Company` and its detail tab should list **only that company's** orders — not every order in the database. The same is true everywhere a screen nests one entity under another: a person's addresses, a policy's documents, a company's subsidiaries. The grid in that tab is not the entity's default query; it is a **custom query source** that knows which parent it is rendered under and filters accordingly.

Vidyano hands the parent to the query through `CustomQueryArgs.Parent`. This is the most common read pattern after the query-model grid itself, and the two compose: a detail tab is usually a query model filtered by the parent's id.

The default `Database.<Entity>` query returns the whole collection. A detail tab needs the subset belonging to the record the user opened. Pushing the filter into the client ships every row to the browser and breaks paging, sorting, and security — the filter has to happen server-side, and the server has to know which parent the tab hangs under. `CustomQueryArgs` carries exactly that.

Not for: the built-in attach/detach buttons that need no custom query (`/vidyano:relationship_editors`), or the denormalized `V*` projection itself (`/vidyano:query_models`).

## When to use

The user wants a detail tab/sub-grid to list only one parent's children, filter a query by the opened record, build a master-detail screen or a parent-scoped picker, or mentions `CustomQueryArgs`/`EnsureParent`/`args.Parent`/`Custom.<source>`/`Empty<T>.Queryable`/a detail query.

## The mechanism

Three pieces wire a parent-scoped detail query together:

```
   Company (parent PO)                Company_Orders                 VOrder / Order
   ──────────────────                 ──────────────                 ──────────────
   detail tab bound to       ──▶      public IQueryable      ──▶      .Where(o =>
   query "Company_Orders"             (CustomQueryArgs args)            o.Company ==
                                      reads args.Parent.ObjectId        args.Parent.ObjectId)
```

1. A **query method** taking `CustomQueryArgs`, returning an `IQueryable` / `IRavenQueryable` / `IEnumerable` of the child type.
2. A **query in the model** whose `Source` is `Custom.<MethodName>`, surfaced as a detail tab on the parent.
3. The framework **sets `args.Parent`** to the open parent PO when it renders that detail query — you never thread the parent through.

`CustomQueryArgs` (namespace `Vidyano.Service.Repository`) carries `Parent` (a `PersistentObject?`), plus `Query`, `Reason` (`ExecuteQueryReason`), `Attribute` (`PersistentObjectAttribute?`, set when the query backs a `SelectInPlace` / `AsDetail.Objects` / `KeyValueList`), and `CustomizeColumns`.

## The query method

The method is a plain public method hosted on the **child** entity's `PersistentObjectActions` class — `OrderActions` for `Order`, the same class that holds its `OnNew`/`OnLoad` hooks — not on the parent's (`CompanyActions`), and not a `CustomAction`. It can also live on the context. Vidyano resolves `Custom.<MethodName>` to the method of that name by reflection, so the name in the model and the method name must match exactly.

```csharp
// The child's PersistentObjectActions (the source generator supplies the base class).
public partial class OrderActions
{
    // Source "Custom.Company_Orders" on the Order PO resolves here.
    public IQueryable<Order> Company_Orders(CustomQueryArgs args) =>
        Context.Orders.Where(o => o.Company == args.Parent!.ObjectId);
}
```

The naming convention is `<Parent>_<Child>` (`Company_Orders`, `Person_Addresses`, `InsurancePolicy_Documents`). It is only a convention — the framework matches on the model's `Source` string, not the name — but following it keeps the wiring obvious.

## Guarding the parent

`args.Parent` is null when the query runs **without** a parent (opened as a standalone menu item, or during a tooling sync). Every parent-scoped query runs parentless at least once. A query that dereferences `args.Parent.ObjectId` unguarded will throw there. There are two idioms, and the choice is about whether running parentless is an *error* or just an *empty result*.

### `EnsureParent` — the strict guard

`args.EnsureParent(Types.Company)` asserts the parent is present and of the expected type, throwing `FaultException("DEV: Expected parent of type …")` otherwise. It carries `[MemberNotNull(nameof(Parent))]`, so after it `args.Parent` is non-null — no `!` needed:

```csharp
public IQueryable<Order> Company_Orders(CustomQueryArgs args)
{
    args.EnsureParent(Types.Company);

    return Context.Orders.Where(o => o.Company == args.Parent.ObjectId);
}
```

It accepts several acceptable parent types — useful when the same detail query hangs under either the editable aggregate or its read-only query model:

```csharp
public IQueryable<VOrder> Person_Orders(CustomQueryArgs args)
{
    args.EnsureParent(Types.Person, Types.VPerson);

    return VOrders.Where(o => o.Person == args.Parent.ObjectId);
}
```

(Always pass at least one type. The arg-less overload is `[Obsolete(…, error: true)]` and throws.)

### The soft guard with an empty fallback

When a missing or wrong-typed parent should yield *no rows* rather than an error, test the type and return an empty queryable:

```csharp
public IQueryable<Order> Company_Orders(CustomQueryArgs args)
{
    if (args.Parent?.Type != Types.Company)
        return Empty<Order>.Queryable;

    return Context.Orders.Where(o => o.Company == args.Parent.ObjectId);
}
```

`Empty<T>.Queryable` (`Vidyano.Service.Repository`) is a cached `Array.Empty<T>().AsQueryable()` — the canonical "no rows" return. It is also how you declare a deliberate placeholder query that a screen fills in later:

```csharp
public IQueryable<Order> Placeholder(CustomQueryArgs e) => Empty<Order>.Queryable;
```

## What you can filter

The parent's id is just a string. What you filter with it depends on where the children live.

### A query model (`V*`) projection

The most common target: a denormalized query model, so the detail grid gets the same flat, searchable, sortable columns as a top-level grid.

```csharp
public IRavenQueryable<VOrder> Company_Orders(CustomQueryArgs args)
{
    args.EnsureParent(Types.Company);

    return Query<VOrder, Orders_Overview>()
        .Where(o => o.Company == args.Parent.ObjectId)
        .AsNoTracking();
}
```

### A sibling collection

When the child is its own document holding the parent's id, filter the context collection directly:

```csharp
public IQueryable<Order> Company_Orders(CustomQueryArgs args) =>
    Context.Orders.Where(o => o.Company == args.Parent!.ObjectId);
```

### An embedded child collection

When the children are embedded value-object children on the parent document (not separate documents), there is nothing to query — load the parent and return its in-memory collection. Pair it with an empty fallback so a missing parent yields no rows instead of a null-reference:

```csharp
public IQueryable<Address> Person_Addresses(CustomQueryArgs args)
{
    args.EnsureParent(Types.Person);

    return Session.Load<Person>(args.Parent.ObjectId)?.Addresses.AsQueryable()
        ?? Empty<Address>.Queryable;
}
```

The same shape works for a collection stored in a split side-document: load the side-document by its derived id and return its collection.

## Wiring it into the model

The detail query is declared on the **child** entity's persistent object as a second query whose `Source` points at your method. The default `Database.*` query stays; the custom one is added alongside it:

```jsonc
"Queries": [
  {
    "Source": "Database.Orders",        // the full collection
    "Name": "Orders",
    "AutoQuery": true
  },
  {
    "Source": "Custom.Company_Orders",   // the parent-scoped subset
    "Name": "Company_Orders",
    "AutoQuery": true
  }
]
```

The parent (`Company`) surfaces `Company_Orders` as a detail tab — it references the child's query by `$ref`. Because it is rendered under an open `Company`, the framework sets `args.Parent` to that company before calling the method — you never wire the parent through by hand.

The query is defined on the **child** PO, but the source generator lists a parent's detail-query names under the **parent's** constant. That is why the New-gating check below reads `QueryNames.Company.Company_Orders`, not `QueryNames.Order.…`, even though the query lives on `Order`.

## Reading more than the parent's id

`args.Parent` is the live parent persistent object, not just an id — you can read its other attributes to shape the query. This is how a detail query filters on a parent field (here: subsidiaries in the same country as the parent), or short-circuits for a brand-new parent that has no children yet:

```csharp
public IQueryable<VCompany> Company_Subsidiaries(CustomQueryArgs args)
{
    args.EnsureParent(Types.Company);

    if (args.Parent.IsNew)                       // unsaved parent → nothing to show
        return Empty<VCompany>.Queryable;

    var country = (string?)args.Parent[AttributeNames.Company.Country];

    return VCompanies.Where(c => c.ParentCompany == args.Parent.ObjectId && c.Country == country);
}
```

## Gating actions on a detail query

A child opened from its full `Database.*` grid and a child opened from a parent's detail tab are the same entity, but the available actions often differ — you usually only want **New** on the parent-scoped tab, since a child created anywhere else has no parent to attach to. Branch on `query.Name` in `OnConstruct`:

```csharp
public override void OnConstruct(Query query, PersistentObject? parent)
{
    base.OnConstruct(query, parent);

    if (query.Name is not QueryNames.Company.Company_Orders)
        query.DisableActions(QueryAction.New);   // New only from the company's tab
}
```

You do **not** seed the parent yourself. When a child is created from a parent's detail tab and the child has a reference attribute to the parent's type, `base.OnNew` sets that reference to the parent automatically and marks it read-only — the new `Order` already points at the `Company` it was opened under. Override `OnNew` only to seed *other* defaults derived from the parent (a start date, a copied setting), with the `obj[attr].SetValue(...)` idiom:

```csharp
public override void OnNew(PersistentObject obj, PersistentObject? parent, Query? query, Dictionary<string, string>? parameters)
{
    base.OnNew(obj, parent, query, parameters);   // parent reference already set + locked here

    obj[AttributeNames.Order.OrderDate].SetValue(Manager.Current.Today);
}
```

## Picker variant: a detail query inside a wizard

A select-from / add-reference picker is the same mechanism with one extra hop. When the query is rendered inside a transient wizard PO, the real record is the *grandparent* (`args.Parent.Parent`). Read the already-linked ids off the grandparent's detail attribute and exclude them, so the picker only offers what is not yet attached:

```csharp
public IQueryable<VSupplier> SupplierSelection_Suppliers(CustomQueryArgs args)
{
    if (args.Parent?.Parent?.Type != Types.NewOrderSupplier)
        return Empty<VSupplier>.Queryable;

    var alreadyLinked = ((PersistentObjectAttributeAsDetail)args.Parent.Parent[AttributeNames.NewOrderSupplier.Suppliers])
        .Objects
        .Select(o => ((PersistentObjectAttributeWithReference)o[AttributeNames.SupplierSelection.Supplier]).ObjectId)
        .ToArray();

    return VSuppliers.Where(s => !s.Id.In(alreadyLinked));
}
```

For the standard attach/detach buttons that do not need a custom picker query, prefer the built-in `AddReference`/`Remove` actions in `/vidyano:relationship_editors`.

## Guidelines, tips, pitfalls

- **Always guard the parent.** Every parent-scoped query runs parentless at least once (tooling sync, standalone menu). Use `EnsureParent` when that is an error, the soft type-check + `Empty<T>.Queryable` when it should be empty.
- **Match the method name to the model `Source`.** `Custom.Company_Orders` resolves to a method literally named `Company_Orders`. A rename on one side without the other silently breaks the query.
- **Filter the query model, not the aggregate.** A detail grid wants denormalized columns; point it at the `V*` projection and `.AsNoTracking()`, exactly like a top-level grid.
- **Embedded children are loaded, not queried.** `Session.Load<Parent>(...)?.Collection.AsQueryable()` — there is no separate document to `Where` over, and the `?? Empty<T>.Queryable` fallback keeps a missing parent from throwing.
- **Don't seed the parent reference yourself.** `base.OnNew` sets and locks the parent reference automatically when the child is created from the parent's detail tab — override `OnNew` only for *other* parent-derived defaults.
- **`args.Parent` is the live PO.** You can read any of its attributes (`args.Parent[AttributeNames...]`), check `IsNew`, or walk to `args.Parent.Parent` inside a wizard — not only `ObjectId`.

## Checklist

- [ ] Query method is `public`, takes `(CustomQueryArgs args)`, returns `IQueryable`/`IRavenQueryable`/`IEnumerable` of the child, and lives on the child's `*Actions` class or the context.
- [ ] Method name matches the model `Source` exactly (`Custom.<Method>`); convention `<Parent>_<Child>`.
- [ ] Parent guarded: `args.EnsureParent(Types.X)` when parentless is an error, or `if (args.Parent?.Type != Types.X) return Empty<T>.Queryable;` when it should mean no rows.
- [ ] Filter target chosen: `V*` projection (`Query<V, Index>().Where(...).AsNoTracking()`), sibling collection, or embedded collection via `Session.Load(...)?.Collection.AsQueryable() ?? Empty<T>.Queryable`.
- [ ] Declared in model.json as a second query (`Source: "Custom.<Method>"`) alongside the default `Database.*`; surfaced as a detail tab on the parent.
- [ ] `New` gated to the parent's tab in `OnConstruct`; parent reference NOT seeded in `OnNew` (the base sets and locks it) — override `OnNew` only for other parent-derived defaults.
- [ ] Picker variant: guard `args.Parent?.Parent?.Type`, read already-linked ids off the grandparent's `PersistentObjectAttributeAsDetail.Objects`, exclude them.

## Verify

1. Build, run a schema synchronize (in CronosCore repos, `/dcg:synchronize`), and confirm the new `Custom.<Method>` query is in the model and the parent PO shows it as a detail tab.
2. Open two different parents: each tab lists only that parent's children (cross-check the count against `from Orders where Company = '<id>'` in RavenDB Studio).
3. Click New on the tab: the parent reference is pre-filled and read-only; open the child's full grid and New is absent there.
4. Negative case: open the custom query standalone (or with the wrong parent type) — `EnsureParent` shows the `DEV: Expected parent of type …` fault, or the soft guard shows an empty grid, whichever you chose.
5. Rename the method without touching the model `Source` and the tab errors at load — rename both sides together.

## Related skills

- `/vidyano:query_models` — the denormalized `V*` projection this query filters, and the click-through reroute from the read-only row to the editable aggregate.
- `/vidyano:lifecycle_hooks` — `OnConstruct` (New-gating), `OnNew` (parent auto-set + other defaults), `QueryExecuted` (grid shaping).
- `/vidyano:wizards` — the transient-PO shell for the picker variant.
- `/vidyano:relationship_editors` — the built-in `AddReference`/`Remove` actions: attach/detach without a custom query.
