---
name: app_shell
description: "Customize a Vidyano web v4 app at the shell level — `AppServiceHooks` overrides (sign-in, session expiry, action confirmation), a custom `AppBase` shell with `<vi-app-route>` pages, dialogs via `app.showDialog`, panes, `<vi-icon>` registration, `skipElements`. PROACTIVE: invoke when the user changes sign-in or session-expiry behaviour, adds a non-model page or URL, builds a public-facing site on Vidyano, adds a modal or pane, registers an icon, or mentions AppBase, AppServiceHooks, vi-app-route or showDialog. Not for how one object, query or field renders (see /vidyano:custom_pages, /vidyano:attribute_presenters)."
user-invocable: true
---

# App Shell: Hooks, Routes, Dialogs, Icons

Most Vidyano apps never need a custom shell. `vi-app` already gives you the menu, program units, routing to every query and object, session handling, notifications and dialogs. Two things genuinely justify shell work:

- **Cross-cutting behaviour** — how sign-in redirects, what happens on session expiry, confirming a destructive action, intercepting an open. That is `AppServiceHooks`, and it needs **no** custom app.
- **A different application shape** — a public-facing site with its own nav, its own routes, and pages that are not model objects at all (a customer-facing webshop, say). That is an `AppBase` subclass.

Reach for the first freely. Reach for the second only when the app genuinely isn't a back-office, because you take on the shell's maintenance permanently.

## When to use

Sign-in / session-expiry behaviour, action confirmation, intercepting open/close or client operations, a URL route that isn't a model object, a custom nav or footer, a modal dialog, a slide-in pane, a named icon, or suppressing a built-in element.

Not for: replacing how one object or query renders (`/vidyano:custom_pages`), one field (`/vidyano:attribute_presenters`), or a server-side button (`/vidyano:custom_actions`).

## Hooks without a custom app

`AppServiceHooksBase` is the app's behavioural seam — ~30 overridable methods, no rendering. Two ways to install your subclass:

**Deferred (no app subclass).** Set the attribute, then assign once the bundle has loaded:

```html
<vi-app label="MyApp" service-hooks="deferred" theme-color="#4682B4"></vi-app>
```

```ts
// wwwroot/web-components/app/app.ts
import { App, AppServiceHooks } from "@vidyano/vidyano";

class MyAppServiceHooks extends AppServiceHooks {   // note: AppServiceHooks, not …Base
    async onActionConfirmation(action: Vidyano.Action, option: number) {
        if (action.name === "Delete" && action.parent?.type === "Dossier")
            return await this.app.showMessageDialog({
                title: "Dossier verwijderen",
                message: "Dit kan niet ongedaan worden gemaakt.",
                actions: ["Verwijderen", "Annuleren"]
            }) === 0;

        return super.onActionConfirmation(action, option);
    }
}

(window.app as App).hooks = new MyAppServiceHooks();
```

**⚠ The base class you extend depends on the app class.** `App` (the concrete `vi-app`) has a setter that throws:

```ts
set hooks(value: AppServiceHooks) {
    if (value && !(value instanceof AppServiceHooks))
        throw new Error("App.hooks must be an instance of AppServiceHooks");
```

So under plain `<vi-app>`, extend **`AppServiceHooks`**. Extending `AppServiceHooksBase` there throws at startup with a message that reads like a framework bug. Under your own `AppBase` subclass there is no check, and `AppServiceHooksBase` is fine — which is why the `MyShellServiceHooks extends AppServiceHooksBase` passed to the custom shell's constructor below works.

Hooks can be assigned **once**; a second assignment logs `"AppBase hooks cannot be altered."` and is ignored. Without `service-hooks="deferred"`, `App.connectedCallback` installs a default `new AppServiceHooks()` before your script runs, so the attribute is what makes the timing work.

**Constructor (with an app subclass).** `super(new MyShellServiceHooks())` — see below.

### What to override

The most-used hooks: `onRedirectToSignIn`/`onRedirectToSignOut` (external IdP), `onSessionExpired` (return `false` to suppress the dialog), `onAppRouteChanging` (return a path to redirect, `null` to proceed), `onActionConfirmation`, `onAction`, `onOpen`/`onClose`, `onClientOperation`, `onGetStream`, and the `get*Config` resolvers that decide which template renders what. The full catalog — every method, its signature, and which of the two base classes carries it — is in `${CLAUDE_PLUGIN_ROOT}/skills/app_shell/references/app-service-hooks.md`.

Always call `super.*` on the path you are not special-casing. These methods carry real default behaviour — swallowing it produces dead buttons and hung dialogs.

Sign-in redirection to an external provider, the most common override:

