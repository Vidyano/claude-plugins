---
name: business_rules
description: "Validate Vidyano input at the right layer: per-attribute rules in a public static <App>BusinessRules class (null-or-message, wired by name in Rules JSON, toggled from PreClient), cross-field and uniqueness checks in CheckRules via AddNotification/ValidationError, in-use delete guards throwing FaultException. PROACTIVE: invoke when the user validates input, adds a business rule, blocks a save/delete, enforces uniqueness, or mentions BusinessRules/CheckRules/ValidationError/AddNotification/FaultException. Not for visibility/read-only toggling (see /vidyano:conditional_attributes)."
user-invocable: true
---

# Business Rules & Validation

Validation in a Vidyano module is not one mechanism but three, each suited to a different scope: a single field's format, a record's cross-field consistency, and a record's relationship to others. Choosing the right layer keeps each rule where a reader expects to find it.

Not for: showing, hiding, or locking a field from a sibling's value (`/vidyano:conditional_attributes`), or the save/delete hook pipeline these rules plug into (`/vidyano:lifecycle_hooks`).

## When to use

The user validates input, adds a business rule, blocks a save/delete, enforces uniqueness, or mentions `BusinessRules`/`CheckRules`/`ValidationError`/`AddNotification`/`FaultException`/`Rules` JSON.

## Three layers of validation

```
   ┌──────────────────────────────────────────────────────────────────┐
   │ scope             where it lives                  fires            │
   ├──────────────────────────────────────────────────────────────────┤
   │ one attribute      <App>BusinessRules method       per field edit  │
   │ one record         CheckRules override              on save        │
   │ across records     CheckRules / OnDelete guard      on save/delete │
   └──────────────────────────────────────────────────────────────────┘
```

A rule about *one value* (is this a valid IBAN?) belongs in a named attribute rule. A rule about *one record* (does the start date precede the end date?) belongs in `CheckRules`. A rule about *this record versus others* (is this code already taken? is this lookup still in use?) needs a query: uniqueness stays in `CheckRules`, an in-use check goes in the lifecycle guard of the operation it protects.

## Layer 1 — named attribute rules: `<App>BusinessRules`

A module gathers its per-attribute rules in one `public static` class named after the module. Each rule is a `public static` method that **returns `null` on success or an error message on failure**:

```csharp
public static class AcmeBusinessRules
{
    public static string? IsDuration(string? value)
        => string.IsNullOrWhiteSpace(value) || TimeSpan.TryParse(value, out _)
            ? null
            : "{0} is not a valid duration.";

    public static string? IsValidIban(string? value)
        => string.IsNullOrWhiteSpace(value) || IbanChecksumPasses(value)
            ? null
            : "{0} is not a valid IBAN.";

    // ISO 7064 mod 97-10: rotate the first four characters to the end,
    // map letters to 10..35, and the remainder must be 1.
    private static bool IbanChecksumPasses(string iban)
    {
        var normalized = iban.Replace(" ", "").ToUpperInvariant();
        if (normalized.Length is < 15 or > 34 || !normalized.All(char.IsLetterOrDigit))
            return false;

        var remainder = 0;
        foreach (var c in normalized[4..] + normalized[..4])
        {
            var digits = char.IsDigit(c) ? (c - '0').ToString() : (c - 'A' + 10).ToString();
            foreach (var d in digits)
                remainder = (remainder * 10 + (d - '0')) % 97;
        }

        return remainder == 1;
    }
}
```

Two conventions make these work:

- **The return contract is null-or-message.** A non-null string fails validation and shows as the error. The framework substitutes `{0}` with the attribute's label (and `{1}`, `{2}`, … with rule arguments), so `"{0} is not a valid duration."` reads as *"IBAN is not a valid duration."* at the field.
- **The method name is the rule name.** `IsValidIban` is referenced as `"IsValidIban"`; no attribute or registration is needed beyond the framework discovering the class's public static methods.

