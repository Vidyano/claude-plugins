# Value-object children with parent recompute

Nested collections (invoice lines) are value objects of a root aggregate — own `<Child>Actions`, but stored inside the parent document; composite `ObjectId` is `{rootId};{collection};{childId}`. Recompute the child's own amount in `OnRefresh`, the parent's totals in `PrePersistChanges`:

```csharp
public override void OnRefresh(RefreshArgs args)
{
    var obj = args.PersistentObject;
    var qty = (decimal?)obj[AttributeNames.InvoiceLine.Quantity];
    var unit = (decimal?)obj[AttributeNames.InvoiceLine.UnitPrice];
    obj[AttributeNames.InvoiceLine.Amount].SetValue(qty * unit);
}

protected override void PrePersistChanges(PersistentObject obj, InvoiceLine entity)
{
    base.PrePersistChanges(obj, entity);
    InvoiceActions.UpdateTotalsForDetail(Context, obj);   // load root, recompute, queue parent refresh
}
```

Deleting a value object must bump the parent and force the write — modules centralize this in a small `ValueObjectPersistentObjectActions` base whose `OnDelete` calls `PersistChanges()` after `base.OnDelete`.
