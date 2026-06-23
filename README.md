# Pake CI — Build desktop apps from any URL, automatically

A minimal GitHub Actions setup for building [Pake](https://github.com/tw93/Pake) apps across **macOS / Windows / Linux** with optional Apple notarization.

## What it does

- Wraps any URL into a Tauri-based native desktop app (`.dmg`, `.msi`, `.deb`)
- Matrix build across all three desktop OSes
- Optionally notarizes macOS apps via App Store Connect API (no Apple ID password needed)
- Optionally attaches artifacts to a GitHub Release

## Quick start

1. **Add apps to `apps.json`** — one entry per app:
   ```json
   [
     {
       "name": "excalidraw",
       "title": "Excalidraw",
       "url": "https://excalidraw.com",
       "width": 1200,
       "height": 800
     }
   ]
   ```
   Supported per-app fields: `name`, `title`, `url`, `icon`, `width`, `height`, `incognito`, `hide_title_bar`, `dark_mode`, `always_on_top`, `enable_find`, `multi_instance`, `safe_domain`.

2. **Trigger a build**:
   - **All apps, all platforms**: GitHub → Actions → "Build All Popular Pake Apps" → Run workflow (optionally set a tag for release)
   - **One app only**: same workflow, fill "only" with the app name
   - **Single ad-hoc build** (no apps.json): "Build Single Pake App" → fill URL + name + flags manually

3. **Download artifacts**: from the workflow run page (retention 30 days).

## Build a release

Trigger the workflow with a tag like `v2026.06.24` and the artifacts get auto-attached to a GitHub Release with that tag. Locally:

```bash
# Build Excalidraw + ChatGPT + YouTube and publish as v2026.06.24
gh workflow run build-popular.yml -f tag=v2026.06.24
```

## Apple notarization (optional)

For macOS apps distributed outside your own machine, enable notarization:

1. Go to https://appstoreconnect.apple.com/access/api → "App Store Connect API" → Generate API Key with **App Manager** role
2. Download the `.p8` file → encode it: `base64 -i AuthKey_XXXXXXXXXX.p8 | pbcopy`
3. Add these secrets in Settings → Secrets and variables → Actions:
   - `PAKE_API_KEY_BASE64` — base64-encoded `.p8` contents
   - `PAKE_API_KEY_ID` — the 10-char Key ID (e.g. `AB12CD34EF`)
   - `PAKE_API_ISSUER_ID` — the Issuer UUID (e.g. `572b8e8f-1234-5678-9abc-def012345678`)
4. Run the workflow — notarization happens automatically, DMG ships with `Developer ID` signature + notarization ticket

If you don't add secrets, builds still work — just signed ad-hoc (works locally only, Gatekeeper will warn on other machines).

## File layout

```
pake-ci/
├── apps.json                              # app list, edit this to add apps
├── .github/
│   ├── actions/pake-setup/action.yml      # composite action: Rust + Node + pake-cli + linux deps
│   └── workflows/
│       ├── build-popular.yml              # matrix build (3 OS × N apps)
│       ├── build-single.yml               # callable single-app workflow
│       └── lint.yml                       # validate apps.json schema
└── README.md
```

## Adapting for your own fork

This repo can be a thin launcher around Pake. To customize the Rust/JS inject code:

1. Fork the repo
2. Clone tw93/Pake locally, apply your changes, publish to your fork
3. Edit `build-single.yml` to `actions/checkout@your-fork` with `repository: your-fork/pake`
4. Continue editing `apps.json` here — the build will pull your custom pake source

## Local build (debug before pushing)

```bash
# Same toolchain CI uses
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
. "$HOME/.cargo/env"
npm install -g pake-cli

# Quick local build
pake https://excalidraw.com --name Excalidraw --width 1200 --height 800

# Full app from apps.json
jq -c '.[] | .url, .name' apps.json | xargs -L 2 pake
```

## Related

- [Pake](https://github.com/tw93/Pake) — the upstream tool
- [Tauri](https://tauri.app/) — the underlying framework
- [Apple Notarization](https://developer.apple.com/documentation/security/notarizing_macos_software_before_distribution) — required for distribution outside your Mac