---
name: conditional_attributes
description: "Drive an attribute's visibility, read-only, and required rule from a sibling field: one Toggle(attribute, visible) helper over Visibility/IsReadOnly/AddRule/RemoveRule, called from PreClient (every round-trip and before OnSave), TriggersRefresh on the driver, OnRefresh to clear stale values. PROACTIVE: invoke when a field should appear/hide/lock/become required based on another field, or the user mentions PreClient/OnRefresh/TriggersRefresh/Visibility/IsReadOnly/AttributeVisibility or a Type-driven form. Not for the validation rules themselves (see /vidyano:business_rules)."
user-invocable: true
---

# Conditional Attribute Behavior

Forms are rarely static. Tick "Has tetanus vaccination" and a date field should appear; choose a `Type` of `Shareholder` and a different set of fields becomes relevant than for `ProfitLoss`; mark a billing period `Excluded` and everything downstream should grey out. The rule is always the same shape: **one driver attribute decides the visibility, read-only state, and required-ness of one or more dependent attributes.**

This is done server-side in lifecycle hooks, so the form stays coherent both when it first loads and on every subsequent change, without leaking the conditional structure to the client.

Not for: the validation rules themselves (`/vidyano:business_rules`), or changing how a field *renders* once visible (`/vidyano:attribute_presenters`).

## When to use

A field should appear/hide/lock/become-required based on another field; the user mentions `PreClient`/`OnRefresh`/`TriggersRefresh`/`Visibility`/`IsReadOnly`/`AttributeVisibility`; or the user builds a `Type`-variant (discriminated-union) form.

## The three knobs

Each attribute on a `PersistentObject` exposes three knobs you set from code:

```csharp
obj[attr].Visibility = AttributeVisibility.Always;   // or .Never
obj[attr].IsReadOnly = true;                         // lock without hiding
obj[attr].AddRule(BusinessRuleNames.NotEmpty);       // require it
obj[attr].RemoveRule(BusinessRuleNames.NotEmpty);    // stop requiring it
```

Read the driver's value with `FromServiceString<T>` (handles enums, dates, primitives). For a nullable bool use the `== true` form — **not** `(bool)obj[attr]!`, which throws on null:

```csharp
var type = obj[AttributeNames.CompanyOverride.Type].FromServiceString<CompanyOverrideType>();
var isVaccinated = (bool?)obj[AttributeNames.MedicalCard.IsTetanusVaccinated] == true;
```

Clearing a now-irrelevant value is `SetValue(null)`. That's the whole vocabulary; the rest is *where* to apply it.

## Where the logic lives

| Hook | Fires | Use for |
|---|---|---|
| `PreClient(PersistentObject obj)` | after `OnNew`, after `OnSelectReference`, on every round-trip to the client, and right before `OnSave` | the **structural state** — visibility, read-only, rules |
| `OnRefresh(RefreshArgs args)` | each time a `TriggersRefresh` attribute changes | **clearing values** the change made irrelevant |

The structural state belongs in **`PreClient`**, not `OnLoad`. `PreClient` is the one hook guaranteed to run on every path the form takes — initial load, a freshly created record (`OnNew`), a re-resolved reference (`OnSelectReference`), and every subsequent round-trip — so the conditional structure can never be lost in a round-trip the way it can if you only set it in `OnLoad`. It also runs **immediately before `OnSave`**, which is what makes a conditionally-required rule actually enforced at save time rather than only reflected in the UI.

Because `PreClient` is called this often, keep it cheap and idempotent: set visibility / read-only / rules and nothing else. Do **not** write attribute values or make database calls from it.

## A single toggle helper

One private helper translates "relevant or not" into the three knobs. Define it once per `*Actions` class and call it for every dependent — shown, editable, and required when `visible`; hidden and un-required otherwise:

```csharp
private static void Toggle(PersistentObjectAttribute attribute, bool visible)
{
    attribute.Visibility = visible ? AttributeVisibility.Always : AttributeVisibility.Never;
    attribute.IsReadOnly = false;

    if (visible)
        attribute.AddRule(BusinessRuleNames.NotEmpty);
    else
        attribute.RemoveRule(BusinessRuleNames.NotEmpty);
}
```

A second private method reads the driver and calls `Toggle` per dependent. It is the single source of truth for the conditional structure:

