---
name: relationship_editors
description: "Attach and detach related records on a parent without custom verbs: enable the built-in `AddReference`/`Remove` query actions in `OnConstruct`, override `OnAddReference`/`OnRemoveReference` to mutate the parent's collection from `selectedItems` and save — the framework supplies the picker. PROACTIVE: invoke when the user wants an Add/Remove button to link or unlink records, manages a many-to-many, or mentions AddReference/OnAddReference/OnRemoveReference/LookupSource. Not for a dialog gathering extra input first (see /vidyano:wizards)."
user-invocable: true
---

# Relationship Editors (Add / Remove Reference)

A parent aggregate shows a grid of *related* records, and the user wants to **attach** another one (pick from a list) or **detach** the selected rows — a `Company` with a grid of `Person`s, an "Add" button that opens a picker and a "Remove" button that drops the selected people.

The instinct is to write two custom actions. Don't. `AddReference` and `Remove` are **stock Vidyano query actions**: you enable them on a detail query and override two hooks. The framework supplies the picker dialog, reads the selected rows, materializes the entities, and hands them to you. Your code does one thing — mutate the parent's collection and save. Nothing here touches CronosCore.

Not for: a dialog that must gather extra input before linking (`/vidyano:wizards`), a toolbar verb of your own (`/vidyano:custom_actions`), or a child grid that merely *shows* related rows (`/vidyano:detail_queries`).

## When to use

The user wants an Add/Remove button to link or unlink related records, attach members to a collection from a picker, manage a many-to-many, or mentions `AddReference`/`RemoveReference`/`OnAddReference`/`OnRemoveReference`/`ActionNames.AddReference`/`LookupSource`/`UseValueSource`.

## The built-in actions, not a custom verb

- **`AddReference`** renders an "Add" toolbar button that opens a *select-reference dialog* — a sub-query listing candidate rows to pick from.
- **`Remove`** operates on the rows currently selected in the detail grid.

On confirm, the framework dispatches to two virtual hooks on the detail entity's `PersistentObjectActions` — `OnAddReference` and `OnRemoveReference` — inside a transaction scope. There is a sensible **default**: override nothing and the base adds/removes the entity to/from the parent's child collection named by `query.Source`, then persists. Override only when the relationship needs more (a parallel id list, a value-object projection, a bidirectional link).

## Turning the actions on

Enable them in `OnConstruct` by assigning `query.Actions`, gated on the parent type and (usually) a permission so the buttons only appear where they make sense:

```csharp
public partial class PersonActions   // actions for the Person detail query under Company
{
    public override void OnConstruct(Query query, PersistentObject? parent)
    {
        base.OnConstruct(query, parent);

        if (parent is not { Type: Types.Company } || !Manager.Current.User!.IsMemberOf(AppRoles.Editors))
            return;

        query.Actions = [ActionNames.AddReference, ActionNames.Remove];
    }
}
```

For a true many-to-many that Vidyano synchronizes automatically, the picker's candidate source (`LookupSource`) is wired for you. For a hand-modeled collection, set `query.Actions` as above and the picker uses the query itself as the candidate source.

## Handling the add

`OnAddReference` receives the parent PO, the materialized picked entities, the query, and the raw selected rows. Load the parent, append the chosen ids/values, save:

```csharp
public override void OnAddReference(PersistentObject? parent, IEnumerable<Person> entities,
                                    Query query, QueryResultItem[] selectedItems)
{
    var company = Context.Session.Load<Company>(parent!.ObjectId);

    company.People.AddRange(selectedItems.Select(it => it.Id!));
    Context.SaveChanges();
}
```

If the collection is a list of reference ids, `selectedItems.Select(it => it.Id)` is all you need. If it's a value-object collection, load the picked entities and project them into your value-object shape before adding.

## Handling the remove

`OnRemoveReference` is the mirror image, operating on the selected grid rows:

```csharp
public override void OnRemoveReference(PersistentObject parent, IEnumerable<Person> entities,
                                       Query query, QueryResultItem[] selectedItems)
{
    var company = Context.Session.Load<Company>(parent.ObjectId);
    var ids = selectedItems.Select(it => it.Id).ToArray();

    company.People.RemoveAll(p => p.In(ids));
    Context.SaveChanges();
}
```

The hook returns nothing; the grid re-reads from the now-mutated parent. Mirror the base signatures' nullability exactly: `parent` is `PersistentObject?` on `OnAddReference` but `PersistentObject` on `OnRemoveReference`.

## Keeping two parallel collections in sync

A frequent model keeps both a **value-object collection** (display: names, dates) *and* a parallel **id list** (fast membership tests, indexing). Every add and remove must touch **both**, or they silently drift:

```csharp
public override void OnAddReference(PersistentObject? parent, IEnumerable<Tag> entities,
                                    Query query, QueryResultItem[] selectedItems)
{
    var node = Context.Session.Load<Node>(parent!.ObjectId);
    var tags = Context.Session.Load<Tag>([.. selectedItems.Select(it => it.Id)]).Select(x => x.Value);

    node.NodeTags.AddRange(tags);                            // value-object collection (display)
    node.Tags.AddRange(selectedItems.Select(it => it.Id!));  // parallel id list (membership/index)
    Context.SaveChanges();
}

public override void OnRemoveReference(PersistentObject parent, IEnumerable<Tag> entities,
                                       Query query, QueryResultItem[] selectedItems)
{
    var node = Context.Session.Load<Node>(parent.ObjectId);
    var ids = selectedItems.Select(it => it.Id).ToArray();

    node.NodeTags.RemoveAll(e => e.Id.In(ids));
    node.Tags.RemoveAll(e => e.In(ids));
    Context.SaveChanges();
}
```

