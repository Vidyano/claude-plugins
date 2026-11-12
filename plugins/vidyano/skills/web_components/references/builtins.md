# Built-in `vi-*` Components

The `@vidyano/vidyano` 4.x source registers 133 `vi-*` tags (counted in `src/vidyano/web-components`). This is the ~100 you will actually meet, curated from that registry and grouped by how you'd meet them. Source of truth: `src/vidyano/web-components/<name>/<name>.ts` in your Vidyano checkout — the `@WebComponent.register({ properties: {…} }, "vi-tag")` block is the API. There is no upstream doc for these.

Read-only properties are marked **(ro)** — bind *out* of them, never into them.

## Composable controls

Nest these. They are themed, keyboard-accessible, and handle the app's sensitive-data mode. Reaching for a raw `<input>` or a hand-rolled dropdown is almost always a mistake.

| Tag | Purpose | Key properties |
|---|---|---|
| `vi-button` | Themed button | `label`, `icon`, `accent`, `inverse`, `disabled`, `busy`, `elevation` |
| `vi-checkbox` | Checkbox, or a radio with `radio` | `checked` (notifies), `label`, `radio`, `disabled` |
| `vi-toggle` | On/off switch, tri-state aware | `toggled` (notifies), `label`, `isNull` **(ro)**, `disabled` |
| `vi-select` | Dropdown with filtering, grouping, suggestions | `options`, `selectedOption` (notifies), `groupSeparator`, `keepUnmatched`, `readonly` |
| `vi-input-search` | Search box with clear affordance | `value` (notifies), `focused` (notifies), `autofocus` |
| `vi-masked-input` | Fixed-format input (times, codes, plates) | `format` (e.g. `__:__:__`), `separator`, `allowed`, `open`, `value` (notifies), `preserve` |
| `vi-date-picker` | Calendar / month picker | `selectedDate` (notifies), `minDate`, `maxDate`, `monthMode`, `zoom` |
| `vi-time-picker` | Clock-face time picker | `time`, `hours`, `minutes`, `state` |
| `vi-tags` | Tag chips, editable or read-only | `tags`, `input`, `readonly`, `sensitive` (defaults **true**) |
| `vi-icon` | Named or inline SVG icon | `name`, `source`, `unresolved` **(ro)** |
| `vi-spinner` | Busy indicator | `color` |
| `vi-popup` | Anchored floating panel | `open` **(ro, notifies)**, `placement` (default `bottom-start`), `openOnHover`, `openDelay`, `closeDelay`, `sticky`, `autoWidth`, `disabled` |
| `vi-popup-menu` + `vi-popup-menu-item` | Menu / context menu | `contextMenuOnly`, `openOnHover`, `placement`, `shiftKey`, `ctrlKey`, `autoWidth` |
| `vi-menu` / `vi-menu-item` | App-level navigation menu | — |
| `vi-sensitive` | Masks content when the app is in sensitive mode | `disabled` (set `disabled="[[!sensitive]]"`), `show` **(ro)** |
| `vi-marked` | Renders markdown (DOMPurify-sanitised) | — |
| `vi-file-drop` | Drop target wrapper | `dragOver` **(ro)**; fires drop events |
| `vi-alert` | Inline alert banner | — |

### Layout & measurement

| Tag | Purpose | Key properties |
|---|---|---|
| `vi-scroller` | The app's scroll container — use instead of `overflow: auto` | `atTop`/`atBottom`/`atStart`/`atEnd` **(ro)**, `outerWidth`/`outerHeight`/`innerWidth`/`innerHeight` **(ro, notify)**, `hovering`/`scrolling` **(ro)** |
| `vi-size-tracker` | Emits its parent's size on resize | `size` **(ro, notifies)**, `deferred`, `triggerZero`, `bubbles` |
| `vi-overflow` | Collapses children past the edge into a `…` popup | `hasOverflow` **(ro)**, `label` (default `…`), `icon`, `type` |
| `vi-list` | Virtualised list (`iron-list`) | `items`, `as`, `parentScroller` |
| `vi-sortable` | Drag-to-reorder (abstract; subclass it) | `group`, `handle`, `filter`, `draggableItems`, `enabled`, `isDragging` **(ro)** |
| `vi-connected-notifier` | Fires once the element is connected | `oneTime` |

