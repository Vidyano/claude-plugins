---
name: map_reduce_indexes
description: "Hand-write the aggregating RavenDB indexes `[GenerateIndex]` never produces: map/reduce (`AbstractIndexCreationTask<TSource, TReduce>` with `Map` + `Reduce`, identical output types), multi-map (`AbstractMultiMapIndexCreationTask<T>` + `AddMap` per collection), seed-with-zero counts, `OutputReduceToCollection`. PROACTIVE: invoke when the user needs totals/counts per key, a roll-up grid, an index spanning collections, a referenced-anywhere delete guard, or mentions Reduce/AddMap/StoreAllFields. Not for the single-map grid projection (see /vidyano:query_models)."
user-invocable: true
---

# Map/Reduce & Multi-Map Indexes

The everyday projection index — one source collection, one row per document, usually emitted by Vidyano's source generator from a `[GenerateIndex]` attribute — is covered by `/vidyano:query_models`. This skill is the two **aggregating** shapes you write by hand:

- **Map/reduce** — group documents by a key and aggregate (count, sum, min/max) per group. "Total revenue per product per quarter", "open tasks per project".
- **Multi-map** — fold *several different collections* into one result set with a shared shape. "Everything that references this record", "documents and their type, counted together".

These are the reporting and roll-up indexes. The `[GenerateIndex]` generator (Vidyano.Service.RavenDB) never produces them — you write the class; the base classes are 100% stock RavenDB. Only the `[FromIndex]`/`[Key]` binding and the `Context.Query<,>()` entry point are Vidyano.

Not for: the single-map projection behind a grid, the `[FromIndex]`/`[QueryType]` binding, or rerouting a row click to the editable aggregate (`/vidyano:query_models`).

## When to use

The user needs totals/counts/sums per key, a roll-up or reporting grid, an index spanning multiple collections, a referenced-anywhere delete guard, or mentions map/reduce, multi-map, `AbstractIndexCreationTask`/`AbstractMultiMapIndexCreationTask`, `Reduce`/`AddMap`, `OutputReduceToCollection`, or `StoreAllFields`.

Reach for map/reduce when the grid shows an **aggregate**, not a row-per-document — precomputed and kept current, versus scanning every document on every load in a read-time query model. Reach for multi-map when one result set must draw from **more than one collection**.

## The map/reduce index

Derive from `AbstractIndexCreationTask<TSource, TReduce>` and set **both** `Map` and `Reduce`. `Map` projects each source document (fanning out over child collections where needed); `Reduce` groups those rows by a key and aggregates:

```csharp
public class Orders_RevenueByProductQuarter
    : AbstractIndexCreationTask<Order, Orders_RevenueByProductQuarter.Result>
{
    public class Result
    {
        public string Product { get; set; } = null!;
        public int YearQuarter { get; set; }
        public decimal Quantity { get; set; }
        public decimal Total { get; set; }
    }

    public Orders_RevenueByProductQuarter()
    {
        Map = orders =>
            from order in orders
            from line in order.OrderLines                 // fan-out: one row per order line
            select new Result
            {
                Product     = line.Product,
                YearQuarter = order.Date.Year * 10 + (order.Date.Month - 1) / 3 + 1,
                Quantity    = line.Quantity,
                Total       = line.Quantity * line.UnitPrice,
            };

        Reduce = results =>
            from result in results
            group result by new { result.Product, result.YearQuarter } into g    // the reduce key
            select new Result
            {
                Product     = g.Key.Product,              // key fields come from g.Key
                YearQuarter = g.Key.YearQuarter,
                Quantity    = g.Sum(x => x.Quantity),     // aggregations
                Total       = g.Sum(x => x.Total),
            };

        StoreAllFields(FieldStorage.Yes);
    }
}
```

## The reduce-key rule

The single rule that governs every map/reduce index: **the `Reduce` output type must be identical to the `Map` output type, and every field must come from either the group key or an aggregate.** RavenDB feeds reduce output back into reduce (it reduces incrementally, in trees), so the shapes must match exactly.

In the `Reduce` select:

