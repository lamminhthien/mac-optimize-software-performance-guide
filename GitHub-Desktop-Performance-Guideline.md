# GitHub Desktop Performance Guideline

> Audit date: 2026-08-23 · macOS (Apple Silicon, Homebrew)

## 1. Summary

GitHub Desktop does **not** store performance-tunable settings on disk.
Its data folder only holds Electron runtime state:

```
~/Library/Application Support/GitHub Desktop/
├── Cache/          (~11 MB — normal)
├── Code Cache/     (tiny)
├── GPUCache/       (~548 KB)
├── logs/
└── window-state.json
```

Real performance is controlled by:

- The **global git config** (`~/.gitconfig`)
- Per-repository git settings
- Whether **Git LFS** is properly installed

## 2. Issue found during audit

`~/.gitconfig` contains LFS filters, but `git-lfs` was **not installed**:

```gitconfig
[filter "lfs"]
    required = true
    clean = git-lfs clean -- %f
    smudge = git-lfs smudge -- %f
    process = git-lfs filter-process
```

This causes errors and slowness in any repo using LFS.

## 3. Fix: install Git LFS

```bash
brew install git-lfs
git lfs install
```

Verify:

```bash
which git-lfs        # should print /opt/homebrew/bin/git-lfs
git lfs version
```

Alternative: if you never use LFS, remove the filter block instead:

```bash
git config --global --remove-section filter.lfs
```

## 4. Optional speedups for large repositories

Apply globally:

```bash
git config --global core.fsmonitor true        # use filesystem watcher for status
git config --global core.untrackedcache true   # cache untracked file scan
git config --global feature.manyFiles true     # better index scaling (big repos)
```

Per noisy repository (many untracked files):

```bash
git config status.showUntrackedFiles normal    # or "no" to hide them entirely
```

Skip expensive checks on huge histories:

```bash
git config --global core.commitGraph true
git config --global gc.writeCommitGraph true
```

## 5. Housekeeping

- Clear GitHub Desktop caches if the app feels sluggish:
  quit the app, then delete `Cache/`, `Code Cache/`, `GPUCache/`
  inside `~/Library/Application Support/GitHub Desktop/`.
- Old logs can be pruned from the same folder's `logs/` directory.

## 6. Checklist

| Item | Command | Status |
| --- | --- | --- |
| Install Git LFS | `brew install git-lfs && git lfs install` | ☐ Pending |
| Enable fsmonitor | `git config --global core.fsmonitor true` | ☐ Pending |
| Enable untracked cache | `git config --global core.untrackedcache true` | ☐ Pending |
| Verify LFS | `git lfs version` | ☐ Pending |