> **`vi-list` vs `@lit-labs/virtualizer`.** Consuming apps use the latter (`virtualize()`) inside Lit components; `vi-list` wraps Polymer's `iron-list`. In a Lit component prefer the virtualizer — `vi-list`'s `as`/template model is Polymer templatize.

### ⚠ The `layout` classes are not available in Lit

`class="layout horizontal center-center flex"`, `wrap`, `justified`, `self-end`, `fit` — these come from **Polymer `dom-module` style modules** (`vi-flex-layout-style-module`, `vi-reset-css-style-module`, `vi-responsive-style-module`) which are pulled in with `<style include="vi-flex-layout-style-module">` inside a Polymer template. A Lit component's shadow root **cannot** include them, so the classes silently do nothing there.

The available classes, for when you *are* in a Polymer template: `layout`, `horizontal`, `vertical`, `horizontal-reverse`, `vertical-reverse`, `inline`, `wrap`, `wrap-reverse`, `flex`, `start`, `center`, `end`, `justified`, `center-justified`, `start-justified`, `end-justified`, `around-justified`, `center-center`, `self-start`, `self-center`, `self-end`, `self-stretch` — plus `fit` and `relative` from reset-css.

In Lit, write the flex rules yourself in the component's own CSS. A Lit component that renders `class="product layout vertical"` only lays out because its own stylesheet declares `display: flex; flex-direction: column` on `.product` — the `layout vertical` classes there are inert. Don't copy the classes; copy the CSS.

## Model presenters — the render tree

These *are* the generated UI. You normally replace one via a config element rather than instantiating it; the exception is nesting `vi-persistent-object-attribute-presenter` inside a custom page to render one field the standard way.

| Tag | Renders | Key properties |
|---|---|---|
| `vi-program-unit-presenter` | A program unit's landing page | `programUnit`, `error` |
| `vi-query-presenter` | A query page (header + grid/chart) | `queryId`, `query`, `loading` **(ro)**, `hasError` **(ro)** |
| `vi-query-items-presenter` | Swaps grid ↔ chart for a query | — |
| `vi-query-grid` | The data grid | `query`, `items`, `columns`, `pinnedColumns`, `rowHeight`, `asLookup`, `noSelection`, `canFilter`, `canReorder`, `noInlineActions`, `hasGrouping` |
| `vi-query-grid-gallery` | Image-gallery view of a query | — |
| `vi-query-chart-selector` | Chart-type picker | `query`, `types` |
| `vi-persistent-object-presenter` | A persistent-object page | `persistentObjectId`, `persistentObjectObjectId`, `persistentObject`, `loading` **(ro)**, `templated` **(ro)**, `error` |
| `vi-persistent-object` | Tabs + groups for one object | — |
| `vi-persistent-object-tab-bar` / `-item` | Tab strip | — |
| `vi-persistent-object-tab-presenter` / `vi-persistent-object-tab` | One tab's body | `tab`, `columns`, `groups`, `noScroll`, `noAutofocus` |
| `vi-persistent-object-group` | An attribute group box | `group`, `columns`, `label`, `noLabel` |
| `vi-persistent-object-attribute-presenter` | **One attribute: label + editor + validation.** The unit to nest in a custom page | `attribute`, `name`, `type`, `noLabel`, `editing`, `nonEdit`, `readOnly`, `disabled`, `bulkEdit`, `height`, `hasError` **(ro)** |
| `vi-persistent-object-attribute-label` | Just the label | — |
| `vi-persistent-object-attribute-edit` | The editor chrome (border, focus ring) a presenter wraps its input in | `attribute` |
| `vi-persistent-object-attribute-validation-error` | The error bubble | — |
| `vi-action-bar` | Pinned + overflow actions for a query or object (**Lit**) | `serviceObject`, `pinnedActions`/`unpinnedActions` **(computed)**, `noActions` **(ro)**, `hasCharts`, `canSearch`, `selectedItemCount` |
| `vi-action-button` | One action | `action`, `item`, `name`, `icon`, `pinned`, `overflow`, `noLabel`, `forceLabel`, `canExecute` **(ro)**, `disabled`, `hidden` |
| `vi-audit` | Audit-trail view | `persistentObject`, `query`, `groups`, `search`, `filter` |
| `vi-user` | Signed-in user chip + menu | `isSignedIn`, `userName`, `hasSensitive`, `hasUserSettings`, `hasProfiler`, `collapsed`, `appSensitive` |
| `vi-notification` | Inline notification strip for a service object | `serviceObject`, `type`, `text`, `hidden` |
| `vi-session-presenter`, `vi-sign-in`, `vi-sign-out`, `vi-error`, `vi-profiler` | Session / auth / diagnostics chrome | — |