- Key fields read from `g.Key.*`.
- Numeric roll-ups use `g.Sum(...)`, `g.Max(...)`, `g.Min(...)`, `g.Count()`.
- A non-key field you only want to *carry through* (a label constant within the group) comes from `g.First().Field` — but be sure it really is constant per key, or the value is arbitrary.

Get a field from neither the key nor an aggregate and the index silently produces wrong numbers.

## Counting and the "seed with zero" trick

A count-per-key index emits `Count = 1` in the map and sums in the reduce:

```csharp
Map = items => from item in items
               select new Result { Type = item.Type, Count = 1 };

Reduce = results => from r in results
                    group r by r.Type into g
                    select new Result { Type = g.Key, Count = g.Sum(x => x.Count) };
```

A pure count drops keys with zero rows — a status with no documents won't appear. To show every key including empties, **seed every key with a zero row** from a second map (this makes it a multi-map):

```csharp
AddMap<OrderStatus>(statuses => from s in statuses           // every status, seeded at 0
    select new Result { Status = s.Id, Count = 0 });

AddMap<Order>(orders => from o in orders                     // each order counts as 1
    select new Result { Status = o.Status, Count = 1 });
// Reduce sums Count per Status — statuses with no orders still show, as 0.
```

## Multi-map: merging several collections

Derive from `AbstractMultiMapIndexCreationTask<TResult>` and call `AddMap<TSource>(...)` once per source collection. **Every `AddMap` must produce the same result type with the same fields populated** — a map that leaves a field unset silently drops that data. The canonical use is a reference-tracking delete guard — "which documents point at record X?":

```csharp
public class UsedRecord
{
    public string Id { get; set; } = null!;      // the referencing document
    public string RefId { get; set; } = null!;   // the document it points at
}

public class UsedRecords : AbstractMultiMapIndexCreationTask<UsedRecord>
{
    public UsedRecords()
    {
        AddMap<Order>(orders => from o in orders
            where o.Customer != null
            select new UsedRecord { Id = o.Id, RefId = o.Customer });

        AddMap<Invoice>(invoices => from i in invoices
            from line in i.Lines                          // fan-out over a child collection
            select new UsedRecord { Id = i.Id, RefId = line.Product });

        StoreAllFields(FieldStorage.Yes);
    }
}
```

Querying it answers "is this id referenced?" without loading every collection: `Context.Query<UsedRecord, UsedRecords>().Any(x => x.RefId.In(idsToDelete))`. (CronosCore ships an `AbstractUsedRecords` convenience over exactly this pattern — but the stock multi-map above is all you need.)

## Multi-map plus reduce

Several maps feed a shared result, then a `Reduce` aggregates across all of them — counting usages across collections, rolling up a junction:

```csharp
public class ProductTypes_DocumentCount
    : AbstractMultiMapIndexCreationTask<ProductTypes_DocumentCount.Result>
{
    public class Result
    {
        public string ProductType { get; set; } = null!;
        public string Description { get; set; } = null!;
        public int Count { get; set; }
    }

    public ProductTypes_DocumentCount()
    {
        AddMap<ProductType>(types => from t in types                 // seed every type with 0
            select new Result { ProductType = t.Id, Description = t.Description, Count = 0 });

        AddMap<Product>(products => from p in products               // count each product as 1
            let type = LoadDocument<ProductType>(p.ProductType)
            select new Result { ProductType = p.ProductType, Description = type.Description, Count = 1 });

        Reduce = results => results
            .GroupBy(r => new { r.ProductType, r.Description })
            .Select(g => new Result
            {
                ProductType = g.Key.ProductType,
                Description = g.Key.Description,
                Count       = g.Sum(x => x.Count),
            });

        StoreAllFields(FieldStorage.Yes);
    }
}
```

(Method syntax `.GroupBy(...).Select(...)` is exactly equivalent to query-syntax `group ... into g`; use whichever reads better.)

## Exposing the result to Vidyano

