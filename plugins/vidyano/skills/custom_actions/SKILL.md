---
name: custom_actions
description: "Build a Vidyano custom action — a server-side verb (toolbar/context button) on a query or persistent object. Covers the CustomAction<TContext> class, syncing it into the model, the per-action right in security.json (granted to a group), Execute() return shapes (refresh, RetryAction dialog, RegisterStream), IsAllowed scoping, and common shapes (bulk status change, clone, AddReference, client-side triggers). PROACTIVE: invoke when the user adds or modifies a toolbar/context/menu action, mentions CustomAction, Execute, RetryAction, RegisterStream from an action, action rights/visibility, or \"add a button that…\"."
user-invocable: true
---

# Build a Vidyano Custom Action

A custom action is a server-side verb bound to a query or persistent object — a toolbar/context button whose `Execute` validates a selection or parent, mutates the model, calls an external system, or opens a follow-up form.

Not for: a multi-step input form before the verb runs (`/vidyano:wizards`), telling the browser to open/copy/refresh (`/vidyano:client_operations`), delivering a file (`/vidyano:exports_downloads`), or the built-in attach/detach buttons (`/vidyano:relationship_editors`).

## When to use

The user wants a button that does something server-side: approve/transition records, clone, attach references, trigger a client operation, or kick off a download/import (those last two delegate — see Related skills).

## The four moving parts (all required)

A working button needs all four, and the names must line up:

1. A `CustomAction<TContext>` class under `Service/CustomActions/` (the logic).
2. The action declared in the **synchronized Vidyano model** (display name, where it shows, options) — done in the management UI and synced into the model, **not** hand-edited into `Actions:[]` in `model.json`.
3. A per-action right `<Action>/<Module>.<Type>` in `App_Data/security.json`, granted to a group.
4. The action attached to the query/PO in the model so it renders.

The class name, the model action key, and the first segment of the security resource **must match exactly** (e.g. `ApproveOrder`). The second resource segment is the module-qualified entity (`Acme.Order`). After the synchronize step the source generator emits an `ActionNames.ApproveOrder` constant alongside `Types`/`QueryNames` — use it wherever code refers to the action, never the string literal.

## 1. The CustomAction class

Subclass `CustomAction<TContext>` (sync) and override `Execute`. For I/O-bound work subclass `AsyncCustomAction<TContext>` and override `ExecuteAsync`. The class is `partial` (a source generator emits the constructor that injects `Context`).

```csharp
namespace Acme.Service.CustomActions;

public partial class ApproveOrder : CustomAction<AcmeContext>
{
    public override PersistentObject? Execute(CustomActionArgs e)
    {
        e.EnsureParent(Types.Order);

        var order = Context.Session.Load<Order>(e.Parent.ObjectId);

        if (order.Status != OrderStatus.Submitted)
            return Notification(Messages.OrderNotApprovable, NotificationType.Error);

        order.Status = OrderStatus.Approved;
        order.ApprovedBy = Manager.Current.UserName;
        order.ApprovedOn = Manager.Current.NowOffset;
        Context.SaveChanges();

        e.Parent.AddNotification(Messages.OrderApproved, NotificationType.OK);
        return e.Parent;
    }
}
```

`Context.Session` is the RavenDB session; `Manager.Current` is the per-request ambient API (current user, settings, today). Keep one verb per class, one file under `Service/CustomActions/`.

### Reading the selection or parent

```csharp
// Single-record action (detail form): assert and read the parent PO.
e.EnsureParent(Types.Order);
var orderId = e.Parent.ObjectId;

// Grid action over a selection: assert the query, then iterate (batch-load to avoid N+1).
e.EnsureQuery();
var orders = Context.Session.Load<Order>(e.SelectedItems.Select(it => it.Id));
foreach (var order in orders.Values)
    order.Status = OrderStatus.Approved;
Context.SaveChanges();

// A split-button menu option arrives as a string:
if (e["MenuOption"] is "0") { /* ... */ }
```

`EnsureParent`/`EnsureQuery` throw a clear fault if invoked from the wrong place — no defensive null-checks needed.