```ts
export class MyShellServiceHooks extends AppServiceHooksBase {
    async onAppRouteChanging(newRoute: AppRoute, currentRoute: AppRoute): Promise<string> {
        if (newRoute != null && newRoute.route !== "" && this.service.isUsingDefaultCredentials) {
            Vidyano.cookie("returnUrl", this.app.path, { expires: 1, force: true });
            this.onRedirectToSignIn();
            return null;                               // we're navigating away; no in-app redirect
        }
        return super.onAppRouteChanging(newRoute, currentRoute);
    }

    async onRedirectToSignIn() {
        Vidyano.cookie("returnUrl", this.app.path, { expires: 1, force: true });
        window.location.assign(this.app.rootPath + "authenticate/sso");
    }

    async onSessionExpired(): Promise<boolean> {
        Vidyano.cookie("returnUrl", this.app.path, { expires: 1, force: true });
        this.onRedirectToSignIn();
        return false;                                  // we're navigating away; no dialog
    }
}
```

## A custom app shell

`AppBase extends WebComponent` (Polymer), so the app class cannot be Lit. Keep it thin — nav, routes, a handful of app-wide getters — and put substance in Lit children.

```ts
@WebComponent.register({
    properties: {
        shoppingCart: { type: Object, readOnly: true },
        userInfo: { type: Object, readOnly: true }
    },
    forwardObservers: [
        "shoppingCart.attributes.TotalItems.value",
        "userInfo.attributes.HasRewards.value"
    ],
    observers: ["_onServiceInitialized(service)"],
    listeners: { "app-route-activated": "_appRouteActivated" }
}, "ca")                                    // ← a PREFIX, not a tag name
export class App extends AppBase {
    static get template() {
        const baseTemplate = AppBase.template as HTMLTemplateElement;
        baseTemplate.content.appendChild(Polymer.html`<link rel="import" href="app.html">`.content);
        return baseTemplate;
    }

    constructor(hooks: AppServiceHooksBase = new MyShellServiceHooks()) {
        super(hooks);
    }

    async _onServiceInitialized(service: Vidyano.Service) {
        await this.initialize;                   // wait for the application model
        if (service != null && this.userInfo == null)
            this._setUserInfo(await service.getPersistentObject(null, "MyUserInfo"));
    }
}
```

Three mechanics in there:

**The second `register` argument is a prefix.** `register(info, "ca")` on class `App` produces `ca-app`. The site's `index.html` is therefore `<ca-app theme-color="#004368">` — **not** `<vi-app>`. A custom shell replaces the app element entirely; `vi-app` is never instantiated. The decorator walks the prototype chain merging `registerAbstract` info, so `AppBase`'s properties and observers are inherited.

**`static get template()` appends, it does not replace.** Take `AppBase.template`, append your own fragment, return the combined template. Your `app.html` renders inside the app's shadow root alongside the framework's chrome. Note that this mutates the shared `AppBase.template` — do it once, in one class.

**`await this.initialize`** resolves when the application model has loaded. Anything that touches `service.getPersistentObject` or program units must wait on it.

Config elements still work — they are light-DOM children of whatever the app element is, because `TemplateConfig` sets `slot="vi-app-config"` on itself and `AppBase`'s template exposes that slot:

```html
<ca-app theme-color="#004368">
  <vi-query-config name="Products"><template>…</template></vi-query-config>
</ca-app>
```

### Media queries

`AppBase` reflects `isPhone` / `isTablet` / `isDesktop`, usable directly in Polymer bindings (`<dom-if if="[[isPhone]]">`) and as host attributes for CSS. Prefer them over your own media queries so breakpoints stay consistent.

## Routes outside the model

`<vi-app-route-presenter>` holds the routes; each `<vi-app-route>` stamps its template when its pattern matches.

```html
<vi-app-route-presenter class="layout horizontal flex">
  <vi-app-route route="(product/:productId)" page="webshop" preserve-content>
    <template><ca-webshop></ca-webshop></template>
  </vi-app-route>
  <vi-app-route route="campaign/:campaignId(/product/:productId)" page="webshop" preserve-content>
    <template><ca-webshop></ca-webshop></template>
  </vi-app-route>
  <vi-app-route route="rewards(/product/:productId)" page="rewards" preserve-content>
    <template><ca-rewards></ca-rewards></template>
  </vi-app-route>
</vi-app-route-presenter>
```

- `:name` captures a segment; `( … )` marks an optional group. Read parameters from the activated route.
- **`preserve-content`** keeps the stamped content alive across matches of the same route, so navigating product → product does not rebuild the page (and does not lose scroll position or in-flight state). Without it, every parameter change restamps.
- `page` is not a framework concept — it is a plain attribute the shell reads to highlight nav:

```ts
private _appRouteActivated(e: CustomEvent) {
    this.set("page", (e.detail.route as AppRoute).getAttribute("page"));
}
```

- `allowSignedOut` lets a route render without a session.
- Navigate with `app.changePath("cart")` (or `changePath(path, true)` to replace history). Plain `<a href="rewards">` also works — `AppBase` intercepts clicks.
- Guard entry in `onAppRouteChanging`, not in the component.

## Dialogs

`Dialog extends WebComponent` (Polymer) — the dialog class cannot be Lit, though its body can host Lit children. It wraps a native `<dialog>`, so modality, focus trapping and Esc come free.

```ts
@WebComponent.register({
    properties: { label: String, sources: Array, source: String },
    keybindings: { "left": "_previous", "right": "_next" }
}, "ca-image-dialog")
export class ImageDialog extends Dialog {
    static get template() {
        return Dialog.dialogTemplate(Polymer.html`<link rel="import" href="image-dialog.html">`);
    }

    constructor(public label: string, ...sources: string[]) {
        super();
        this.source = sources[0];
    }
}
```

