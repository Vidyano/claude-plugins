---
name: attribute_presenters
description: "Change how one value renders and edits in a Vidyano web v4 app — a form field via `<vi-persistent-object-attribute-config>` (Lit) or `registerAttributeType` (Polymer), a grid column via `QueryGridCell.registerCellType`. PROACTIVE: invoke when the user changes how a field looks or is edited, adds a presenter for an unsupported DataType (a raw TimeSpan on screen), formats or colours a grid column, or mentions registerAttributeType, registerCellType, QueryGridCellDefault, DisplayFormat or typeHints. Not for a whole page (see /vidyano:custom_pages) or for visibility/validation rules, which are server-side (see /vidyano:conditional_attributes)."
user-invocable: true
---

# Attribute & Cell Presenters

One value, two places it appears: an editable field on a form, and a read-only cell in a grid. They are separate components with separate registries, and customising one does **not** customise the other. Making `TimeSpan` display correctly takes both — that is the normal amount of work, not a sign of doing it wrong.

Before writing either, ask whether the problem is actually a presenter. A `DisplayFormat` typeHint set server-side, a `KeyValueList` instead of a free string, or the right `DataType` will often do the job with no frontend code at all. A presenter is for when no built-in DataType renders your value acceptably.

## When to use

An unsupported DataType renders as a raw string; a field needs a different editor than the framework picked; a grid column needs formatting, colour, alignment or a tag pill that typeHints can't express. Or the user mentions `registerAttributeType`, `registerCellType`, `<vi-persistent-object-attribute-config>`, `QueryGridCellDefault`, or `DisplayFormat` not being honoured.

Not for: a whole page or dashboard (`/vidyano:custom_pages`), a field that should only sometimes be visible or read-only (`/vidyano:conditional_attributes` — server-side), or validation (`/vidyano:business_rules`).

## First: which presenter is even consulted

`persistent-object-attribute-presenter.ts` normalizes `attribute.type` **before** looking anything up. Registering the un-normalized name is a silent no-op:

| `attribute.type` | Normalizes to |
|---|---|
| `Decimal`, `Single`, `Double`, `Int64`, `UInt64`, `Int32`, `UInt32`, … (+ `Nullable*`) | `Numeric` |
| `Date`, `DateTime`, `DateTimeOffset` (+ `Nullable*`) | `DateTime` |
| `YesNo` | `Boolean` |
| `YesNo` / `Boolean` **while `parent.isBulkEdit`** | `NullableBoolean` |
| `Enum` | `DropDown` |
| `Guid`, `NullableGuid` | `String` |
| `NullableUser` | `User` |
| anything else | itself, verbatim |

So `registerAttributeType("Decimal", …)` never fires — register `"Numeric"`. And note what is **absent**: `Time` and `NullableTime` are in neither the numeric nor the date-time set, so they pass through verbatim, find nothing registered, and hit the fallback:

```ts
new (PersistentObjectAttribute.getAttributeConstructor(attributeType) ?? PersistentObjectAttributeString)()
```

An unregistered type is rendered by the **string** presenter. That is why an untouched `TimeSpan` shows as `0:14:20:00.0000000` on screen — not a bug to hunt, a missing registration.

### Resolution order

1. A `<template>` passed as light DOM to the presenter element itself (rare; used by `vi-persistent-object-attribute-as-detail`).
2. **`<vi-persistent-object-attribute-config>` with a template** — `config.stamp(attribute, config.as || "attribute")`.
3. The type registry, else `PersistentObjectAttributeString`.

**The config template beats the registry.** That is the lever that makes Lit possible.

## Route A — config template (Lit-capable, scoped)

```html
<!-- App_Data/Websites/Default/index.html, inside <vi-app> -->
<vi-persistent-object-attribute-config type="Time">
  <template>
    <my-timespan-field attribute="[[attribute]]"></my-timespan-field>
  </template>
</vi-persistent-object-attribute-config>
```

