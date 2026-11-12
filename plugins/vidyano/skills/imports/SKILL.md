---
name: imports
description: "Get external data and files into the app: spreadsheet/CSV import via RegisterImport + OnImport (parse-and-upsert), paste-a-list bulk wizards, large-file upload to blob storage, and async job-based ingestion. PROACTIVE: invoke when the user adds an import/upload action, mentions RegisterImport/OnImport, parses an uploaded spreadsheet, or bulk-creates records from pasted/external data."
user-invocable: true
---

# Imports & Uploads

The inbound counterpart to exports. Two recipes you write here, plus pointers for the two shapes that lean on infrastructure outside Vidyano. All are custom actions — see `/vidyano:custom_actions` and `/vidyano:lifecycle_hooks` for the basics.

Not for: delivering a file to the browser (`/vidyano:exports_downloads`), keeping an uploaded file on a record as an attachment (`/vidyano:file_storage`), or the button itself (`/vidyano:custom_actions`).

## When to use

| You have… | Use… |
|-----------|------|
| A spreadsheet/CSV, parse synchronously | `RegisterImport` + `OnImport` |
| A column of ids/codes the user will paste | Paste-a-list virtual PO |
| A large binary (PDF, image, archive) | Blob storage outside RavenDB — pointer in §3 |
| A slow import, or one needing an audit trail/review | Job document + background worker — pointer in §4 |

## 1. Spreadsheet/CSV import (RegisterImport + OnImport)

The dominant pattern. Three parts: a query action that calls `RegisterImport` (pops a file picker), a right that gates it, and an `OnImport` override that parses-and-upserts.

```csharp
using Vidyano.Service.Repository;

public partial class Import : CustomAction<MyAppContext>
{
    public override PersistentObject? Execute(CustomActionArgs e) =>
        Manager.Current.RegisterImport(e.Query!.PersistentObject, accept: ".xlsx");
}
```

`RegisterImport` returns a transient PO instructing the client to upload; when the upload arrives, the framework calls `OnImport` on the `Actions` class of the entity behind that query. Full signature:

```csharp
PersistentObject RegisterImport(
    PersistentObject obj, string? key = null, string? label = null,
    string? accept = null,            // ".xlsx", ".csv", or Excel.ContentType (comma-separated)
    bool refreshAfterSave = true,
    ImportArgs.CustomOption[]? customOptions = null);   // extra inputs (dry-run toggle, type dropdown)
```

Gate it with a right `<ActionName>/<Schema>.<Entity>` granted to a group (`UserId` is a group GUID). A user without it never sees the button.

### OnImport — parse and upsert

Lives on the entity's `PersistentObjectActions` (the `partial` half you own). Read the file, walk rows, resolve each to a master record, create-or-update:

```csharp
using Vidyano.Service.Actions;

public override void OnImport(ImportArgs e)
{
    var sheet = e.ReadAsExcelSheet();
    if (sheet is null)
        throw new FaultException(Messages.EmployeeActions_OnImport_CantReadFile);

    var imported = 0;
    var skipped = new List<string>();

    for (var r = 1; r < sheet.RowCount; r++)   // row 0 is the header
    {
        var companyNumber = sheet[r, 0];
        var email = sheet[r, 1];
        var firstName = sheet[r, 2];
        var lastName = sheet[r, 3];

        if (string.IsNullOrWhiteSpace(email)) continue;   // blank line

        // Excel strips leading zeros from numeric-looking columns; repair them.
        if (!string.IsNullOrWhiteSpace(companyNumber) && companyNumber.Length < 3)
            companyNumber = companyNumber.PadLeft(3, '0');

        var company = Context.Companies.FirstOrDefault(c => c.SubNumber == companyNumber);
        if (company is null) { skipped.Add($"{email} (unknown company {companyNumber})"); continue; }

        var employee = Context.Employees.FirstOrDefault(emp => emp.Email == email);
        if (employee is null) { employee = new Employee { Email = email, Company = company.Id }; Context.Session.Store(employee); }
        employee.FirstName = firstName;
        employee.LastName = lastName;
        imported++;
    }

    Context.SaveChanges();   // once, after the loop — not per row

    e.SetNotification(skipped.Count == 0
        ? $"Imported {imported} employees."
        : $"Imported {imported}; skipped {skipped.Count}: {string.Join(", ", skipped)}",
        NotificationType.OK);
}
```

Conventions:
- **Match keys vary by domain** — resolve by the most stable identifier (sub-number, VAT, email, code); fall back through several where there is no single key.
- **Don't hard-delete** — upsert and skip unresolved rows; report them rather than throwing on the first bad line.
- **Report the outcome** — `e.SetNotification` shows a toast but **rejects `NotificationType.Error`**; for a genuinely unreadable file, throw `FaultException` with a translated `Messages` key (the framework surfaces it and rolls back).
- **Persist once** after the loop.

`ImportArgs` members: `e.Key`, `e.BinaryFile`/`e.FileName`/`e.FileData`, `e.CustomOptions`, `e.ReadAsExcelSheet(sheetName?)`, `e.ReadAsString(encoding?)`, `e.SetNotification(...)`. For CSV, `e.ReadAsString()` (strip BOM/zero-width: `csv.Trim('\uFEFF', '\u200B')`) and feed your CSV reader.

**Reference entity with no query of its own:** register against a fresh PO instead. The framework's default import maps columns to attributes by name, so a custom `OnImport` is often unnecessary:

```csharp
return Manager.Current.RegisterImport(
    Manager.Current.GetPersistentObject(Types.Education, isNew: true, ignoreQueries: true)!,
    accept: Excel.ContentType);
```