`Dialog.dialogTemplate(inner, { omitStyle? })` wraps your template in the dialog chrome. Pass data through the **constructor**, not attributes — you instantiate it yourself:

```ts
const picked = await app.showDialog(new ImageDialog("Product", ...urls));
if (picked === undefined) return;                 // cancelled
```

`showDialog` appends to the app's shadow root, awaits `open()`, and always removes the element afterwards. Inside the dialog, `close(result)` resolves with the result and `cancel()` resolves with nothing; Esc calls `cancel` unless `noCancelOnEscKey`. `noCancelOnOutsideClick` defaults to **true**. Dialogs are draggable by their `anchorTag` (default `header`).

For the common cases skip the subclass entirely:

```ts
const choice = await app.showMessageDialog({ title, message, rich: false, actions: ["OK", "Cancel"] });  // → index
app.showAlert("Saved", "Notice", 3000);                                                                  // transient toast
```

### Slide-in panes

Not a framework feature — an app-level pattern. Expose a `<slot name="panes">` in the shell template and manage the stack on the app:

```ts
async showPane<T>(pane: Pane): Promise<T> {
    pane.setAttribute("slot", "panes");
    this.appendChild(pane);
    openPanes.push(pane);
    const result = await pane.open();
    openPanes.remove(pane);
    pane.animateOut().then(() => this.removeChild(pane));
    return result as T;
}
```

The pane element itself is yours (a `ca-pane`, say), so its **content can be Lit**. Use panes rather than dialogs when the user should keep seeing context behind the panel.

## Icons

Declare inline SVG as a light-DOM child of the app element:

```html
<vi-app label="MyApp">
  <vi-icon name="ProgramUnit_Dashboard">
    <svg viewBox="0 0 24 24"><path fill="#888888" d="…"/></svg>
  </vi-icon>
</vi-app>
```

`ProgramUnit_<Name>` is picked up automatically as a program unit's menu icon — the cheapest way to brand the nav. Reference an icon anywhere with `<vi-icon source="my-icon">`.

Programmatic alternative: `IconRegister.add(name, innerHTML)` / `add(element)` / `add(template)`, plus `exists(name)`, `load(name)`, `fetchIcon(name)`, `all()`.

A global `vi-icon { display: none }` in `index.html`'s `<style>` hides unresolved icons during load — worth adding if icon pop-in is visible.

## Suppressing a built-in

```html
<script>window.VidyanoSettings = { skipElements: ["vi-profiler"] };</script>
<script src="index.js" type="module"></script>
```

`customElements.define` is patched in both bases (`web-component.ts`, `web-component-lit.ts`) and silently skips listed tags. Must be set **before** the bundle loads. The element never upgrades, so any parent expecting it gets an inert node — use it to suppress, not to replace.

## Checklist

1. Is a custom shell actually needed, or do hooks suffice? Prefer hooks.
2. Hooks: extend `AppServiceHooks` under plain `vi-app`, `AppServiceHooksBase` under your own `AppBase` subclass. Assign once. `service-hooks="deferred"` if not passing via constructor.
3. Every override calls `super.*` on the paths it does not handle.
4. Custom shell: `register(info, "<prefix>")`, append to `AppBase.template`, and change `index.html` to `<prefix-app>`.
5. Routes declared under one `<vi-app-route-presenter>`; `preserve-content` on any route whose parameters change during normal use.
6. Dialog data via constructor; `await app.showDialog(...)`; handle `undefined` as cancel.
7. `ProgramUnit_<Name>` icons registered for each program unit.
8. Imports in `app.ts` carry `.js`; run your repo's frontend build script (typically sass + `npm run build`); read `tsc` output — it does not fail the build.

## Verify

1. Build the frontend and read the `tsc` output; then hard-reload the site (the bundle is one file, `wwwroot/index.js`).
2. Hooks: the console shows neither `App.hooks must be an instance of AppServiceHooks` nor `AppBase hooks cannot be altered.` on startup, and the overridden path fires (sign out and back in, trigger the confirmed action).
3. Custom shell: `index.html` carries `<prefix-app>`, the page renders your nav plus the framework chrome, and a `<vi-*-config>` child still stamps.
4. Routes: navigate product → product; with `preserve-content` the page keeps state, and `app-route-activated` highlights the nav.
5. Dialogs: Esc resolves the awaited `showDialog` to `undefined`; a chosen action resolves the value passed to `close(result)`.

## Related

`/vidyano:web_components` (pick the extension point) · `/vidyano:custom_pages` (replace one presenter instead of the shell) · `/vidyano:attribute_presenters` · `/vidyano:client_operations` (the server side of `onClientOperation`) · `/vidyano:exports_downloads` (the server side of `onGetStream`) · `/vidyano:custom_actions` · SSO app tiles and Playwright end-to-end tests are CronosCore concerns (in CronosCore repos, the dcg plugin's sso_app_tiles and playwright skills)
