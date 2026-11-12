---
name: custom_pages
description: "Replace Vidyano's generated rendering of a whole persistent object, query, tab, program-unit page or chart with a Lit component: the `<vi-*-config>` template family in a site's `index.html`, backed by a model-only persistent object carrying the `Queries`. PROACTIVE: invoke when the user builds a dashboard, landing page, card/tile/map/kanban view or bespoke editor, or mentions `vi-persistent-object-config`, `vi-query-config`, `persistentObject.queries` or `forwardObservers`. Not for one field (see /vidyano:attribute_presenters) or the app shell (see /vidyano:app_shell)."
user-invocable: true
---

# Custom Pages: Replacing a Whole Presenter

A dashboard is not a page you add to a Vidyano app. It is a **persistent object whose rendering you replaced**. The server still owns the object — its attributes, its queries, its actions, its security. You are supplying a different set of pixels for it.

That framing matters because it tells you where each half of the work goes. Data, counts, filtering defaults, and permissions are backend concerns. Layout, interaction, and visual grouping are the component's. If you find yourself fetching data in the component with raw HTTP, you have crossed the line — the object already carries what you need.

## When to use

A dashboard, a landing page, a card/tile/map/kanban/calendar view over a query, a bespoke editor screen, or any case where the generated tabs-and-groups form (or the grid) is the wrong shape. Also `<vi-app-setting>` per-site knobs.

Not for: changing how a single field renders (`/vidyano:attribute_presenters`), adding a button (`/vidyano:custom_actions` — that is server-side), or changing the app's nav/routes (`/vidyano:app_shell`).

## The five config elements

All extend `TemplateConfig<T>`, all go directly inside `<vi-app>` in `App_Data/Websites/<site>/index.html`, all take **one** optional `<template>`:

| Element | Matches on | `as` defaults to |
|---|---|---|
| `<vi-persistent-object-config>` | `type`, `id`, `objectId` | `persistentObject` |
| `<vi-query-config>` | `name`, `id`, `type` | `query` |
| `<vi-persistent-object-tab-config>` | `id`, `name`, `type`, `objectId` | `tab` |
| `<vi-program-unit-config>` | `name` | `programUnit` |
| `<vi-query-chart-config>` | `type` | `chart` |

```html
<vi-app label="MyApp" class="fit" theme-color="#4682B4" theme-accent-color="#009688">
  <vi-persistent-object-config type="Dashboard">
    <template>
      <my-dashboard persistent-object="[[persistentObject]]"></my-dashboard>
    </template>
  </vi-persistent-object-config>

  <vi-query-config name="Orders">
    <template>
      <my-order-cards query="[[query]]"></my-order-cards>
    </template>
  </vi-query-config>
</vi-app>
```

`type` is the model type name, `name` the query name — both as the server declares them. Get one wrong and there is no error: the default rendering appears, which reads as "my component is broken" when in fact it was never asked for. **Check the model JSON, don't guess the casing.**

Each `App_Data/Websites/<site>/` has its own `index.html`. An app often has several (`Default`, `App`, a customer portal); a config added to one does not exist in the others.

### Tab and chart configs

Same shape, one node smaller. A tab config replaces the body of one tab while the rest of the object stays generated:

```html
<vi-persistent-object-tab-config type="Order" name="Timeline">
  <template><my-order-timeline tab="[[tab]]"></my-order-timeline></template>
</vi-persistent-object-tab-config>
```

The model is `tab`: `tab.parent` is the persistent object, and on a query tab `tab.query` is the query. Scope it with `type` (and `objectId` for one instance) or it matches every tab of that name.

A chart config replaces the rendering of one chart type wherever a query declares it:

```html
<vi-query-chart-config type="Funnel">
  <template><my-funnel-chart chart="[[chart]]"></my-funnel-chart></template>
</vi-query-chart-config>
```

The model is `chart`, a `Vidyano.QueryChart`: `chart.query` is the parent query, `chart.type` the string matched on, and `await chart.execute()` returns the data points. The grid, filters and action bar around it stay generated.

### Per-site knobs

```html
<vi-query-config name="Products">
  <vi-config key="pageSize" value="50"></vi-config>
  <template><my-product-list query="[[query]]"></my-product-list></template>
</vi-query-config>

<vi-app-setting key="mapProvider" value="osm"></vi-app-setting>
```

`<vi-config>` children land on the config object as `config.configs["pageSize"]`; `<vi-app-setting>` is app-wide via `app.configuration.getSetting("mapProvider", "default")`. Prefer these over hard-coding a value that differs per site.

## The backend half

A dashboard has no entity behind it. It is a **model-only persistent object** — declared in the model JSON, not generated from a C# type:

```jsonc
// App_Data/Model/MyApp/Dashboard.json
{
  "PersistentObject": {
    "Type": "Dashboard",
    "SchemaName": "MyApp",
    "IsReadOnly": true,
    "Attributes": [ { "Name": "MineOnly", "DataType": { "$ref": "DataType:Boolean" }, … } ],
    "Queries": [ "037df888-…", "2963b745-…", … ]   // the data the page renders
  }
}
```