## 2. The model declaration

Declared in the synchronized model with these knobs (run your Vidyano synchronize step after adding the action class so the model picks it up):

| Key | Meaning |
| --- | --- |
| `ShowedOn` | `1` = detail/PO toolbar, `2` = query (grid) toolbar (flags; combine for both). |
| `SelectionRule` | Row-count constraint for grid actions: `"=1"`, `">0"`, `"=0"`. Button stays disabled until met. |
| `RefreshQueryOnCompleted` | Refresh the originating query after the action runs. |
| `IsPinned` | Pin to the toolbar instead of the overflow menu. |
| `IconName` / `Icon`, `Offset`, `Intent` | Icon, sort order, visual emphasis. |
| `ConfirmationMessageKey` | Translation key for a simple "are you sure?" prompt (use `RetryAction` when you need input). |

Refine at runtime from the entity's `*Actions` class — e.g. grey out the button on certain rows in `OnLoad`:

```csharp
public override void OnLoad(PersistentObject obj, PersistentObject? parent)
{
    base.OnLoad(obj, parent);
    var order = GetEntity(obj)!;
    if (order.Status != OrderStatus.Submitted)
        obj.DisableActions(ActionNames.ApproveOrder);
}
```

## 3. The right in security.json

Each action gets its own resource `<Action>/<Module>.<Type>`, granted to group(s). **`UserId` holds a group GUID** (a key from the `Groups` map), not a user:

```json
{
  "Groups": {
    "a5a8585a-cfea-4b75-b62b-fcaf0e2b05f1": "Admins",
    "d3bd3312-0730-43d9-9bf4-9e14c75b00f7": "Users"
  },
  "Rights": [
    {
      "Id": "8b5695c2-e833-45bf-8f7c-413787e600f2",
      "Resource": "ApproveOrder/Acme.Order",
      "UserId": "a5a8585a-cfea-4b75-b62b-fcaf0e2b05f1"
    }
  ]
}
```

Add a second `Rights` entry with the same `Resource` and a different `UserId` to grant another group. A missing right is the most common reason a working action "doesn't appear" for a non-admin — also grant any right a dialog/picker it opens needs (`New` on the dialog PO, `Query` on a picker query).

## 4. Execute() return shapes

What you return decides what the client does next.

- **Refresh / notification:** return `null`, a `Notification(msg, type)` (the one static factory on the action base class), or the parent PO (re-render). For finer control queue a client operation — `Manager.Current.QueueClientOperation(new RefreshOperation(e.Parent))`; the full menu is `/vidyano:client_operations`. After mutating behind a form, reflect it without a reload: `e.Parent[AttributeNames.Order.Status].SetOriginalValue(OrderStatus.Approved.ToString())`.

- **Parameter dialog with RetryAction** — when `Manager.Current.RetryResult` is null this is the first call: register the dialog with `RetryAction` and return `null`. The client shows the popup and re-invokes the action with the answer in `RetryResult`, on which you branch:

```csharp
public override PersistentObject? Execute(CustomActionArgs e)
{
    if (Manager.Current.RetryResult is null)
    {
        var dialog = Manager.Current.GetPersistentObject(Types.ApproveOrderDialog, isNew: true);
        Manager.Current.RetryAction(
            Messages.ApproveOrderTitle, Messages.SelectApprovalReason,
            dialog, Messages.Approve, Messages.Cancel);  // title, message, PO, option buttons
        return null;
    }

    var retryResult = Manager.Current.RetryResult;
    if (string.Equals(retryResult.Option, Messages.Approve, StringComparison.Ordinal))
    {
        var reason = (string)retryResult.PersistentObject[AttributeNames.ApproveOrderDialog.Reason]!;
        e.EnsureParent(Types.Order);
        var order = Context.Session.Load<Order>(e.Parent.ObjectId);
        order.Status = OrderStatus.Approved;
        order.ApprovalReason = reason;
        Context.SaveChanges();
        return Notification(Messages.OrderApproved, NotificationType.OK);
    }
    return null; // cancelled
}
```

