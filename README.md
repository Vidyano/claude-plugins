# 2sky Claude Code plugins

Public [Claude Code](https://claude.com/claude-code) plugins by [2sky](https://www.2sky.be).

## Install

Add the marketplace, then install the plugin:

```
/plugin marketplace add 2sky/claude-plugins
/plugin install vidyano@2sky
```

Skills are then available as `/vidyano:<skill_name>` and also fire proactively when you work on the matching feature.

## `vidyano`

Usage patterns and recipes for the [Vidyano](https://www.vidyano.com) application framework — the C# backend and the web v4 (`@vidyano/vidyano` 4.x) frontend. For frontend work start at `/vidyano:web_components`, which routes to the other three frontend skills.

| Skill | Description |
|-------|-------------|
| `/vidyano:custom_actions` | Server-side toolbar/context/menu actions (`CustomAction<TContext>`), rights, and `Execute` shapes |
| `/vidyano:lifecycle_hooks` | Entity create/load/validate/save/delete hooks via `PersistentObjectActions` |
| `/vidyano:app_shell` | **(frontend)** `AppServiceHooks` overrides, custom `AppBase` shell, `vi-app-route` pages, dialogs, panes, icons |
| `/vidyano:attribute_presenters` | **(frontend)** Custom field editors and grid cells: `registerAttributeType`/`registerCellType` vs `vi-persistent-object-attribute-config` |
| `/vidyano:business_rules` | Declarative validation: the `<App>BusinessRules` class, `CheckRules`, uniqueness/referential-integrity guards |
| `/vidyano:client_operations` | Trigger the browser from a server action via `QueueClientOperation` (open URL, copy, message box, refresh, self-polling) |
| `/vidyano:clone_merge_erase` | Clone, merge duplicates, or GDPR-erase aggregates over a session (no FK cascade; repoint-before-delete; provenance) |
| `/vidyano:conditional_attributes` | Drive `IsVisible`/`IsReadOnly`/rules from sibling field values; refresh-on-change |
| `/vidyano:custom_pages` | **(frontend)** Replace a whole presenter with a Lit component via `vi-persistent-object-config` / `vi-query-config` templates |
| `/vidyano:data_migrations` | Versioned startup data migrations with Raven.Migrations (`[Migration]`, `MigrationRunner`) |
| `/vidyano:detail_queries` | Parent-scoped detail grids: `Custom.<Method>` query source + `CustomQueryArgs.Parent`, `EnsureParent`, `Empty<T>.Queryable` |
| `/vidyano:exports_downloads` | Deliver files to the browser: export/print via `RegisterStream` + `OnGetStream` |
| `/vidyano:file_storage` | Store/serve files as RavenDB attachments (`BinaryFile`, `Attachments.Store`, SkiaSharp resize) |
| `/vidyano:imports` | Ingest external data and files: `RegisterImport` + `OnImport`, paste-a-list wizards, uploads |
| `/vidyano:map_reduce_indexes` | Hand-written aggregating indexes: map/reduce + multi-map, `OutputReduceToCollection` |
| `/vidyano:query_models` | Denormalized `V*` read models over indexes (`[FromIndex]`) with reroute-to-editable |
| `/vidyano:relationship_editors` | Built-in `AddReference`/`Remove` query actions + `OnAddReference`/`OnRemoveReference` hooks |
| `/vidyano:row_security` | Framework-general row filtering (`Where`/`GetEntity`/`ApplyDataSecurity`); owner/self-service scoping |
| `/vidyano:sequential_numbering` | Gapless, concurrency-safe human-facing numbers (optimistic-concurrency sequence document) |
| `/vidyano:web_components` | **(frontend, start here)** Router over the 13 extension points + the `vi-*` built-in catalog and build pipeline |
| `/vidyano:wizards` | Transient-PO modal dialogs to gather input/selection before an action (`RetryAction`, bulk-assign) |

Model changes (new entities, attributes, actions) need a schema synchronize before `Types.*` / `AttributeNames.*` resolve; most skills say where.

## Contributing

- Skills live in `plugins/<plugin>/skills/<skill_name>/SKILL.md` (`snake_case` directories), with optional `references/` and `scripts/`.
- Bump `version` in the plugin's `.claude-plugin/plugin.json` on every change — installs auto-update by version.
- The `vidyano` plugin is framework-level only: samples name Vidyano and RavenDB types, never a downstream framework's types, machine paths, or private repositories.

Questions: info@vidyano.com

## License

[MIT](LICENSE)