## Attribute presenters, by DataType

Registered via `PersistentObjectAttribute.registerAttributeType(type, ctor)`. Re-registering a type replaces it app-wide — that is the mechanism in `/vidyano:attribute_presenters`.

| DataType | Tag |
|---|---|
| `String` | `vi-persistent-object-attribute-string` |
| `MultiLineString` | `vi-persistent-object-attribute-multi-line-string` |
| `MultiString` | `vi-persistent-object-attribute-multi-string` (+ `-item`, `-items`) |
| `TranslatedString` | `vi-persistent-object-attribute-translated-string` (+ `-dialog`) |
| `Numeric` | `vi-persistent-object-attribute-numeric` |
| `Boolean` | `vi-persistent-object-attribute-boolean` |
| `NullableBoolean` | `vi-persistent-object-attribute-nullable-boolean` |
| `DateTime` | `vi-persistent-object-attribute-date-time` |
| `Reference` | `vi-persistent-object-attribute-reference` |
| `KeyValueList` | `vi-persistent-object-attribute-key-value-list` |
| `DropDown` | `vi-persistent-object-attribute-drop-down` |
| `ComboBox` | `vi-persistent-object-attribute-combo-box` |
| `FlagsEnum` | `vi-persistent-object-attribute-flags-enum` (+ `-flag`) |
| `Image` | `vi-persistent-object-attribute-image` (+ `-dialog`) |
| `BinaryFile` | `vi-persistent-object-attribute-binary-file` |
| `CommonMark` | `vi-persistent-object-attribute-common-mark` |
| `Password` | `vi-persistent-object-attribute-password` |
| `Icon` | `vi-persistent-object-attribute-icon` |
| `User` | `vi-persistent-object-attribute-user` |
| `AsDetail` | `vi-persistent-object-attribute-as-detail` (+ `-row`, `-cell` — the cell is **Lit**) |

**No built-in presenter** for `Time` / `NullableTime`. Consuming apps supply their own (see `/vidyano:attribute_presenters`); if an app shows a raw TimeSpan string, that is why.

## Grid cell types

Registered via `QueryGridCell.registerCellType(type, ctor)`. `vi-query-grid-cell-default` handles everything unregistered.

| Type | Tag |
|---|---|
| *(fallback)* | `vi-query-grid-cell-default` |
| `Boolean`, `NullableBoolean`, `YesNo` | `vi-query-grid-cell-boolean` |
| `CommonMark` | `vi-query-grid-cell-common-mark` |
| `Image` | `vi-query-grid-cell-image` |

Grid internals you may need to recognise but should not instantiate: `vi-query-grid-row`, `-row-group`, `-column-header`, `-column-filter`, `-column-measure`, `-filters`, `-footer`, `-grouping`, `-select-all`, `-sortable`, `-configure-dialog(+ -column, -column-list)`, `-filter-dialog(+ -name)`.

## Dialogs

All extend `Dialog` (Polymer). Show with `await app.showDialog(new X(...))` → resolves to the dialog's result; `null`/`undefined` on cancel.

| Tag | Purpose |
|---|---|
| `vi-message-dialog` | Title + message + buttons. Prefer the helper: `app.showMessageDialog({ title, message, rich, actions })` → index of the chosen action |
| `vi-persistent-object-dialog` | Edit an object in a modal |
| `vi-persistent-object-wizard-dialog` | Multi-step object wizard (see `/vidyano:wizards`) |
| `vi-select-reference-dialog` | Pick row(s) from a lookup query |
| `vi-retry-action-dialog` | Server-driven retry prompt |
| `vi-streaming-action-dialog` (+ `-busy-indicator`) | Progress for a streaming action |
| `vi-persistent-object-attribute-image-dialog` | Full-size image viewer |
| `vi-query-grid-gallery-image-viewer` | Gallery lightbox |