`RetryAction` is also the idiomatic Yes/No confirm for an irreversible step (two option buttons, no PO, branch on `retryResult.Option`). A dialog with several steps or its own save logic is a `/vidyano:wizards`.

- **Stream a file:** return `Manager.Current.RegisterStream(parent, key)` and produce bytes in the entity's `OnGetStream`. Full recipe in `/vidyano:exports_downloads`.

## 5. Scoping with IsAllowed

`IsAllowed` is a server-side gate on *where* the action may run (independent of the right, which controls *who* sees it):

```csharp
public override bool IsAllowed(PersistentObject? parent, Query? query)
    => query?.Name == QueryNames.Invoice.FailedInvoices;
```

## Common shapes

- **Bulk status transition:** `ShowedOn: 2`, `SelectionRule: ">0"`, `RefreshQueryOnCompleted: true`. Validate precondition, batch-load, flip status, `SaveChanges`, return a summary `Notification`. Where the transition must fire lifecycle rules, save *through* the PO rather than `Context.SaveChanges()` — see `/vidyano:lifecycle_hooks`.
- **Clone/duplicate:** load source `AsNoTracking`, null `Id` (RavenDB mints a fresh one), rename, reset state fields, `Store` + `SaveChanges`. Embedded value-object collections come along; attachments must be re-uploaded for fresh keys.
- **Attach/detach related records:** not a custom verb — the built-in `AddReference`/`Remove` query actions and their `OnAddReference`/`OnRemoveReference` hooks are covered in `/vidyano:relationship_editors`.
- **Client-side trigger:** compute a value, queue a client operation, return a notification:

```csharp
using Vidyano.Service.ClientOperations;

public override PersistentObject? Execute(CustomActionArgs e)
{
    e.EnsureParent(Types.Order);
    var url = Context.Session.Load<Order>(e.Parent.ObjectId).PortalUrl;   // the customer-facing link

    Manager.Current.QueueClientOperation(ExecuteMethodOperation.CopyToClipboard(url));
    return Notification(Messages.LinkCopied, NotificationType.OK);
}
```

The other factories (`OpenUrl`, `ShowMessageBox`, `ReloadPage`), `RefreshOperation`, `OpenOperation`, and the polling loop are in `/vidyano:client_operations`.

## Checklist

- [ ] `CustomAction<TContext>` class under `Service/CustomActions/` (one verb per file).
- [ ] Action synced into the Vidyano model (synchronize step), attached to the query/PO.
- [ ] Right `<Action>/<Module>.<Type>` in `security.json` granted to the right group(s) (plus any dialog/picker rights).
- [ ] `Execute` return shape matches the intended UX; preconditions validated; batch-load over selections.

## Verify

1. Build, run a schema synchronize (in CronosCore repos, `/dcg:synchronize`), and confirm the action appears in the model with the intended `ShowedOn`/`SelectionRule`.
2. Sign in as a member of the granted group: the button renders on the query or PO, disabled until the selection rule is met; sign in as a user outside the group and it is absent.
3. Run it: the entity changes as intended in RavenDB Studio and the notification renders in the right colour (green for `NotificationType.OK`).
4. Precondition case: run it on a record that fails the check and confirm the error notification shows and the document is unchanged.
5. `RetryAction` dialog: the first click opens the popup; Cancel leaves the data untouched; the confirm option applies the change.

## Related skills

- `/vidyano:lifecycle_hooks` — `OnLoad`/`OnSave`/`CheckRules`/`OnDelete`; saving through the PO so rules fire.
- `/vidyano:wizards` — a transient-PO dialog that gathers input before the verb runs.
- `/vidyano:client_operations` — `QueueClientOperation`: open URL, copy, message box, refresh, poll.
- `/vidyano:exports_downloads` — `RegisterStream`/`OnGetStream` for download actions.
- `/vidyano:imports` — `RegisterImport`/`OnImport` for upload actions and paste-a-list wizards.
- `/vidyano:relationship_editors` — the built-in `AddReference`/`Remove` actions.
- `/vidyano:row_security` — restricting *which records* an action may touch.