The `Queries` list is the point: the page's data arrives as `persistentObject.queries`, already secured and already shaped by whatever `V*` query models back them (`/vidyano:query_models`). Attributes carry page-level state — a filter toggle, a selected period.

Behaviour goes in the matching actions partial:

```csharp
// <Project>/Service/Actions/DashboardActions.cs
public partial class DashboardActions
{
    public override void OnConstruct(PersistentObject obj)
    {
        base.OnConstruct(obj);
        // Counts are fetched by the component when it wants them, not eagerly with the object.
        obj.Queries.Run(q => q.IsCountIncludedInParentObject = false);
    }

    public override void OnLoad(PersistentObject obj, PersistentObject? parent)
    {
        base.OnLoad(obj, parent);
        if (!bool.TryParse(Manager.Current.User?.Profile["MineOnly"], out var mineOnly))
            mineOnly = true;
        obj.SetAttributeValue(AttributeNames.Dashboard.MineOnly, mineOnly);
    }
}
```

Run a schema synchronize (in CronosCore repos, `/dcg:synchronize`) after touching the model so `Types.*` / `AttributeNames.*` exist. See `/vidyano:lifecycle_hooks` for the hook surface.

## Writing the component in Lit

The stamped element is unconstrained — no base class required — so this is where Lit belongs. A card view over a query, stamped by `<vi-query-config name="Orders">`:

```ts
import { Vidyano } from "@vidyano/vidyano";
import { css, html, LitElement, nothing } from "lit";
import { customElement, property, state } from "lit/decorators.js";

@customElement("my-order-cards")
class OrderCards extends LitElement {
    static styles = css`
        :host { display: flex; flex-direction: column; min-height: 0; }
        .facets { display: flex; flex-wrap: wrap; gap: var(--theme-h4); padding: var(--theme-h3); }
        .chip[selected] { border-color: var(--theme-color); background: var(--theme-color); color: white; }
        vi-scroller { flex: 1; min-height: 0; }
    `;

    @property({ type: Object }) query: Vidyano.Query;
    @state() private selectedStatuses = new Set<string>();

    #itemsObserver: Vidyano.ISubjectDisposer;

    updated(changed) {
        if (changed.has("query") && this.query) {
            this.#itemsObserver?.();
            this.#itemsObserver = this.query.propertyChanged.attach((_, args: Vidyano.PropertyChangedArgs) => {
                if (args.propertyName === "items")
                    this.requestUpdate();
            });

            if (!this.query.hasSearched && !this.query.isBusy)
                this.query.search();
        }
    }

    disconnectedCallback() {
        super.disconnectedCallback();
        this.#itemsObserver?.();
    }

    render() { /* … */ }
}
```

Four things in that snippet are the whole skill:

**1. The binding works, and here is why.** `query="[[query]]"` in a Polymer `<template>` is a *property* binding: Polymer assigns `element[dashToCamelCase("query")] = value`. Lit's `@property` accessor sees the assignment and schedules a render. Multi-word attributes convert as expected — `<my-product display-price="[[displayPrice]]">` binds into `@property({type: Boolean}) displayPrice`. Nothing special is needed on either side.

**2. You must subscribe to the model yourself.** This is the single biggest difference from existing Polymer components. Polymer's `forwardObservers: ["query.items", "selectedQuery.isBusy"]` wired deep model changes to re-render for free. **Lit has no equivalent.** `Vidyano.Query` and `PersistentObject` mutate in place — `items` fills in after `search()`, `isBusy` flips, attribute values change — and Lit sees none of it, because the object reference never changes. Attach to `propertyChanged`, call `requestUpdate()`, and dispose in `disconnectedCallback`. Symptom of getting this wrong: the page renders empty, then never updates, and works only if you happen to click something.

**3. Trigger the search.** A query handed to a config template has usually not searched yet. `if (!query.hasSearched && !query.isBusy) query.search();` in `updated` is the idiom. Guard both flags or you will fire duplicate round-trips on every render.

**4. `@state` needs a new reference.** `Set`/`Map`/array mutation does not trigger Lit. Build a fresh `Set` per toggle:

```ts
const next = new Set(this.selectedStatuses);
next.has(v) ? next.delete(v) : next.add(v);
this.selectedStatuses = next;
```

### Styling

