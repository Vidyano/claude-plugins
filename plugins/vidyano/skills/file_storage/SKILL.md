---
name: file_storage
description: "Store and serve files that belong to one RavenDB document as attachments: read the BinaryFile attribute (name|base64; null means unchanged) in SaveNew, store after base.SaveNew via Attachments.Store, serve in OnGetStream via Attachments.Get(...).Stream, resize images with SkiaSharp first. PROACTIVE: invoke when the user attaches a file/photo/document to an entity, uploads via a BinaryFile attribute, resizes an image before storing, or mentions BinaryFile/Attachments.Store/StoreAsAttachment/GetNames/SKBitmap. Not for the download recipe (see /vidyano:exports_downloads)."
user-invocable: true
---

# File Storage: RavenDB Attachments

A document store holds JSON, not megabytes of PDF. When a user attaches a contract, a photo, or a generated document to an entity, the first decision is *where the bytes physically live* — and that choice drives how you save, serve, and clean them up. The simplest backend, and the right default for small files that belong inseparably to one document, is a **RavenDB attachment**: the bytes hang off the entity's own document, back up and replicate with it, and disappear when it's deleted.

This is the storage half of file handling. Getting bytes *in* (parsing spreadsheets, paste-a-list wizards) is `/vidyano:imports`; streaming bytes *out* to the browser is `/vidyano:exports_downloads`. Here we focus on what happens between.

Not for: the `RegisterStream`/`OnGetStream` download recipe (`/vidyano:exports_downloads`), parsing uploaded data (`/vidyano:imports`), or files too large or numerous for the database (blob storage — see the end of this skill).

## When to use

The user attaches a file/photo/document to an entity, uploads via a `BinaryFile` attribute, resizes an image before storing, or mentions `BinaryFile`/`Attachments.Store`/`StoreAsAttachment`/`GetNames`/`SKBitmap`/`ScaleImage`.

Reach for an attachment when the file is **small or medium, few per entity, and one-to-one with its document** — it should back up and replicate with the document, and vanish when the document is deleted. When files get large or numerous, keep them out of the database (that's blob storage — described at the end).

## The `BinaryFile` attribute

Every file backend starts the same way: a `BinaryFile` attribute on the entity (declared in the model JSON) carries the upload from the client as `name|base64`. In a lifecycle hook (`/vidyano:lifecycle_hooks`) — usually `SaveNew` or `PrePersistChanges` — you read it and decide where the bytes go. Deconstruct it into its filename and data:

```csharp
var (fileName, fileData) = (BinaryFile?)obj[AttributeNames.Order.Upload] ?? default!;

if (fileName is not null && fileData is not null)
{
    // ... store the bytes somewhere ...
}
```

A `null` deconstruction means the user didn't change the file on this save — **leave the existing stored file alone.** This is the single most important guard: without it, every save with an untouched file would overwrite or clear the stored attachment.

## Storing the bytes as a named attachment

Store the bytes as a *named attachment* on the entity's own document. The one ordering rule: call `base.SaveNew(obj)` **first** — the document must exist before an attachment can hang off it — then store:

```csharp
protected override void SaveNew(PersistentObject obj)
{
    var (fileName, fileData) = (BinaryFile?)obj[AttributeNames.Order.Upload] ?? default!;

    base.SaveNew(obj);   // the document must exist before an attachment can hang off it

    if (fileName is not null && fileData is not null)
        Context.Session.Advanced.Attachments.Store(obj.ObjectId, fileName, new MemoryStream(fileData));
}
```

`BinaryFile` also offers a one-call helper that stores the upload under a conventional name — use it when you don't need to control the attachment name yourself. The receiver is the `BinaryFile` read off the attribute; the entity is the just-saved document:

```csharp
protected override void SaveNew(PersistentObject obj)
{
    var upload = (BinaryFile?)obj[AttributeNames.Order.Upload];

    base.SaveNew(obj);

    if (upload is not null)
        upload.StoreAsAttachment(Context.Session, GetEntity(obj)!, contentType: "application/pdf");
}
```

## Serving it back

Read the attachment in `OnGetStream` (see `/vidyano:exports_downloads` for the `RegisterStream`/`OnGetStream` two-step that exposes the download). Resolve the entity, get the named attachment, set the filename, return its stream:

```csharp
public override Stream OnGetStream(GetStreamArgs e)
{
    var order = Context.Session.Load<Order>(e.Key);
    var attachment = Context.Session.Advanced.Attachments.Get(order.Id, "contract.pdf");

    e.FileName = "contract.pdf";
    return attachment.Stream;
}
```

## Rehydrating the attribute on load

On load, rehydrate the `BinaryFile` attribute to *filename-only* (no base64) so the UI shows the existing file without re-downloading its content. Enumerate the document's attachments with `Session.Advanced.Attachments.GetNames(entity)` and set the attribute to just the name. The bytes travel only when the user explicitly downloads via `OnGetStream`:

```csharp
public override void OnLoad(PersistentObject obj, PersistentObject? parent)
{
    base.OnLoad(obj, parent);

    var order = GetEntity(obj)!;
    var stored = Context.Session.Advanced.Attachments.GetNames(order).FirstOrDefault();
    if (stored is null)
        return;

    // Filename only: the UI shows "contract.pdf" with a download link; no bytes in the response.
    // Let BinaryFile write the value — a bare string is not a valid BinaryFile
    // service value, the wire form is always "name|" or "name|base64".
    obj[AttributeNames.Order.Upload].SetValue(new BinaryFile(stored.Name));
}
```

The attribute is transient on load — nothing is written back unless the user picks a new file, and the `null`-deconstruction guard in `SaveNew` keeps an untouched value from overwriting the attachment. When the UI must render the bytes inline (a small avatar), rehydrate the full `name|base64` value instead — load the attachment, copy its stream to a buffer, encode:

```csharp
using var attachment = Context.Session.Advanced.Attachments.Get(order.Id, stored.Name);
using var buffer = new MemoryStream();
attachment.Stream.CopyTo(buffer);

obj[AttributeNames.Order.Photo].SetValue(new BinaryFile(stored.Name, buffer.ToArray()));
```

Construct a `BinaryFile` rather than formatting `name|base64` by hand: the same type reads the attribute and writes it, so the two ends can never drift.

**An `Image` attribute is not a `BinaryFile`** — it carries bare base64, no filename. Hand `SetValue` the `byte[]` and it is encoded for you:

```csharp
obj[AttributeNames.Order.Thumbnail].SetValue(buffer.ToArray());
```

Do this only for images already scaled down on the way in (next section) — a full-size PDF in every load response defeats the point of attachments.

## Resizing images on the way in

When the file is an image you'll show in a grid or card, store a scaled copy so the client never downloads full-resolution bytes for a thumbnail. Resize before storing with `SkiaSharp`, short-circuiting when the source is already small enough:

```csharp
public static byte[] ScaleImage(byte[] originalFileBytes, int targetWidthInPixels = 400)
{
    using var original = SKBitmap.Decode(originalFileBytes);

    if (original.Width <= targetWidthInPixels)
        return originalFileBytes;

    var widthScale = targetWidthInPixels / (double)original.Width;
    var newHeight = (int)(original.Height * widthScale);

    using var resized = original.Resize(new SKImageInfo(targetWidthInPixels, newHeight), SKFilterQuality.High);
    using var image = SKImage.FromBitmap(resized);

    return image.Encode(SKEncodedImageFormat.Jpeg, 90).ToArray();
}
```

`SKBitmap.Decode` → early-return if already under the target width → `original.Resize(new SKImageInfo(w, h), SKFilterQuality.High)` → `SKImage.FromBitmap` → `Encode(Jpeg, 90)`. Every `SK*` object is `using`-scoped — these wrap unmanaged bitmaps, so leaking them leaks native memory.

## When attachments aren't the right backend

Attachments shine when the file is small and one-to-one with its document. When files get **large or numerous** — and you'd rather keep them out of the database's backup and replication footprint — the bytes belong in external blob storage instead, with only an opaque key kept on the entity, and (unlike attachments) cleanup you perform yourself on delete. When even the *upload* is large enough that it shouldn't pass through the request as base64, the upload itself is staged client-side first. Both of those are out of scope here (in CronosCore repos, the dcg plugin's `blob_storage` skill covers them); the `SaveNew`/`OnGetStream` shape is the same, only the storage call changes.