```ts
@customElement("my-timespan-field")
class TimeSpanField extends LitElement {
    @property({ type: Object }) attribute: Vidyano.PersistentObjectAttribute;

    #observer: Vidyano.ISubjectDisposer;

    get #display(): string {
        return this.attribute.displayValue ?? "";           // what the model already formatted
    }

    #parse(input: string): string {
        return input.trim();                                 // convert the typed text to the attribute's wire format
    }

    updated(changed) {
        if (changed.has("attribute") && this.attribute) {
            this.#observer?.();
            this.#observer = this.attribute.propertyChanged.attach(() => this.requestUpdate());
        }
    }

    disconnectedCallback() {
        super.disconnectedCallback();
        this.#observer?.();
    }

    render() {
        if (!this.attribute)
            return nothing;

        return this.attribute.parent.isEditing
            ? html`<input .value=${this.#display} @change=${this.#commit}>`
            : html`<span>${this.#display}</span>`;
    }

    async #commit(e: Event) {
        await this.attribute.setValue(this.#parse((e.target as HTMLInputElement).value));
    }
}
```

Matching attributes: `type` (normalized name), `name` (the attribute's name), `parentId` / `parentObjectId` (scope to one object type or instance). Also on the config: `noLabel`, and `height` / `width` as **JS expressions over `attr`** (`width="attr.columnSpan"`), evaluated with `new Function`.

| | Config template | Type registry |
|---|---|---|
| Base class | none — **Lit welcome** | Polymer `PersistentObjectAttribute` |
| Reach | one site, filtered by type/name/parent | app-wide, every occurrence of the DataType |
| Declared in | each `App_Data/Websites/<site>/index.html` | one `registerAttributeType` call |
| Gets `editing`/`readOnly`/`sensitive` computed for you | **no** — read `attribute.parent.isEditing`, `attribute.isReadOnly`, `attribute.isSensitive` yourself | yes |
| Reusable as a NuGet plugin | no | yes (how the `TwoSky.WebComponents.*` packages ship theirs) |

**Pick the config route** when the customisation is app-specific, or when you want Lit. **Pick the registry** when you are filling a genuine framework gap that every screen should get, or shipping a reusable presenter.

## Route B — type registry (Polymer, app-wide)

`PersistentObjectAttributeConstructor` is `new (...args: any[]) => PersistentObjectAttribute` (`persistent-object-attribute.ts`), and `PersistentObjectAttribute extends WebComponent` — the Polymer base. A `WebComponentLit` subclass does not satisfy the type. This is a structural constraint of web v4, not a style choice; keep the presenter thin and delegate real rendering to a Lit child if it grows.

The base gives you these, all computed and kept current — do not recompute them:

| Property | Meaning |
|---|---|
| `attribute` | the `Vidyano.PersistentObjectAttribute` |
| `value` | mirror of `attribute.value`, notifies |
| `editing` | `!nonEdit && attribute.parent.isEditing` |
| `readOnly` | `attribute.isReadOnly \|\| disabled \|\| sensitive` |
| `readOnlyTabIndex` | bind to `tabindex$` so read-only fields leave the tab order |
| `sensitive` | attribute is sensitive **and** the app is masking |
| `required`, `frozen`, `placeholder`, `options` | from the model |
| `validationError`, `hasError` | reflected to attributes for styling |

Overridable hooks: `_attributeChanged`, `_valueChanged(new, old)`, `_editingChanged()`, `_optionsChanged`. `forwardObservers` already covers `attribute.displayValue`, `isRequired`, `isReadOnly`, `isSensitive`, `options`, `typeHints`, `validationError`, `parent.isFrozen`, `parent.isEditing` and `attribute.value`.

```ts
import { Polymer, Vidyano, PersistentObjectAttribute, WebComponent } from "@vidyano/vidyano";