```csharp
private static void ToggleByType(PersistentObject obj)
{
    var type = obj[AttributeNames.CompanyOverride.Type].FromServiceString<CompanyOverrideType>();
    var isShareholder = type == CompanyOverrideType.Shareholder;

    Toggle(obj[AttributeNames.CompanyOverride.Holder],     isShareholder);
    Toggle(obj[AttributeNames.CompanyOverride.Ownership],  isShareholder);
    Toggle(obj[AttributeNames.CompanyOverride.ProfitLoss], type == CompanyOverrideType.ProfitLoss);
}
```

For a simple boolean driver there is nothing to switch on — one `Toggle` call from the flag:

```csharp
Toggle(obj[AttributeNames.MedicalCard.TetanusDate],
       (bool?)obj[AttributeNames.MedicalCard.IsTetanusVaccinated] == true);
```

## Applying it: `PreClient`

Call the driver-reading method from `PreClient` so the structure is correct on first paint, survives every round-trip, and is in force when `OnSave` validates:

```csharp
public override void PreClient(PersistentObject obj)
{
    base.PreClient(obj);
    ToggleByType(obj);
}
```

That single call covers load, `OnNew`, `OnSelectReference`, and pre-save — there is no separate "apply on load" step to keep in sync.

## Reacting to edits: `TriggersRefresh` + clearing in `OnRefresh`

For the form to react *as the user edits the driver*, the driver attribute must ask the client to call back to the server when it changes — set `TriggersRefresh` on it (in code during construction, or in the attribute's model JSON):

```csharp
obj[AttributeNames.CompanyOverride.Type].TriggersRefresh = true;
```

That callback is itself a round-trip, so `PreClient` re-runs and **re-applies the visibility/read-only/rules automatically** — you don't re-toggle the structure in `OnRefresh`. What `OnRefresh` is for is the one thing `PreClient` must not do: clearing the values the change just made irrelevant.

```csharp
public override void OnRefresh(RefreshArgs args)
{
    base.OnRefresh(args);

    if (args.Attribute.Name != AttributeNames.CompanyOverride.Type)
        return;

    // structure is handled in PreClient; here we only drop values
    // that no longer belong to the chosen type
    var type = args.PersistentObject[AttributeNames.CompanyOverride.Type]
        .FromServiceString<CompanyOverrideType>();

    if (type == CompanyOverrideType.ProfitLoss)
    {
        args.PersistentObject[AttributeNames.CompanyOverride.Holder].SetValue(null);
        args.PersistentObject[AttributeNames.CompanyOverride.Ownership].SetValue(null);
    }
}
```

`args.Attribute.Name` tells you which attribute the user touched; guarding on it keeps the cleanup from running on unrelated edits. (`RefreshArgs` exposes `.Attribute` and `.PersistentObject`.) Clearing hidden values matters: a field the user can no longer see should not silently keep a stale value into the save.

## The discriminated-union editor

The richest version of this pattern is a single PO that edits one of several variants chosen by a `Type` attribute — an `Authentication` that is `OAuth`, `ApiKey`, or `Basic`, each with its own fields. It is exactly `ToggleByType` above with one `Toggle(..., type == Variant)` line per field: the chosen variant's fields are shown and required, the rest hidden and un-required. A discriminated-union editor is not a special feature, just the conditional-attribute pattern with the driver being a `Type` enum.

## Checklist

- [ ] Driver read in one method (`FromServiceString<T>()`; nullable bool via `== true`); every dependent set through `Toggle(attribute, visible)`.
- [ ] That method called from `PreClient`, not `OnLoad`; it writes no values and makes no DB calls.
- [ ] `TriggersRefresh = true` on the driver; `OnRefresh` guarded on `args.Attribute.Name` and only clears values (`SetValue(null)`) — `PreClient` re-applies the structure.

## Verify

1. Build, then open an existing record: the dependents match the stored driver value on first paint (no flash of the wrong field set).
2. Change the driver in the form: the browser round-trips (network tab shows the refresh call), the dependents flip, and any now-hidden field's value is cleared in the request that follows.
3. Save with a required dependent empty: the save is refused with the `NotEmpty` message; flip the driver so it is not required and the same save succeeds.
4. Negative case: remove `TriggersRefresh` on the driver and the form no longer reacts until Save — put it back.
5. Create via New: `PreClient` runs after `OnNew`, so a fresh record already shows the right dependents for the default driver value.

## Related skills

- `/vidyano:business_rules` — the named validation rules being toggled with `AddRule`/`RemoveRule`.
- `/vidyano:lifecycle_hooks` — the hook pipeline `PreClient`/`OnRefresh` sit in.
