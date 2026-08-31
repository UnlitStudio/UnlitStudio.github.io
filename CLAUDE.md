# Unlit Studio GitHub Pages — Static Config & Content

Public static repo served at `unlitstudio.github.io`. It is the launcher's bootstrap: the prelauncher
cannot start without it, and every dynamic download URL in the ecosystem comes out of one file here.
Pushing to `master` publishes immediately — **a bad push breaks every installed client**, with no release
required and no rollback but another push.

Jekyll builds the site (`_config.yml` sets the title and excludes `CLAUDE.md`, `.idea`, `.claude`).
`index.html` is a placeholder containing the words "Hello world"; nobody is meant to visit the site.

## Repo Layout

```
unlit-studio-github/
├── _config.yml                          # Jekyll: title + exclude list
├── index.html                           # placeholder, "Hello world"
└── Public/
    ├── Data/                            # ~2.6 MB
    │   ├── Links.txt                    # AES-256-ECB config  ← prelauncher + launcher
    │   ├── Links-Test.txt               # TEST counterpart
    │   ├── Content.json                 # LIVE catalogue, 155 entries
    │   ├── Content-Test.json            # TEST catalogue, 159 entries
    │   ├── ChunkAndBrand.json           # orphaned AND malformed — see below
    │   ├── ReleaseNotes.json            # Updates screen feed
    │   └── Homepage/
    │       ├── CarousselData.json       # home carousel slides
    │       └── *.jpg                    # 11 carousel images
    └── Images/                          # ~27 MB
        ├── Thumbnails/                  # 216 PNGs — brand tiles
        └── *.jpg / *.png                # ~30 render / spotlight / marketing images
```

## Who Reads What

| File | Consumer | How |
|---|---|---|
| `Links.txt` | **Prelauncher** (hardcoded URL) and **launcher** (`AppEnvironment.ConfigUrl`) | GET → AES-256-ECB decrypt → `linkData` |
| `Links-Test.txt` | Launcher only, when `AppEnvironment.IsTest` | Prelauncher has no TEST branch |
| `Content.json` | Launcher — Library grid, Home strip, onboarding picker, update scan | `AppEnvironment.ContentJsonUrl` |
| `Content-Test.json` | Launcher, TEST users only | same resolver |
| `ReleaseNotes.json` | Launcher `ReleaseNotesMenu` — Updates screen | hardcoded URL |
| `Homepage/CarousselData.json` | Launcher `CarrouselViewModel` — home carousel | hardcoded URL |
| `Images/Thumbnails/*.png` | Launcher, via the `Image` field in Content.json | **`raw.githubusercontent.com`**, not this domain |
| `Homepage/*.jpg` | Launcher, via `ImageURL` in CarousselData.json | `unlitstudio.github.io` |
| `ChunkAndBrand.json` | **Nothing.** No C#, Python or JS in the workspace fetches it | — |
| `Images/*.jpg` (root) | **Nothing in this workspace.** Presumably the Webflow site or legacy | — |

Earlier docs said the dashboard references `Public/Images/`. It does not — `unlit-dashboard/` contains no
reference to either GitHub host.

## File Formats

### `Links.txt` / `Links-Test.txt`

A single base64 blob, AES-256-ECB, zero padding. The key is a 32-character literal hardcoded **identically**
in `Scripts/AesOperation.cs` (launcher) and `AesOperation.cs` (prelauncher) — rotating it means shipping
both apps and re-encrypting this file in the same moment.

Both readers trim everything after the final `}` before deserializing, because zero padding leaves trailing
bytes. Decrypts to `RootObject` → `Launcher` / `Prelauncher` / `Window_Install`, carrying the game version
and download URLs, the launcher update URLs, and the DLC pak/version endpoints.

**The prelauncher does not use its URLs verbatim.** It rewrites `.txt` → `-live.txt` and `.zip` →
`-live.zip` before downloading, so both filenames must exist on the Space. See
`unlit-studio-prelauncher/CLAUDE.md`.

### `Content.json` / `Content-Test.json`

```json
{ "Content": [ {
    "brand": "Koopmans Meubelen",
    "chunkid": 10292,
    "downloadlink": "https://unlitstudio.ams3.digitaloceanspaces.com/content/paks/pakchunk10292-Windows.pak",
    "Image": "https://raw.githubusercontent.com/UnlitStudio/UnlitStudio.github.io/master/Public/Images/Thumbnails/KoopmansMeubelen.png",
    "filters": ["Meubels"]
} ] }
```

LIVE has **155 entries**, ids 10101–10292. TEST has **159**, ids 10101–10296 — the same list plus four
pak entries under test.

- **`downloadlink` is dead metadata.** No code path reads it. Pak and version-file URLs are always composed
  from `AppEnvironment.PaksBaseUrl` / `VersionFilesBaseUrl`, so a TEST entry downloads from
  `unlitstudiodevelopment` regardless of the production URL sitting in this field. (Every TEST entry
  currently carries a production `downloadlink` — harmless, but do not trust it.)
