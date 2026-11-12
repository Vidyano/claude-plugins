---
name: sequential_numbering
description: "Allocate gapless, never-duplicated human-facing numbers (invoice/order/ticket): a sequence document per prefix+period (`INV/25`), allocation on finalize behind a `??=` guard, RavenDB arbitrating via `UseOptimisticConcurrency` + retry on `ConcurrencyException`; alternatives: `CounterBatchOperation`, a `ClusterWide` compare-exchange. PROACTIVE: invoke when the user needs a sequence or per-year number, concurrency-safe numbering, or mentions optimistic concurrency, RavenDB counters, IncrementAndGet, a sequence document. Not for the document id itself (see /vidyano:lifecycle_hooks)."
user-invocable: true
---

# Sequential Numbering & Counters

Many aggregates need a **human-facing sequence number** distinct from their storage id: an invoice number `INV/25/0042`, an order number, a ticket number. It must be gapless or near-gapless, reset per year, and — above all — **never duplicate**, because two invoices sharing a number is a real accounting problem.

This is harder than it looks: the obvious "load a counter, increment it, save it" races under concurrency — *unless* you let the database arbitrate the race. The counter, operation, and optimistic-concurrency APIs here are stock RavenDB; the one CronosCore add-on (`IncrementAndGet`) is the trap to avoid.

Not for: the document id itself — deterministic or natural ids are assigned in `AddToContext` (`/vidyano:lifecycle_hooks`) — or the save hook the allocation runs from (`/vidyano:lifecycle_hooks`, `OnSave`).

## When to use

The user needs an invoice/order/ticket/sequence number, a per-year counter, concurrency-safe numbering with no duplicates, or mentions optimistic concurrency / `UseOptimisticConcurrency` / `ConcurrencyException` / RavenDB counters / `CounterBatchOperation` / `IncrementAndGet` / a sequence document.

## Why the document id won't do

A natural id or a server-generated id is fine for *addressing* a document, but it's a poor human-facing number: opaque, doesn't reset per year, and you often want the number assigned only when the document is **finalized** (sent), not when the draft is created. So the sequence number is a **separate, allocated value** stored on the aggregate. (This is the human-facing number — distinct from a derived document id, which is a different concern.)

## A counter per prefix and period

Whatever stores the running number, **key it by prefix and period** — a separate counter for each `prefix/yy` — so a new year is just a new key starting at 0, with no reset job:

```
INV/25  → 42
INV/26  → 7
SBI/25  → 13
```

Either way, the same helper turns the value into the final string:

```csharp
private static string FormatReference(string prefix, DateTime period, long count) =>
    $"{prefix}/{period:yy}/{count:D4}";   // INV/25/0042
```

## Allocate on finalize, reuse on resend

Assign the number when the aggregate is **finalized** (saved/sent), guarded so it only allocates if a number isn't already set:

```csharp
invoice.Reference ??= AllocateReference(prefix, invoice.Date);
```

The `??=` makes allocation happen **once**: an abandoned draft never burns a number, and a resend sees a non-null `Reference` and reuses it instead of allocating again.

## The core tension: gaps, duplicates, and contention

Three failure modes pull against each other:

- **Gaps** — a number is allocated, the surrounding save fails, the number is burned. Harmless for accounting, ugly in an audit.
- **Duplicates** — two writers read the same value and both claim the next number. Never acceptable.
- **Contention** — serializing allocation so neither happens, paid for in retries or waiting.

You can't zero all three — but you don't have to treat "gaps or duplicates" as the only choice. For normal volumes, eliminate *both* and absorb the cost as **occasional retries**: that's Option A, and it's what to reach for first. The fallback judgement, when you genuinely must trade: **gaps are harmless; duplicates are not.**

## Option A — sequence document with optimistic concurrency (recommended)

Store the running number as a plain property on a small **sequence document**, one per prefix+period, and let RavenDB's **optimistic concurrency** arbitrate. The number is allocated, stamped onto the entity, and the incremented counter saved **in the same `SaveChanges` batch** — they commit together or not at all. A concurrent bump throws `ConcurrencyException`; reload, take the next number, retry. A failed attempt commits nothing, so there's neither a gap nor a duplicate.

```csharp
public class NumberSequence
{
    public string Id { get; set; } = null!;   // "sequences/INV/25"
    public long Last { get; set; }
}
```