## Checklist

- [ ] `BinaryFile` attribute declared in the model JSON; deconstructed `var (fileName, fileData) = (BinaryFile?)obj[attr] ?? default!;` in `SaveNew`/`PrePersistChanges`.
- [ ] Null deconstruction guarded — a `null` filename/data means "unchanged", so the stored file is left alone.
- [ ] `base.SaveNew(obj)` called **before** `Attachments.Store(obj.ObjectId, fileName, new MemoryStream(fileData))` (document must exist first); or the one-call `BinaryFile.StoreAsAttachment(session, entity, contentType)`.
- [ ] Served in `OnGetStream` via `Attachments.Get(id, name).Stream` with `e.FileName` set.
- [ ] Attribute rehydrated on load via `Attachments.GetNames(entity)` into a `BinaryFile` — `new BinaryFile(name)` for filename-only, `new BinaryFile(name, bytes)` when the bytes must travel; an `Image` attribute takes the `byte[]` straight.
- [ ] Images resized before store with the `ScaleImage` SkiaSharp helper; every `SK*` object `using`-scoped; short-circuit when already small.

## Verify

1. Build; if you added the `BinaryFile` attribute, run a schema synchronize (in CronosCore repos, `/dcg:synchronize`) and confirm the attribute renders as a file picker.
2. Upload a file and save: in RavenDB Studio the document's Attachments tab lists it under the expected name with the right size and content type.
3. Reopen the record: the attribute shows the filename without a base64 payload in the load response (network tab), and the download link streams the same bytes back.
4. Edit an unrelated field and save: the attachment is still there — if it vanished, the `null`-deconstruction guard is missing.
5. Image path: upload a 4000px photo and confirm the stored attachment is at most the target width; upload a 200px one and confirm it is stored byte-for-byte (the short-circuit).

## Related skills

- `/vidyano:exports_downloads` — the `RegisterStream`/`OnGetStream` recipe that exposes the attachment as a download.
- `/vidyano:imports` — the intake side: parsing and uploading file content into the model.
- `/vidyano:lifecycle_hooks` — the `SaveNew`/`OnGetStream` hooks the whole pattern lives in.