**Thousands of rows:** `BulkInsert` is far faster and sidesteps the session request limit. Dedupe parents with a `HashSet`:

```csharp
public override void OnImport(ImportArgs e)
{
    var csv = e.ReadAsString().Trim('\uFEFF', '\u200B');

    using var bulkInsert = Context.Session.Advanced.DocumentStore.BulkInsert(
        new BulkInsertOptions { SkipOverwriteIfUnchanged = true });
    var seenCompanies = new HashSet<string>();
    foreach (var line in CsvReader.ReadFromText(csv))   // Csv NuGet package; line[header] reads a cell
    {
        var companyId = "companies/" + line["company_number"];
        bulkInsert.Store(new Employee { Id = "employees/" + line["employee_id"], Email = line["email"], Company = companyId });
        if (seenCompanies.Add(companyId)) bulkInsert.Store(new Company { Id = companyId, Name = line["company_name"] });
    }
}
```

## 2. Paste-a-list bulk action (transient virtual PO)

When the input is a list already on the clipboard (sub-numbers, logins), a file upload is overkill. Open a one-field dialog and fan out on save. The dialog PO has no backing entity — it lives only in `model.json` (a `StateBehavior` dialog with one `MultiLineString` + `NotEmpty`).

```csharp
public partial class ImportEmployees : CustomAction<MyAppContext>
{
    public override PersistentObject? Execute(CustomActionArgs e) =>
        Manager.Current.GetPersistentObject(Types.ImportEmployees, isNew: true, targetContext: Context);
}
```

Grant `New/<App>.ImportEmployees` (open the dialog) plus the operation right on the target entity. The wizard's `Actions.OnSave` reads, splits, resolves, reports — call `CheckRules` first so `NotEmpty` is enforced:

```csharp
public override void OnSave(PersistentObject obj)
{
    if (!CheckRules(obj)) return;

    var pasted = (string[])obj[AttributeNames.ImportEmployees.EmployeesToImport]!;
    var emails = pasted.Where(l => !string.IsNullOrWhiteSpace(l)).Select(l => l.Trim()).ToArray();
    var existing = Context.Employees.Where(emp => emp.Email.In(emails)).ToArray();
    var notFound = new List<string>();

    foreach (var email in emails)
    {
        if (existing.Any(emp => emp.Email == email)) continue;
        var person = Context.People.FirstOrDefault(p => p.Email == email);
        if (person is null) { notFound.Add(email); continue; }
        Context.Session.Store(new Employee { Email = email, FirstName = person.FirstName, LastName = person.LastName });
    }
    Context.SaveChanges();

    if (notFound.Count > 0)
        Manager.Current.QueueClientOperation(ExecuteMethodOperation.ShowMessageBox(
            "Import employees", "Could not be matched: " + string.Join(", ", notFound)));
}
```

Same gotchas: `PadLeft` fixed-width codes back to width; report unmatched tokens after the save (don't throw) so the partial import commits.

## 3. Large file upload to blob storage

For large binaries you do **not** want the bytes in RavenDB: Vidyano's chunked `BigFileUpload` attribute type stages the upload, and on `SaveNew` you move the staged bytes into external storage and keep only a key on the entity. The storage container, the staging client and the `GenerateBigFileSession`/`CloseBigFileSession` rights are infrastructure outside Vidyano — in CronosCore repos, the dcg plugin's `blob_storage` skill has the full recipe. For files that can live as RavenDB attachments, use `/vidyano:file_storage` instead.

## 4. Asynchronous import via a Job document

For slow or reviewable imports, decouple upload from processing: `OnImport` stores a small job document, attaches the file to it (`e.BinaryFile.StoreAsAttachment(Context.Session, job, contentType: "text/csv")`), saves, and returns immediately; a background worker picks up new jobs in a fresh context under a service identity and runs the same parse/resolve/upsert as the synchronous `OnImport` — only *where* it runs differs. The worker is a RavenDB subscription — in CronosCore repos, the dcg plugin's `subscription_workers` skill covers it.

## Checklist

- [ ] Import action returns `RegisterImport` (or opens the wizard PO) with the right `accept`/PO.
- [ ] Right `<Action>/<Schema>.<Entity>` (plus dialog/big-file rights) granted to the group.
- [ ] `OnImport` upserts (no hard-delete), skips+reports unresolved rows, `Context.SaveChanges()` once.
- [ ] Unreadable file → `FaultException`; partial-success notes → `SetNotification`/`ShowMessageBox` (not Error notification).
- [ ] Big files go to blob storage (allow-list validated), never into the document.

## Verify

1. Build; when the action or dialog PO is new, run a schema synchronize (in CronosCore repos, `/dcg:synchronize`) so `Types.*`/`ActionNames.*` resolve, then grant the right.
2. Open the grid as a member of the granted group: the Import button is there; as a user outside the group it is not.
3. Upload a small valid file: the toast reports the imported count, and RavenDB Studio shows the new or updated documents.
4. Upload a file with one unresolvable row: that row is reported as skipped and the remaining rows are committed.
5. Upload a file of the wrong type: the `FaultException` message is shown and no document changed.

## Related skills

- `/vidyano:exports_downloads` — the outbound counterpart; `OnGetStream` for re-downloading uploads.
- `/vidyano:file_storage` — keeping an uploaded file on a record as a RavenDB attachment.
- `/vidyano:custom_actions` — `CustomAction<TContext>` and the action lifecycle.
- `/vidyano:lifecycle_hooks` — `SaveNew`, `OnSave`, `CheckRules`, `OnDelete`.
- `/vidyano:row_security` — gating which rows an import may touch.
