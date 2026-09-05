# Mo (Mole) — Complete Manual Guide (via Terminal CLI)

> **Tool:** Mole — binary `mo` / `mole` (`/opt/homebrew/bin/mo` → `../Cellar/mole/1.50.0/bin/mo`)
> **Installed version:** 1.50.0 (update 1.53.0 available) · macOS 27.0 arm64 · Homebrew
> **Explored via:** `mo --help`, `mo --version`, `mo clean --help`, `mo <subcmd> --help` (all 9), `cat /opt/homebrew/Cellar/mole/1.50.0/libexec/bin/clean.sh`, `ls /opt/homebrew/Cellar/mole/1.50.0/libexec/lib/clean/*`, `cat README.md`, `brew info mole`, `mo status --json`
> **Date:** Sep 05, 2026 · Status: ✅ Verified locally
> **Home:** https://mole.fit · GitHub: https://github.com/tw93/mole · License: GPL-3.0

> **Correction:** Previous doc mistakenly targeted `moe` (GNU Moe editor). User clarified tool is `mo` (Mole) — this guide replaces it and documents `mo clean` fully.

---

## 1. What is Mole / `mo`?

Mole is an **all-in-one Mac cleaner** that replaces CleanMyMac + AppCleaner + DaisyDisk + iStat Menus in a **single terminal binary**. Tagline: *Deep clean and optimize your Mac* (https://mole.fit).

Key claims from `README.md:1`:
- Deep cleaning: caches, logs, leftovers, orphaned app data → reclaim GBs
- Smart uninstaller: app + launch agents, prefs, hidden remnants
- Disk insights: visualizes usage, finds large files, rebuilds caches
- Live monitoring: real-time CPU/GPU/mem/disk/network

Install paths:

```bash
brew install mole          # recommended, macOS 14+ (you used this)
curl -fsSL https://raw.githubusercontent.com/tw93/mole/main/install.sh | bash
# optional: bash install.sh -s latest  (main branch) or -s 1.17.0
```

Binaries are `mo` (short) and `mole` (long) — identical. This guide uses `mo`.

---

## 2. Verify Installation (Terminal CLI)

Commands executed on this Mac (zsh, darwin, Homebrew):

```bash
which mo
# /opt/homebrew/bin/mo -> ../Cellar/mole/1.50.0/bin/mo

mo --version        # also `mole --version`
# Mole version 1.50.0
# macOS: 27.0 / Architecture: arm64 / Kernel: 27.0.0 / SIP: Enabled
# Disk Free: 269.21GB / Install: Homebrew / Shell: /bin/zsh

mo --help           # overview of 13 commands (see §3)
mo clean --help     # see §4
brew info mole      # 1.50.0 → 1.53.0 bottled, 63 files, 9.7MB, GPL-3.0
ls /opt/homebrew/Cellar/mole/1.50.0/libexec/{bin,lib/clean,lib/core}
```

No man page (`man mo` → not found); help is via `--help` and Texinfo-like README.

---

## 3. Command Overview (`mo --help`)

From `mo --help` (colors stripped):

```
COMMANDS
  mo                          Main menu (interactive TUI)
  mo clean                    Free up disk space
  mo uninstall                Remove apps completely
  mo optimize                 Refresh caches and services
  mo analyze                  Explore disk usage
  mo status                   Monitor system health
  mo history                  Review cleanup activity
  mo purge                    Remove old project artifacts
  mo installer                Find and remove installer files
  mo touchid                  Configure Touch ID for sudo
  mo completion               Setup shell tab completion
  mo update                   Update to latest version
  mo remove                   Remove Mole from system
  mo --help                   Show help
  mo --version                Show version

  mo clean --dry-run          Preview cleanup
  mo clean --whitelist        Manage protected caches
  mo optimize --dry-run       Preview optimization
  mo optimize --whitelist     Manage protected items
  mo uninstall --dry-run      Preview app uninstall
  mo history --json           Export cleanup history
  mo purge --dry-run          Preview project purge
  mo installer --dry-run      Preview installer cleanup
  mo touchid enable --dry-run Preview Touch ID setup
  mo completion --dry-run     Preview shell completion edits
  mo purge --paths            Configure scan directories
  mo analyze /Volumes         Analyze external drives only
  mo update --force           Force reinstall latest stable version
  mo update --nightly         Install latest unreleased main branch build
  mo remove --dry-run         Preview Mole removal

OPTIONS
  --debug                     Show detailed operation logs
```

Global `--debug` works with any subcommand.

Interactive TUI (`mo` alone) menu: 1 Clean 2 Uninstall 3 Optimize 4 Analyze 5 Status — navigable with arrows / vim `h/j/k/l`.

---

## 4. `mo clean` — Deep Clean (Focus of this Guide)

### 4.1 Syntax

From `mo clean --help` (and `lib/core/help.sh:show_clean_help`):

```bash
Usage: mo clean [OPTIONS]

Clean up disk space by removing caches, logs, temporary files,
and app leftovers from already-uninstalled apps.

Options:
  --dry-run, -n     Preview cleanup without making changes
  --external PATH   Clean OS metadata from a mounted external volume
  --whitelist       Manage protected paths
  --debug           Show detailed operation logs
  -h, --help        Show this help message
```

**Semantics:**
- `mo clean` targets **leftovers of already-uninstalled apps** + caches/logs/temp. It does NOT uninstall still-installed apps — for that use `mo uninstall` (see §5).
- Safety: conservative boundaries, path validation, protected-directory rules, explicit confirmation for risky deletes. Process guards skip Xcode/Simulator/Final Cut/Autodesk when those apps are running.
- Preview: `~/.config/mole/clean-list.txt` is written on dry-run; deduplicated via inode ledger (see `libexec/bin/clean.sh`).
- Logs: `~/Library/Logs/mole/operations.log` (disable `MO_NO_OPLOG=1`), reviewable via `mo history`.

### 4.2 What `mo clean` Cleans — Modules

From `libexec/lib/clean/*.sh` and `lib/manage/whitelist.sh:get_all_cache_items` (whitelist catalog):

**System & User caches (`caches.sh`, `system.sh`, `user.sh`):**
- User app cache, Trash, iOS firmware `.ipsw`, Font caches, Spotlight, CloudKit, Apple Mail via lib/clean/system.

**App caches (`app_caches.sh`) — examples:**
- **Xcode:** DerivedData `~/Library/Developer/Xcode/DerivedData/*`, `~/Library/Caches/com.apple.dt.Xcode/*`, `Products/*`, DeviceSupport symbols, Caches. Guarded: skips if Xcode/Simulator running.
- **Editors:** VS Code `Code/Cache/*`, `CachedData/*`, `CachedExtensions/*`, `WebStorage/*/CacheStorage/*`; Cursor, Sublime, Zed (`Application Support/Zed/node/cache`), CodeBuddy (Electron fork) caches/logs.
- **Communication:** Discord/Legcord/Slack cache, Zoom, WeChat, Teams (current + legacy `Application Support/Microsoft/Teams/*`), WhatsApp, Skype, Tencent Meeting/WeCom/QQ, Feishu.
- **DingTalk:** `dd.work.exclusive4aliding`, AliLang, `iDingTalk/log/*`.
- **AI:** ChatGPT, Claude desktop, LM Studio, Gemini/ClearcutLogger; Codex desktop *protected* by default.
- **Design:** Sketch, Adobe `Caches/Adobe/*`, `com.adobe.*`, Figma, `Adobe/Common/Media Cache Files/*`.
- **Video:** ScreenFlow, Final Cut Pro cache + **generated caches** under `~/Movies/*.fcpbundle` (`Render Files/High Quality Media`, `Transcoded Media/Proxy Media`) — guarded, only `~/Movies`, never Original Media/Backups. JianyingPro (CapCut CN) `~/Movies/JianyingPro/User Data/Cache/{recognize,frameThumbnail,audioWave,AlgorithmCache,...}` (≈11 subdirs), DaVinci Resolve.
- **3D/CAD:** Blender, Cinema 4D, Autodesk `com.autodesk.*` (guarded: AcCoreConsole/ADPClientService), SketchUp.
- **Productivity:** MiaoYan, Klee, Ora, Filo, Flomo, Quark videoCache, NetNewsWire/MindNode containers, Kaku, Spacedrive thumbnails, Folo cache.
- **Media players:** Spotify (protected if offline `offline.bnk` >1KB or `*.file`), Apple Music/Podcasts/TV, Plex, NetEase/QQMusic.
- **And many more** via `clean/app_caches.sh` (~800 lines) — safely guarded per family.

**Compiler / package / AI caches (whitelist catalog, clean via `caches.sh`/`dev.sh`):**
- Gradle `~/.gradle/{caches/build-cache-*,daemon,workers}`, Maven `~/.m2/repository/*` (protected by default), JetBrains `Library/Caches/JetBrains`, Android Studio/SDK, Bazaar `~/.cache/bazel`, Go `go-build`, Cargo, RustUp, ccache/sccache, SBT/Ivy, Turbo `.turbo`, Next `.next`, Vite `.vite`, Parcel, pre-commit, ruff/mypy/pytest, Flutter, SwiftPM, Zig, Deno, CocoaPods, npm/pnpm/yarn, pip/uv, Conda, HuggingFace/torch/tensorflow, Playwright `ms-playwright*`, Selenium, Ollama `~/.ollama/models` (protected), wandb, Surge, Docker BuildX, Podman, Tart.

**Browser caches:** Safari, Chrome, Firefox, Brave — plus Chrome on-device AI models `OptGuideOnDevice*`, Surge proxy.

> **Default whitelist** (never cleaned unless you remove protection via `mo clean --whitelist`): Playwright browsers, HuggingFace models, Maven repo, Ollama models, Surge Mac, R `renv`, Finder `.DS_Store`. Config: `~/.config/mole/whitelist` (see §4.4).

Example run (from README, illustrative because optimize counts vary by state):

```bash
$ mo clean

Scanning cache directories...

  ✓ User app cache                                           45.2GB
  ✓ Browser cache (Chrome, Safari, Firefox)                  10.5GB
  ✓ Developer tools (Xcode, Node.js, npm)                    23.3GB
  ✓ System logs and temp files                                3.8GB
  ✓ App-specific cache (Spotify, Dropbox, Slack)              8.4GB
  ✓ Trash                                                    12.3GB

====================================================================
Space freed: 95.5GB | Free space now: 223.5GB
====================================================================
```

Note: `IN_USE` CoreSimulator volumes skipped; developer-tools section defers if Xcode running.

### 4.3 Common Workflows

```bash
# Safest: preview first
mo clean --dry-run --debug   # detailed logs + list
cat ~/.config/mole/clean-list.txt  # preview file

# Interactive protection
mo clean --whitelist         # TUI picker (see §4.4)

# Real clean
mo clean                     # interactive confirm per section
mo clean --external /Volumes/MyDrive  # clean OS metadata on mounted volume

# Check what was done
mo history                   # TUI log
mo history --json | jq .
mo history --limit 20
cat ~/Library/Logs/mole/operations.log

# External drives (clean OS dotfiles)
mo clean --external /Volumes/ADATA_SC750
```

**App leftovers rule of thumb:** `mo clean` when app *already* gone; `mo uninstall` when app *still* installed.

### 4.4 Whitelist — Protect Paths (`--whitelist`)

From `lib/manage/whitelist.sh:manage_whitelist` and defaults in `lib/core/base.sh`:

- **Config file:** `~/.config/mole/whitelist` (clean) and `~/.config/mole/whitelist_optimize` (optimize). Legacy `whitelist_checks` auto-migrated.
- **Format:** one absolute path or pattern per line; `#` comments; `~`/`$HOME` expanded; no `..`, control chars, `//`, or protected system roots (`/`, `/System`, `/bin`, `/sbin`, `/usr/bin`, `/etc`, `/var/db`).
- **Safety whitelist always merged** even if you override file (ensures `FINDER_METADATA_SENTINEL` etc. remain).
- **Defaults:** Playwright `ms-playwright*`, HuggingFace `~/.cache/huggingface/*`, Maven `~/.m2/repository/*`, Ollama `~/.ollama/models/*`, Surge, R renv, plus `FINDER_METADATA` (`.DS_Store`).

**TUI:** `mo clean --whitelist` shows categorized picker (system_cache, ide_cache, ai_ml_cache, browser_cache, package_manager, compiler_cache, container_cache). Already-enabled items appear first; custom patterns listed separately. Active config path displayed in title (`~/.config/mole/whitelist`).

```bash
mo clean --whitelist
# → "Whitelist Manager, Select caches to protect"
#   [x] HuggingFace models and datasets
#   [x]  Ollama local AI models
#   etc. — toggle with space, save to whitelist file

# Manual edit:
mkdir -p ~/.config/mole
cat ~/.config/mole/whitelist
# # Mole Whitelist - Protected paths won't be deleted
# /Users/thienlam/Library/Caches/ms-playwright
# $HOME/.cache/huggingface/*
```

Patterns are exact-string matched (no glob expansion at check time) — use the exact catalog string from `get_all_cache_items` or a custom absolute path.

### 4.5 Safety Design (from `README` + `SECURITY.md` refs + `clean.sh`)

- Never broadens scope under uncertainty → skips/refuses/requires stronger confirmation.
- Process guards: Xcode, Simulator, Final Cut, Autodesk — probes via `pgrep`; deferred families reported as `mole_defer_cleanup_family`.
- Path validation: `validate_path_for_deletion`, `is_path_whitelisted`, `should_protect_path`, `holds_compiled_model_cache`, `is_final_cut_pro_generated_cache_target` (only `~/Movies/*.fcpbundle` + exact subdirs).
- DRY_RUN ledger deduplicates by inode (`stat -f '%d:%i'`) — prevents double-count from symlinks/case-insensitive aliases.
- Root dry-run staging: root-owned temp → publish via invoking user (`sudo -u $SUDO_USER`) to avoid symlink TOCTOU (#1210).
- SQLite live DB gate: `#1390` — refuses to delete open `*.db` if `sqlite_database_in_use`.

Tip from README: `mo analyze` moves to Trash via Finder (safer for ad-hoc); `mo clean` deletes directly (use `--dry-run` first).

---

## 5. Other Commands (Full `--help` Dump)

### `mo uninstall` — Smart Uninstaller

```bash
Usage: mo uninstall [OPTIONS] [APP_NAME ...]
  mo uninstall                   Open interactive app selector
  mo uninstall slack             Uninstall Slack
  mo uninstall slack zoom        Uninstall multiple
  mo uninstall --dry-run slack   Preview
  mo uninstall --list            Show exact names mo accepts
Options: --list, --dry-run, --permanent (bypass Trash → rm -rf), --debug, -h
# By default moves to Trash; use --permanent to skip Trash.
# For already-removed apps, use `mo clean` instead.
```

Removes app + 52+ related files across 12 locations (Application Support, Caches, Preferences, Logs, WebKit, Cookies, Extensions, Plugins, LaunchDaemons) and removes Dock entries via `PlistBuddy` then `killall Dock`.

### `mo optimize` — Refresh System

```bash
Usage: mo optimize [OPTIONS]
  --dry-run   Preview
  --whitelist Manage protected items
  --debug     Detailed logs
```

Catalog from `lib/optimize/catalog.sh` + tasks in `lib/optimize/tasks.sh`: inspects/repairs maintenance items, refreshes Finder/network/database state, skips unnecessary/unsafe/unavailable. Counts are illustrative (e.g., Applied 8, 9 unchanged, 4 skipped, 2 unavailable) — depends on Mac state. Whitelist at `~/.config/mole/whitelist_optimize`.

### `mo analyze` — Disk Insights

```bash
# Go binary: libexec/bin/analyze-go
Usage: analyze-go -json
# Shell wrapper: mo analyze [PATH]
#   mo analyze                 # interactive TUI (≈75GB Home example)
#   mo analyze /Volumes        # external drives (skipped by default)
#   mo analyze /private/tmp    # user tmp (moves to Trash after confirm)
#   mo analyze --json ~/Documents  → JSON { path, entries[], large_files[], total_size, total_files }
```

TUI keys: `↑↓→ Enter | R Refresh | O Open | P Preview | F File | Esc/Q Quit`. Moves selected to Trash via Finder (safer than `clean`).

### `mo status` — Live Health

```bash
# Go binary: libexec/bin/status-go
Usage: status-go [-interval 1s] [-json] [-proc-cpu-alerts=true] [-proc-cpu-threshold 100] [-proc-cpu-window 5m0s] [-watch]
# Wrapper: mo status [--json]  # auto-detects pipe → JSON
```

Dashboard: Health score 0-100 (CPU/mem/disk/temp/I-O, color-coded), hardware (model M4, RAM, disk, OS, 60Hz), per-core CPU (toggle `c` cycles 2/4/8/all), mem/disk/IO/network/processes top consumers. Shortcuts: `k` toggle cat, `c` cycle cores, `q` quit (prefs saved). Alert banner for sustained high-CPU processes (`--proc-cpu-threshold`, `--proc-cpu-window`, `--proc-cpu-alerts=false`).

Piped JSON example (verified locally):

```json
{
  "health_score": 100,
  "health_score_msg": "Excellent",
  "host": "MacBook-Air-M4-ThienLam.local",
  "hardware": { "model": "MacBook Air", "cpu_model": "Apple M4", "total_ram": "24.0 GB", "disk_size": "460.4 GB", "os_version": "macOS 27.0" },
  "cpu": { "usage": 6.74, "per_core": [19.77, 18.18, ...], "load1": 2.98 },
  "memory": { "used": 14954037248, "total": 25769803776, "used_percent": 58.0, "swap_used": 0 },
  "disks": [{ "mount": "/", "used_percent": 45.5 }, { "mount": "/Volumes/ADATA_SC750", "external": true }]
}
```

### `mo history` — Audit

```bash
Usage: mo history [OPTIONS]
  --json     JSON output
  --limit N  1..200 recent entries
```

Backed by `~/Library/Logs/mole/{operations.log,deletions.log,mole.log}`. Respects `MO_NO_OPLOG=1` to disable. Example: `mo history --json | jq '.[].reclaimed_bytes'`.

### `mo purge` — Project Artifacts

```bash
Mole Purge — Clean old project build artifacts
Usage: mo purge [options]
  --paths         Edit custom scan directories
  --dry-run       Preview without deleting
  --include-empty Show zero-size dirs
  --debug         Debug logs
Default Paths: ~/www, ~/dev, ~/Projects, ~/GitHub, ~/Code, ~/Workspace, ~/Repos, ~/Development, ~/Library/CloudStorage, ~/.codex/worktrees, ~/.claude/worktrees
```

Targets: `node_modules`, `target` (Rust), `.build`, `build`, `dist`, `venv`, etc. >7-day-old selected by default (recent unselected). Recommend `brew install fd` for speed. Safety: permanent delete, review before confirm. Custom paths: `mo purge --paths` edits `~/.config/mole/purge_paths`.

### `mo installer` — Installer Files

```bash
Usage: mo installer [OPTIONS]
  --dry-run Preview without deleting
  --debug   Detailed logs
# Finds .dmg .pkg .iso .xip .zip in Downloads/Desktop/Homebrew/iCloud/Mail, labeled by source
```

### `mo touchid` — sudo Touch ID

```bash
Usage: mo touchid [COMMAND]
  enable / disable / status
  --dry-run Preview without modifying /etc/pam.d/sudo*
# Interactive menu if no command
```

### `mo completion` — Shell Completion

```bash
Usage: mole completion [bash|zsh|fish]
  mole completion              # auto-detect + install
  mole completion --dry-run    # preview edits
  mole completion bash         # generate script
# Manual: eval "$(mole completion bash)" etc.
# Installs to etc/bash_completion.d, share/zsh/site-functions, share/fish/vendor_completions.d
```

### `mo update` / `mo remove`

```bash
mo update [--force] [--nightly]   # --force reinstall stable, --nightly main branch (script install only)
# currently 1.50.0 → 1.53.0 available (brew)
mo remove [--dry-run]             # preview removal
```

---

## 6. Configuration & State Files

| Path | Purpose |
|------|---------|
| `~/.config/mole/whitelist` | clean protections (see §4.4) |
| `~/.config/mole/whitelist_optimize` | optimize protections (legacy `whitelist_checks` auto-migrated) |
| `~/.config/mole/purge_paths` | custom purge scan dirs (see `mo purge --paths`) |
| `~/.config/mole/clean-list.txt` | last dry-run preview |
| `~/.cache/mole/version_check`, `update_message` | update cache (cleared on `mo update`) |
| `~/Library/Logs/mole/operations.log` | operation log (oplog, `MO_NO_OPLOG=1` disables) |
| `~/Library/Logs/mole/deletions.log`, `mole.log` | deletions + general log |
| `/opt/homebrew/Cellar/mole/1.50.0/` | install prefix (63 files) |

Current `~/.config/mole/purge_paths` on this Mac (from `mo purge --help` default resolution):
```
# (managed via `mo purge --paths`)
# custom file exists: ~/.config/mole/purge_paths → 219 bytes
```

---

## 7. Quick Cheat Sheet

```bash
mo                         # TUI menu
mo clean --dry-run --debug # SAFEST first run
mo clean --whitelist        # protect Playwright/HF/Maven/Ollama/Surge before real clean
mo clean                   # real clean (confirm)
mo uninstall --list && mo uninstall --dry-run Slack
mo analyze                 # explore; moves to Trash via Finder
mo analyze /Volumes/ADATA_SC750
mo status                  # live dashboard (k/c/q), or --json | jq
mo purge --dry-run         # review node_modules/target >7d
mo installer --dry-run
mo history --limit 20 && mo history --json
mo completion && exec $SHELL  # install tab completion
mo update --force          # upgrade 1.50.0 → 1.53.0
```

Tips from README:
- Always `--dry-run` first for destructive `clean/uninstall/purge/installer/remove`.
- Add `--debug` when troubleshooting.
- Navigation supports Vim `h/j/k/l`.
- iTerm2 has issues; recommend Kaku/Alacritty/kitty/WezTerm/Ghostty/Warp (`MO_LAUNCHER_APP` override).
- Quick launchers: `curl -fsSL https://raw.githubusercontent.com/tw93/Mole/main/scripts/setup-quick-launchers.sh | bash` → Raycast/Alfred `Mole Clean` etc.

---

## 8. Troubleshooting

- **Process defer:** If `mo clean` says `Deferred cleanup while active: Xcode/Simulator/Final Cut/Autodesk`, quit that app and re-run — guard is intentional to avoid deleting live caches/SQLite DBs.
- **Whitelist not sticking:** Ensure `~/.config/mole/whitelist` has absolute paths, no `..`, no control chars, no `//`. Check `MOLE_USER_HOME` vs `$HOME` when using `sudo`.
- **Update stale:** `mo update` uses Homebrew; if `1.53.0 available` persists, run `brew update && brew upgrade mole` manually (timeouts `MOLE_HOMEBREW_*_TIMEOUT=120`).
- **Logs empty:** Check `MO_NO_OPLOG` not set; logs written via `operations.log`.
- **Analyze external fast skip:** By design skips `/Volumes`; run `mo analyze /Volumes` explicitly.

---

## 9. Terminal Verification Log (Copy-Paste)

```text
$ which mo
/opt/homebrew/bin/mo -> ../Cellar/mole/1.50.0/bin/mo

$ mo --version
Mole version 1.50.0
macOS: 27.0 / Architecture: arm64 / Kernel: 27.0.0 / SIP: Enabled
Disk Free: 269.21GB / Install: Homebrew / Shell: /bin/zsh

$ mo --help  # 13 commands + 7 preview variants (see §3)

$ for c in clean uninstall optimize analyze status history purge installer touchid completion; do mo $c --help; done
# outputs captured in §4-§5

$ brew info mole
mole: 1.50.0 → 1.53.0 (bottled), HEAD
https://mole.fit  GPL-3.0-or-later  63 files, 9.7MB

$ mo status --json | python3 -m json.tool
{ "health_score": 100, "health_score_msg": "Excellent", ... }

$ cat /opt/homebrew/Cellar/mole/1.50.0/README.md  # Features, Quick Start, Security
```

All explored **via terminal CLI** as requested; builds on real local state.

---

*Rendered by Astro `src/pages/[slug].astro:2` globbing `../../*.md`. Edit with `mo` or `mole` — e.g., `mo clean --dry-run`.*
