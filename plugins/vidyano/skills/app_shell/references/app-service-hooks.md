# `AppServiceHooks` override catalog

The hooks class is the app's behavioural seam — ~30 overridable methods, no rendering. Which base you extend depends on the app class (see the SKILL.md trap): `AppServiceHooks` under plain `<vi-app>`, `AppServiceHooksBase` under your own `AppBase` subclass.

Two layers, one catalog. The first block lives on `AppServiceHooksBase`; the second on the concrete `AppServiceHooks` that `vi-app` installs. Under a custom shell extending `AppServiceHooksBase` you get the first block only — the second exists on `AppServiceHooks` because `vi-app`'s default behaviour (opening objects, navigating, client operations) lives there.

## On `AppServiceHooksBase`

| Concern | Method |
|---|---|
| Redirect to an external IdP instead of the built-in sign-in | `onRedirectToSignIn(keepUrl)`, `onRedirectToSignOut(keepUrl)` |
| Session expired mid-session | `onSessionExpired(): Promise<boolean>` — `false` suppresses the default dialog |
| Block navigation, or force sign-in per route | `onAppRouteChanging(newRoute, currentRoute): Promise<string>` — return a path to redirect, `null` to proceed |
| Confirm a destructive action | `onActionConfirmation(action, option): Promise<boolean>` |
| Wrap every action execution | `onAction(args): Promise<PersistentObject>` |
| Downloaded file arrives | `onGetStream(blob, filename)` |
| Streaming action progress | `onStreamingAction(action, messages, abort)` |
| Reference-picker UI | `onSelectReference(query): Promise<QueryResultItem[]>` |
| Server asks to retry | `onRetryAction(retry): Promise<string>` |
| Messages, notifications | `onMessageDialog(title, message, rich, ...actions)`, `onShowNotification(notification, type, duration)` |
| Programmatic navigation | `onNavigate(path, replaceCurrent?)` |
| Which template renders what | `getPersistentObjectConfig`, `getQueryConfig`, `getAttributeConfig`, `getTabConfig`, `getProgramUnitConfig`, `getQueryChartConfig` |
| Construct the model objects | `onConstructApplication(response)`, `onConstructQuery(service, query, parent?, asLookup?, maxSelectedItems?)` |
| Startup / landing | `onBeforeAppInitialized`, `onAppInitializeFailed(message)`, `onInitial(initial)` |
| Telemetry identity | `getTrackUserId()` |

## On the concrete `AppServiceHooks` (plain `vi-app`)

| Concern | Method |
|---|---|
| Intercept opening / closing an object | `onOpen(obj, replaceCurrent?, forceFromAction?)`, `onClose(parent)` |
| Server-pushed client operations (navigate, refresh, execute JS) | `onClientOperation(operation)` |
| Files dropped on a query | `onQueryFileDrop(query, name, contents): Promise<boolean>` |
| Telemetry | `onPersistentObjectActivated`/`Deactivated`, `onQueryActivated`/`Deactivated`, `onProgramUnitActivated`/`Deactivated` |

`onSessionExpired`, `onAction`, `onRedirectToSignIn`/`Out` are overridden again on the concrete class with the real `vi-app` behaviour — call `super.*` on the path you are not special-casing. These methods carry real default behaviour; swallowing it produces dead buttons and hung dialogs.

Source of truth: `src/vidyano/web-components/app-service-hooks/app-service-hooks-base.ts` and `app-service-hooks.ts` in your Vidyano checkout.