- **Theme variables work** — `--theme-h1`…`--theme-h5` (40/32/22/12/6 px spacing scale), `--theme-color` and its `-light`/`-dark`/`-faint`/`-rgb` variants, `--theme-foreground`, `--theme-light-border`, `--theme-read-only`, `--theme-box-shadow`. Custom properties inherit through shadow boundaries. Use them instead of literals.
- **`layout` / `horizontal` / `vertical` / `flex` / `center-center` / `fit` classes do NOT work.** They live in Polymer `dom-module` style modules that a Lit shadow root cannot include. Write the flex rules in your own CSS. A Lit template carrying `class="product layout vertical"` has inert classes; it only lays out if its own CSS declares `display:flex; flex-direction:column`.
- Two ways to get CSS in: `static styles = css\`…\`` inline, or `import styles from "./x.css"` + `static styles = unsafeCSS(styles)` from a compiled `.scss`. The second needs `declare module '*.css'` in `global.d.ts` and `rollup-plugin-postcss` with `inject: false`.
- Nesting built-ins is encouraged — `vi-scroller`, `vi-button`, `vi-select`, `vi-spinner`, `vi-icon` all work fine inside a Lit template. See `${CLAUDE_PLUGIN_ROOT}/skills/web_components/references/builtins.md`.

### Rendering fields the standard way

Inside a custom page you can still delegate one field to the framework rather than reimplementing its editor and validation:

```html
<vi-persistent-object-attribute-presenter attribute="[[persistentObject.attributes.MineOnly]]">
</vi-persistent-object-attribute-presenter>
```

In Lit: `.attribute=${this.persistentObject?.attributes.MineOnly}`. This gives you the label, editor, dirty tracking and validation bubble for free — worth it for anything editable.

## Reading the model

| Want | Path |
|---|---|
| The page's queries | `persistentObject.queries` (array, ordered as in the model JSON) |
| One query by name | `persistentObject.queries.find(q => q.name === "X")` |
| Page-level state | `persistentObject.attributes.MineOnly.value` |
| Set page state | `await attribute.setValue(v)` — do not assign `.value` |
| Rows | `query.items` (sparse while loading — `.filter(Boolean)`) |
| A cell | `item.values.Status`, or `item.getValue("Status")` |
| Counts / busy | `query.totalItems`, `query.isBusy`, `query.labelWithTotalItems` |
| Open a row | `app.hooks.onOpen(item)` — keeps routing and caching intact |

`query.items` being sparse is not a bug: the grid virtualises, so unfetched pages read `undefined`. Always `.filter(Boolean)` before mapping.

## Overriding which config wins

Attribute matching (`type` / `name` / `objectId`) is declarative and covers nearly everything. When it cannot express the rule — pick a template by user role, by a value on the object, by feature flag — override the resolver instead of adding a wrapper component:

```ts
export class MyShellServiceHooks extends AppServiceHooksBase {
    getPersistentObjectConfig(po: Vidyano.PersistentObject, configs: PersistentObjectConfig[]) {
        if (po.type === "Dashboard" && !po.attributes.MineOnly.value)
            return configs.find(c => c.id === "dashboard-all");
        return super.getPersistentObjectConfig(po, configs);
    }
}
```

Which base you extend depends on the app element: under plain `<vi-app>` extend `AppServiceHooks` (the `App.hooks` setter rejects anything else); `AppServiceHooksBase` is only valid under your own `AppBase` subclass — see `/vidyano:app_shell`.

The same shape exists for `getQueryConfig`, `getAttributeConfig`, `getTabConfig`, `getProgramUnitConfig`, `getQueryChartConfig`. Wiring the hooks class is `/vidyano:app_shell`.

## Checklist

1. Model JSON declares the type (or the query exists) — confirm the exact `Type`/`name` string.
2. `*Actions` partial supplies defaults and query config in `OnConstruct`/`OnLoad`.
3. Schema synchronize run (in CronosCore repos, `/dcg:synchronize`); `Types.*`/`AttributeNames.*` resolve.
4. Component written in Lit; subscribes to `propertyChanged`, disposes on disconnect, triggers `search()`.
5. Imported from `wwwroot/web-components/app/app.ts` **with a `.js` extension**.
6. `<vi-*-config>` added to every `App_Data/Websites/<site>/index.html` that needs it.
7. Your repo's frontend build script (typically sass + `npm run build`) run; check `tsc` output, it does not fail the build.

## Verify

1. Run the frontend build and read the `tsc` output (a type error does not fail the bundle); hard-reload the site whose `index.html` you edited.
2. Open the object or query and inspect the DOM: your element sits where the generated form or grid used to be. Seeing the default rendering means the `type`/`name` string does not match the model JSON.
3. The page fills without a click: one search request in the network tab, then a re-render when `items` arrives through `propertyChanged`.
4. After a model change and the schema synchronize, `Types.*`/`AttributeNames.*` compile and the model-only object opens with its `Queries` populated.
5. Negative case: remove the `<vi-*-config>` from one site's `index.html`; that site falls back to the generated rendering while the other sites keep the custom page.

## Related

`/vidyano:web_components` (pick the extension point) · `/vidyano:attribute_presenters` (one field, not a page) · `/vidyano:app_shell` (nav, routes, hooks, dialogs) · `/vidyano:query_models` (the read model behind the queries) · `/vidyano:lifecycle_hooks` (`OnConstruct`/`OnLoad`) · schema synchronize and Playwright end-to-end tests are CronosCore concerns (in CronosCore repos, the dcg plugin's synchronize and playwright skills)