Localize messages with the generated `Messages.*` resources or `Manager.Current.GetTranslatedMessage("Key", "{0}", arg)`.

**Wiring a rule to an attribute.** Declaratively, list the rule by name in the attribute's `Rules` string in its model JSON — semicolon-separated, alongside the built-ins:

```json
{
  "Name": "Iban",
  "Rules": "NotEmpty; IsValidIban; MaxLength(40)"
}
```

`NotEmpty` and `MaxLength(40)` are framework rules; `IsValidIban` resolves to your method. Order does not matter; all rules run and all failures are reported.

**Context-aware rules.** A rule that needs a sibling field takes a `BusinessRulesArgs` first parameter. `args.PersistentObject` gives the whole record, so the rule can read another attribute:

```csharp
public static string? RequiredWhenDkvWanted(BusinessRulesArgs args, DateTime? value)
{
    var wantsDkv = (bool?)args.PersistentObject[AttributeNames.FamilyMember.WantsDkv];
    return wantsDkv == true && value is null
        ? Messages.AttributeIsRequired(args.PersistentObject[AttributeNames.FamilyMember.DkvFrom].Label)
        : null;
}
```

`BusinessRulesArgs` exposes `PersistentObject` (read siblings via the indexer), `Attribute` (the field being validated — has `Label`, `IsValueChanged`), and `Context` (for data access).

**Toggling rules at runtime.** When whether a field is required depends on another field's value, don't hard-code the rule in JSON — add or remove it from `PreClient`, which runs on every round-trip and right before `OnSave`, so the rule is in force when validation runs. `AddRule` / `RemoveRule` set it unconditionally; `ToggleRule` takes a boolean:

```csharp
// require a reason only when the location is "external"
obj[AttributeNames.Booking.Reason]
    .ToggleRule(BusinessRuleNames.NotEmpty, locationType == LocationType.External);
```

This is the validation half of `/vidyano:conditional_attributes`, whose `Toggle(attribute, visible)` helper flips the required rule together with visibility and read-only state from the same `PreClient` call.

## Layer 2 — cross-field validation: `CheckRules`

When a rule spans several fields of one record — a date order, a range overlap, a total that must not exceed a cap — override `CheckRules`. It runs on every save and receives the typed entity. **Null-guard `entity` first:**

```csharp
protected override void CheckRules(PersistentObject obj, Type entityType, BudgetPeriod? entity)
{
    base.CheckRules(obj, entityType, entity);

    if (entity is null)
        return;

    // date order
    if (entity.StartDate >= entity.EndDate)
        obj.AddNotification(
            Messages.DateCheck(obj[AttributeNames.BudgetPeriod.EndDate].Label,
                               obj[AttributeNames.BudgetPeriod.StartDate].Label),
            NotificationType.Error);

    // overlap with a sibling period under the same parent
    var overlaps = entity.Parent.Periods
        .Where(p => p.Id != entity.Id)
        .Any(p => p.StartDate < entity.EndDate && entity.StartDate < p.EndDate);

    if (overlaps)
        obj.AddNotification(Messages.PeriodOverlaps, NotificationType.Error);
}
```

Report failures one of two ways, both blocking the save at `Error` severity:

- `obj.AddNotification(message, NotificationType.Error)` — a form-level banner; right for cross-field rules where no single attribute is at fault. (`NotificationType` also has `Warning`, `Notice`, `OK` for non-blocking messages.)
- `obj[attr].ValidationError = message` — highlights one attribute; right when a specific field is to blame. Often paired with a summary `obj.AddNotification(Messages.ValidationRulesFailed)`.

## Layer 3 — guarding deletes and uniqueness

Rules that compare a record to *other* records need a query against the session. Uniqueness belongs in `CheckRules` — it fires on save with the typed entity in hand, exactly like layer 2. An in-use guard lives in the lifecycle hook for the operation it protects, throwing `FaultException` to hard-stop it.

