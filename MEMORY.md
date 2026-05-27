---
canvas:
  - "[[UTI-roadmap.canvas]]"
UTI-roadmap: []
---
# Kindle Zotero Importer Memory

## Goal

Import Kindle `My Clippings.txt` highlights/notes into the corresponding Zotero library items as native Zotero annotations where possible.

Primary source of Kindle data:

- `/Users/ubd/Library/Mobile Documents/com~apple~CloudDocs/Projects/test/My Clippings.txt`

Zotero database discovered at:

- `/Users/ubd/Zotero/zotero.sqlite`

Do not write directly to SQLite. Use Zotero's local JavaScript API/data layer.

Confirmed title-to-Zotero mappings must be stored in `match-overrides.json` so they persist across cumulative Kindle `My Clippings.txt` exports. `match-overrides.generated.json` is disposable and should not be treated as authoritative.

## Confirmed Architecture

Use a two-step importer:

1. Python-side preview/import-plan generator.
   - Parse `My Clippings.txt`.
   - Match clipping titles to Zotero items/attachments.
   - For EPUB, compute EPUB CFI selectors.
   - For PDF, compute page rect coordinates.
   - Emit JSON import plan.
2. Zotero-side JavaScript writer.
   - Run in Zotero via `Tools > Developer > Run JavaScript` or later as a plugin.
   - Load/import generated plan.
   - Call `Zotero.Annotations.saveFromJSON(attachment, annotation)`.

## Working Zotero API

Native annotations can be created safely with:

```js
const attachment = Zotero.Items.get(ATTACHMENT_ITEM_ID);
const saved = await Zotero.Annotations.saveFromJSON(attachment, annotation);
```

Annotation item type exists internally as Zotero item type `annotation`.

Native annotation table observed:

- `itemAnnotations`
- columns include `itemID`, `parentItemID`, `type`, `text`, `comment`, `color`, `pageLabel`, `sortIndex`, `position`, `isExternal`

## EPUB Test: Cyclonopedia

Zotero parent item:

- itemID: `51395`
- key: `JTDWDKRH`
- type: `book`
- citationKey: `negarestani2008`
- title: `Cyclonopedia: complicity with anonymous materials`

Zotero EPUB attachment:

- itemID: `51402`
- key: `P6XQUQXF`
- contentType: `application/epub+zip`
- path: `/Users/ubd/Library/Mobile Documents/com~apple~CloudDocs/03_Academia/ZoteroFiles/Negarestani_2008_Cyclonopedia complicity with anonymous materials.epub`

Kindle clipping title:

- `(fixed) Negarestani_2008_Cyclonopedia complicity with anonymous materials (Reza Negarestani)`

Text matching against the Zotero-linked EPUB worked:

- Cyclonopedia clipping bodies: `79`
- Highlights: `69`
- Notes: `10`
- Exact text matches: `66`
- Partial text matches: `5`
- Unmatched: `8`, all standalone notes.

EPUB spine inspection showed `EPUB/paleopetrology.html` is spine slot `/6/30`.

Working native EPUB test annotation:

```js
const attachment = Zotero.Items.get(51402);

const annotation = {
  key: Zotero.DataObjectUtilities.generateKey(),
  type: "highlight",
  text: "Oil as a lubricant or Tellurian Lube, upon which everything moves forward,",
  comment: "Kindle import test annotation. Delete after verification.",
  color: "#ffd400",
  pageLabel: "",
  sortIndex: "00014|00047607",
  position: {
    type: "FragmentSelector",
    conformsTo: "http://www.idpf.org/epub/linking/cfi/epub-cfi.html",
    value: "epubcfi(/6/30!/4/90/2,/1:3,/1:77)"
  },
  tags: [{ name: "kindle-import-test" }]
};

await Zotero.Annotations.saveFromJSON(attachment, annotation);
```

Important EPUB CFI lesson:

- Bad CFI navigated but did not render highlight: `epubcfi(/6/30!/4/90/2/1:3,/4/90/2/1:77)`
- Good CFI rendered highlight: `epubcfi(/6/30!/4/90/2,/1:3,/1:77)`
- Zotero's EPUB CFI uses a common parent element plus relative text-node offsets.

Manual Zotero highlight over a longer range produced:

```json
{
  "value": "epubcfi(/6/30!/4/90/2,/1:3,/1:317)"
}
```

## PDF Test: Rethinking Algorithmic Regulation

Chose this because it is a normal Zotero item, has a real PDF path, directly matches Kindle clipping text, and had zero existing annotations before the test.

Zotero parent item:

- itemID: `350`
- key: `SGZMZ9X7`
- type: `journalArticle`
- citationKey: `medina2015`
- title: `Rethinking algorithmic regulation`
- publicationTitle: `Kybernetes`

Zotero PDF attachment:

- itemID: `673`
- key: `4BPF7ZUR`
- contentType: `application/pdf`
- attachment title/path in DB: `attachments:Medina_2015_Rethinking algorithmic regulation.pdf`
- actual file found at: `/Users/ubd/Library/Mobile Documents/com~apple~CloudDocs/03_Academia/ZoteroFiles/Medina_2015_Rethinking algorithmic regulation.pdf`
- existing annotations before test: `0`

Kindle clipping title:

- `Rethinking Algorithmic Regulation Kybern - Unknown`

Tested clipping text:

