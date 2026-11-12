---
name: web_components
description: "Router for customizing the Vidyano web v4 frontend: pick the smallest node of the generated render tree to replace, then load the deep skill for it. Read this first. PROACTIVE: invoke when the user changes how something looks or behaves on screen in a Vidyano app — dashboard, grid cell, attribute editor, chart, dialog, pane — or mentions `vi-*` tags, `wwwroot/web-components`, `WebComponentLit`, `registerAttributeType`, `vi-persistent-object-config` or the rollup build. Not for backend data shaping (see /vidyano:query_models, /vidyano:lifecycle_hooks)."
user-invocable: true
---

# Web Components: Choosing an Extension Point

A Vidyano app has a working UI before anyone writes a line of frontend code. The server ships a model; the client walks it and renders — `vi-program-unit-presenter` → `vi-query-presenter` → `vi-query-grid` → `vi-query-grid-cell`, or `vi-persistent-object-presenter` → `vi-persistent-object-tab` → `vi-persistent-object-group` → `vi-persistent-object-attribute-presenter` → `vi-persistent-object-attribute-string`.

So frontend work here is **never "build a page"**. It is: find the node in that tree that renders the thing you want to change, and replace exactly that node. Everything else keeps working — the action bar, the dirty tracking, the validation, the routing, the sensitive-data masking.

The cost of getting this wrong is asymmetric. Replace too small a node and you fight the parent's layout. Replace too large a node — the whole page, when you wanted one cell — and you inherit the maintenance of every feature you just cut out. **Pick the smallest node that contains your change.**

## When to use

The user wants to change how something *looks or behaves on screen* in a Vidyano app, or mentions `vi-*` tags, `wwwroot/web-components`, `WebComponentLit`, `registerAttributeType`, `registerCellType`, `<vi-persistent-object-config>`, `<vi-query-config>`, `<vi-app-route>`, `AppServiceHooks`, or the rollup/vulcanize build.

Read this skill to pick the extension point, then load the deep skill it routes to.

**Not for:** changes that are really about *data* — which columns exist, what the grid selects, server-side validation. Those are backend changes: see `/vidyano:query_models`, `/vidyano:lifecycle_hooks`, `/vidyano:business_rules`.

## Pick the node

| You want to change… | Replace | Mechanism | Skill |
|---|---|---|---|
| How **one attribute** renders/edits, everywhere that type appears | `vi-persistent-object-attribute-<type>` | `PersistentObjectAttribute.registerAttributeType("Time", X)` | `/vidyano:attribute_presenters` |
| How **one attribute** renders, on one type/name/parent only | same node, scoped | `<vi-persistent-object-attribute-config type\|name>` + `<template>` | `/vidyano:attribute_presenters` |
| How a value renders **in a grid cell** | `vi-query-grid-cell-default` | `QueryGridCell.registerCellType("Time", X)` | `/vidyano:attribute_presenters` |
| A **whole persistent-object page** (dashboard, custom editor) | `vi-persistent-object-presenter`'s content | `<vi-persistent-object-config type="X"><template>` | `/vidyano:custom_pages` |
| A **whole query page** (grid → cards, map, chart) | `vi-query-presenter`'s content | `<vi-query-config name="X"><template>` | `/vidyano:custom_pages` |
| One **tab** inside an otherwise normal object | `vi-persistent-object-tab`'s content | `<vi-persistent-object-tab-config>` | `/vidyano:custom_pages` |
| A **program unit's landing page** | `vi-program-unit-presenter`'s content | `<vi-program-unit-config name="X">` | `/vidyano:custom_pages` |
| A **query chart** type | chart body | `<vi-query-chart-config type="X">` | `/vidyano:custom_pages` |
| The **app shell** — nav, header, footer, custom routes | `vi-app` | `class App extends AppBase` + `<vi-app-route>` | `/vidyano:app_shell` |
| **Cross-cutting behaviour** — sign-in redirect, session expiry, action confirmation, open/close | nothing visual | `class X extends AppServiceHooksBase` | `/vidyano:app_shell` |
| A **modal or slide-in pane** | — (additive) | `class X extends Dialog` + `app.showDialog(...)` | `/vidyano:app_shell` |
| An **icon** (incl. `ProgramUnit_<Name>`) | — (additive) | `<vi-icon name="X">` in `<vi-app>`, or `IconRegister.add` | `/vidyano:app_shell` |
| A **built-in** you want gone entirely | any `vi-*` | `window.VidyanoSettings.skipElements` | below |

