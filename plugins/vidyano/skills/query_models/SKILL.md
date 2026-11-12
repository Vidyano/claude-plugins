---
name: query_models
description: "Back a Vidyano grid with a `V<Entity>` query model over a RavenDB index — the read side of CQRS: `[FromIndex]` on the `V*`, `[QueryType]` on the aggregate, a single-map index denormalizing with `LoadDocument` + `StoreAllFields`, an `AsNoTracking()` context property, `e.Reroute` in `OnPreviewLoad` to open the editable record. PROACTIVE: invoke when the user builds an overview/grid/report, denormalizes columns, mentions V*/[FromIndex]/[QueryType]/LoadDocument/Reroute, or asks why a list is slow. Not for map/reduce or multi-map aggregation (see /vidyano:map_reduce_indexes)."
user-invocable: true
---

# Query Models: `V*` View Models over Indexes

A grid that lists `Order`s wants the customer's name, the company's name, a line count and a total — values living on three different documents. Loading each `Order`, then each referenced document, to render one row does not scale; pushing that join into every query duplicates it everywhere. The answer is a dedicated **query model**: a `V`-prefixed class (`VOrder`) projected by a RavenDB index that does the denormalization once, at index time. The grid binds to the query model and reads a single flat row; the editable aggregate stays clean.

> **Query model = the R in CQRS.** This is the read side of a read/write split. We call it a *query model*, not a "read model", because Vidyano already uses "read" for a specific right: the **Read** right governs opening a single persistent object (`GetPersistentObject` → `OnLoad`), while the **Query** right governs listing. A `V*` binds with `[QueryType]`, surfaces in lists, and is governed by the **Query** right — you query it, you never *open* one. A click on a `V*` row reroutes to the editable aggregate instead (below).

Not for: aggregating indexes — map/reduce, multi-map, `OutputReduceToCollection` (`/vidyano:map_reduce_indexes`) — or the hooks of the editable aggregate the row reroutes to (`/vidyano:lifecycle_hooks`).

## When to use

The user builds an overview/grid/report, denormalizes columns for listing, asks why a list query is slow, or mentions `V*`/`VOrder`/`[FromIndex]`/`[QueryType]`/`AbstractIndexCreationTask`/`LoadDocument`/`OnPreviewLoad`/`Reroute`/`AsNoTracking`.

The write side and the read side want different shapes. The `Order` aggregate is normalized (stores a `Customer` id, not the name) because that keeps writes correct and small. A grid wants the opposite — every column flattened onto one row, already sorted and searchable. A query model resolves the tension by giving the read side its own type:

| | Aggregate (`Order`) | Query model (`VOrder`) |
|---|---|---|
| Purpose | Edited, validated, persisted | Listed, searched, sorted, reported |
| Shape | Normalized (ids, references) | Denormalized (names, totals inlined) |
| Source | A document | An index projection over documents |
| Vidyano right | Read (open one) | Query (list many) |
| Writable | Yes | No (read-only) |

## The three pieces

```
   Order (document)          Orders_Overview (index)         VOrder (query model)
   ─────────────────         ──────────────────────          ────────────────────
   Customer = "cust/7"  ──▶   Map: load + flatten     ──▶     CustomerName = "Acme"
   Lines = [ … ]              StoreAllFields                  LineCount = 3
   [QueryType(VOrder)]                                        Total = 1499.00
                                                              [FromIndex(Orders_Overview)]
```

The entity points "forward" to its query model with `[QueryType]`; the query model points "back" to its index with `[FromIndex]`; the index does the work in between.

### 1. The view model `V<Entity>`

A plain class carrying exactly the columns the grid shows, decorated `[FromIndex]` naming the index that fills it. On the editable aggregate, declare the matching `[QueryType]` so the framework knows which query model lists it:

```csharp
[FromIndex(typeof(Orders_Overview))]
public class VOrder
{
    public string Id { get; set; } = default!;
    public string OrderNumber { get; set; } = default!;
    public string CustomerName { get; set; } = default!;   // denormalized from Customer
    public string CompanyName { get; set; } = default!;    // denormalized from Company
    public int LineCount { get; set; }
    public decimal Total { get; set; }
    public string Status { get; set; } = default!;
}

[QueryType(typeof(VOrder))]
public class Order
{
    public string Id { get; set; } = default!;
    public string OrderNumber { get; set; } = default!;
    [Reference(typeof(Customer))] public string Customer { get; set; } = default!;
    public IList<OrderLine> Lines { get; set; } = new List<OrderLine>();
    public string Status { get; set; } = "Draft";
}
```

### 2. The index that projects it

An `AbstractIndexCreationTask` whose `Map` selects into the `V*` shape. Two settings make it grid-ready:

- `StoreAllFields(FieldStorage.Yes)` — store every projected field in the index so the grid is served straight from the projection, with no document load per row.
- `Index(nameof(VOrder.Field), FieldIndexing.Search)` — enable full-text search on the columns the user filters on.

**Single-map: denormalize with `LoadDocument`.** When the query model flattens one entity plus a few referenced documents, a single-map index is enough:

```csharp
public class Orders_Overview : AbstractIndexCreationTask<Order, VOrder>
{
    public Orders_Overview()
    {
        Map = orders =>
            from order in orders
            let customer = LoadDocument<Customer>(order.Customer)
            let company = LoadDocument<Company>(customer.Company)
            select new VOrder
            {
                Id = order.Id,
                OrderNumber = order.OrderNumber,
                CustomerName = customer.Name,
                CompanyName = company.Name,
                LineCount = order.Lines.Count,
                Total = order.Lines.Sum(l => l.Amount),
                Status = order.Status,
            };

        Index(nameof(VOrder.CustomerName), FieldIndexing.Search);
        Index(nameof(VOrder.CompanyName), FieldIndexing.Search);

        StoreAllFields(FieldStorage.Yes);
    }
}
```

`LoadDocument` creates an index-side dependency: when the referenced `Customer` or `Company` changes, RavenDB re-indexes the affected `VOrder` rows automatically, so the denormalized name never goes stale.

> Do not over-guard the projection. RavenDB's index LINQ null-propagates — a missing or deleted reference makes customer.Name just null, never a throw. Do not add defensive null checks to a plain projection. Guard explicitly only when null would otherwise produce a wrong value rather than a null, e.g. Weight = r != null ? r.Mass + "kg" : null (without the check a missing r yields the misleading "kg").

**Aggregation: multi-map and map/reduce.** When the grid shows a count or sum per key, or joins several source collections, the index is a hand-written `AbstractMultiMapIndexCreationTask<V>` with a `Reduce` — the shapes, the reduce-key rule and `OutputReduceToCollection` are `/vidyano:map_reduce_indexes`. The `V*` side is the same as above, except that the reduce key carries `[Key]` so Vidyano knows what identifies a row:

```csharp
[FromIndex(typeof(Orders_PerCustomer))]
public class VCustomerOrders
{
    [Key] public string Customer { get; set; } = default!;   // the reduce key identifies the row
    public int OrderCount { get; set; }
    public decimal Total { get; set; }
}
```

Indexes deploy at startup along with the rest of the module; a `V*` index is registered no differently from a plain one.

### 3. Exposing it as a query

The context exposes the query model as a read-only queryable, bound to its index and `.AsNoTracking()` (these rows are never saved):

```csharp
public IRavenQueryable<VOrder> VOrders =>
    Query<VOrder, Orders_Overview>().AsNoTracking();
```

That property is the query Vidyano lists in the UI, under the **Query** right. The `V*` class and the `[QueryType]` are model changes: run a schema synchronize (in CronosCore repos, `/dcg:synchronize`) so `Types.VOrder` and the grid's attributes exist before you wire rights or actions to them.

## Rerouting from the grid to the editable record

A query model is read-only, so a user double-clicking a `VOrder` row should land on the editable `Order`, not a dead read-only page. The query model's actions class redirects them by overriding `OnPreviewLoad` (and `OnPreviewAction`):

```csharp
public partial class VOrderActions
{
    public override void OnPreviewLoad(PreviewLoadArgs e) => e.Reroute(Types.Order);

    public override void OnPreviewAction(PreviewActionArgs e) => e.Reroute(Types.Order);
}
```

`e.Reroute(Types.Order)` sends the user to the `Order` whose id matches the row. When the query model's id is composite — a multi-map row keyed by `"{week};{child}"`, say — split it and reroute with an explicit id:

```csharp
public override void OnPreviewLoad(PreviewLoadArgs e)
{
    var parts = e.ObjectId!.Split(';');
    e.Reroute(Types.Order, parts[0]);
}
```

Occasionally you *want* the read row to open as-is — a pure report with nothing editable behind it. Override `OnPreviewLoad` and do nothing (no `Reroute`): it documents the intent and stops the framework from guessing.

## Checklist

- [ ] `V*` class decorated `[FromIndex(typeof(TheIndex))]`; editable aggregate marked `[QueryType(typeof(VEntity))]`.
- [ ] Index `Map`s into the `V*` shape; `LoadDocument` to denormalize references; `StoreAllFields(FieldStorage.Yes)` + `Index(field, FieldIndexing.Search)` on searchable columns.
- [ ] No defensive null checks on a plain projection (index LINQ null-propagates); guard only where null would yield a *wrong* value.
- [ ] Aggregating grids: `[Key]` on the reduce-key property; the index itself follows `/vidyano:map_reduce_indexes`.
- [ ] Exposed read-only via `Query<VEntity, Index>().AsNoTracking()`; schema synchronized after adding the `V*`.
- [ ] `OnPreviewLoad`/`OnPreviewAction` reroute to the editable aggregate (`e.Reroute(Types.Entity[, id])`, splitting composite ids on `;`), or do nothing for a pure report.

## Verify

1. Build and start the app: RavenDB Studio → Indexes lists `Orders/Overview` in state Normal, and querying it in Studio returns flattened rows with `CustomerName` filled in.
2. Run a schema synchronize (in CronosCore repos, `/dcg:synchronize`): the `VOrders` grid lists one row per order with the denormalized columns, and the filter box finds rows by a `Search`-indexed column.
3. Rename a customer: its `VOrder` rows show the new name once the index has caught up.
4. Click a row: the editable `Order` opens (the URL carries the `Order` type, not `VOrder`), and saving it goes through the aggregate's hooks.
5. Negative: a user without the Query right on `VOrder` has no menu entry for the grid.

## Related skills

- `/vidyano:map_reduce_indexes` — the hand-written aggregating indexes behind a roll-up grid.
- `/vidyano:lifecycle_hooks` — the hooks that fire when the rerouted editable aggregate opens and saves.
