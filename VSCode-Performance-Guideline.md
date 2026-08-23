# VS Code Performance Guideline

> Audit date: 2026-08-23 · VS Code 1.134.0 · macOS (Apple Silicon)

## 1. Summary

Baseline state before optimization:

| Metric | Value |
|---|---|
| Code processes | 14 |
| Total RAM (all Code processes) | ~2.8 GB |
| Main window process | 815 MB |
| Extension host helper | 600 MB |
| Installed extensions | 29 |
| `~/.vscode/extensions` size | 1.1 GB |
| `CachedExtensionVSIXs` (stale installers) | 382 MB |
| `workspaceStorage` | 250 MB |

The base `settings.json` was already well tuned (minimap off, smooth
scrolling off, watcher excludes, editor limit, telemetry off). The main
costs were **extension-related**, not settings-related.

## 2. Issues found during audit

### 2.1 Six AI assistants sharing one extension host

Claude Code, Codium, Google Antigravity, Ollama, ChatGPT and OpenCode
were all installed at once. Every extension by default runs inside the
**single** extension-host process — one busy extension can freeze
completions, linting and the whole workbench.

### 2.2 Invalid tsserver watch options

```jsonc
"js/ts.tsserver.watchOptions": "vscode"   // ❌ not a valid value
```

The setting expects an object; the string was silently ignored.

### 2.3 Redundant extensions duplicating built-ins

| Extension | Built-in replacement |
|---|---|
| Auto Rename Tag | `editor.linkedEditing` |
| Trailing Spaces | `files.trimTrailingWhitespace` |
| Todo Highlight | Todo Tree (kept) |

### 2.4 Stale VSIX cache

382 MB of old extension installer archives in
`~/Library/Application Support/Code/CachedExtensionVSIXs/`.

## 3. Fixes applied

### 3.1 Isolate heavy extensions with affinity

Each number is a separate extension-host process, so AI tools can no
longer starve the main one:

```jsonc
"extensions.experimental.affinity": {
  "anthropic.claude-code":     1,
  "codium.codium":             1,
  "google.google-antigravity": 1,
  "ollama.ollama":             1,
  "openai.chatgpt":            1,
  "sst-dev.opencode":          1,
  "eamodio.gitlens":           2,
  "bradlc.vscode-tailwindcss": 2,
  "yoavbls.pretty-ts-errors":  2
}
```

Requires a full restart (`Cmd+Q`, reopen). Verify in Help → Open
Process Explorer: each group gets its own `extensionHost` entry.

### 3.2 Fixed tsserver watch options

```jsonc
"typescript.tsserver.watchOptions": {
  "watchFile":       "useFsEvents",
  "watchDirectory":  "useFsEvents",
  "fallbackPolling": "dynamicPriority"
}
```

Uses native FSEvents instead of polling → lower idle CPU in large repos.

### 3.3 Disabled automatic type acquisition

```jsonc
"typescript.disableAutomaticTypeAcquisition": true
```

Stops background downloads of `@types/*` packages for plain-JS files.
TypeScript projects are unaffected — types still come from
`node_modules`.

### 3.4 Expanded file watchers / search excludes

Added `.astro`, `.vercel`, `.output`, `.svelte-kit`, `.nuxt`,
`.parcel-cache`, `.expo`, `*.log` to `files.watcherExclude`; added
`.astro`, `.vercel`, `.output` to `search.exclude`; plus:

```jsonc
"search.followSymlinks": false
```

### 3.5 Replaced redundant extensions with built-ins

```bash
code --uninstall-extension formulahendry.auto-rename-tag
code --uninstall-extension shardulm94.trailing-spaces
code --uninstall-extension wayou.vscode-todo-highlight
```

Enabled in settings:

```jsonc
"editor.linkedEditing": true,
"files.trimTrailingWhitespace": true
```

### 3.6 Cleared stale VSIX cache

```bash
rm -rf ~/Library/Application\ Support/Code/CachedExtensionVSIXs/*
```

Freed **382 MB**. Safe — only old installer archives.

## 4. Maintenance routine

Run every few months:

```bash
# Find memory hogs among Code processes
ps aux | grep "Code Helper" | awk '{print $6/1024 " MB", $11}' | sort -rn

# Check biggest extensions on disk
du -sh ~/.vscode/extensions/* | sort -rh | head -15

# Clean caches (close VS Code first)
rm -rf ~/Library/Application\ Support/Code/CachedExtensionVSIXs/*

# Audit installed extensions
code --list-extensions
```

Rules of thumb:

- One or two AI assistants maximum per workflow; keep the rest disabled
  until needed (`Extensions` panel → Disable).
- Never let `git.autofetch` stay on in very large repos (already off).
- After major updates, check Process Explorer for a runaway helper.

## 5. Expected result

| Metric | Before | After |
|---|---|---|
| Extensions | 29 | 26 |
| Stale cache | 382 MB | 0 MB |
| AI extensions in main host | 6 | 0 (isolated) |
| tsserver watch mode | invalid (polling fallback) | FSEvents |

Restart VS Code fully once so the affinity groups take effect.
