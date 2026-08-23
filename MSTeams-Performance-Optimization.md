# Microsoft Teams Performance Optimization (macOS)

> Audit date: 2026-08-23 · Teams 26183.1901.4874.5228 (new client) · macOS 27.0 · Apple M4 · 24 GB RAM

## 1. Summary

Baseline state before optimization:

| Metric | Value |
|---|---|
| Total Teams data footprint | 2.4 GB |
| EBWebView browser cache (`Caches/Microsoft/MSTeams`) | 1.4 GB |
| Service Worker CacheStorage | 511 MB |
| Local chat database (`IndexedDB` leveldb) | 119 MB |
| WebStorage + File System cache | ~230 MB |
| Accounts signed in | 2 tenants (`a`, `b`) |
| `keep_app_running_on_close` | `true` (runs in background forever) |

After cleanup the data footprint dropped to **127 MB (~95% reduction)**.

## 2. Symptoms reported

1. Mac runs **hot during screen share / calls**
2. Slow **app loading time** on launch
3. **Laggy switching** between chats/workspaces when using 2 accounts
4. General slowness

## 3. Root causes found

### 3.1 Bloated Chromium cache (1.4 GB)

Teams (new client) is an Electron-style app using an embedded Edge WebView
(`EBWebView`). Its HTTP/browser cache had grown unbounded:

```
~/Library/Containers/com.microsoft.teams2/Data/Library/Caches/Microsoft/MSTeams/EBWebView   → 1.4 GB
```

Every launch scans and validates this cache → slow startup.

### 3.2 Stale Service Worker cache (511 MB)

```
.../MSTeams/EBWebView/WV2Profile_tfw/Service Worker/CacheStorage → 511 MB
```

Stale service-worker payloads conflict with updated app code after each
Teams update — a classic cause of slow loading and broken UI state.

### 3.3 Oversized local chat database (119 MB LevelDB)

Both tenants share one IndexedDB store:

```
.../WV2Profile_tfw/IndexedDB/https_teams.microsoft.com_0.indexeddb.leveldb → 119 MB
```

A fragmented/bloated leveldb makes every chat/workspace switch issue slow
queries → **this was the direct cause of laggy account switching**.

### 3.4 Teams never actually quits

`app_settings.json` contained `"keep_app_running_on_close": true`.
Closing the window left renderer processes running in background,
contributing heat and CPU even when "not using" Teams.

### 3.5 Heat during calls/screen share

On Apple Silicon this is usually caused by in-app ML effects, not the app
itself: background blur/replace, noise suppression, camera preview, and
4K screen-share encoding. See §6 for settings.

## 4. Fixes applied (terminal)

> Teams was fully quit first (`osascript -e 'quit app "Microsoft Teams"'`
> then `pkill -f com.microsoft.teams2`). The only surviving process is the
> system Core Audio driver (`MSTeamsAudioDevice.driver`) which is normal.

### 4.1 Clear caches (safe, re-created automatically)

```bash
T=~/Library/Containers/com.microsoft.teams2/Data/Library

rm -rf "$T/Caches/Microsoft/MSTeams" \
       "$T/Caches/WebKit" \
       "$T/Caches/com.plausiblelabs.crashreporter.data"

rm -rf "$T/Application Support/Microsoft/MSTeams/EBWebView/WV2Profile_tfw/Service Worker/CacheStorage" \
       "$T/Application Support/Microsoft/MSTeams/EBWebView/WV2Profile_tfw/GPUCache" \
       "$T/Application Support/Microsoft/MSTeams/EBWebView/WV2Profile_tfw/DawnWebGPUCache" \
       "$T/Application Support/Microsoft/MSTeams/EBWebView/WV2Profile_tfw/DawnGraphiteCache" \
       "$T/Application Support/Microsoft/MSTeams/EBWebView/WV2Profile_tfw/File System"
```

### 4.2 Reset local chat database (fixes laggy account switching)

Chat history re-syncs from the server automatically; login session is kept.

```bash
T=~/Library/Containers/com.microsoft.teams2/Data/Library/Application Support/Microsoft/MSTeams/EBWebView/WV2Profile_tfw

rm -rf "$T/IndexedDB" "$T/WebStorage" "$T/Local Storage/leveldb"
rm -rf ~/Library/Containers/com.microsoft.teams2/Data/Library/Application\ Support/Microsoft/MSTeams/media-stack
```

### 4.3 Quit Teams fully on window close

```bash
python3 - <<'EOF'
import json, os
p = os.path.expanduser("~/Library/Containers/com.microsoft.teams2/Data/"
                       "Library/Application Support/Microsoft/MSTeams/app_settings.json")
d = json.load(open(p))
d["keep_app_running_on_close"] = False
json.dump(d, open(p, "w"))
EOF
```

## 5. Results

| Metric | Before | After |
|---|---|---|
| Data footprint | 2.4 GB | 127 MB |
| Browser cache | 1.4 GB | rebuilt fresh |
| Chat DB (leveldb) | 119 MB | clean re-sync |
| Background after close | running | quits |

First launch after cleanup re-downloads chat history — expect a few
minutes of syncing; subsequent launches are fast.

## 6. In-app settings for cooler calls/screen share

These cannot all be set from terminal — change once inside Teams:
Settings → Devices / Calls:

| Setting | Recommended |
|---|---|
| Background effects | None or static blur (blur is cheaper than image/video) |
| Noise suppression | Medium (High costs more CPU) |
| Camera resolution | 1080p or lower; off when not needed |
| Hardware acceleration | **Keep ON** (Apple Silicon Metal GPU encoding is efficient) |

macOS-level (optional, System Settings):

- Displays → reduce brightness slightly during long calls
- Accessibility → Display → Reduce Motion / Reduce Transparency
- Battery/Energy: keep MacBook ventilated; avoid charging at 100% under load

## 7. Maintenance routine (run monthly or when slow again)

```bash
#!/bin/zsh
# teams-clean.sh — quit Teams, purge caches, relaunch
osascript -e 'quit app "Microsoft Teams"'; sleep 3
pkill -f com.microsoft.teams2 2>/dev/null; sleep 1
T=~/Library/Containers/com.microsoft.teams2/Data/Library
rm -rf "$T/Caches/Microsoft/MSTeams" \
       "$T/Application Support/Microsoft/MSTeams/EBWebView/WV2Profile_tfw/Service Worker/CacheStorage" \
       "$T/Application Support/Microsoft/MSTeams/EBWebView/WV2Profile_tfw/GPUCache"
open -a "Microsoft Teams"
echo "Teams cache purged."
```

Do **not** delete `IndexedDB` more often than needed — it forces a full
chat re-sync each time.

## 8. Notes

- macOS 27.0 is a beta OS; Electron apps occasionally run hotter on betas.
  If heat persists after §6, compare against the stable macOS release.
- Only the new Teams client was installed (no classic duplicate) — good.
- Login items are minimal (Cloudflare WARP, GoTiengViet) — not a factor.
