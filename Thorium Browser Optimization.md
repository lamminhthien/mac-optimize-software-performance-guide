# Thorium Browser — Optimization Log

**Date:** 2026-08-23
**System:** macOS · Apple M4 (arm64)
**Thorium version:** 151.0.7922.72 (`/Applications/Thorium.app`)

---

## Audit Findings (before changes)

| Check | Finding | Status |
|---|---|---|
| Binary architecture | Native arm64 on Apple M4 | OK (no Rosetta) |
| Hardware acceleration | Not overridden → default ON | OK |
| Launch flags | None | Untuned |
| chrome://flags experiments | Empty | Untuned |
| In-app performance settings | Defaults | Untuned |
| DNS prefetch / network prediction | Defaults | OK |
| Profiles | Default + Profile 1, both stock | OK |

**Verdict:** nothing broken, but completely stock — no runtime tuning beyond Thorium's built-in compile-level optimizations.

---

## Changes Applied

### 1. Feature flags in `Local State`

File: `~/Library/Application Support/Thorium/Local State`
Key: `browser.enabled_labs_experiments` (equivalent to setting them in `chrome://flags`)
Active regardless of how the browser is launched.

| Flag | Effect |
|---|---|
| `enable-gpu-rasterization@1` | Force GPU rasterization |
| `enable-zero-copy@1` | Zero-copy raster path (less memory copying) |
| `canvas-oop-rasterization@1` | Hardware-accelerated 2D canvas |
| `smooth-scrolling@1` | Compositor smooth scrolling |

### 2. Command-line launch flags

This build has no `chrome_command_line` support, so CLI flags require a launcher.

| Flag | Effect |
|---|---|
| `--process-per-site` | Fewer renderer processes → lower RAM usage |
| `--disk-cache-size=1073741824` | 1 GB disk cache |
| `--media-cache-size=536870912` | 512 MB media cache |
| `--enable-features=ParallelDownloading` | Parallel download connections |

### 3. In-app settings (UI)

These settings are configured inside Thorium and apply even when launched from the original app icon.

Open `chrome://settings/performance`:

- **Memory Saver:** `On` (suspends inactive tabs to reduce RAM pressure)
- **Preload pages:** `Off` (reduces speculative background requests / memory use)

Open `chrome://settings/system`:

- **Continue running background apps when Thorium is closed:** `Off`

These are high-impact user-facing settings and should be tuned together with flags.

### 4. Launchers created

- **`~/bin/thorium-fast`** — terminal launcher with all CLI flags above
- **`~/Applications/Thorium-Fast.app`** — app wrapper (same icon) for Dock use

Both run the real binary at `/Applications/Thorium.app/Contents/MacOS/Thorium`.

### 5. Backup

Pre-change configs copied to:
`~/.thorium-backup-<timestamp>/`
Contains: `Local State`, `Default-Preferences`, `Profile1-Preferences`.
Delete this folder anytime to discard the rollback option.

---

## How to Use

- **Full optimizations:** drag `Thorium-Fast` from `~/Applications` into the Dock and launch from it.
- **Terminal:** run `~/bin/thorium-fast [url]`.
- **Original Thorium icon:** groups 1 and 3 apply; CLI flags are skipped.
- To verify flags are live: open `chrome://version` → check *Command Line*; open `chrome://flags` → enabled entries listed at top.
- To verify in-app tuning: open `chrome://settings/performance` and confirm **Memory Saver = On**, **Preload pages = Off**.

## Rollback

```sh
cp ~/.thorium-backup-*/Local\ State "$HOME/Library/Application Support/Thorium/Local State"
rm ~/bin/thorium-fast
rm -r ~/Applications/Thorium-Fast.app
```

(Quit Thorium before restoring the backup.)

Also revert UI settings manually if needed:

- `chrome://settings/performance` → set **Memory Saver** and **Preload pages** back to your preferred defaults.
- `chrome://settings/system` → re-enable background apps if you rely on them.
