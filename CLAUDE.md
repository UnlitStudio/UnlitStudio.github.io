# Unlit Studio GitHub Pages — Static Content Repository

Static content repository hosted on GitHub Pages at `unlitstudio.github.io`. Serves as a public distribution point for configuration data and assets consumed by the desktop applications.

## Purpose

- Hosts **encrypted configuration** (URLs, version info) fetched by the prelauncher
- Provides **content metadata** (brand/chunk data) for the main launcher
- Serves **public images** used in the dashboard and launcher UI

## Structure

```
unlit-studio-github/
├── index.html                          # Landing page
└── Public/
    ├── Data/
    │   ├── Links.txt                   # AES-256 encrypted config URLs (fetched by prelauncher)
    │   ├── Links-Test.txt              # TEST-environment counterpart of Links.txt
    │   ├── ChunkAndBrand.json          # Chunk-to-brand mapping data
    │   ├── Content.json                # Content metadata (LIVE brand list)
    │   ├── Content-Test.json           # TEST brand list — new paks under test go here
    │   └── ReleaseNotes.json           # Release notes feed (launcher Updates screen)
    └── Images/
        └── Homepage_*.jpg              # Brand/homepage images
```

### TEST-environment manifests

`Links-Test.txt` and `Content-Test.json` are the TEST counterparts, selected at runtime by
`AppEnvironment.ConfigUrl` / `AppEnvironment.ContentJsonUrl` in the launcher when an
internal (`@unlit.studio`, excluding `pilot@`) user turns the TEST switch on.

To put a new pak in front of internal testers, add its entry to `Content-Test.json` and
push — it shows up in the Library for TEST users only, and stays out of `Content.json`
until it's ready to go live. The entry's `downloadlink` field is ignored by the launcher;
pak and version-file URLs are composed from the environment's DigitalOcean Space
(`unlitstudiodevelopment` for TEST), so the pak must be uploaded there.

## How It's Used

1. **Prelauncher** fetches `https://unlitstudio.github.io/Public/Data/Links.txt`, decrypts with AES-256 ECB to get config URLs (version check, download links)
2. **Main launcher** reads content metadata from `Public/Data/` for chunk/brand information
3. **Dashboard** may reference images from `Public/Images/`

## Key Details

- `Links.txt` is **AES-256 ECB encrypted** — the decryption key is in `AesOperation.cs` (both prelauncher and main launcher)
- This is a **separate Git repo** from the main launcher repository
- Changes here affect what the prelauncher and launcher fetch at runtime — update carefully