Bind the index to a grid the same way as any query model (`/vidyano:query_models`): `[FromIndex(typeof(TheIndex))]` on the result class, `[Key]` on the reduce-key fields (so Vidyano knows an aggregate row's identity), and a context property to query it. `StoreAllFields(FieldStorage.Yes)` lets queries read every projected field straight from the index; `Index(nameof(Result.Name), FieldIndexing.Search)` enables full-text search on a projected field.

```csharp
[FromIndex(typeof(Orders_RevenueByProductQuarter))]
public class VProductQuarterRevenue
{
    [Key] public string Product { get; set; } = null!;
    [Key] public int YearQuarter { get; set; }
    public decimal Quantity { get; set; }
    public decimal Total { get; set; }
}

// On the context:
public IRavenQueryable<VProductQuarterRevenue> VProductQuarterRevenues
    => Query<VProductQuarterRevenue, Orders_RevenueByProductQuarter>().AsNoTracking();
```

## Materializing results: OutputReduceToCollection

A reduce index can write its output rows into a **real collection** as artificial documents, so *another* index or query can read the aggregate as ordinary data:

```csharp
OutputReduceToCollection = "ProductQuarterRevenues";
PatternForOutputReduceToCollectionReferences = r => $"products/{r.Product}/revenue";
// optionally: PatternReferencesCollectionName = "ProductRevenueRefs";
```

Powerful, but it creates an index → collection → index dependency chain (the artificial documents trigger further indexing), so deploy and rebuild in dependency order, and never let an index recursively reference its own output collection. Use it only when a downstream consumer genuinely needs the aggregate as documents; a plain queryable result is simpler otherwise.

## Counting archived documents

RavenDB's data archival excludes archived documents from indexes by default, so a roll-up silently loses history. Opt the index in from its constructor:

```csharp
using Raven.Client.Documents.DataArchival;

public class Orders_CountByStatus : AbstractIndexCreationTask<Order, Orders_CountByStatus.Result>
{
    public class Result
    {
        public string Status { get; set; } = null!;
        public int Count { get; set; }
    }

    public Orders_CountByStatus()
    {
        Map = orders => from o in orders select new Result { Status = o.Status, Count = 1 };
        Reduce = results => from r in results
                            group r by r.Status into g
                            select new Result { Status = g.Key, Count = g.Sum(x => x.Count) };

        ArchivedDataProcessingBehavior = ArchivedDataProcessingBehavior.IncludeArchived;   // or ArchivedOnly
    }
}
```

## Checklist

- [ ] Map/reduce: `Map` and `Reduce` produce the **identical** type; every reduce field is `g.Key.*`, an aggregate, or a constant-per-key `g.First()`.
- [ ] Counting: emit `Count = 1` in map, `g.Sum(x => x.Count)` in reduce; add a "seed with 0" map to keep empty keys visible.
- [ ] Multi-map: all `AddMap` lambdas emit the same type with the same fields populated (an unset field drops data).
- [ ] Grid binding: `[FromIndex(typeof(TIndex))]`, `[Key]` on reduce-key fields, `Query<TResult, TIndex>().AsNoTracking()` context property, `StoreAllFields(FieldStorage.Yes)`.
- [ ] Fan-out (`from x in doc.Children`) is intended for line items — watch index size on high-cardinality collections.
- [ ] `OutputReduceToCollection`: pair with `PatternForOutputReduceToCollectionReferences`; deploy the index→collection→index chain in order, no cycles.
- [ ] Archived documents count only with `ArchivedDataProcessingBehavior = IncludeArchived` set in the index constructor.

## Verify

1. Build and start the app (indexes deploy at startup): RavenDB Studio → Indexes lists the new index as Map-Reduce or Multi-Map, state Normal, no indexing errors.
2. Query it in Studio (`from index 'Orders/RevenueByProductQuarter'`): one row per key, and a spot-checked total matches a hand sum over the source documents.
3. Edit one source document: the affected row changes once the index is no longer stale.
4. Seed-with-zero: a key with no source documents still appears with `Count` 0.
5. Grid: after adding the `V*` class and its context property, run a schema synchronize (in CronosCore repos, `/dcg:synchronize`); the grid lists the aggregate rows. With `OutputReduceToCollection`, the artificial collection shows under Documents.

## Related skills

- `/vidyano:query_models` — binding the `V*` result to a grid and rerouting to the editable aggregate; the base projection index and `[GenerateIndex]`.