```csharp
private Invoice CreateNumberedInvoice(string prefix, DateTime period)
{
    // Make the save fail if the sequence document changed under us.
    Context.Session.Advanced.UseOptimisticConcurrency = true;
    var seqId = $"sequences/{prefix}/{period:yy}";

    for (var attempt = 0; attempt < 5; attempt++)   // bounded: a pathological hotspot errors instead of spinning
    {
        var seq = Context.Session.Load<NumberSequence>(seqId);
        if (seq is null)
        {
            seq = new NumberSequence { Id = seqId };
            Context.Session.Store(seq);   // a null change vector enforces "must not already exist"
        }

        seq.Last++;
        var invoice = new Invoice { Reference = FormatReference(prefix, period, seq.Last) };
        Context.Session.Store(invoice);

        try
        {
            Context.SaveChanges();        // counter + invoice committed atomically
            return invoice;
        }
        catch (ConcurrencyException)
        {
            // Someone took a number first — nothing was committed (no gap).
            // Evict only the stale pair (not Clear(), so other pending work survives) and retry.
            Context.Session.Advanced.Evict(seq);
            Context.Session.Advanced.Evict(invoice);
        }
    }

    throw new InvalidOperationException($"Could not allocate a number for {seqId} after 5 attempts.");
}
```

Why reach for it first: **gapless** (increment shares the transaction with the document that uses it), **concurrency-safe** (the loser retries), **multi-threaded** (the retry, not a lock, serializes the race), **no special primitive** (a plain `long` + `try`/`catch`). The cost is **contention** — the sequence document is a write hotspot — but for invoice/order/ticket volumes that's negligible.

## The duplicate trap: increment-then-get in one session

If you use RavenDB counters (Options B/C), there's a tempting wrong turn that ships duplicates — incrementing and then reading back in the same session:

```csharp
// LOOKS fine, IS a trap when allocating several numbers in one session.
session.CountersFor(docId).Increment(counterName);          // deferred until SaveChanges
var count = session.CountersFor(docId).Get(counterName);    // reads the last PERSISTED value
```

`Increment` is **deferred to `SaveChanges`**, but `Get` reads the last *persisted* value. Allocate three numbers before one `SaveChanges` and all three read the same base — `42, 42, 42`. This is exactly how a batch produces duplicate references. (CronosCore wraps this pair as an `IncrementAndGet` extension — same trap.) Don't allocate multiple numbers per session this way.

## Option B — immediate atomic increment (when gaps are acceptable)

Increment the counter on the server *immediately*, via a store operation outside the session transaction, and use what it returns:

```csharp
private string AllocateReference(string prefix, DateTime period)
{
    var counterName = $"{prefix}/{period:yy}";
    var result = Context.Session.Advanced.DocumentStore.Operations.Send(
        new CounterBatchOperation(new CounterBatch
        {
            Documents =
            [
                new DocumentCountersOperation
                {
                    DocumentId = "Settings/InvoiceNumbers",
                    Operations = [ new CounterOperation
                    {
                        CounterName = counterName,
                        Delta = 1,
                        Type = CounterOperationType.Increment,
                    } ],
                }
            ],
        }));

    var count = result.Counters[0].CounterValues.First().Value;
    return FormatReference(prefix, period, count);
}
```

Because the increment is server-side and atomic, two concurrent callers always get distinct numbers. The cost: if `SaveChanges()` later throws, that number is gone — a gap. Choose this over Option A only when you specifically want to avoid the optimistic-concurrency retry (very high contention) and gaps are fine.

## Option C — deferred-delta scope (single-threaded batches)

When you must avoid gaps in a **single-threaded batch** that allocates many numbers, read each counter's base once, accumulate deltas in memory, and flush them as part of the unit of work:

```csharp
internal sealed class CounterScope(IDocumentSession session) : IDisposable
{
    private readonly Dictionary<(string counter, string doc), (long baseValue, long delta)> _counters = [];

    public long Next(string counterName, string documentId)
    {
        var key = (counterName, documentId);
        if (!_counters.TryGetValue(key, out var c))
            c = (session.CountersFor(documentId).Get(counterName) ?? 0, 0);   // read persisted base once
        c.delta++;
        _counters[key] = c;
        return c.baseValue + c.delta;                                        // base + running local delta
    }

    public void Dispose()
    {
        foreach (var ((counter, doc), (_, delta)) in _counters)
            session.CountersFor(doc).Increment(counter, delta);              // queue the real increment
    }
}
```

The `using` block **must close before `SaveChanges()`**, so the deferred increments are queued into that save:

```csharp
private void NumberBatch(IEnumerable<Invoice> batch, string prefix)
{
    using (var scope = new CounterScope(Context.Session))
    {
        foreach (var invoice in batch)
            invoice.Reference = FormatReference(prefix, invoice.Date,
                scope.Next($"{prefix}/{invoice.Date:yy}", "Settings/InvoiceNumbers"));
    }                       // Dispose() queues the increments...
    Context.SaveChanges();  // ...and this persists invoices + counter deltas atomically.
}
```

Gapless because nothing commits until `SaveChanges`. **Only safe single-threaded** — two concurrent batches read the same base and collide. Reserve it for a high-volume single-threaded batch where per-number retries would be wasteful.

## Option D — cluster-wide transaction with compare-exchange (strictest)

Option A's optimistic concurrency is arbitrated by the node that owns the document; in a multi-node cluster with failover a rare split-second duplicate is theoretically possible. When a gap *and* a duplicate are both unacceptable, hold the running number in a **compare-exchange** key and commit inside a **cluster-wide** session — its `SaveChanges` is a Raft consensus, so the key and the invoice commit together on every node or not at all:

```csharp
using Raven.Client.Documents.Session;
using Raven.Client.Exceptions;

private Invoice CreateNumberedInvoiceClusterWide(string prefix, DateTime period)
{
    var store = Context.Session.Advanced.DocumentStore;
    var key = $"sequences/{prefix}/{period:yy}";

    for (var attempt = 0; attempt < 5; attempt++)
    {
        // A separate session: cluster-wide mode cannot be switched on for the request's own session.
        using var session = store.OpenSession(new SessionOptions { TransactionMode = TransactionMode.ClusterWide });

        var seq = session.Advanced.ClusterTransaction.GetCompareExchangeValue<long>(key);
        var next = (seq?.Value ?? 0) + 1;
        if (seq is null)
            session.Advanced.ClusterTransaction.CreateCompareExchangeValue(key, next);
        else
            seq.Value = next;                                   // tracked: SaveChanges sends it with its index

        var invoice = new Invoice { Reference = FormatReference(prefix, period, next) };
        session.Store(invoice);

        try
        {
            session.SaveChanges();                              // key + invoice commit together, cluster-wide
            return invoice;
        }
        catch (ConcurrencyException)
        {
            // another writer moved the key first — the whole transaction was rejected; retry
        }
    }

    throw new InvalidOperationException($"Could not allocate a number for {key} after 5 attempts.");
}
```

The invoice lives in that separate session, not in `Context` — return its id and `Context.Session.Load` it if the request keeps working on it. Heavier than Option A (every allocation is a cluster round-trip), so reserve it for the case where the strict guarantee is a requirement, not a preference.

## Checklist

- [ ] Number keyed by **prefix + period** (`INV/25`) so year rollover is free — period is part of the key/name, never a reset job.
- [ ] Allocated on **finalize** (save/send) with a `??=` guard — drafts don't burn numbers, resends reuse.
- [ ] **Option A** preferred: sequence document, `UseOptimisticConcurrency = true`, increment + stamp + save in one batch, retry on `ConcurrencyException`, **bounded** loop, `Evict` (not `Clear`) on retry.
- [ ] **Never** `IncrementAndGet` (or unguarded load-increment-save) for several numbers in one session — deferred increment + stale `Get` yields duplicates.
- [ ] Option B (immediate `CounterBatchOperation`) only when gaps are acceptable; Option C (deferred `CounterScope`, dispose **before** `SaveChanges`) only single-threaded.
- [ ] Cluster-strict need: Option D (`TransactionMode.ClusterWide` + compare-exchange) — heavier; only when a gap *and* a rare duplicate are both unacceptable.

## Verify

1. Build; finalize one invoice: its `Reference` reads `INV/25/0001` and RavenDB Studio shows `sequences/INV/25` with `Last: 1` (Option D: the compare-exchange key under Documents → Compare Exchange).
2. Finalize a second invoice and resend the first: the second is `0002`, the first still `0001`, `Last` is 2.
3. Concurrency: finalize two invoices at the same moment (two browser sessions, or a small parallel test): both numbers are distinct, `Last` equals the count of numbered invoices, and the log shows a retry rather than an error.
4. Roll the period: finalize an invoice dated next year and a new `sequences/INV/26` document appears at `Last: 1` with no reset job.
5. Negative: make the save fail after allocation (trip a rule): with Option A or D no sequence changed; with Option B the counter advanced — the accepted gap.

## Related skills

- `/vidyano:lifecycle_hooks` — allocation usually runs in `OnSave` (or the action that sends). The human-facing sequence number is separate from any derived document id.
