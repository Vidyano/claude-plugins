---
name: wizards
description: "Collect input or a selection before an operation with a transient PO opened as a dialog: declared in model JSON only (`StateBehavior: 8`), returned from a custom action's `Execute` via `GetPersistentObject(..., isNew: true)`, `ActionLabels` in `OnNew`, the work run in `OnSave` after `CheckRules`; plus the two-pass `RetryAction`/`RetryResult` and the bulk-assign carrier. PROACTIVE: invoke when the user needs a popup/dialog/wizard before an action, a bulk assign, or mentions transient PO, RetryAction, ActionLabels. Not for the button itself (see /vidyano:custom_actions)."
user-invocable: true
---

# Wizards & Multi-Step Dialogs

Some operations need input before they can run. Recalculating a month needs to know *which* month. Bulk-assigning people to a department needs the department. None of these is a record the user is creating — there is nothing to persist. What you want is a small modal form that collects a parameter (or a selection), runs a domain operation when the user confirms, then refreshes the screen.

A wizard is built from pieces you already know: a custom action returns a **transient** PersistentObject — a PO declared only in model JSON, with no backing C# entity — and the framework opens it as a dialog. The PO is never stored. Its `OnSave` hook is simply *where the real work runs* once the user has filled the form and pressed the (relabelled) Save button. The key shift from an ordinary New flow: **the dialog PO is a form, not a record.**

Not for: the toolbar button itself and its right (`/vidyano:custom_actions`), the general `OnNew`/`OnSave` pipeline of a *real* record (`/vidyano:lifecycle_hooks`), or a paste-a-list intake dialog (`/vidyano:imports`).

## When to use

The user needs a popup/dialog/wizard to collect a parameter or selection before an action, a confirm-then-apply prompt, or a bulk-assign over selected rows — or mentions a transient PO, `StateBehavior 8`/`OpenAsDialog`, `GetPersistentObject`, `RetryAction`/`RetryResult`, `ActionLabels`, or `QueueClientOperation`.

The flow is always the same:

1. A custom action's `Execute` returns the transient PO → the framework opens it as a modal dialog.
2. `OnNew` seeds defaults and relabels the Save button to a verb.
3. The user fills the form and clicks the verb.
4. `OnSave` validates, runs the domain operation, and refreshes the client.

The worked example below: an action on a `Company` that recalculates a chosen month's headcount.

## 1 — Declare the dialog in model JSON

The dialog has no backing entity, so it lives entirely in model JSON. Two fields make it behave as a dialog: `"StateBehavior": 8` (`OpenAsDialog`) renders it as a modal popup instead of a full page, and `"NewBreadcrumb"` sets the header text:

```json
{
  "PersistentObject": {
    "SchemaName": "Acme",
    "Type": "RecalculateMonth",
    "NewBreadcrumb": { "en": "Select month" },
    "StateBehavior": 8,
    "Attributes": [
      {
        "Name": "Month",
        "Offset": 10,
        "DataType": { "$ref": "DataType:Date" },
        "DataTypeHints": "DisplayFormat={0:y}",
        "Label": { "en": "Month" }
      }
    ],
    "Queries": []
  },
  "Queries": [],
  "Reports": []
}
```

Attribute `Required`/`NotEmpty` rules belong here too — `CheckRules` (step 4) enforces them server-side. If the PO type happened to carry queries you don't want, suppress them at open time with `ignoreQueries: true`. The dialog type and its attributes are model changes: run a schema synchronize (in CronosCore repos, `/dcg:synchronize`) so `Types.RecalculateMonth` and `AttributeNames.RecalculateMonth.*` exist before the C# below compiles.

## 2 — The action that opens it

The action is a `/vidyano:custom_actions` whose `Execute` returns the transient PO. Because the returned PO's `StateBehavior` is `OpenAsDialog`, the framework pops it as a modal:

```csharp
namespace Acme.Service.CustomActions;

public partial class RecalculateMonth : CustomAction<AcmeContext>
{
    // The action lives on a Company grid/record; the dialog gathers one parameter (a month).
    public override PersistentObject? Execute(CustomActionArgs e) =>
        Manager.Current.GetPersistentObject(Types.RecalculateMonth, e.Parent?.ObjectId, isNew: true, targetContext: Context);
}
```