- **TEST entries use the chunk id as the brand name** (`"brand": "10296"`) and reuse an arbitrary existing
  thumbnail as a placeholder. Give an entry a real name and image when it graduates to `Content.json`.
- 14 filter categories in use: Behang, Buiten, Deuren, Haarden, Meubels, Raamdecoratie, Textiel, Trappen,
  Verf, Verlichting, Vloeren, Vloerkleden, Wandafwerking, Woonaccessoires.
- All 155 referenced thumbnails exist. 61 of the 216 PNGs in `Thumbnails/` are unreferenced — the numbered
  `1.png`–`129.png` set appears to be legacy, superseded by the brand-named files.

### `ReleaseNotes.json`

Array, newest first. Generated from launcher git history by `tools/Generate-ReleaseNotes.ps1`; the launcher
only renders it.

```json
[ { "product": "studio", "version": "2026.1.0", "date": "30 maart 2026",
    "channel": "stable", "commit": "fb0c5e6…",
    "items": [ { "kind": "new", "text": "…" } ] } ]
```

`kind` is `new` / `imp` / `fix`. `product` distinguishes the game from the launcher. Dates are Dutch prose,
not ISO. Currently one release entry.

### `Homepage/CarousselData.json`

Array of slides: `Title`, `Subtitle`, `ButtonText`, `ButtonLink`, `ImageURL`. Copy is Dutch; links point at
the help centre, the WhatsApp community and similar. Images live beside it in `Public/Data/Homepage/`.

## Known Problems

**`ChunkAndBrand.json` is orphaned.** No C#, Python or JS in the workspace fetches it — the launcher
never has, and the dashboard keeps its own copy of the mapping in `brands.py`. It parses (157 ids, 0–10289;
a missing comma on line 157 was fixed on 2026-08-24, before which no parser could read it at all), but it
has no consumer. Either give it an owner or delete it. The root workspace `CLAUDE.md` used to list it as
launcher content metadata; that has been corrected.

**The chunk→brand mapping exists in three divergent places**, with no shared source:

| Source | Ids | Style | Example for 10001 |
|---|---|---|---|
| `ChunkAndBrand.json` (here) | 157 | code-ish keys | `CoreMaterials` |
| `Content.json` `brand` field (here) | 155 | display names | *not in the catalogue* |
| `unlit-dashboard/brands.py` | 134 | display names | `Default - Models` |

Measured divergence (2026-08-24):

- **`brands.py` is missing 29 ids that are live in `Content.json`** — 10212, 10221, and everything from
  10255 up. `brand_for_chunk()` falls back to `"Chunk {id}"`, so the dashboard's brand-level charts label
  those 29 brands by number instead of name.
- **26 of the 130 ids shared by `ChunkAndBrand.json` and `brands.py` disagree on the name.** Most are
  cosmetic (`WADM` vs `Werk Aan De Muur`, `WOOOD` vs `WOOD`), but at least one is a straight conflict:
  10154 is `Pastoe` in one and `Atelier Artiforte` in the other. And 10001/10002 are **inverted** —
  `CoreMaterials`/`CoreModels` here versus `Default - Models`/`Default - Materials` there.
- `ChunkAndBrand.json` lacks 10290–10292, the three newest catalogue entries.

Adding a brand means touching `Content.json` here and `brands.py` in the dashboard, and uploading the pak to
the Space. Getting one wrong shows a brand in the Library that cannot install, or usage attributed to the
wrong name.

**Brand tiles are served from `raw.githubusercontent.com`, not GitHub Pages.** Every `Image` URL in both
catalogues points at the raw content host. That is a source-code endpoint with its own rate limits and no
CDN guarantees, and it bypasses the Pages domain entirely — worth moving to `unlitstudio.github.io/Public/…`
or the Spaces CDN if tile loading is ever flaky.

**27 MB of images are versioned in git**, most of them unreferenced by any code here.

## Publishing

```bash
git add .
git commit -m "chore: update config"
git push origin master        # live within a minute or two
```

There is no staging, no preview and no CI. Validate JSON before pushing:

```bash
python -c "import json,sys; [json.load(open(f,encoding='utf-8')) for f in sys.argv[1:]]" \
  Public/Data/Content.json Public/Data/Content-Test.json Public/Data/ChunkAndBrand.json \
  Public/Data/ReleaseNotes.json Public/Data/Homepage/CarousselData.json
```

All five parse as of 2026-08-24. Run this before every push — a broken catalogue reaches every client.

To put a new pak in front of internal testers: upload it to the `unlitstudiodevelopment` Space, add an entry
to `Content-Test.json`, push. It appears in the Library only for `@unlit.studio` accounts with the TEST
switch on (`pilot@unlit.studio` excluded), and stays out of `Content.json` until it goes live.