**Before writing any control, check `${CLAUDE_PLUGIN_ROOT}/skills/web_components/references/builtins.md`** — the ~100 built-ins you will actually meet, curated from the full registry (the 4.x source registers 133 `vi-*` tags). `vi-select`, `vi-popup`, `vi-masked-input`, `vi-scroller`, `vi-checkbox`, `vi-toggle`, `vi-date-picker`, `vi-tags`, `vi-sensitive`, `vi-spinner`, `vi-size-tracker` and `vi-overflow` all exist and are themed. Hand-rolling one of these is the single most common waste in Vidyano frontends.

## Which base class — and where you get no choice

The framework carries two authoring bases. **Use `WebComponentLit` for everything you can.** But four extension points inherit from Polymer-based classes, and the registries type-check against those classes, so Lit is structurally impossible there — not a style preference:

| Extension point | Base you must extend | Lit? |
|---|---|---|
| Stamped page / dashboard (`<vi-*-config><template>`) | none — any element | **Lit** |
| Leaf / presentational component (card, chart, stepper) | none | **Lit** |
| Pane / drawer *content* | none | **Lit** |
| Attribute presenter via `registerAttributeType` | `PersistentObjectAttribute extends WebComponent` | No — Polymer |
| Query-grid cell via `registerCellType` | `QueryGridCell extends WebComponent` | No — Polymer |
| Dialog via `app.showDialog` | `Dialog extends WebComponent` | No — Polymer |
| App shell | `AppBase extends WebComponent` | No — Polymer |

`PersistentObjectAttributeConstructor` is `new (...args: any[]) => PersistentObjectAttribute` (`persistent-object-attribute.ts`); `QueryGridCellConstructor` is the same shape. A `WebComponentLit` subclass does not satisfy either.

Two consequences worth internalising:

1. **A Polymer node can host a Lit child, and the bindings work.** Polymer's `attr="[[expr]]"` is a *property* binding: it assigns `node[dashToCamelCase(attr)] = value`, which a Lit `@property` accessor picks up and turns into a re-render. Proven in production: a Polymer page template stamps `<my-product product="[[product]]" display-price="[[displayPrice]]">`, and `my-product` is a `WebComponentLit` with `@property({type: Object}) product` / `@property({type: Boolean}) displayPrice`. So keep the Polymer shell thin and put the substance in Lit children.
2. **The attribute-config route sidesteps the Polymer requirement entirely.** `<vi-persistent-object-attribute-config>` stamps whatever element you name, so a Lit element can present an attribute — you just give up global-by-type registration in exchange. `/vidyano:attribute_presenters` covers that trade.

### Recognising legacy Polymer components

Nearly every existing component in consuming apps is Polymer. The tell is `@WebComponent.register({ properties: {...} }, "tag-name")` with a sibling `.html` template pulled in via `static get template() { return Polymer.html`<link rel="import" href="x.html">`; }`. Do not mix idioms inside one component, and do not port one opportunistically — a rewrite is a behaviour change on a file with no tests. New components go in Lit; existing ones stay as they are unless the task is explicitly to migrate.

## Where the files live

```
<Project>/wwwroot/
  web-components/
    app/app.ts            # the import barrel — every component must be reachable from here
    app/app.html          # (Polymer shells only) template appended to AppBase's
    <feature>/<feature>.ts
    <feature>/<feature>.scss   # compiled to .css beside it
  index.js                # rollup output — the only script the page loads
  global.d.ts             # declare module '*.css' — required for Lit's `import styles`
<Project>/App_Data/Websites/<site>/index.html   # <vi-app> + all the config elements
```

An app may host several sites under `App_Data/Websites/<site>/` — each with its own `index.html` and therefore its own set of config elements. Changing one does not change the others.

## The build

```bash
cd <Project>/
npm ci
find wwwroot -type f -name "*.scss" -print -execdir sh -c 'sass "{}:${1%.scss}.css"' _ {} \;   # scss → css, in place
tsc --project ./tsconfig.json                                                                  # ts → js, in place (no outDir)
npm run build                                                                                  # rollup → wwwroot/index.js
```

