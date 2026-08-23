# MacBook Air M4 — Performance Optimization Guide

> **Device:** MacBook Air M4 · 24 GB RAM · macOS 26.6.2
> **Date:** Aug 23, 2026
> **Status:** ✅ System healthy — 88% free memory, zero swap, disk only 6% used

---

## 1. Diagnostics Summary (Before Optimization)

| Area | Result | Verdict |
|------|--------|---------|
| Memory | 88% free, swap = 0 MB | Excellent |
| Disk | 12 Gi used / 460 Gi (216 Gi free) | Excellent |
| CPU load | ~1.9 avg on 10 cores | Very light |
| Thermal | No warnings recorded | Healthy |
| Spotlight indexing | Already disabled | Saves resources |

**Conclusion:** No performance bottleneck existed. Optimizations below trim background noise.

---

## 2. Applied Changes (Aug 23, 2026)

Disabled 4 unnecessary login-time Launch Agents:

| Agent | Purpose |
|-------|---------|
| `com.google.keystone.agent` | Chrome background updater |
| `com.google.GoogleUpdater.wake` | New Google updater wake job |
| `io.sideloadly.daemon` | Sideloadly iOS sideloading daemon |
| `com.valvesoftware.steamclean` | Steam leftover cleaner |

Commands used:

```bash
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/<name>.plist
launchctl disable gui/$(id -u)/<label>
```

Verify disabled state:

```bash
launchctl print-disabled gui/$(id -u) | grep -iE "google|sideload|steam"
```

---

## 3. Pending Step — Requires Your Password

Faster sleep/wake on battery (safe-hibernate off):

```bash
sudo pmset -b hibernatemode 0
```

- **Benefit:** Faster sleep & wake, no large `sleepimage` written to disk on battery.
- **Tradeoff:** If battery drains to 0% while "asleep", unsaved work is lost.
- Revert with: `sudo pmset -b hibernatemode 3`

---

## 4. Rollback Instructions

Re-enable any agent:

```bash
launchctl enable gui/$(id -u)/com.google.keystone.agent
launchctl kickstart gui/$(id -u)/com.google.keystone.agent   # optional: run now
```

Notes:
- Plist files were **not deleted**, only disabled.
- Chrome still updates itself when opened — Keystone just ran periodic checks in the background.

---

## 5. Ongoing Maintenance Tips

1. **Check what's slowing you down:**
   ```bash
   ps aux -r | head -10        # top CPU consumers
   ps aux -m | head -10        # top memory consumers
   ```

2. **Memory pressure check:**
   ```bash
   memory_pressure -Q | tail -1
   ```
   > Keep free memory above ~20%. Swap usage > 0 means RAM pressure.

3. **Disk health:** Keep at least 10–15% free for optimal SSD performance.

4. **Login items review:** System Settings → General → Login Items — remove apps you don't need at startup.

5. **Restart weekly** — clears leaked memory from long-running apps.

---

## 6. Current Login Items (Reviewed — All Fine)

| Item | Verdict |
|------|---------|
| Cloudflare WARP | ✅ Keep (VPN, lightweight) |
| GoTiengViet | ✅ Keep (Vietnamese input method) |
| AltTab | ✅ Keep (window switcher) |
