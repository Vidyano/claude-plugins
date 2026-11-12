---
name: exports_downloads
description: "Deliver files to the browser with Vidyano's RegisterStream + OnGetStream recipe: Excel/CSV export, template-generated documents (Word/PDF), stored-attachment download, ZIP bundles. Covers key encoding, GetStreamArgs (GetBytes/GetText/GetFile, or a raw stream + FileName/ContentType), and the per-action right. PROACTIVE: invoke when the user adds a download/export/print action, mentions RegisterStream/OnGetStream/GetStreamArgs, generates a document to download, or exports query rows to a file."
user-invocable: true
---

# Exports, Downloads & Streams

Delivering a file to the browser is a two-step recipe. A custom action does **not** return bytes directly; it calls `Manager.Current.RegisterStream(po, key)`, which tells the client to issue a *second* request. The owning entity's `OnGetStream(GetStreamArgs)` answers that request: decode the key, materialize the bytes, set name/type, return a `Stream`. Splitting registration from generation means heavy binaries are produced only when the user actually downloads.

Not for: storing uploaded bytes on a document (`/vidyano:file_storage`), or getting files *in* (`/vidyano:imports`).

## When to use

The user adds a download/export/print button, generates a document, or serves a stored attachment.

## Step 1: register the stream in an action

The action (a `CustomAction<TContext>` — see `/vidyano:custom_actions`) validates, computes a **key**, and returns `RegisterStream`. The key is an opaque string you choose — it is the only thing that travels to `OnGetStream`, so encode everything that method needs.

```csharp
public partial class DownloadContract : CustomAction<MyModuleContext>
{
    public override PersistentObject? Execute(CustomActionArgs e)
        => Manager.Current.RegisterStream(e.Parent!, e.Parent!.ObjectId);
}
```

Two overloads — `RegisterStream(po, key)` (most common) and `RegisterStream(Types.Invoice, key)` (a type name when there is no concrete PO). The key is optional and defaults to the object id. The stream is handled by the `OnGetStream` of the *registered PO's type*. Handle both single-record and grid invocations, encoding multiple ids into the key:

```csharp
public override PersistentObject? Execute(CustomActionArgs e)
{
    if (e.Query is null && e.Parent is not null)
        return Manager.Current.RegisterStream(e.Parent, e.Parent.ObjectId);
    if (e.Query is not null && e.SelectedItems is not null)
        return Manager.Current.RegisterStream(
            e.Query.PersistentObject, string.Join(";", e.SelectedItems.Select(si => si.Id)));
    return null;
}
```

## Step 2: produce the bytes in OnGetStream

Override `OnGetStream` on the entity's `Actions` class, decode `e.Key`, build/load bytes, return a `Stream`. The base returns `Stream.Null`, so the override is mandatory for any PO that registers a stream. **The key you write in `Execute` is the key you parse here** — a single delimiter + `e.Key.Split(...)` is the idiomatic encoding for multiple values.

```csharp
public override Stream OnGetStream(GetStreamArgs e)
{
    var contract = Context.Session.Load<Contract>(e.Key);
    var bytes = BuildContractPdf(contract);   // your generator — see the template variant below
    return e.GetBytes(bytes, $"Contract-{contract.Number}.pdf", "application/pdf");
}
```

### GetStreamArgs

| Member | Purpose |
| --- | --- |
| `e.Key` | The key from `RegisterStream` (read-only). Decode it. |
| `e.FileName`, `e.ContentType` | Download name / MIME (defaults `application/octet-stream`). |
| `e.GetBytes(bytes, fileName?, contentType?)` | Wrap a `byte[]` and set name/type in one call. |
| `e.GetFile(path, fileName?, contentType?)` | Open a file on disk. |
| `e.GetText(contents, fileName?, encoding?, contentType?)` | Return text (UTF-8, `text/plain` defaults). |

The helpers assign name/type only when you supply them and return the `Stream`. You can also set the properties yourself and return a raw `MemoryStream` — useful when building incrementally (e.g. a ZIP).

### Declare the action and right

Declare the action in the model (`ShowedOn`, `SelectionRule`), gate it with a right `<Action>/<Module>.<Type>` granted to a group (`UserId` is a group GUID). `OnGetStream` inherits the session identity — a user without the right never registers the stream.

## Variants

**Excel export.** For a plain "all columns as shown" grid export, use the built-in `ExportToExcel` query action — enable it in the model and grant `ExportToExcel/<Module>.<Query>`; no code. Reach for custom code only for a fixed external schema, multiple sheets, or a side effect:

```csharp
using Vidyano.Service.Actions;

// Re-run the registered query:
var query = Context.GetQuery(e.Key);
var bytes = Excel.Generate(parent: null, query, sheetName: "Employees");
return e.GetBytes(bytes, "Employees.xlsx", Excel.ContentType);

// Or a fixed-schema workbook:
var invoices = Context.Invoices.Where(i => i.Status == InvoiceStatus.Open).ToArray();
var bytes = Excel.WriteSpreadsheet("Invoices", sheet =>
{
    sheet.WriteRow("Client", "Invoice", "Amount");
    foreach (var inv in invoices) sheet.WriteRow(inv.ClientName, inv.Number, inv.Amount);
});
```

