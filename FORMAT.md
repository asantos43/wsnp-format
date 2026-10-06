# WSNP — Web SNaPshot file format, version 1.0

A `.wsnp` file is the "photo" of one web page, to keep and read offline: the page as it was on
screen, every file it needs, and the small local scripts that give its carousels, tabs and menus
their behaviour back. Like `.docx` or `.epub`, it is a ZIP archive with a fixed structure. PageKeep
writes it (popup → **Save as** → `.wsnp`), and the PageKeep viewer, a separate application, reads it.

A `.wsnpx` file is its sibling for applications: the same container, but it may carry scripts of
its own that turn the saved pages into an application, run only when the user allows them
(section 8), as `.xlsm` is to `.xlsx`.

The reference tools are in the `tests/` folder of the
[PageKeep repository](https://github.com/asantos43/webpage-snapshot/tree/main/tests): `wsnp-check.mjs`
validates a file against this document, and `wsnp-crypt.mjs` implements password protection. Both are
plain Node modules with no dependency, for readers to reuse.

The key words **must**, **must not**, **should** and **may** are used as in RFC 2119.

This description lives in one place, this repository (`wsnp-format`), and is the source of truth for the format. The programs that
use it, PageKeep (`webpage-snapshot`, which writes the files) and the viewer (`wsnp-viewer`, which reads them), link to it and keep no copy.

## 1. Summary

| Item | Value |
| --- | --- |
| Extension | `.wsnp` |
| Media type | `application/vnd.wsnp+zip` (not registered with IANA yet) |
| Container | ZIP, entries stored or DEFLATE-compressed; no ZIP64, no ZIP encryption |
| Identification | first entry `mimetype`, stored, holding the media type |
| Description | `manifest.json` |
| Entry point | `index.html` |
| Version | `format_version` `"1.0"` in the manifest (`"1.1"` when the file is signed) |
| Signature | optional `signature.json` (1.1): the manifest signed by the writer's key (section 12) |
| Family | `.wsnpx` (`application/vnd.wsnp.x+zip`): the profile with an application's scripts (section 8) |

No format was found using `.wsnp` or `.wsnpx` when they were chosen (September 2026); the nearest
are `.snp`, `.wsn`, `.npx` and `.wspx`, all unrelated.

## 2. Container

- A `.wsnp` is a standard ZIP archive that any unzip tool opens. Renamed to `.zip` and unzipped, it
  gives a folder whose `index.html` opens in a browser and works offline.
- Every entry **must** be stored (method 0) or DEFLATE-compressed (method 8), so a browser can read
  it natively (`DecompressionStream('deflate-raw')`).
- The archive **must not** use ZIP64 (so it stays under 4 GB) or ZIP-level encryption (the old
  ZipCrypto is weak, and WinZip AES is not something a browser reads natively). Password protection
  is the format's own layer (section 9).
- File names are UTF-8 (flag bit 11), although the path rules (section 5) keep them to ASCII.
- Readers find the entries through the ZIP's central directory, so they can read any one entry
  without reading the rest: from a local file (the File API) or from a server (HTTP Range requests).

## 3. Identification

The first entry of the archive **must** be named `mimetype`, be stored (not compressed), have no
extra field, and contain exactly the media type in ASCII, with no line break. This is the rule
EPUB and OpenDocument use: a program recognises the file from its first bytes, whatever its name.

| Offset | Bytes | Meaning |
| --- | --- | --- |
| 0 | `50 4B 03 04` | ZIP local file header |
| 8 | `00 00` | method: stored |
| 26 | `08 00` | name length: 8 |
| 28 | `00 00` | extra field length: 0 |
| 30 | `mimetype` | the entry's name |
| 38 | `application/vnd.wsnp+zip` | the entry's content, the media type |

```
00000000  50 4b 03 04 14 00 00 08  00 00 .. .. .. .. .. ..  |PK..............|
00000010  .. .. 18 00 00 00 18 00  00 00 08 00 00 00 6d 69  |..............mi|
00000020  6d 65 74 79 70 65 61 70  70 6c 69 63 61 74 69 6f  |metypeapplicatio|
00000030  6e 2f 76 6e 64 2e 77 73  6e 70 2b 7a 69 70 ..     |n/vnd.wsnp+zip..|
```

(`..` are the date, time and CRC, which vary.)

## 4. Layout

```
mimetype              application/vnd.wsnp+zip (first, stored)
manifest.json         describes the file (section 6)
signature.json        the signature of the manifest (optional, 1.1; section 12)
index.html            the page
assets/               every file the page uses, by kind:
  images/               pictures, icons, SVG, pictures of frames from other sites
  styles/               stylesheets
  fonts/                fonts
  media/                video, audio, subtitles
  files/                files the page links to for download (PDF, archives, office documents…),
                        and anything else
_wsnp/                the format's own files:
  offline.js            the offline scripts of the page (when it needs them)
  offline-2.js …        the offline scripts of a frame that needs a different set
  preview.jpg           a picture of the page as it was on screen (optional)
```

- Every file of the snapshot **must** be under `assets/`, in one of the five folders. The folder
  is a convenience for people browsing an unzipped copy; readers **must** rely on the manifest's
  `media_type`, never on the folder.
- The folder `_wsnp/` is reserved for the format's own files.
- A reader **must** ignore entries it does not know, so later 1.x versions can add some.

## 5. Paths

Every entry name **must**:

- use only `A–Z a–z 0–9 . _ - /`;
- be relative: no leading `/`, no empty segment, no `.` or `..` segment;
- be at most 255 characters;
- be unique even when upper and lower case are treated as the same, so it unzips the same way on
  Windows and macOS.

References inside the page and its stylesheets are relative too: `index.html` refers to
`assets/images/logo-1k3f9a.png`, and a stylesheet in `assets/styles/` refers to
`../fonts/serif-2b7c.woff2`. A snapshot unzipped into a folder of any static web server works as is.

## 6. The manifest

`manifest.json` is UTF-8 JSON. Readers **must** ignore fields they do not know.

| Field | Type | Meaning |
| --- | --- | --- |
| `format` | `"wsnp"` | required |
| `format_version` | string `"major.minor"` | `"1.0"`, or `"1.1"` for a signed file; see section 11 |
| `generator` | `{ name, version }` | the program that wrote the file, e.g. `{ "name": "PageKeep", "version": "1.5.0" }` |
| `created` | string, ISO 8601 | when the page was captured |
| `title` | string | the original page's title (`<title>`, else its first `<h1>`, else its address) |
| `description` | string | the original page's description (`<meta name="description">`, else `<meta property="og:description">`); `""` when it has none |
| `source` | object | where the page came from: `url` (required, the address in the browser's tab), `canonical` (the page's `<link rel="canonical">`, or `""`), `language` (the page's `lang`, or `""`) |
| `pages` | array | the pages of the file: `[{ entry, title, description, source }]`; a `.wsnp` holds exactly one, whose `entry` is `"index.html"` |
| `preview` | string, optional | `"_wsnp/preview.jpg"` when there is one |
| `viewport` | `{ width, height, device_pixel_ratio }` | the browser window's size in CSS pixels, and its pixel ratio, at capture time |
| `capture` | object | how it was captured: `load_whole_page` (true when the page was scrolled through and "Load more" pressed first) |
| `converted_from` | object, optional | present when a program made the file from another format, such as a plain ZIP saved by PageKeep: `format` (`"zip"`) and `tool` (the program named in the original's `snapshot.json`, e.g. `"Page Snapshot 1.0.0"`). Facts the original did not record (`viewport`, `capture`) then hold defaults. |
| `files` | array | one entry per file of the archive except `mimetype` and `manifest.json` (below) |
| `failed` | array | what could not be saved: `[{ url, reason }]`; those references were removed from the page |

Each entry of `files`:

| Field | Type | Meaning |
| --- | --- | --- |
| `path` | string | the entry's name in the archive |
| `original_url` | string, optional | the address it was saved from, for files that came from the web |
| `media_type` | string | required: the type to serve it with (`text/css`, `image/png`…) |
| `bytes` | number | its size, uncompressed |
| `sha256` | string | the SHA-256 of its uncompressed bytes, in lowercase hex |
| `source` | string | how it was obtained: `page` (the bytes the browser had already loaded), `tab` (requested by the tab from its own site), `network` (downloaded in the tab's context), `picture` (a picture the capture took, e.g. of a frame from another site), `generated` (made by the writer: `index.html`, `_wsnp/*`) |

The manifest describes the file, not the page's behaviour: what the offline scripts need (the
recorded carousels, for instance) lives in the page itself, in `<script type="application/json">`
elements that never run.

### Sample

```json
{
  "format": "wsnp",
  "format_version": "1.0",
  "generator": { "name": "PageKeep", "version": "1.5.0" },
  "created": "2026-09-29T14:45:12.345Z",
  "title": "Harbor Times — Local news",
  "description": "News from the harbor and the old town.",
  "source": { "url": "https://harbortimes.example/", "canonical": "https://harbortimes.example/", "language": "en" },
  "pages": [{
    "entry": "index.html",
    "title": "Harbor Times — Local news",
    "description": "News from the harbor and the old town.",
    "source": { "url": "https://harbortimes.example/", "canonical": "https://harbortimes.example/", "language": "en" }
  }],
  "preview": "_wsnp/preview.jpg",
  "viewport": { "width": 1280, "height": 800, "device_pixel_ratio": 1 },
  "capture": { "load_whole_page": true },
  "files": [
    { "path": "index.html", "media_type": "text/html", "bytes": 48213, "sha256": "9f2c…", "source": "generated" },
    { "path": "assets/styles/site-1x05wni.css", "original_url": "https://harbortimes.example/site.css", "media_type": "text/css", "bytes": 10422, "sha256": "4b1e…", "source": "page" },
    { "path": "assets/images/logo-1qg48nw.svg", "original_url": "https://harbortimes.example/logo.svg", "media_type": "image/svg+xml", "bytes": 1804, "sha256": "c07a…", "source": "page" },
    { "path": "_wsnp/offline.js", "media_type": "text/javascript", "bytes": 21877, "sha256": "e5d3…", "source": "generated" },
    { "path": "_wsnp/preview.jpg", "media_type": "image/jpeg", "bytes": 142380, "sha256": "71aa…", "source": "generated" }
  ],
  "failed": [
    { "url": "https://cdn.example/old-banner.png", "reason": "HTTP 404" }
  ]
}
```

## 7. The page

- `index.html` is the page as it was on screen, with every reference to a saved file rewritten to
  its path in the archive. Opening it **must not** contact the network: the page's own scripts,
  `ping` attributes, cross-origin frames and every reference to something that could not be saved
  are removed. Links (`<a href>`) and `<link rel="canonical">` keep their absolute addresses: they
  load nothing by themselves.
- The page **must not** contain inline script. The only scripts are the format's own files in
  `_wsnp/` (the writer's offline scripts, never the page's own), loaded with `<script src>`; inline
  event handlers (`onclick=` …) are removed. So a reader can show the page under a strict Content
  Security Policy that allows scripts only from itself (section 10). Data for those scripts is in
  `<script type="application/json">`, which never runs.
- A frame of the page whose content could be read is saved inline (`<iframe srcdoc>`); its
  `<script src="_wsnp/…">` resolves against the page's address, as `srcdoc` documents do.
- Inline styles (`<style>`, `style=""`) are allowed.

## 8. Profiles: `.wsnp` and `.wsnpx`

As `.xlsx` and `.xlsm` share one container and differ in what they may contain (macros), the WSNP
family has two profiles, told apart by the extension **and** by the `mimetype` entry:

| | `.wsnp` | `.wsnpx` |
| --- | --- | --- |
| Purpose | the "photo" of one page, to read offline | an application: one or more pages **with scripts of their own** that give them features (notes, highlights, quizzes, calculators, dashboards over saved data…) |
| Media type | `application/vnd.wsnp+zip` | `application/vnd.wsnp.x+zip` |
| `format` in the manifest | `"wsnp"` | `"wsnpx"` |
| Scripts | only the writer's offline scripts in `_wsnp/`, which restore the page's behaviour | any scripts in `app/`, under the rules of section 8.2 |
| Pages | exactly one | one or more |
| Opening | scripts run at once (they are the writer's own, known and small) | scripts run only after the user allows them (section 8.3) |
| Readers | any WSNP reader | a `.wsnpx`-aware reader only |

A reader that only knows `.wsnp` **must** refuse a `.wsnpx` and say that it is a snapshot with an
application it cannot open (as office programs warn about macros), never "broken file".

Everything in sections 2 to 7 and 9 to 11 applies to `.wsnpx` too, except what this section
changes.

### 8.1 Layout and manifest of a `.wsnpx`

```
mimetype              application/vnd.wsnp.x+zip (first, stored)
manifest.json         as in section 6, with "format": "wsnpx" and the "app" object below
index.html            the first page (other pages: any .html file listed in "pages")
assets/               the pages' files, as in a .wsnp
app/                  the application: its scripts (classic or modules), styles, pictures, fonts
data/                 data the application reads and writes (JSON, text, CSV…)
_wsnp/                the format's own files, as in a .wsnp
```

The manifest adds:

| Field | Type | Meaning |
| --- | --- | --- |
| `app` | object | required: the application the file carries |
| `app.name` | string | its name, shown to the user before its scripts run |
| `app.version` | string | its version |
| `app.description` | string | what it does, in one or two sentences, shown with the name |
| `app.entry` | string | the page opened first (default: `pages[0].entry`) |
| `app.permissions` | array | what it asks for beyond running offline (section 8.4); `[]` when nothing |
| `app.origins` | array of strings | with the `network` permission only: the origins it may contact (`https://api.example.com`) |

`pages` may list several pages, each `{ entry, title, description, source }`; `source` is optional
for pages the application made itself.

### 8.2 Scripts

- Scripts **must** be files of the archive, in `app/` (or `_wsnp/`), listed in `files` with their
  SHA-256 and `text/javascript`. Loading code from the network is forbidden, even with the
  `network` permission: every line that runs is in the file and covered by its hashes.
- Pages load them with `<script src>` or `<script type="module" src>`, and modules import each
  other by relative paths. Inline scripts and inline event handlers stay forbidden, so readers can
  keep `script-src 'self'`.
- Code **must not** build code from strings (`eval`, `new Function`, `setTimeout("…")`): readers
  do not allow `'unsafe-eval'`. WebAssembly modules in `app/` **may** be used; readers that allow
  them add `'wasm-unsafe-eval'`.
- The application talks to the reader only through the reader API of section 8.5; it sees nothing
  of the reader, of other snapshots or of the user's computer.
- A writer **should not** copy a site's own scripts into `app/`: they expect their server and
  would fail offline. `.wsnpx` scripts are written for the snapshot.

### 8.3 Opening: the user decides

- A reader **must** open a `.wsnpx` with its scripts off, showing its pages as they are, and a bar
  naming the application (`app.name`, `app.version`, `app.description`) and what it asks for
  (`app.permissions`, `app.origins`), with a button to enable it.
- Scripts run only after the user enables them. The reader **may** remember the choice for that
  exact application: the SHA-256 of every file in `app/` and of the manifest's `app` object. If any
  of them changes, it asks again.
- A file whose hashes do not match its manifest **must** never run its scripts.
- The user can turn the application off again at any time.

### 8.4 Permissions

An application with no permission runs offline, in memory, and forgets everything when closed. It
can ask for:

| Permission | What the reader grants |
| --- | --- |
| `storage` | keep data between openings, through the reader API (`storage.*`), in a store of its own tied to `app.name` and the file |
| `save` | write its data back into the file: the reader saves a new `.wsnpx` with the changed files of `data/` (and new hashes), after the user confirms |
| `download` | offer files to the user (`download`), who chooses where to save them |
| `clipboard` | write text to the clipboard, only in answer to a click |
| `network` | contact the origins in `app.origins`, and only those (`connect-src`); never to load code |

A reader grants only the permissions it knows and the user accepted; the others are refused, and
the application must cope without them. Unknown permissions are shown to the user as such.

### 8.5 The reader API

The application runs in a sandboxed frame without `allow-same-origin`, so it has no storage of its
own. It asks the reader with `window.parent.postMessage`, and the reader answers the same way:

```js
// request                                   // answer
{ wsnpx: 1, id: 7, call: 'storage.get',      { wsnpx: 1, id: 7, result: { … } }
  args: { key: 'notes' } }                   { wsnpx: 1, id: 7, error: 'not allowed' }
```

| Call | Arguments | Permission |
| --- | --- | --- |
| `info` | — | none: returns `{ reader, version, permissions }` (what was granted) |
| `data.read` | `{ path }` (inside `data/`) | none: reads a file of `data/` as it is in the archive, or as `data.write` left it |
| `data.write` | `{ path, text }` or `{ path, bytes }` | none: changes `data/` in memory; kept only with `save` |
| `save` | — | `save`: the reader writes the file back (after the user confirms) |
| `storage.get` / `storage.set` / `storage.remove` | `{ key }` / `{ key, value }` / `{ key }` | `storage` |
| `download` | `{ name, media_type, text }` or `{ name, media_type, bytes }` | `download` |
| `clipboard.write` | `{ text }` | `clipboard` |

Values are anything structured clone accepts. Readers **must** check the message's source frame,
answer only calls from the snapshot they show, and refuse calls that need a permission not granted.
New calls come in later minor versions; an application calls `info` first and relies only on what
it lists.

### 8.6 Security policy of a `.wsnpx`

As in section 10, with scripts from the file only, and the network opened only to `app.origins`
when `network` is granted:

```
default-src 'self' data:; script-src 'self'; style-src 'self' 'unsafe-inline' data:;
connect-src 'self' <app.origins, when granted>
```

(plus `'wasm-unsafe-eval'` in `script-src` for readers that allow WebAssembly). The frame's sandbox
is `allow-scripts`, plus `allow-downloads` with the `download` permission and `allow-modals` when
the reader wants to allow `alert` and `confirm`.

### 8.7 From one to the other

A `.wsnp` becomes a `.wsnpx` when a program (the viewer, a web app) adds an application to it:
`mimetype` and `format` change, `app/`, `data/` and `app` are added, and the page stays as it was.
Removing `app/`, `data/` and the extra pages turns it back into a `.wsnp`. PageKeep, the
extension, only writes `.wsnp`.

## 9. Password protection

A `.wsnp` can be protected with a password, like an encrypted PDF, so it can be sent to someone and
read only by who knows the password. Only standard algorithms available in every browser (Web
Crypto) and in Node are used, so any reader can do it without a library. PageKeep always writes open
files; the viewer saves and opens protected ones.

A protected file is still a ZIP, still `.wsnp`, and still starts with the same `mimetype` entry. It
holds exactly three entries, in this order, all stored:

1. `mimetype`, as in section 3;
2. `encryption.json`, readable, with the parameters only:
   ```json
   {
     "encryption_version": "1.0",
     "kdf": { "name": "PBKDF2", "hash": "SHA-256", "iterations": 600000, "salt": "<16 bytes, base64>" },
     "cipher": { "name": "AES-256-GCM", "chunk_size": 1048576, "nonce_prefix": "<8 bytes, base64>" },
     "payload": "_wsnp/encrypted",
     "payload_bytes": 1532871
   }
   ```
3. `_wsnp/encrypted`: the **whole open `.wsnp`** (its own `mimetype`, manifest, page and files),
   encrypted. Decrypting it gives back an ordinary open `.wsnp`, to which everything else in this
   document applies.

Encryption:

- **Key**: PBKDF2 with SHA-256 over the password (UTF-8, Unicode NFC), the random 16-byte `salt`
  and `iterations` rounds, giving a 256-bit AES key. Writers **must** use at least 600 000 rounds
  (the OWASP figure in 2026); the number is in the file so it can grow. Readers **must** refuse
  fewer than 100 000.
- **Chunks**: the open file is cut into chunks of `chunk_size` bytes (1 MiB; between 1 KiB and
  16 MiB), the last one shorter (an empty file is one empty chunk). Chunk *i* (from 0) is encrypted
  with AES-256-GCM, a 128-bit tag, the nonce `nonce_prefix` (8 random bytes) followed by *i* as a
  4-byte big-endian number, and as additional data *i* (4 bytes, big-endian) followed by one byte,
  1 for the last chunk and 0 for the others. `_wsnp/encrypted` is the encrypted chunks one after
  the other, each `chunk_size + 16` bytes long except the last.
- Because of the additional data, chunks cannot be reordered, dropped or the file cut short
  without the reader noticing. `payload_bytes`, the size of the open file, is checked after
  decryption.
- A wrong password makes the first chunk fail; readers **should** then say "wrong password" rather
  than "damaged file".

Rules:

- Nothing about the page is readable in a protected file: title, description, address, preview and
  file names are all inside the encrypted content. A reader can only say that the file is
  protected until the password is given.
- A forgotten password cannot be recovered. Programs that protect files **must** say so, and
  **must not** store the password.
- A reader that does not support protection **must** say "this snapshot is password-protected",
  never "broken file".
- Readers **should** decrypt in memory and never write the open content to disk unless the user
  asks for an unprotected copy.

## 10. Readers and web apps

A reader, whether a desktop program, an installable web app or a web site, **should**:

- read entries straight from the archive by its central directory, without unzipping to disk;
- serve each file with the `media_type` of the manifest;
- show the page in a sandboxed frame without `allow-same-origin` (or on an origin of its own), so a
  snapshot can never reach the reader or other snapshots;
- apply a Content Security Policy that keeps everything inside the snapshot, for example
  `default-src 'self' data:; script-src 'self'; style-src 'self' 'unsafe-inline' data:`
  (`'self'` being wherever the reader serves the snapshot's files from). A valid `.wsnp` works
  under it with no violation.

An installable web app can register the file types in its web app manifest:

```json
"file_handlers": [{
  "action": "/open",
  "accept": {
    "application/vnd.wsnp+zip": [".wsnp"],
    "application/vnd.wsnp.x+zip": [".wsnpx"]
  }
}]
```

(the second line for readers that support `.wsnpx`).

### Validation

A reader **must** check, and refuse the file if any check fails (saying why):

1. It is a ZIP, and its first entry is `mimetype`, stored, with no extra field.
2. The media type is `application/vnd.wsnp+zip` (a `.wsnpx` is refused as in section 8 by readers
   that do not support it, and checked as `.wsnpx` by those that do; anything
   else is not a WSNP file).
3. If `encryption.json` is present: the protected layout of section 9, then, with the password,
   the decrypted file from step 1.
4. `manifest.json` is present and valid JSON, `format` is `"wsnp"` and the major version of
   `format_version` is one the reader knows.
5. The required fields are present with the right types, and `source.url` is an address.
6. Every entry name follows section 5; no entry is compressed with another method than stored or
   DEFLATE; no ZIP-level encryption.
7. Every entry (except `mimetype`, `manifest.json` and, from 1.1, `signature.json`) is listed in `files`, and every file listed
   is present, with the same size, the same SHA-256 and a `media_type`.
8. The page (`pages[0].entry`) and the `preview`, when declared, are present.

If `signature.json` is present (1.1), a reader that knows 1.1 **must** also check it as section 12 says, and a file whose signature does not check is **not valid**. A file with no
`signature.json` is valid: every file written before 1.1 is such a file, and a reader tells its user that the metadata is not protected.

A reader **should** also check that no page contains inline script or a reference that would load
from the network, and treat a file that fails as unsafe to show with scripts.

For a `.wsnpx`, step 4 expects `format` `"wsnpx"`, step 8 checks every page of `pages` and
`app.entry`, and a reader also checks that:

9. `app` has a `name`, a `version`, a `description` and a `permissions` list, and `app.origins`
   lists only `https://` (or `http://localhost`) origins, present only with `network`.
10. Every script the pages load is a file of the archive listed in `files`; no page loads a script
    from elsewhere, not even with `network`.

The reference validator (`wsnp-check.mjs`) checks `.wsnp` files; the `.wsnpx` checks come with the first reader that
supports them.

## 11. Versions

`format_version` is `"major.minor"`. A reader opens any file whose major version it knows and
ignores fields and entries it does not know; a new minor version only adds. A new major version
may change anything, and readers refuse majors they do not know. `encryption_version` follows the
same rule for section 9.

1.1 adds the signature of section 12 and nothing else. Possible additions in a later 1.x: several pages in `.wsnp`, text
extracted for search.

## 12. Signing the manifest (1.1)

The hashes of section 6 protect the files, but a `.wsnp` is a ZIP: anyone can unzip it, change the manifest (its title, the address the page came from, the date) and zip it again, and nothing in the file tells.
A signature does, because it can be made only by someone who holds a key that is not in the file. The design, the reasons and what it does not cover are in
[`MANIFEST-SIGNING.md`](MANIFEST-SIGNING.md).

### The file

`signature.json` is at the root, beside `manifest.json`, and like `manifest.json` it is **not listed in `files`** (it signs the manifest that would list it). It is UTF-8 JSON:

```json
{
  "signature_version": "1.0",
  "algorithm": "Ed25519",
  "public_key": "A6EHv/POEL4dcN0Y50vAmWfk1jCbpQ1fHdyGZBJVMbg=",
  "signed": "manifest.json",
  "manifest_sha256": "5d7ad4fd156257bf10d2772ee5a6c2f41e7cb835e0732c3d3fc0f7e765db9c60",
  "signature": "wVPgpLe7+q5VYI58HQ2USJbELBdfrPo1hkQzauk0lzTLLo5V0jL/2fumEDdo+rjgdFKAf6SM5Bei8Tx7VY33AA=="
}
```

| Field | Meaning |
| --- | --- |
| `signature_version` | `"1.0"` |
| `algorithm` | `"Ed25519"` (the key is 32 bytes, the signature 64) or `"ECDSA-P256-SHA256"` (the key is the 65-byte uncompressed point, the signature the 64 bytes `r‖s` that Web Crypto gives) |
| `public_key` | the writer's public key, raw, in base64 |
| `signed` | always `"manifest.json"` |
| `manifest_sha256` | the SHA-256 of the exact bytes of `manifest.json` that were signed, in hex: a reader says "the manifest was edited" from this before it checks the signature |
| `signature` | the signature over the exact bytes of `manifest.json` as stored in the archive (for ECDSA, over their SHA-256), in base64 |

Because the manifest holds the SHA-256 of every other file, the signature covers the whole archive. The signature is made **after** the manifest is final: any later change to the manifest's bytes, even of spacing, breaks it.
In a protected file (section 9) `signature.json` is inside the encrypted content, part of the open file.

### The signer

The signer is identified by the **fingerprint**: the SHA-256 of the raw public key, in lowercase hex. To show it to people, the first 128 bits, in groups of four hex digits, upper case: `5647-5AA7-5463-474C-0285-DF5D-BF2B-CAB7`.
A fingerprint says *which key* signed, not who the person is: a reader keeps the fingerprints its user has chosen to trust (trust on first use) and tells a key it does not know from one it does.

A writer **should** make its key with the private part not extractable (Web Crypto `extractable: false`), keep it in its own storage, and **must not** write the private key into any file.

### Checking

A reader that knows 1.1:

1. finds no `signature.json`: the file is *unsigned* (valid; the metadata is not protected);
2. cannot read it (not JSON, a field missing, a `signed` other than `manifest.json`, a key or a signature of the wrong size, bytes that are not base64): *not valid*;
3. does not know the `algorithm`: *not valid* (a reader cannot tell a signature it cannot check from one that would fail);
4. finds that the SHA-256 of `manifest.json` is not `manifest_sha256`: *not valid*, the manifest was edited after it was signed;
5. finds that the signature does not check with `public_key`: *not valid*;
6. otherwise the file is *signed* by that fingerprint, and the manifest is what they signed.

### Known answer

An implementation can check itself against this: the Ed25519 key made from the seed `00 01 02 … 1f` (32 bytes) signs the manifest bytes
`{"format":"wsnp","format_version":"1.1","title":"Known answer"}` followed by a line feed (`0a`), and gives exactly the `public_key`, `manifest_sha256` and `signature` of the example above.
The fingerprint of that key is `56475aa75463474c0285df5dbf2bcab73da651358839e9b77481b2eab107708c`. (Ed25519 is deterministic: the same key and bytes always give the same signature.)

