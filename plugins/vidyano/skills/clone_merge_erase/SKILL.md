---
name: clone_merge_erase
description: "Clone an aggregate, merge duplicates, or erase/anonymize data as custom actions over a RavenDB session (no FK cascade, no audit/id auto-reset): detach with Evict/AsNoTracking and reset before Store, repoint every reference (session or PatchByQueryOperation) before deleting losers, merged-from provenance in @metadata. PROACTIVE: invoke when the user duplicates/clones a record, merges or deduplicates records, repoints references, GDPR-anonymizes or erases data, or mentions clone/merge/Evict/PatchByQueryOperation/merged-from. Not for a key re-shape (see /vidyano:data_migrations)."
user-invocable: true
---

# Clone, Merge & Erase Aggregates

Three operations show up on almost every aggregate once it's been in production a while:

- **Clone** — "make me another one like this", copying the aggregate and its children into a fresh document.
- **Merge** — two records turn out to be the same entity; pick a survivor, repoint everything that referenced the loser, delete the loser.
- **Erase / anonymize** — a GDPR request, or scrubbing personal data while keeping the document for referential integrity.

All three are custom actions over a RavenDB session, and all three are shaped by the same two facts about RavenDB. There is **no built-in "clone", "merge", or "erase"** — you compose them from primitives. This skill gives the real patterns for clone and merge, and is honest about erase: it's a composition of the other two's building blocks, not a feature. (The merge delete here uses stock `Session.Delete`; CronosCore's `Context.MarkForDeletion` is a convenience over it.)

Not for: a one-shot re-shape of stored keys across a collection (`/vidyano:data_migrations`), or the action plumbing itself — right, model declaration, return shapes (`/vidyano:custom_actions`).

## When to use

The user wants to duplicate/clone a record and its children, merge or deduplicate records, repoint references and delete the losers, GDPR-anonymize or erase personal data, or mentions clone/copy/duplicate, merge, `PatchByQueryOperation`, `Evict`, deep-copy, or merged-from provenance.

## The shared problem: no cascade, no auto-reset

Two things the framework will *not* do, and forgetting either is the usual bug:

1. **RavenDB has no foreign-key cascade.** Deleting a document does nothing to documents that reference it. Before deleting a merge loser, *you* repoint every reference yourself.
2. **Nothing resets audit fields or ids on a copy.** A naive copy carries the original's `Id`, `CreatedBy`, `CreatedOn`, status, and secrets. Reset them by hand.

Everything below is downstream of these two facts.

## Clone: entity-level

The lightest clone loads the entity, **detaches it from the session**, clears the id and audit/status fields, and stores it as a new document. Detaching is essential — mutating a *tracked* entity's id and saving overwrites the original instead of copying:

```csharp
public override PersistentObject? Execute(CustomActionArgs e)
{
    var template = e.GetEntities<OrderTemplate>().First();

    Context.Session.Advanced.Evict(template);   // detach: don't overwrite the original

    template.Id = null!;                         // RavenDB assigns a fresh id on Store
    template.Name += " (Copy)";
    template.IsEnabled = false;                  // reset status
    template.CreatedBy = null!;                  // reset audit — see below
    template.ModifiedBy = null;
    template.ModifiedOn = null;

    Context.Session.Store(template);
    Context.SaveChanges();
    return null;
}
```

Loading with `.AsNoTracking()` is an equivalent way to get a detached copy. `Id = null` lets RavenDB allocate a server id; assign an explicit natural id instead when the clone's children must cross-reference it *within the same unit of work* before save.

## Clone: persistent-object level (recursive)

When the aggregate has detail collections that must themselves be deep-copied, clone at the **`PersistentObject` layer**. Build a new PO, copy every attribute except audit fields, and recurse into detail collections:

```csharp
private static readonly HashSet<string> AuditFields = ["CreatedBy", "CreatedOn", "ModifiedBy", "ModifiedOn"];

private PersistentObject Clone(PersistentObject obj)
{
    var copy = Manager.Current.GetPersistentObject(obj.FullTypeName, isNew: true, targetContext: Context);

    foreach (var attribute in obj.Attributes)
    {
        var name = attribute.Name;
        if (AuditFields.Contains(name) || !copy.TryGetAttribute(name, out var target))
            continue;

        if (target is PersistentObjectAttributeAsDetail targetDetail && attribute is PersistentObjectAttributeAsDetail detail)
            targetDetail.Objects = detail.Objects.Select(Clone).ToArray();   // recurse into child collection
        else if (target.Type == DataTypes.TranslatedString)
            target.SetValue(attribute.Options[0]);                            // copy the raw translated value
        else
            target.SetValue(attribute.GetValue(Context));
    }

    return copy;
}
```

Return the cloned PO as `isNew: true` and the normal new-object path persists it (and opens it) on Save — the id is allocated for you. The recursion gives a genuinely deep copy; an attribute-by-attribute copy that skips detail collections shares children by reference.

## Resetting audit fields and scrubbing secrets

On every clone:

- **Audit fields.** No auto-reset. Null them individually (`CreatedBy`/`CreatedOn`/`ModifiedBy`/`ModifiedOn`) or, cleaner, keep a shared `AuditFields` allow-set and skip those names in the copy loop. Stamp fresh values if the copy should record who cloned it.
- **Secrets.** A cloned child often carries a credential, API key, or token. Clear them — a clone should not silently duplicate a live secret. (Fields stored encrypted at rest are a separate concern — in CronosCore repos, the dcg plugin's `field_encryption` skill.)
- **Value-object children.** A JSON value-object collection must be **deep-cloned** (`(JObject)child.DeepClone()`) or the copy shares the same instance by reference; detail-collection POs are recursively cloned as above.

## Merge: the reconcile dialog

A merge has two halves. First, a **reconcile dialog** lets the user pick the surviving value per field; second, the real repoint-and-delete runs in that dialog's `OnSave` (`/vidyano:wizards`). The action just builds the dialog from the selected duplicates — each field read-only when all sources agree, a dropdown of the distinct values when they differ:

```csharp
public override PersistentObject? Execute(CustomActionArgs e)
{
    e.EnsureQuery();
    var ids = e.SelectedItems.Select(it => it.Id!).ToArray();

    // ObjectId carries the duplicate set; OnNew builds one attribute per field,
    // read-only if the values agree, a DropDown of distinct values if they differ.
    return Manager.Current.GetPersistentObject(Types.MergePeople, string.Join(";", ids),
        isNew: true, targetContext: Context, ignoreQueries: true);
}
```

Pre-selecting a "preferred" survivor with a small heuristic (the most complete record, the non-synced one) saves clicks. The survivor's id is whatever the dialog resolves; the rest of the selected ids are the losers.

## Merge: repointing references

The heart of the merge. Before deleting the losers, repoint everything that points at them. Both shapes below run in the reconcile dialog's `OnSave`, where `obj` is the dialog PO: its `ObjectId` carries the duplicate set and one attribute holds the chosen survivor:

```csharp
var survivorId = (string)obj[AttributeNames.MergePeople.Survivor]!;
var loserIds   = obj.ObjectId!.Split(';').Where(id => id != survivorId).ToArray();
var survivor   = Context.Session.Load<Person>(survivorId);
var losers     = Context.Session.Load<Person>(loserIds).Values;
```

Two approaches by scale.

**Doc-by-doc in the session** — when references live in a known, small set of collections:

```csharp
foreach (var loser in losers)
{
    foreach (var order in Context.Orders.Where(o => o.Customer == loser.Id))
        order.Customer = survivorId;

    survivor.Notes.AddRange(loser.Notes);          // merge child collections
    Context.Session.Delete(loser);                 // delete the loser
}
Context.SaveChanges();
```

**Patch-by-query across the whole database** — when references could be anywhere. A server-side `PatchByQueryOperation` runs a JS patch over every collection, skipping system documents (ids starting with `Raven/`), swapping any top-level field equal to a loser id over to the survivor id, and you wait for it:

```csharp
using Raven.Client;
using Raven.Client.Documents.Operations;
using Raven.Client.Documents.Queries;

var op = Context.Session.Advanced.DocumentStore.Operations.Send(new PatchByQueryOperation(new IndexQuery
{
    Query = """
            from @all_docs as doc
            where not startsWith(id(), 'Raven/')
            update {
                for (var field in doc) {
                    if (field !== '@metadata' && $losers.indexOf(doc[field]) !== -1)
                        doc[field] = $survivor;
                }
            }
            """,
    QueryParameters = new Parameters
    {
        ["losers"]   = loserIds,
        ["survivor"] = survivorId,
    },
}));
op.WaitForCompletion();
```

Extend the loop for nested paths your model uses (`doc.Lines.forEach(...)`), then delete the losers in the session as above. Photos/attachments are merged by streaming the chosen one onto the survivor. Whichever approach, **repoint before delete** — there's no cascade to catch a missed reference. Watch unique-key ordering: if survivor and loser both hold a value under a unique constraint, **delete the loser first**, then move the value onto the survivor, or the constraint trips.

## Recording merge provenance

Once the losers are deleted they're gone, and "why does this customer have that order?" becomes unanswerable. **Record the loser ids on the survivor** before deleting. Crucially, a loser may itself be the **survivor of an earlier merge** — it already carries its own merged-from trail. Record only the loser's own id and that earlier lineage is lost. So fold in **each loser's id *and* each loser's accumulated trail**, transitively.

The cleanest place is the survivor's **`@metadata`** — a string array there is generic (any entity, no model change) and stays off the document body and the PO surface:

```csharp
string[] ReadTrail(object doc) =>
    Context.Session.Advanced.GetMetadataFor(doc).TryGetValue("Merged-From", out var v) && v is IEnumerable<object> trail
        ? trail.Select(o => (string)o!).ToArray()
        : [];

var survivorMeta = Context.Session.Advanced.GetMetadataFor(survivor);
survivorMeta["Merged-From"] = ReadTrail(survivor)                     // survivor's existing trail
    .Concat(loserIds)                                                // the losers themselves
    .Concat(losers.SelectMany(ReadTrail))                            // + each loser's inherited trail
    .Distinct()
    .ToArray();
```

For a strongly-typed alternative, a property marked `[IgnoreProperty]` persists on the document but is hidden from the PO:

```csharp
[IgnoreProperty]
public List<string> MergedFrom { get; set; } = [];
// ... during the merge:
foreach (var loser in losers)
{
    survivor.MergedFrom.Add(loser.Id);
    survivor.MergedFrom.AddRange(loser.MergedFrom);   // inherit the loser's own lineage
}
survivor.MergedFrom = survivor.MergedFrom.Distinct().ToList();
```

Prefer `@metadata` for one generic mechanism across many merge-able entities; prefer `[IgnoreProperty]` when a single aggregate wants a typed, queryable field. Either way, **append and inherit** rather than overwrite.

## Erase / anonymize: a composition, not a feature

There is no dedicated "GDPR erase" action — be wary of any guide implying otherwise. What exists is two genuinely different needs, both built from the primitives above:

- **Anonymize in place** — scrub personal fields but keep the document so references stay intact. This is the entity-level field reset from the clone section, applied to the live document:

  ```csharp
  var person = Context.Session.Load<Person>(e.Parent!.ObjectId);

  person.FirstName = "REDACTED";
  person.LastName  = "REDACTED";
  person.Email     = null;
  person.NationalNumber = null;
  Context.SaveChanges();
  ```

- **Hard delete** — when nothing references the person, `Context.Session.Delete(person)` (the same delete used in merge). If references exist, repoint or anonymize them first, exactly as in a merge.

So "erase" is not a new mechanism: it's *field-reset* (from clone) plus optionally *delete + repoint* (from merge). Decide per field whether erasure means null, a tombstone value, or document removal, and reuse the building blocks.

## Checklist

- [ ] Clone: **detach first** (`Evict` or `AsNoTracking()`) before changing id/fields, or you overwrite the original.
- [ ] Reset `Id` + audit fields manually (or skip an `AuditFields` allow-set in the copy loop); **scrub secrets**; **deep-copy** detail POs (recurse) and JSON value-objects (`DeepClone()`).
- [ ] Merge: reconcile dialog (read-only when sources agree, dropdown when they differ); **repoint every reference before delete** — doc-by-doc for known collections, `PatchByQueryOperation` (excluding system collections, `WaitForCompletion()`) for database-wide.
- [ ] Unique-key ordering: delete the loser before moving a unique value onto the survivor.
- [ ] Provenance recorded **transitively** (loser id + each loser's own prior trail, `Distinct()`) on `@metadata` or an `[IgnoreProperty]` property — append-and-inherit, never overwrite.
- [ ] Erase modeled as field-reset + optional delete/repoint — not a magic framework call; decide per field what "erased" means.

## Verify

1. Clone: run the action, then check RavenDB Studio — a new document exists with a fresh id, the original is unchanged, and audit fields, status, and secrets on the copy are reset; detail children have their own ids (not shared with the original).
2. Merge: after the dialog saves, the loser documents are gone, every document that referenced a loser id now carries the survivor id (query `where Customer = '<loserId>'` returns nothing), and the survivor's `@metadata` or `MergedFrom` lists the losers plus their inherited trail.
3. Unique key: merge two records that both hold a unique value and confirm the constraint does not trip (loser deleted before the value moves).
4. Erase: the anonymized document keeps its id and references resolve, while the personal fields read `REDACTED`/null.
5. Negative case: clone without `Evict`/`AsNoTracking()` and watch the original get overwritten — then put the detach back.

## Related skills

- `/vidyano:custom_actions` — clone, merge, and erase are all custom actions over the session.
- `/vidyano:wizards` — the merge reconcile dialog (a transient PO whose `OnSave` runs the repoint-and-delete).
- `/vidyano:lifecycle_hooks` — `OnNew`/`OnSave` and the audit fields a clone must reset.
- `/vidyano:data_migrations` — a key re-shape (vs a one-off clone/merge) is a migration.