Your repo's frontend build script (typically sass + `npm run build`) is exactly this. Four things about it bite:

- **`tsc` emits beside the source, with no `outDir`.** So intra-project imports must carry a **`.js` extension** even from `.ts` files: `import "../dashboard/dashboard.js"`. Omit it and rollup cannot resolve the module. Package imports (`@vidyano/vidyano`, `lit`) take no extension.
- **Rollup's input is `app.js`, not `app.ts`.** Rollup never sees TypeScript. A type error does not fail the build (`noEmitOnError: false`) — it fails at runtime instead. Read `tsc` output.
- **The `vulcanize` rollup plugin inlines Polymer's `<link rel="import" href="x.html">` and `<link rel="stylesheet" href="x.css">` into the JS at bundle time.** That is why Polymer components can reference sibling HTML/CSS files at all. It only applies to paths containing `web-components`.
- **Lit styles come through `rollup-plugin-postcss` with `inject: false`**, which turns `import styles from "./x.css"` into a string for `unsafeCSS(styles)`. This needs `declare module '*.css'` in `global.d.ts` *and* the postcss plugin in `rollup.config.js`. Check both are present in the consuming app repo before writing the first Lit component.

Anything not reachable from `app/app.ts` is not in the bundle and will silently not exist. Registering a component is two steps, and the second is easy to forget:

```ts
// wwwroot/web-components/app/app.ts
import "@vidyano/vidyano";
import "../dashboard/dashboard.js";                 // 1. pull it into the bundle
import "../timespan-presenter/timespan-presenter.js";
```

```html
<!-- App_Data/Websites/Default/index.html -->
<vi-app label="MyApp" theme-color="#4682B4">
  <vi-persistent-object-config type="Dashboard">   <!-- 2. tell Vidyano where to use it -->
    <template><my-dashboard persistent-object="[[persistentObject]]"></my-dashboard></template>
  </vi-persistent-object-config>
</vi-app>
```

Step 2 is unnecessary only for components that register themselves into a registry (`registerAttributeType`, `registerCellType`) or that are nested inside another component's template.

## Removing a built-in

`customElements.define` is monkey-patched (`web-component.ts`, `web-component-lit.ts`): a tag listed in `window.VidyanoSettings.skipElements` is never defined. Set it **before** the bundle loads.

```html
<script>window.VidyanoSettings = { skipElements: ["vi-profiler"] };</script>
<script src="index.js" type="module"></script>
```

This is a blunt instrument — the element simply does not upgrade, and any parent that expects it gets an inert `<div>`-alike. Reach for it only to suppress something you cannot otherwise disable; to *replace* behaviour, use one of the other twelve rows of the Pick-the-node table above.

## References

- `${CLAUDE_PLUGIN_ROOT}/skills/web_components/references/builtins.md` — the `vi-*` catalog: tag, what it does, the properties you'd actually bind. **Consult before building any control.**
- Grounding source is `src/vidyano/web-components/` in your Vidyano checkout (one folder per component: `.ts` required, `.scss` required, `.html` optional). The checkout's `docs/` covers **only** `src/core` — there is no upstream component-authoring doc, so the source is the contract.
- Consumers to copy patterns from: the consuming app repos your organisation already ships — look for dashboards and Lit charts, attribute + grid-cell presenters, a custom app shell with routes and panes — and the `TwoSky.WebComponents.*` NuGet packages (reusable presenters).

## Verify

1. Run the frontend build (sass → `tsc` → `npm run build`) and read the `tsc` output — a type error does not fail the bundle.
2. Open the site whose `index.html` you edited; the replaced node renders your element (inspect the DOM: your tag sits where the `vi-*` node used to).
3. The framework chrome around it still works — action bar, dirty tracking, validation, routing back and forth.
4. Other sites under `App_Data/Websites/` are unchanged unless you edited their `index.html` too.

## Related

`/vidyano:attribute_presenters` · `/vidyano:custom_pages` · `/vidyano:app_shell` · `/vidyano:query_models` (the backend read model behind a grid) · `/vidyano:custom_actions` (a server-side button, not a component) · Playwright end-to-end tests for the result (in CronosCore repos, the dcg plugin's playwright skill)