@WebComponent.register({
    properties: {
        inputValue: { type: String, observer: "_inputValueChanged" },
        displayValue: { type: String, computed: "_computeDisplayValue(attribute.value, attribute, isConnected)" },
        inputFormat: { type: String, value: "__:__:__" }
    },
    forwardObservers: ["attribute.typeHints"]
}, "my-timespan-presenter")
export class TimeSpanPresenter extends PersistentObjectAttribute {
    static get template() {
        return Polymer.html`<link rel="import" href="timespan-presenter.html">`;
    }

    protected _valueChanged(newValue: any, oldValue: any) {
        // Deliberately NOT calling super — this presenter owns the conversion.
        …
    }

    protected _editingChanged() {
        super._editingChanged();
        if (this.editing) {
            Polymer.flush();                              // template must exist before we touch the input
            this._valueChanged(this.attribute?.value, null);
        }
    }
}

PersistentObjectAttribute.registerAttributeType("Time", TimeSpanPresenter);
PersistentObjectAttribute.registerAttributeType("NullableTime", TimeSpanPresenter);
```

Register **every** type variant. `Time` and `NullableTime` are separate keys; there is no nullable-folding for types outside the normalization table.

The template splits on `editing` and reuses framework chrome — do this rather than styling your own border and mask:

```html
<link rel="stylesheet" href="timespan-presenter.css" />

<dom-if if="[[!editing]]">
  <template>
    <vi-sensitive disabled="[[!sensitive]]"><span>[[displayValue]]</span></vi-sensitive>
  </template>
</dom-if>
<dom-if if="[[editing]]">
  <template>
    <vi-persistent-object-attribute-edit attribute="[[attribute]]">
      <vi-sensitive disabled="[[!sensitive]]">
        <vi-masked-input id="input" format="[[inputFormat]]" separator=":"
                         value="{{inputValue}}" disabled="[[readOnly]]"
                         tabindex$="[[readOnlyTabIndex]]"></vi-masked-input>
      </vi-sensitive>
    </vi-persistent-object-attribute-edit>
  </template>
</dom-if>
```

`vi-persistent-object-attribute-edit` supplies the focus ring, error state and read-only background. `vi-sensitive` with `disabled="[[!sensitive]]"` — note the inversion — masks the value when the user toggles sensitive mode.

### Writing values

```ts
this.attribute.setValue(value, false).catch(Vidyano.noop);   // false = don't trigger a server refresh
```

Always `setValue`; never assign `attribute.value`. Compare before writing — a `setValue` with an equal value still dirties the object, so a naive observer loop marks every opened record as modified. A TimeSpan presenter must normalize both sides before comparing because two different TimeSpan strings can mean the same instant.

Guard re-entrancy. Model → input → model is a live cycle:

```ts
private _valueChangedBlock = false;
// set true around programmatic input updates, check it at the top of the input observer
```

## Grid cells

Separate registry, separate base:

```ts
import { WebComponent, QueryGridCell, QueryGridCellDefault } from "@vidyano/vidyano";

@WebComponent.register({
    properties: {
        value: { type: Object, observer: "_valueChanged" },
        column: Object,
        right: { type: Boolean, reflectToAttribute: true },
        tag: { type: Boolean, reflectToAttribute: true }
    },
    sensitive: true
}, "my-timespan-query-grid-cell-presenter")
export class TimeSpanQueryGridCellPresenter extends QueryGridCellDefault {
    protected _valueChanged(itemValue: any, oldValue: any) {
        // Skip QueryGridCellDefault's own _valueChanged, keep QueryGridCell's typeHint handling.
        (QueryGridCell.prototype as any)._valueChanged?.call(this, itemValue, oldValue);
        …
    }
}

