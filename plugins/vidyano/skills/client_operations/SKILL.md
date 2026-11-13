---
name: client_operations
description: "Make the browser act from a server-side action or hook via Manager.Current.QueueClientOperation: ExecuteMethodOperation factories (OpenUrl, Navigate, CopyToClipboard, ShowMessageBox, ReloadPage), OpenOperation, RefreshOperation, and delayed-refresh polling. PROACTIVE: invoke when an action must open a link, copy text, pop a dialog/toast, refresh or reload the screen, auto-refresh until done, or the user mentions QueueClientOperation/ExecuteMethodOperation/RefreshOperation/OpenOperation/NotificationType. Not for the action itself (see /vidyano:custom_actions)."
user-invocable: true
---

# Client-Side Operation Triggers

A custom action runs on the server. Most of the time its only job is to mutate the model and let Vidyano re-render — but sometimes the server needs to *tell the browser to do something*: open a URL, copy a value, pop a message box, navigate, or refresh a grid the action didn't touch. Those are **client operations**: small instructions the server queues during an action and the client replays when the response lands. All of this is stock Vidyano (`Vidyano.Service.ClientOperations` / `Vidyano.Service.Repository`) — no CronosCore API.

Not for: the action class, its right, and its return shapes (`/vidyano:custom_actions`); the browser-side handler that receives a custom operation (`/vidyano:app_shell`).

## When to use

The user wants an action to open a link, copy text, pop a dialog/toast, refresh or reload the screen, auto-refresh/poll a PO or query until done, or mentions `QueueClientOperation`/`ClientOperation`/`ExecuteMethodOperation`/`RefreshOperation`/`OpenOperation`/`AddNotification`/`NotificationType`/`OpenUrl`/`ReloadPage`.

## Two channels: the returned PO vs the operation queue

Two completely independent ways to push something to the client, and conflating them is the most common mistake:

1. **The returned `PersistentObject`.** Whatever the action returns becomes the client's focus — Vidyano opens it, or shows the **notification** attached to it. Return `null` and nothing is opened.
2. **The client-operation queue.** `Manager.Current.QueueClientOperation(op)` appends an operation attached to the response *regardless of what the action returns*.

Rule of thumb: a **notification** rides on the object you return; an **operation** is queued and is orthogonal to the return value. You will very often queue an operation and then `return null;`.

## Notifications: a message on the returned object

For a one-line "it worked" / "that failed" message, attach a notification to a PO and return it. The simplest source is the static helper on the action base class:

```csharp
public override PersistentObject? Execute(CustomActionArgs e)
{
    // ... do the work ...
    return Notification("The order has been confirmed.", NotificationType.OK);
}
```

`NotificationType` is `Error`, `Notice`, `OK`, `Warning`, or `Critical`. **The default is `Error`** — `Notification("done")` with no type renders as a red error toast. Always use `NotificationType.OK` for success.

To attach to a *real* object you're already returning:

```csharp
obj.AddNotification("Imported 42 rows, skipped 3 duplicates.", NotificationType.Notice);
return obj;
```

`AddNotification` is **additive and severity-sticky**, not last-wins: calling it twice concatenates the messages (newline-separated) and keeps the *highest* severity. A later `OK` will not downgrade an earlier `Warning`. A notification can carry a clickable link with `[url:Label|href]` markup (`href` absolute or an in-app route).

## Queuing a client operation

Everything else goes through one entry point. Because the operation rides on the response rather than the returned object, the canonical shape is *queue then return null*:

```csharp
public override PersistentObject? Execute(CustomActionArgs e)
{
    e.EnsureParent(Types.Order);
    var url = Context.Session.Load<Order>(e.Parent.ObjectId).TrackingUrl;   // the carrier's page for this order

    Manager.Current.QueueClientOperation(ExecuteMethodOperation.OpenUrl(url));
    return null;
}
```

Three concrete operation types. `ExecuteMethodOperation` is the workhorse — a named client method plus arguments, with static factories for each (prefer them over the raw `new ExecuteMethodOperation("methodName", arg)`, whose casing is easy to get wrong). `RefreshOperation` re-fetches a cached PO or query. `OpenOperation` opens a `PersistentObject` in the client.

## Open a URL, navigate, copy, message box

```csharp
// Inside Execute(CustomActionArgs e) on a Person action:
e.EnsureParent(Types.Person);
var person    = Context.Session.Load<Person>(e.Parent.ObjectId);
var reference = Guid.NewGuid().ToString("N")[..8].ToUpperInvariant();

// External link — opens in a new browser context.
Manager.Current.QueueClientOperation(ExecuteMethodOperation.OpenUrl(person.Website));

// In-app navigation — pushes a history entry; set `replace: true` to swap the current one instead.
Manager.Current.QueueClientOperation(ExecuteMethodOperation.Navigate("orders"));

// Clipboard — second arg is the confirmation toast (defaults to "Copied to clipboard!"). Needs a v4+ client.
Manager.Current.QueueClientOperation(
    ExecuteMethodOperation.CopyToClipboard(person.Iban, "IBAN copied to clipboard."));

// Modal dialog — rich:true renders HTML; an optional delay (ms) defers the popup.
Manager.Current.QueueClientOperation(
    ExecuteMethodOperation.ShowMessageBox(
        title:   "Booking confirmed",
        message: $"Your booking reference is <b>{reference}</b>. Quote it when you contact support.",
        rich:    true));
```

These also work from a lifecycle hook (`/vidyano:lifecycle_hooks`) like `OnLoad`, not just actions — anywhere the request flow is active. Prefer a **notification** for short transient feedback and a **message box** only when the user genuinely must acknowledge or copy something.