`Excel.WriteSpreadsheets` writes several named sheets; `Excel.ContentType` is the `.xlsx` MIME constant.

**CSV.** Just text — append the lines to a `StringBuilder sb`, then `e.GetText(sb.ToString(), "people.csv", contentType: "text/csv")`.

**Template document (Word/PDF).** Merge entity data into a `.docx` template, optionally convert to PDF:

```csharp
using Vidyano.Service.Actions;

var contract = Context.Session.Load<Contract>(e.Key);
var template = File.ReadAllBytes(HostingEnvironment.MapPath("~/App_Data/Templates/Contract.docx"));
var docx = MergeContract(template, contract);            // your merge code: fills the content controls
var pdf  = Word.ConvertToPdf(docx);
return e.GetBytes(pdf, $"Contract-{contract.Number}.pdf", "application/pdf");
```

`Word.ContentType` is the `.docx` MIME; `Word.Combine(IList<byte[]>)` merges several rendered docs into one. Templates can live under `App_Data`, as RavenDB attachments, or in blob storage — `OnGetStream` is just code.

**Stored attachment (RavenDB).** `Context.Session.Advanced.Attachments.Get(id, name).Stream` with `e.FileName` set — the full store/serve recipe, including rehydrating the `BinaryFile` attribute, is `/vidyano:file_storage`.

**Blob / object storage.** When the bytes live outside RavenDB, the entity holds only a storage key (often composite for value objects). Decode the key, fetch the bytes from your store, wrap them:

```csharp
var keys = e.Key.Split(';');                                   // "<contractId>;<documentIndex>"
var contract = Context.Session.Load<Contract>(keys[0]);
var document = contract.Documents[int.Parse(keys[1])];

var blobRoot = HostingEnvironment.MapPath("~/App_Data/Blobs");   // a local-folder store
var bytes = File.ReadAllBytes(Path.Combine(blobRoot, document.StorageKey));

e.FileName = document.FileName;
return new MemoryStream(bytes) { Position = 0 };
```

Only the "fetch bytes" line changes per backend — an Azure/S3 client, a typed `HttpClient`, an external DMS. In CronosCore repos that line is `IStorageContainer` from the dcg plugin's `blob_storage` skill, which also covers deleting the blob when the document goes.

**ZIP bundle.** Build a `ZipArchive` over a `MemoryStream`; idiomatic refinement is a single-file fast path (return the bare file when there is exactly one):

```csharp
var memoryStream = new MemoryStream();
var order = Context.Session.Load<Order>(e.Key);
using (var zip = new ZipArchive(memoryStream, ZipArchiveMode.Create, leaveOpen: true))
{
    foreach (var name in Context.Session.Advanced.Attachments.GetNames(order).Select(a => a.Name))
    {
        var file = Context.Session.Advanced.Attachments.Get(order.Id, name);
        using var entryStream = zip.CreateEntry(name, CompressionLevel.Fastest).Open();
        file.Stream.CopyTo(entryStream);
    }
}
e.FileName = $"order-{order.Number}.zip"; e.ContentType = "application/zip";
memoryStream.Position = 0;
return memoryStream;
```

## Checklist

- [ ] Action returns `RegisterStream` with a key encoding everything `OnGetStream` needs.
- [ ] `OnGetStream` on the registered PO's `Actions` class parses the same key shape.
- [ ] `FileName`/`ContentType` set (via helper or directly).
- [ ] Action declared in the model + right `<Action>/<Module>.<Type>` granted to a group.
- [ ] For a plain grid export, prefer built-in `ExportToExcel` over custom code.

## Verify

1. Build, run a schema synchronize (in CronosCore repos, `/dcg:synchronize`), and confirm the action is in the model and its right is granted to the test user's group.
2. Click the action: the network tab shows the action call followed by a second `GetStream` request, and the browser saves a file with the intended name and extension.
3. Open the file: it has the expected content (rows, merged fields, all ZIP entries) and the correct MIME type — a wrong `ContentType` shows up as a `.bin`/`.txt` download.
4. Grid invocation: select several rows and confirm the key round-trips (`e.Key.Split(...)` yields every id) and each row's data is in the output.
5. Negative case: sign in as a user without the right — the button is absent and a hand-crafted `GetStream` request is refused.

## Related skills

- `/vidyano:custom_actions` — the action lifecycle, model declaration, `SelectionRule`.
- `/vidyano:file_storage` — storing uploaded files as RavenDB attachments before they can be downloaded.
- `/vidyano:imports` — the inbound counterpart (`RegisterImport`/`OnImport`); often shares a column shape.
- `/vidyano:row_security` — so users only export rows they may see.