For a transient message use `app.showAlert(text, type?, duration?)` — no dialog needed.

## Declarative config elements

Only valid as direct children of `<vi-app>` in `App_Data/Websites/<site>/index.html`. Each sets `slot="vi-app-config"` on itself, so ordering is free. All the `*-config` elements extend `TemplateConfig<T>`: they take an optional single `<template>` plus any number of `<vi-config key value>` children, and stamp with a model named by `as`.

| Tag | Targets | Attributes |
|---|---|---|
| `vi-app-config` | container (implicit) | — |
| `vi-app-setting` | a named string setting, read via `app.configuration.getSetting(key)` | `key`, `value` |
| `vi-config` | a key/value on the enclosing config | `key`, `value` |
| `vi-persistent-object-config` | a persistent object; `as` defaults to `persistentObject` | `type`, `id`, `objectId` |
| `vi-query-config` | a query; `as` defaults to `query` | `name`, `id`, `type`, `defaultChart`, `hideHeader`, `selectAll`, `rowHeight`, `fileDropAttribute`, `fileDropAction` |
| `vi-persistent-object-attribute-config` | one attribute; `as` defaults to `attribute` | `type`, `name`, `parentId`, `parentObjectId`, `noLabel`, `height`, `width` (the last two are JS expressions over `attr`) |
| `vi-persistent-object-tab-config` | one tab | `id`, `name`, `type`, `objectId`, `hideActionBar` |
| `vi-program-unit-config` | a program unit's landing page | `name` |
| `vi-query-chart-config` | a chart type | `type` |
| `vi-app-route` / `vi-app-route-presenter` | custom URL routes | `route`, `routeAlt`, `preserveContent`, `allowSignedOut`; `active`/`path` **(ro)** |
| `vi-icon` | registers a named icon from inline SVG | `name` |

Resolution order is overridable: `AppServiceHooksBase.getPersistentObjectConfig` / `getQueryConfig` / `getAttributeConfig` / `getTabConfig` / `getProgramUnitConfig` / `getQueryChartConfig` each receive the candidate list and pick. Override one when attribute-matching isn't expressive enough — see `/vidyano:app_shell`.

## Theme variables

Unlike the `layout` classes, CSS custom properties **do** cross shadow boundaries — so these are available in Lit components and are the right way to stay on-theme.

`<vi-app theme-color="#4682B4" theme-accent-color="#009688">` generates, as inline styles on the app element (`app-base.ts:441`):

| Property | From |
|---|---|
| `--theme-color` + `-light`, `-lighter`, `-dark`, `-darker`, `-faint`, `-semi-faint`, `-rgb` | `theme-color` |
| `--theme-accent-color` + the same seven suffixes | `theme-accent-color` |

Static, from `_vars.scss`:

| Property | Value | Use |
|---|---|---|
| `--theme-h1` … `--theme-h5` | 40 / 32 / 22 / 12 / 6 px | The spacing & control-height scale. Use these for padding and gaps instead of literals |
| `--theme-foreground` | `#333` | Body text |
| `--theme-light-border` / `--theme-dark-border` | `#ddd` / darkened `#EAEAEA` | Separators |
| `--theme-read-only` | `#f7f7f7` | Read-only field background |
| `--theme-box-shadow` | `0 0 4px 0 rgba(0,0,0,.5)` | Elevation |
| `--theme-color-error` / `-warning` / `-notice` / `-ok` | notification palette | Status colours |

## App shell & caching

| Tag / export | Purpose |
|---|---|
| `vi-app` | The app root (`App extends AppBase`) |
| `AppBase` API | `service`, `hooks`, `configuration`, `initialize`, `path`, `changePath(path, replaceCurrent?)`, `showDialog(dialog)`, `showMessageDialog(opts)`, `showAlert(text, type?, duration?)`, `redirectToSignIn()`, `redirectToSignOut()`, `activeElement` |
| `AppCacheEntry` + `-query`, `-persistent-object`, `-persistent-object-from-action` | Back-navigation state cache |
| `IconRegister` | `add(name, innerHTML)` / `add(element)` / `add(template)`, `load(name)`, `exists(name)`, `fetchIcon(name)`, `all()` |
| `window.VidyanoSettings.skipElements` | `string[]` of tags to never define — set before the bundle loads |