## Open another persistent object

To open a *different* aggregate than the one you return — e.g. an action on a draft that creates the real order and opens it:

```csharp
e.EnsureParent(Types.Draft);
var draft = Context.Session.Load<Draft>(e.Parent.ObjectId);

var order = new Order { Customer = draft.Customer, Lines = draft.Lines };
Context.Session.Store(order);
Context.SaveChanges();

var po = Manager.Current.GetPersistentObject(Types.Order, objectId: order.Id)!;
Manager.Current.QueueClientOperation(new OpenOperation(po));      // open it
// new OpenOperation(po, replace: true)  — replace the current screen instead of stacking
```

Use `OpenOperation.WithPreClient(po)` when the target needs its `PreClient` hook to run before it's shown (it computes client-only state); the plain constructor skips that hook. This is distinct from *returning* a PO — `OpenOperation` lets you open one thing while returning another (or `null`).

## Refresh and reload

After an action changes data the *current* screen doesn't reflect, two tools in order of preference:

**`RefreshOperation` — surgical.** Re-fetches a single cached PO or query if (and only if) the client has it loaded:

```csharp
// Inside Execute(CustomActionArgs e), after the action changed `order`:
var order = Context.Session.Load<Order>(e.Parent!.ObjectId);

Manager.Current.QueueClientOperation(new RefreshOperation(e.Parent));            // by PO
Manager.Current.QueueClientOperation(RefreshOperation.Query(QueryNames.Orders)); // by query name
Manager.Current.QueueClientOperation(new RefreshOperation(Types.Order, order.Id)); // by type + id
```

**`ReloadPage` — the hammer.** Forces a full reload. Reserve it for when the *set of visible actions* must be recomputed (an action that, once run, should disappear):

```csharp
// After cancelling, the Edit/Cancel actions must vanish — a targeted refresh won't recompute visibility.
Manager.Current.QueueClientOperation(ExecuteMethodOperation.ReloadPage());
```

A `RefreshOperation` is a no-op if the target isn't loaded client-side — safe, but it won't recompute action visibility. `RefreshOperation.Query(string)` throws on an ambiguous or unknown name.

## Polling: refresh with a delay

Every `RefreshOperation` overload takes an optional **`delay`** (`TimeSpan`). Because a refresh re-runs the target's load path, queuing a delayed refresh *from inside that very load path* makes the screen **poll itself** — and you stop the loop simply by not queuing once a terminal state is reached.

**A PO that polls itself** — a long-running `Job` keeps refreshing every 30s from `OnLoad` until it finishes:

```csharp
public override void OnLoad(PersistentObject obj, PersistentObject? parent)
{
    base.OnLoad(obj, parent);

    var job = GetEntity(obj)!;
    if (job.Status is JobStatus.Queued or JobStatus.Running)   // not finished — keep watching
        Manager.Current.QueueClientOperation(new RefreshOperation(obj, TimeSpan.FromSeconds(30)));
    // Succeeded / Failed: queue nothing, the loop ends.
}
```

**A query that polls itself** — a jobs grid refreshes every 30s *only while* at least one row is unfinished (the mechanism behind Vidyano's own auto-refreshing Logs query):

```csharp
public override void QueryExecuted(QueryExecutedArgs args)
{
    base.QueryExecuted(args);

    var anyRunning = args.Items.Any(it => it.GetValue(AttributeNames.Job.Status) is "Running" or "Queued");
    if (anyRunning)
        Manager.Current.QueueClientOperation(RefreshOperation.Query(args.Query, TimeSpan.FromSeconds(30)));
}
```

Near-live screens with no websocket or client timer: the cadence and its **stop condition** live server-side, next to the state they watch. Keep the interval in seconds (each tick is a round-trip) and make the terminal condition reachable, or the screen polls forever.

## Checklist

- [ ] Operation vs notification picked correctly: a queued operation is attached regardless of return value; a notification only shows on the **returned** object.
- [ ] Success notifications use `NotificationType.OK` (default is `Error`); remember `AddNotification` escalates severity and concatenates.
- [ ] `ExecuteMethodOperation` built via the static factories, not the raw constructor; usually followed by `return null;`.
- [ ] Refresh **surgically** with `RefreshOperation`; `ReloadPage()` only to recompute the visible action set.
- [ ] Polling: delayed `RefreshOperation` queued from `OnLoad`/`QueryExecuted` ONLY while a non-terminal condition holds; interval in seconds; stop condition reachable.
- [ ] Operations replay in queue order — queue a refresh before a navigate if both must happen. Tests stub the queue.

## Verify

1. Build, run the action from the UI: the browser does the queued thing (new tab opens, clipboard holds the value, dialog pops) and the response in the network tab carries a `clientOperations` array even though the action returned `null`.
2. Notification: a success message renders green — if it renders red, the `NotificationType.OK` argument is missing.
3. Refresh: change data behind an open grid, run the action, and the grid updates without a page reload; an action that should hide itself afterwards needs `ReloadPage()` instead.
4. Polling: open a running job and watch the network tab refresh it on the chosen interval; mark the job finished in RavenDB Studio and the next tick is the last one.
5. Negative case: `RefreshOperation.Query` with a misspelled query name throws at execute time — read the fault text.

## Related skills

- `/vidyano:custom_actions` — the action that emits these operations and returns the focused PO.
- `/vidyano:lifecycle_hooks` — `OnLoad`/`QueryExecuted`, where the self-polling delayed refresh is queued.