The second argument is the **parent ObjectId** — the id of the record the action was invoked on. The framework stamps it onto the transient PO's `ObjectId`, which is how `OnSave` later knows *which* `Company` to recompute. On a grid there is no parent, so `e.Parent` is null and `ObjectId` comes through empty — branch on that to recompute for one company versus all of them. Assert placement at the top of `Execute` with `e.EnsureParent(Types.Company)` or `e.EnsureQuery()`.

## 3 — Seed defaults and relabel Save in OnNew

`OnNew` on the dialog's `*Actions` partial runs as the transient PO is created. Give parameters sensible defaults and — importantly — relabel the Save button. The user is triggering an operation, not saving a record, so the button should read as a verb:

```csharp
namespace Acme.Service.Actions;

public partial class RecalculateMonthActions
{
    public override void OnNew(PersistentObject obj, PersistentObject? parent, Query? query, Dictionary<string, string>? parameters)
    {
        base.OnNew(obj, parent, query, parameters);

        var today = Manager.Current.Today;
        obj[AttributeNames.RecalculateMonth.Month].SetValue(new DateTime(today.Year, today.Month, 1));
        obj.ActionLabels[ActionNames.Save] = "Recalculate";   // verb instead of "Save"
    }
}
```

`ActionLabels[ActionNames.Save]` is the lever — set it to `"Run"`, `"Assign"`, `"Recalculate"`, whatever names the operation. `OnNew` is also where you seed an attribute's selectable `Options` at runtime if a field's choices depend on context.

## 4 — Run the operation in OnSave

When the user presses the verb, `OnSave` fires. There is nothing to persist about the PO itself — this hook is simply *where the operation runs*. **Always call `CheckRules(obj)` first and return early if it fails** — that is what enforces the attribute `Required`/`NotEmpty` rules declared in model JSON (they are NOT applied automatically here). Then read the parameter, load the real aggregate from `ObjectId`, do the work, and save the *domain* changes:

```csharp
public partial class RecalculateMonthActions
{
    // A form, not a record: no base.OnSave here — it would run SaveNew on a PO with no backing entity.
    public override void OnSave(PersistentObject obj)
    {
        if (!CheckRules(obj))
            return;

        var month = (DateTime)obj[AttributeNames.RecalculateMonth.Month];
        var companyId = obj.ObjectId;                          // the source record id; empty when opened from a grid

        RecomputeHeadcount(Context, month, companyId);         // the real domain operation
        Context.SaveChanges();

        Manager.Current.QueueClientOperation(RefreshOperation.Query(QueryNames.CompanyHeadcountPeriods));
    }

    // Shared with the scheduled job, which calls it headlessly with no dialog.
    public static void RecomputeHeadcount(AcmeContext context, DateTime month, string? companyId)
    {
        var companies = string.IsNullOrEmpty(companyId)
            ? context.Companies.ToArray()
            : [context.Session.Load<Company>(companyId)];

        foreach (var company in companies)
            company.Headcount = context.Employees.Count(emp => emp.Company == company.Id && emp.StartDate <= month);
    }
}
```

`obj.ObjectId` here is the **source record's** id (the parent passed into `GetPersistentObject`), not an id belonging to the dialog. Load the aggregate from it with `Context.Session.Load<Company>(obj.ObjectId)` when you need the full entity. Keep the real work in a **shared/static method** so the dialog and any scheduled job run identical logic.

## Refreshing the client

The operation mutated data the user is looking at, so tell the client to refresh — `QueueClientOperation` queues a refresh that runs once `OnSave` returns:

```csharp
// Refresh a grid the operation touched.
Manager.Current.QueueClientOperation(RefreshOperation.Query(QueryNames.CompanyHeadcountPeriods));

// Or refresh the parent record itself after mutating it.
Manager.Current.QueueClientOperation(new RefreshOperation(e.Parent));
```

Inside `OnSave` (or `OnAddReference`) you can also call `obj.QueueQueryRefresh(QueryNames.CompanyHeadcountPeriods)` directly on the PO — same effect, fewer keystrokes. Refresh the query when a grid changed; refresh the parent PO when you wrote values back onto it.