QueryGridCell.registerCellType("Time", TimeSpanQueryGridCellPresenter);
QueryGridCell.registerCellType("NullableTime", TimeSpanQueryGridCellPresenter);
```

Only `Boolean`/`NullableBoolean`/`YesNo`, `CommonMark` and `Image` have dedicated cells; everything else is `vi-query-grid-cell-default`.

**The trap worth knowing before you debug it.** `QueryGridCellDefault._valueChanged` normalizes a Time value *before* applying `DisplayFormat`: `trimEnd("0")`, `trimEnd(".")`, drop a leading `0:`, drop a trailing `:00`. So `01:48:00` becomes `1:48`, and a `{0:hh\:mm\:ss}` format then has nothing well-formed to work on. There is no way to opt out of the mangling — hence the grandparent call above, which keeps `QueryGridCell`'s common typeHint handling and replaces only the value logic.

`itemValue` is `{ item, column }`; get the raw value with `itemValue.item.getValue(itemValue.column.name)` — the pre-mangled truth.

TypeHints on a cell are read **lowercase** via `this._getTypeHint(column, "displayformat", null)`. Ones worth honouring, because the framework's own cells do and users expect consistency: `displayformat`, `extraclass` (space-separated classes), `tag` (sets `--tag-background`, renders as a pill), `horizontalcontentalignment` (numeric-ish types default `right`), `foreground`. Treat a `displayformat` of `"{0}"` as "no format".

Cells are recycled across rows as the grid virtualises. Every typeHint you apply must be **undone** when the value goes null — set a property and you own clearing it. Keep an `originalValue` per hint for exactly this. Skipping it produces the classic symptom: colours and alignment from one row bleeding onto another after scrolling.

## Wiring

Both registries are populated as an import side effect, so the import is the registration:

```ts
// wwwroot/web-components/app/app.ts
import "../timespan-presenter/timespan-presenter.js";                     // .js extension required
import "../timespan-presenter/timespan-query-grid-cell-presenter.js";
```

Nothing goes in `index.html` for the registry route. The config route needs its `<vi-persistent-object-attribute-config>` in **every** site's `index.html`.

Reusable presenters ship as NuGet packages — the `TwoSky.WebComponents.*` packages are the pattern: a `.targets` file copies `web-components/<name>/` into the consumer's `wwwroot/`, and an `IPlugin` class injects the web content. Follow their README if the presenter should be shared across projects rather than living in one app.

## Checklist

1. Confirm the **normalized** type name — is a presenter even consulted for it?
2. Choose route: config template (Lit, scoped) or registry (Polymer, global).
3. Registry route: register every type variant, including `Nullable*`.
4. Read/write via `attribute.value` / `attribute.setValue(v, false)`; compare before writing; guard re-entrancy.
5. Wrap editors in `vi-persistent-object-attribute-edit` + `vi-sensitive`; bind `tabindex$="[[readOnlyTabIndex]]"`.
6. Grid cell too, if the value appears in a grid — separate registry.
7. Cells: reset every typeHint you set when the value is null (recycling).
8. Imported from `app.ts` with `.js`; your repo's frontend build script (typically sass + `npm run build`) run; `tsc` output read.

## Verify

1. Build the frontend, read the `tsc` output, hard-reload.
2. Open an object carrying the attribute: read mode shows the formatted value, edit mode shows your editor inside the framework's focus ring, and toggling sensitive mode masks it.
3. Type a value, save: the object goes dirty only when the value actually changed, and reopening shows the persisted value.
4. Open a grid with the column: the cell is formatted, and scrolling a long list does not bleed colour or alignment onto other rows.

## Related

`/vidyano:web_components` (pick the extension point; `${CLAUDE_PLUGIN_ROOT}/skills/web_components/references/builtins.md` lists the built-in presenters and cells you will meet) · `/vidyano:custom_pages` (a whole screen instead) · `/vidyano:conditional_attributes` and `/vidyano:business_rules` (visibility, read-only, validation — server-side, usually the better fix) · per-language columns and labels are a CronosCore concern (in CronosCore repos, the dcg plugin's multilingual_content skill) · `/vidyano:query_models` (the column has to exist first)
