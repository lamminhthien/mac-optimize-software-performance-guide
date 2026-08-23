# DBeaver Performance Optimization Guide (macOS)

> Machine: Apple M4, 24 GB RAM, 10 cores · DBeaver Community 26.1.3 · Temurin JRE 25
> Date: 2026-08-23

## 1. Diagnosis Summary

| Area | Finding | Impact |
|---|---|---|
| JVM heap | `-Xmx1024m` (default 1 GB) with `-Xms64m` | GC pressure / UI freezes on large result sets; likely cause of the Aug-21 force-quit |
| Navigator statistics | Redshift `svv_table_info` permission-denied errors repeating in logs | Wasted queries + retries every time schemas expand in navigator |
| Result set row count | Auto `COUNT(*)` fired when opening any table | Slow table opens on large production tables |
| Auto-fetch next segment | Enabled by default | Scrolling to bottom keeps pulling data into RAM unbounded |
| Workspace size | 1.3 MB, logs clean | ✅ No cleanup needed |
| Disk / system | 213 GB free, zero swap | ✅ Healthy |

Connection pattern observed in logs: multiple simultaneous production RDS (PostgreSQL) and Redshift connections, each opening Main + Metadata contexts — heap headroom matters.

## 2. Changes Applied

### 2.1 JVM tuning — `/Applications/DBeaver.app/Contents/Eclipse/dbeaver.ini`

```ini
-Xms512m                        # was 64m – avoids repeated heap resizing
-Xmx4096m                       # was 1024m – headroom for multi-connection workloads
-XX:+UseG1GC                    # explicit; low-pause collector for UI responsiveness
-XX:+UseStringDeduplication     # G1 feature; SQL text/identifiers have many duplicate strings
-XX:MaxGCPauseMillis=100        # pause goal: keep UI snappy
-XX:+ExitOnOutOfMemoryError     # fail fast instead of hanging frozen
```

Verified live via `jcmd <pid> VM.flags`: InitialHeapSize=512 MB, MaxHeapSize=4 GB, G1GC + StringDeduplication active.

### 2.2 App preferences — `~/Library/DBeaverData/workspace6/.metadata/.plugins/org.eclipse.core.runtime/.settings/`

`org.jkiss.dbeaver.ui.navigator.prefs`
```
navigator.show.statistics.info=false   # stops failing/expensive stats queries (Redshift svv_table_info)
```

`org.jkiss.dbeaver.ui.editors.data.prefs`
```
resultset.automatic.row.count=false    # no COUNT(*) on table open
resultset.autofetch.next.segment=false # fetch next page only on demand
```

### 2.3 Backups created

- `/Applications/DBeaver.app/Contents/Eclipse/dbeaver.ini.bak.20260823`
- `~/Library/DBeaverData/metadata.backup.20260823/`

## 3. Rollback

```bash
cp "/Applications/DBeaver.app/Contents/Eclipse/dbeaver.ini.bak.20260823" \
   "/Applications/DBeaver.app/Contents/Eclipse/dbeaver.ini"

rm ~/Library/DBeaverData/workspace6/.metadata/.plugins/org.eclipse.core.runtime/.settings/org.jkiss.dbeaver.ui.navigator.prefs \
   ~/Library/DBeaverData/workspace6/.metadata/.plugins/org.eclipse.core.runtime/.settings/org.jkiss.dbeaver.ui.editors.data.prefs
```

Or restore individual toggles in-app:
- **Settings → User Interface → Navigator →** check *Show statistics*
- **Settings → Editors → Data Editor →** re-enable *Automatic row count* / *Auto-fetch next segment*

## 4. Recommended In-App Settings (manual, optional)

These change UX behavior, so they were left to your choice:

1. **Keep metadata on a separate connection** *(Connections → Edit → General)* — already active per logs; keeps long queries from blocking tree browsing.
2. **Disable tooltips for slow hosts** *(Settings → User Interface → Navigator → Show tooltips)* — each hover can trigger metadata queries over VPN.
3. **Set JDBC fetch size** per connection *(driver properties → `defaultRowFetchSize=1000` for PostgreSQL)* — fewer round-trips for wide scans.
4. **Close idle connections**: *Settings → Connections →* "Keep-alive" and auto-close after N minutes — production DBs see fewer idle sessions.
5. **Limit result-set max rows** in SQL editor toolbar (e.g., 500) when exploring big tables.
6. For the Redshift user without `svv_table_info` access: either grant `SELECT` on `pg_catalog.svv_table_info`, or leave statistics off (as configured).

## 5. Maintenance Notes

- **App updates overwrite `dbeaver.ini`** — after updating DBeaver, re-check/re-apply Section 2.1.
- Watch log growth at `~/Library/DBeaverData/workspace6/.metadata/dbeaver-debug*.log`; old rotated logs are safe to delete.
- If DBeaver ever hangs again, capture state before force-quitting:
  ```bash
  jcmd $(pgrep -x dbeaver) GC.heap_info
  ```