Forgetting one side is the classic bug — the grid looks right but a membership query returns the wrong answer.

## Composite ids on remove

When the detail query is itself a projection, `selectedItems[].Id` can be a **composite** like `"<parentSegment>;<childSegment>"`. Don't match the raw composite against your collection key — split out the part you store:

```csharp
public override void OnRemoveReference(PersistentObject parent, IEnumerable<CompanyLink> entities,
                                       Query query, QueryResultItem[] selectedItems)
{
    // Id is "<appRequestId>;<companyId>" — keep the trailing segment.
    var companyIds = selectedItems.Select(it => it.Id!.Split(";")[^1]).ToArray();

    Context.GetRequiredEntity<AppRequest>(parent).Companies.RemoveAll(x => x.Company.In(companyIds));
    Context.SaveChanges();
}
```

## Preventing duplicates

Two cases, two mechanisms:

- **Framework picker (`LookupSource` set).** The dialog filters out already-referenced rows, so the user can't pick a duplicate. Nothing to do.
- **Anything you build yourself.** You own dedup. Filter candidates when you populate the picker *and* guard the write with `.Distinct()` — belt and braces, because nothing stops a stale double-submit.

## When there is no detail query: the wizard variant

`AddReference`/`Remove` assume a navigable detail query. When the relationship isn't one — stored as a string array on a user profile, or the add needs a confirmation form with extra fields — fall back to a transient-PO wizard (`/vidyano:wizards`). The custom action just opens a transient PO; a paired actions class does the linking in `OnNew` (populate the picker, minus already-linked) and `OnSave` (mutate + save):

```csharp
// The action is a one-liner that opens the transient picker dialog.
public override PersistentObject? Execute(CustomActionArgs e) =>
    Manager.Current.GetPersistentObject(Types.AddToTeam, e.Parent!.ObjectId,
        isNew: true, targetContext: Context, ignoreQueries: true);
```

```csharp
public partial class AddToTeamActions
{
    public override void OnNew(PersistentObject obj, PersistentObject? parent, Query? query,
                               Dictionary<string, string>? parameters)
    {
        base.OnNew(obj, parent, query, parameters);

        var team = Context.Session.Load<Team>(obj.ObjectId);                 // the parent id passed into GetPersistentObject
        var candidates = Context.People.Where(p => !p.Id.In(team.Members))   // dedup: hide already-linked
                                       .Select(p => new { p.Id, p.Name }).ToArray();

        obj[AttributeNames.AddToTeam.Person].UseValueSource(candidates, it => it.Id, it => it.Name);
    }

    // The dialog is a form, not a record: do the work here and do not call base.OnSave
    // (it would run SaveNew on a PO with no backing entity).
    public override void OnSave(PersistentObject obj)
    {
        if (!CheckRules(obj)) return;

        var team = Context.Session.Load<Team>(obj.ObjectId);
        var personId = (string)obj[AttributeNames.AddToTeam.Person]!;
        team.Members = team.Members.Append(personId).Distinct().ToList();   // a double-submit can't duplicate
        Context.SaveChanges();
    }
}
```

The bulk shape is the same, seeded from the selected grid rows in `Execute` (`e.EnsureQuery()` then read `e.SelectedItems`).

## Checklist

- [ ] Actions enabled in `OnConstruct`: `query.Actions = [ActionNames.AddReference, ActionNames.Remove]`, gated on parent `Type` + permission.
- [ ] `OnAddReference` / `OnRemoveReference` overridden only when the default `query.Source` add/remove isn't enough; otherwise rely on the base.
- [ ] Parent loaded, collection mutated from `selectedItems`, `Context.SaveChanges()` issued inside the hook.
- [ ] **Every parallel collection** (value-object list + id list) updated on each add/remove.
- [ ] Composite ids split (`it.Id!.Split(";")[^1]`) before matching on remove.
- [ ] Dedup: rely on `LookupSource` for the picker; in the wizard path filter in `OnNew` AND `.Distinct()` in `OnSave`.
- [ ] Bidirectional links: mutate both ends before saving. Order-sensitive collections: sort explicitly after mutating.

## Verify

1. Build; open the parent as a member of the gated group: the detail grid's toolbar shows Add and Remove. Outside the group, or under a parent of another type, neither appears.
2. Add: the picker lists candidates (minus already-linked rows when `LookupSource` is set); confirm, and the grid shows the new rows while RavenDB Studio shows the parent document's collection — and any parallel id list — grown by exactly those ids.
3. Remove: select rows and Remove; the grid drops them and both collections shrink in Studio.
4. Add the same row again, or double-submit the wizard: the document holds no duplicate entry.
5. Composite ids: a Remove on a projected detail query removes the intended child, not zero rows.

## Related skills

- `/vidyano:lifecycle_hooks` — `OnConstruct` (enabling the actions) and the `OnAddReference`/`OnRemoveReference` hooks live on `PersistentObjectActions`.
- `/vidyano:wizards` — the transient-PO fallback for when there is no navigable detail query.
- `/vidyano:custom_actions` — the opener verb, when the wizard variant needs one.