## Variant — the lighter RetryAction / RetryResult confirm-and-apply

When the dialog is a single confirm-and-apply — no separate save round-trip, often just an option button or two — `RetryAction`/`RetryResult` is lighter. It needs only the dialog PO type; you write **no `OnSave` at all**. The catch is that it is a **two-pass protocol**: `Execute` runs twice.

```csharp
public partial class ChangeOrderDiscount : CustomAction<AcmeContext>
{
    public override PersistentObject? Execute(CustomActionArgs e)
    {
        e.EnsureParent(Types.Order);

        // First pass: no answer yet — pop the dialog with Apply / Cancel buttons and return.
        if (Manager.Current.RetryResult is null)
        {
            var dialog = Manager.Current.GetPersistentObject(Types.ChangeOrderDiscountDialog, isNew: true);
            Manager.Current.RetryAction("Change order discount", "Enter the new discount percentage.", dialog, "Apply", "Cancel");
            return null;
        }

        // Second pass: the framework re-invoked Execute with the user's answer.
        var result = Manager.Current.RetryResult;
        if (result.Option == "Apply")
        {
            var percent = (decimal)result.PersistentObject[AttributeNames.ChangeOrderDiscountDialog.Percent];
            var order = Context.Session.Load<Order>(e.Parent!.ObjectId);
            order.DiscountPercent = percent;
            Context.SaveChanges();

            Manager.Current.QueueClientOperation(new RefreshOperation(e.Parent));
        }

        return null;
    }
}
```

On the **first pass** `RetryResult` is null, so you call `RetryAction(title, message, dialogPO, "Option1", "Option2")` and **return `null`** — the framework shows the modal with those named buttons. When the user answers, the framework **re-invokes `Execute`**; now `RetryResult` is non-null and carries `.Option` (the button pressed) and `.PersistentObject` (the filled-in dialog). Branch on `.Option`, do the work, refresh. Ideal for a confirm-then-apply where the whole operation fits in the action and there is no separate dialog `*Actions` class.

## Variant — multi-select bulk assign

The transient-dialog pattern extends to bulk operations: the user multi-selects grid rows, the action collects them, and the dialog asks for the *one* thing the operation still needs — a target. The only new wrinkle is carrying the selection into the transient PO.

The selected rows arrive on `e.SelectedItems` — each `QueryResultItem` exposes `.Id` (the document id) and `.GetValue(...)` for any column. The idiomatic carrier is a **newline-joined id string in a hidden `MultiLineString` attribute**; pair it with a read-only attribute holding human-readable names so the user can verify the selection:

```csharp
public partial class AssignToDepartment : CustomAction<AcmeContext>
{
    public override PersistentObject? Execute(CustomActionArgs e)
    {
        e.EnsureQuery();

        var personIds   = e.SelectedItems!.Select(si => si.Id!);
        var personNames = e.SelectedItems!.Select(si => si.GetValue(AttributeNames.Person.FullName)!);

        var dialog = Manager.Current.GetPersistentObject(Types.AssignToDepartment, isNew: true, ignoreQueries: true, targetContext: Context)!;

        // Selection ride-along: ids in a hidden carrier, names in a read-only display attribute.
        dialog[AttributeNames.AssignToDepartment.People].SetValue(string.Join('\n', personIds));
        dialog[AttributeNames.AssignToDepartment.Display].SetValue(string.Join('\n', personNames));

        return dialog;
    }
}
```

`ignoreQueries: true` keeps the dialog clean by suppressing any grids the PO type might carry. Use `SetOriginalValue(...)` instead of `SetValue(...)` for the hidden carrier when you want to seed it without flagging the attribute dirty. For a soft conflict, surface it non-blockingly with `dialog.AddNotification(text, NotificationType.Warning)`.

`OnSave` splits the carrier back into ids, loads the target, and folds the selection in. **Filter against what the target already has** before appending — that keeps re-running the wizard idempotent, so you never create duplicate links:

```csharp
public partial class AssignToDepartmentActions
{
    public override void OnSave(PersistentObject obj)   // again: no base.OnSave on a transient form
    {
        if (!CheckRules(obj))
            return;

        // The framework auto-splits the newline-joined MultiLineString back into a string[];
        // cast directly — do NOT ((string)...).Split('\n').
        var personIds  = (string[])obj[AttributeNames.AssignToDepartment.People]!;
        var department = Context.Session.Load<Department>((string)obj[AttributeNames.AssignToDepartment.Department]!);

        var existing = department.Members.Select(m => m.Person).ToArray();
        foreach (var personId in personIds.Where(id => !existing.Contains(id)))
            department.Members.Add(new DepartmentMember { Person = personId });

        Context.SaveChanges();
    }
}
```

A heavier alternative exists when you want the framework's built-in reference-picker grid rather than a pre-collected selection: `AddReference(QueryNames.SelectPeople_Items)` opens a selection grid and the chosen rows arrive in `OnAddReference(parent, entities, query, selectedItems)`. To survive a confirm round-trip (an `AddReference` followed by a `RetryAction`), stash the in-flight selection in `Manager.Current.User.Profile["key"]` and read it back with `.Profile["key"].FromServiceString<T>()`. Reach for the simpler transient-PO carrier above unless you specifically need that built-in picker grid.

## Declaring the rights

A wizard needs **two** security rights, not one — forgetting the first makes the button appear but do nothing:

- `New/<Schema>.<TransientPO>` — so the user is allowed to open the dialog PO (e.g. `New/Acme.RecalculateMonth`).
- `<ActionName>/<Schema>.<SourceType>` — the per-action right that makes the button visible on the source query/record (e.g. `RecalculateMonth/Acme.Company`).

Grant **both** to the action's group in `security.json`.

## Checklist

- [ ] Dialog declared in model JSON only — `"StateBehavior": 8`, `"NewBreadcrumb"`, attribute `Required`/`NotEmpty` rules; no backing C# entity; schema synchronized so `Types.*`/`AttributeNames.*` resolve.
- [ ] `Execute` returns `GetPersistentObject(Types.X, e.Parent?.ObjectId, isNew: true, targetContext: Context)`; `ignoreQueries: true` to suppress grids; assert with `EnsureParent`/`EnsureQuery`.
- [ ] `OnNew`: seed defaults, `obj.ActionLabels[ActionNames.Save] = "<verb>"`.
- [ ] `OnSave`: `if (!CheckRules(obj)) return;` FIRST, load aggregate from `obj.ObjectId`, run a **shared static** operation, `Context.SaveChanges()`, then `QueueClientOperation` / `QueueQueryRefresh`; no `base.OnSave`.
- [ ] Confirm-and-apply alternative: `if (Manager.Current.RetryResult is null) { RetryAction(...); return null; }`, then branch on `.Option` and read `.PersistentObject`; no `OnSave`.
- [ ] Bulk assign: ids in a hidden `MultiLineString`, cast back with `(string[])obj[attr]!` (no manual `Split`), filter against existing members.
- [ ] Rights: BOTH `New/<Schema>.<TransientPO>` and `<ActionName>/<Schema>.<SourceType>` in `security.json`.

## Verify

1. Build; run a schema synchronize (in CronosCore repos, `/dcg:synchronize`) so `Types.RecalculateMonth` resolves, and grant both rights.
2. As a member of the group, the action button shows on the `Company` record; clicking it opens a modal with the `NewBreadcrumb` header, the seeded default and the verb in place of Save.
3. Clear a required field and press the verb: the model-JSON rule message appears and RavenDB Studio shows nothing changed.
4. Fill it in and press the verb: the dialog closes, the refreshed grid shows the recomputed rows, and Studio has no collection named after the dialog type.
5. Negative: without the `New/<Schema>.<TransientPO>` right the button is visible but the dialog never opens; grant the right and retry.

## Related skills

- `/vidyano:custom_actions` — the action that opens the dialog; action rights and button visibility.
- `/vidyano:lifecycle_hooks` — the `OnNew`/`OnSave` save pipeline and `CheckRules`.
- `/vidyano:imports` — a paste-a-list intake wizard is the ingestion cousin of the bulk-assign shape.