**Uniqueness** — reject a duplicate business key on save. Set the field error and throw:

```csharp
protected override void CheckRules(PersistentObject obj, Type entityType, ProcedureState? entity)
{
    base.CheckRules(obj, entityType, entity);

    if (entity is null)
        return;

    if (obj.IsNew && Context.ProcedureStates.Any(s => s.Code == entity.Code))
    {
        obj[AttributeNames.ProcedureState.Code].ValidationError =
            Messages.ProcedureState_CodeAlreadyInUse(entity.Code);
        throw new FaultException(Messages.ValidationRulesFailed);
    }
}
```

**In-use guard** — block deleting reference data still referenced elsewhere. Scan the consuming query-model index for live references and throw when any are found:

```csharp
public override void OnDelete(PersistentObject? parent, IEnumerable<ProcedureState> entities,
                              Query query, QueryResultItem[] selectedItems)
{
    var inUse = Context.Query<VProcedure, Procedures_Overview>()
        .Select(p => p.State)
        .Distinct()
        .ToArray();

    var blocked = selectedItems.Where(i => inUse.Contains(i.Id)).ToArray();
    if (blocked.Length > 0)
        throw new FaultException(
            Messages.ProcedureState_CouldNotDelete(
                string.Join(", ", blocked.Select(i => i[AttributeNames.ProcedureState.Code]))));

    base.OnDelete(parent, entities, query, selectedItems);
}
```

`FaultException` is the idiom here: a hard, immediate rejection of an operation that must not proceed, distinct from the soft `AddNotification` used for editable-form validation. These guards sit in `/vidyano:lifecycle_hooks`; the in-use scan reads a `/vidyano:query_models` index.

## Shared rules across modules

Both styles work — the rules engine treats a thrown `ValidationException` and a returned message identically. A per-module `<App>BusinessRules` method returns `string?`; CronosCore repos ship shared checksum rules (VAT, IBAN, digit-length, not-in-the-past) in `CronosBusinessRules`, whose methods throw `ValidationException` instead. Match whichever class you are adding to.

## Checklist

- [ ] One value → a named `<App>BusinessRules` `public static string? Method(...)`, null-or-message (`{0}` = label), wired by name in the attribute's `Rules` JSON.
- [ ] Add `BusinessRulesArgs` first parameter to read siblings; toggle from `PreClient` with `AddRule`/`RemoveRule`/`ToggleRule(BusinessRuleNames.X, cond)`.
- [ ] One record → `CheckRules(obj, entityType, entity)`, null-guard `entity` first; report via `AddNotification` (form) or `ValidationError` (field) — both block at `Error`.
- [ ] Across records → uniqueness in `CheckRules`; in-use delete guard in `OnDelete` throwing `FaultException`, scanning a query-model index for live references.

## Verify

1. Build; if you edited an attribute's `Rules` JSON, run a schema synchronize (in CronosCore repos, `/dcg:synchronize`) and confirm the rule name appears on the attribute in the model.
2. Type an invalid value in the field: the message shows under it with `{0}` replaced by the attribute label, and Save is refused until it is fixed.
3. Save a record that violates a `CheckRules` rule: the form banner (`AddNotification`) or field highlight (`ValidationError`) appears, and the document in RavenDB Studio is unchanged.
4. Toggled rule: flip the driver both ways; the field is required only in the intended state, and saving with it empty in that state fails.
5. Negative case: deleting a lookup that is still referenced shows the `FaultException` message and the document stays; deleting an unreferenced one succeeds.

## Related skills

- `/vidyano:conditional_attributes` — the `Toggle(attribute, visible)` helper that adds/removes rules at runtime from `PreClient`.
- `/vidyano:lifecycle_hooks` — where `CheckRules` and the `OnDelete`/`OnSave` guards live.
- `/vidyano:query_models` — the index the in-use delete guard scans for live references.