```text
increase worker participation in the economy and preserve the autonomy of factory managers, even with the expansion of state influence.
```

Located in PDF:

- PDF page: `5`
- Zotero `pageIndex`: `4`
- PDF page label: `1009`
- `pdftohtml` XML page dimensions: width `739`, height `1020`
- `pdfinfo` PDF page size: width `493.228`, height `680.315` pts

Working native PDF test annotation:

```js
const attachment = Zotero.Items.get(673);

const annotation = {
  key: Zotero.DataObjectUtilities.generateKey(),
  type: "highlight",
  text: "increase worker participation in the economy and preserve the autonomy of factory managers, even with the expansion of state influence.",
  comment: "Kindle import PDF test annotation. Delete after verification.",
  color: "#ffd400",
  pageLabel: "1009",
  sortIndex: "00004|000435|00197",
  position: {
    pageIndex: 4,
    rects: [
      [197.252, 380.843, 445.841, 390.181],
      [99.447, 369.504, 403.125, 378.842]
    ]
  },
  tags: [{ name: "kindle-import-pdf-test" }]
};

await Zotero.Annotations.saveFromJSON(attachment, annotation);
```

Important PDF lessons:

- Native PDF annotation writing works.
- PDF positions use `{ pageIndex, rects }`.
- Rect coordinates are in PDF points, origin bottom-left.
- `pdftohtml -xml` coordinates are top-left in its own scaled XML coordinate system.
- Convert from `pdftohtml` XML to PDF points using:
  - `scaleX = pdfPageWidth / xmlPageWidth`
  - `scaleY = pdfPageHeight / xmlPageHeight`
  - `x1 = xmlLeft * scaleX`
  - `x2 = (xmlLeft + xmlWidth) * scaleX`
  - `y1 = pdfPageHeight - (xmlTop + xmlHeight) * scaleY`
  - `y2 = pdfPageHeight - xmlTop * scaleY`
- PDF `sortIndex` needs three numeric parts: `pageIndex|vertical|horizontal`, e.g. `00004|000435|00197`.
- EPUB `sortIndex` accepted two numeric parts, e.g. `00014|00047607`.

## Test Scripts Created During Exploration

These were created outside this repo during probing and may be copied/adapted:

- `/Users/ubd/Library/Mobile Documents/com~apple~CloudDocs/Projects/test/create-cyclonopedia-test-annotation.js`
- `/Users/ubd/Library/Mobile Documents/com~apple~CloudDocs/Projects/test/list-cyclonopedia-annotations.js`
- `/Users/ubd/Library/Mobile Documents/com~apple~CloudDocs/Projects/test/create-latzer-pdf-test-annotation.js`
- `/Users/ubd/Library/Mobile Documents/com~apple~CloudDocs/Projects/test/select-medina-pdf-zotero-item.js`
- `/Users/ubd/Library/Mobile Documents/com~apple~CloudDocs/Projects/test/select-howaldt-pdf-zotero-item.js`

## Useful Commands/Tools

Query Zotero DB read-only while Zotero is running:

```sh
sqlite3 "file:/Users/ubd/Zotero/zotero.sqlite?mode=ro&immutable=1" "SQL HERE"
```

PDF text extraction available:

```sh
pdftotext -enc UTF-8 -f PAGE -l PAGE file.pdf -
```

PDF coordinate extraction available:

```sh
pdftohtml -f PAGE -l PAGE -xml -stdout file.pdf
```

PDF page size:

```sh
pdfinfo -f PAGE -l PAGE file.pdf
```

Python PDF libraries were not installed at exploration time:

- `fitz` / PyMuPDF: unavailable
- `pypdf`: unavailable

## Repo/Workspace Decision

This project lives in the Obsidian vault project area:

- `/Users/ubd/Library/Mobile Documents/iCloud~md~obsidian/Documents/rhizome/06_projects/UTI/kindle-zotero-importer`

The vault `.gitignore` ignores it:

- `06_projects/UTI/kindle-zotero-importer`

The importer directory has its own independent Git repo:

- `## No commits yet on main`

## Next Roadmap

1. Create project scaffold.
2. Implement Kindle clipping parser.
3. Implement Zotero DB read-only indexer.
4. Implement matching between Kindle titles and Zotero items/attachments.
5. Implement EPUB text matching and CFI generation.
6. Implement PDF text matching and rect generation using `pdftotext`, `pdftohtml`, and `pdfinfo` first.
7. Emit import preview JSON with confidence flags.
8. Build Zotero JavaScript writer that reads preview JSON and calls `Zotero.Annotations.saveFromJSON()`.
9. Add deduplication. Candidate keys should include attachment ID/key, clipping text hash, Kindle location/page, timestamp, and annotation type.
10. Attach standalone Kindle notes to nearest highlight/location as annotation comments where possible; otherwise import as separate text annotations or report unmatched.
11. Generate import report: matched, ambiguous, unmatched, skipped duplicates, created annotations.

## Cautions

- Never write directly to Zotero SQLite.
- Test on one item before bulk writes.
- Keep a dry-run/import-preview mode as default.
- Keep tags on imported annotations, e.g. `kindle-import`, for easy deletion/filtering.
- PDF text matching will be more fragile than EPUB because of hyphenation, line breaks, OCR quality, and coordinate extraction.
- Some Kindle notes are standalone and have no quote text; attach by location/page proximity.
