---
name: filesystem-maintenance
description: Keep disk usage under control on a Hermes VM. Use when disk is filling up, the agent has accumulated stale artifacts, or as a periodic housekeeping check. Covers safe cleanup targets, commands, and things to never delete.
version: 1.2.0
metadata:
  hermes:
    tags: [disk, cleanup, maintenance, ops]
    category: ops
---

# Filesystem Maintenance (Hermes)

Agents generate a surprising amount of disk waste — build artifacts, downloaded models, browser caches, orphaned containers, package caches, and session logs add up fast. This skill tells you what to clean, what to leave alone, and how to stay on top of it.

## Quick health check

Start broad — the biggest offenders are often outside `~/.hermes`:

```sh
HERMES_HOME="${HERMES_HOME:-$HOME/.hermes}"

df -h /                                                    # overall disk usage
du -xh -d 1 "$HOME" 2>/dev/null | sort -h | tail -20      # top-level breakdown
du -xh -d 1 ~/.cache "$HERMES_HOME" 2>/dev/null | sort -h  # cache + hermes detail
du -sh /tmp/* 2>/dev/null | sort -h                        # temp file accumulation
```

If `/` is above 80%, start cleaning. Above 90% is urgent.

## Safe cleanup targets (biggest wins first)

### 1. Browser automation caches

Often the single largest waste item — Playwright, Camoufox, Puppeteer, and Selenium each download full browser binaries.

```sh
du -sh ~/.cache/ms-playwright ~/.cache/camoufox ~/.cache/puppeteer ~/.cache/selenium 2>/dev/null
# Safe to remove — browsers re-download on next run (may be slow on metered connections)
# rm -rf ~/.cache/ms-playwright ~/.cache/camoufox ~/.cache/puppeteer ~/.cache/selenium
```

### 2. Package manager caches

```sh
# Python (uv / pip)
command -v uv >/dev/null && { du -sh ~/.cache/uv 2>/dev/null; uv cache clean; }
command -v pip >/dev/null && { du -sh ~/.cache/pip 2>/dev/null; pip cache purge 2>/dev/null; }

# npm — cache clean only clears _cacache, not _npx
du -sh ~/.npm/_cacache ~/.npm/_npx 2>/dev/null
command -v npm >/dev/null && npm cache clean --force 2>/dev/null
# _npx accumulates full package installs that npm cache clean doesn't touch
du -sh ~/.npm/_npx 2>/dev/null
# rm -rf ~/.npm/_npx/*/    # uncomment after reviewing

# node-gyp (cached Node.js headers — safe to remove, re-downloads on demand)
du -sh ~/.cache/node-gyp 2>/dev/null
# rm -rf ~/.cache/node-gyp  # uncomment after reviewing

# Yarn (can be much larger than npm)
if command -v yarn >/dev/null; then
  yarn_dir="$(yarn cache dir 2>/dev/null)"
  [ -n "$yarn_dir" ] && du -sh "$yarn_dir" 2>/dev/null
  yarn cache clean 2>/dev/null
fi

# apt (requires sudo — skip if unavailable)
command -v sudo >/dev/null && sudo apt clean 2>/dev/null
```

### 3. Temp files and working copies

Agents clone repos, download files, and extract archives into `/tmp`, then forget about them.

```sh
# Review what's in /tmp before removing — never delete blindly
du -sh /tmp/* 2>/dev/null | sort -h

# Find stale working dirs (older than 3 days) — dry run first
find /tmp -mindepth 1 -maxdepth 1 -type d -mtime +3 \
  \( -name "hermes-*" -o -name "repo-*" -o -name "*.tmp" \) -print

# Delete only after reviewing the list above
# find ... -delete  (or pipe to xargs rm -rf)
```

**Rule:** Never `rm -rf /tmp/*`. Other processes (including the Hermes gateway) may have active files there. Target specific stale patterns by age.

### 4. Container and image sprawl

```sh
if command -v docker >/dev/null; then
  docker system df                   # see what Docker is using
  docker system prune -f             # remove stopped containers, dangling images, unused networks
  docker builder prune -f            # remove build cache
  # For aggressive cleanup (removes ALL unused images, not just dangling):
  # docker system prune -a -f       # ← ask before running this
fi
```

### 5. Old log files

```sh
# Journal logs (systemd)
if command -v journalctl >/dev/null; then
  journalctl --disk-usage
  # Vacuum requires sudo on some VMs — skip if unavailable
  sudo journalctl --vacuum-size=200M 2>/dev/null || journalctl --vacuum-size=200M 2>/dev/null
fi

# Hermes session logs — old sessions accumulate
HERMES_HOME="${HERMES_HOME:-$HOME/.hermes}"
du -sh "$HERMES_HOME/sessions/" 2>/dev/null
# CAUTION: session JSONL files power session_search (cross-session recall).
# Deleting them means losing transcript history. Only remove if you accept
# that trade-off, or archive them first.
find "$HERMES_HOME/sessions/" -name "*.jsonl" -mtime +30 | wc -l   # count first
# find "$HERMES_HOME/sessions/" -name "*.jsonl" -mtime +30 -delete  # uncomment after reviewing

# Hermes logs directory
du -sh "$HERMES_HOME/logs/" 2>/dev/null
find "$HERMES_HOME/logs/" -name "*.log" -mtime +14 2>/dev/null | wc -l
# find "$HERMES_HOME/logs/" -name "*.log" -mtime +14 -delete  # uncomment after reviewing

# Hermes media cache (downloaded images, PDFs, attachments)
du -sh "$HERMES_HOME/media/" 2>/dev/null
find "$HERMES_HOME/media/" -type f -mtime +30 2>/dev/null | wc -l
# find "$HERMES_HOME/media/" -type f -mtime +30 -delete  # uncomment after reviewing

# Audio / TTS cache
du -sh "$HERMES_HOME/audio_cache" 2>/dev/null
find "$HERMES_HOME/audio_cache" -type f -mtime +7 2>/dev/null | wc -l
# find "$HERMES_HOME/audio_cache" -type f -mtime +7 -delete  # uncomment after reviewing
```

### 6. Stale database backups

Migrations and transplants leave behind large `.bak` files under the Hermes directory.

```sh
HERMES_HOME="${HERMES_HOME:-$HOME/.hermes}"
find "$HERMES_HOME" -maxdepth 1 \( -name "*.bak" -o -name "*backup*" -o -name "*.pretransplant*" \) | xargs -r ls -lh
# Review the list, then remove confirmed stale backups
```

### 7. Python virtual environments

Agents create venvs for one-off tasks and never clean them up.

```sh
find ~/ -maxdepth 4 -name "pyvenv.cfg" -exec dirname {} \; 2>/dev/null | while read d; do
  echo "$(du -sh "$d") — last used $(stat -c %y "$d/pyvenv.cfg" | cut -d' ' -f1)"
done
# Delete the ones you don't recognise or haven't touched in weeks
```

### 8. Git LFS objects

LFS keeps old versions of large files locally even after they've been pushed. Across multiple repos this adds up quickly.

```sh
find ~/ -maxdepth 4 -name ".git" -type d 2>/dev/null | while read gitdir; do
  repo="$(dirname "$gitdir")"
  [ -d "$repo/.git/lfs" ] || continue
  size_before=$(du -sh "$repo/.git/lfs" 2>/dev/null | cut -f1)
  echo "=== $repo (LFS: $size_before) ==="
  git -C "$repo" lfs prune --dry-run
done
# Review the dry-run output, then re-run without --dry-run to actually prune
```

**Note:** `git lfs prune` deletes local LFS objects that have been pushed and are not referenced by the current checkout. Usually safe, but always dry-run first — especially if you have unpushed branches or work offline.

### 9. Git and build artifacts

```sh
# Run git gc across all repos (skip --aggressive for routine maintenance — it's slow)
find "$HOME" -maxdepth 4 -name .git -type d 2>/dev/null | while read gitdir; do
  repo="$(dirname "$gitdir")"
  git -C "$repo" gc --prune=2.weeks.ago 2>/dev/null
done

# Prune stale worktrees
find "$HOME" -maxdepth 4 -name .git -type d 2>/dev/null | while read gitdir; do
  repo="$(dirname "$gitdir")"
  git -C "$repo" worktree prune 2>/dev/null
done

# Stale node_modules (including inside worktrees)
find ~/ -maxdepth 6 -type d -name "node_modules" 2>/dev/null | xargs -r du -sh | sort -h

# Build outputs and caches
find ~/ -maxdepth 4 -type d \( -name "__pycache__" -o -name ".mypy_cache" -o -name "dist" -o -name "build" \) 2>/dev/null | xargs -r du -sh | sort -h
```

### 10. Stale repo clones

Agents clone repos for one-off lookups and forget about them.

```sh
# Find repos with no recent activity (last commit or access > 30 days ago)
find ~/ -maxdepth 3 -name ".git" -type d 2>/dev/null | while read gitdir; do
  repo="$(dirname "$gitdir")"
  last_mod=$(stat -c %Y "$gitdir/FETCH_HEAD" 2>/dev/null || stat -c %Y "$gitdir/HEAD")
  age=$(( ($(date +%s) - last_mod) / 86400 ))
  [ "$age" -gt 30 ] && echo "${age}d ago — $(du -sh "$repo" | cut -f1) — $repo"
done | sort -rn
# Candidates can be re-cloned on demand — delete after reviewing
```

### 11. Other toolchain caches

Agents may accumulate caches from less common package managers. Inspect before cleaning — some are expensive to rebuild.

```sh
# Rust
du -sh ~/.cargo/registry ~/.rustup/toolchains 2>/dev/null

# pnpm
du -sh ~/.pnpm-store ~/.local/share/pnpm/store 2>/dev/null
command -v pnpm >/dev/null && pnpm store prune 2>/dev/null

# Poetry
du -sh ~/.cache/pypoetry 2>/dev/null

# Conda / Mamba
command -v conda >/dev/null && conda clean --all --dry-run 2>/dev/null

# Go
du -sh ~/go/pkg/mod 2>/dev/null
command -v go >/dev/null && go clean -modcache 2>/dev/null
```

### 12. ML / model caches

Large model downloads that are expensive to re-fetch — be cautious here.

```sh
du -sh ~/.cache/huggingface ~/.cache/torch ~/.cache/ollama ~/.ollama 2>/dev/null
# Only remove models you're certain you won't need again soon
# These can be many GB each and slow to re-download
```

## Things to NEVER delete

| Path | Why |
|------|-----|
| `$HERMES_HOME/.env` | Runtime secrets — gateway won't start |
| `$HERMES_HOME/config.yaml` | Agent configuration |
| `$HERMES_HOME/auth.json` | Platform authentication credentials |
| `$HERMES_HOME/state.db` (`-shm`, `-wal`) | Session state — include SQLite sidecars |
| `$HERMES_HOME/memory_store.db` | Persistent memory (DB form) |
| `$HERMES_HOME/memories/` | Persistent memory (file form — some installs use this instead of the DB) |
| `$HERMES_HOME/sessions/` | Session history — `session_search` depends on these |
| `$HERMES_HOME/SOUL.md` | Agent identity / persona — equivalent to config |
| `$HERMES_HOME/skills/` | Your skill library |
| `$HERMES_HOME/workspace/` | Your workspace (repos, case files, etc.) |
| `$HERMES_HOME/cron/` | Scheduled job definitions |
| `$HERMES_HOME/hooks/` | Event hooks |
| `$HERMES_HOME/scripts/` | Custom scripts (cron, watchdogs, etc.) — not cache |
| `$HERMES_HOME/bin/` | Custom executables (CLIs, tools) — not easily recoverable |
| `$HERMES_HOME/hermes-agent/` | Agent runtime (venv, node_modules, TUI) — often 1-2G but required |
| `$HERMES_HOME/gateway_state.json` | Gateway runtime state — deleting mid-run corrupts connections |
| `$HERMES_HOME/channel_directory.json` | Connected platform/channel registry |
| `$HERMES_HOME/kanban.db` | Local task/kanban state |
| `$HERMES_HOME/mcp-tokens/` | MCP server auth tokens — losing these breaks tool connections |
| `$HERMES_HOME/pairing/` | Pairing/integration state |
| `$HERMES_HOME/plugins/` | Installed extensions/plugins |
| `~/.password-store/` | GPG-encrypted backup vault |
| `~/.ssh/` | SSH keys and config — not recoverable without backup |
| `~/.gnupg/` | GPG keys — required to decrypt `~/.password-store/` |
| `~/.claude/` | Claude CLI credentials |
| `~/.git-credentials`, `~/.netrc` | Git/HTTP credentials |
| `~/.config/gh/` | GitHub CLI auth |
| `~/.local/bin/`, `~/.local/opt/` | User-local tools (especially on no-sudo VMs) |
| `~/workspace/` | Your workspace repo — `git gc` is fine, deleting the dir is not |
| `~/workspace/.argit/` | Backup manifest overlays — needed by argit |
| `~/workspace/secrets/` | Encrypted service/channel secrets |

**Warning:** `$HERMES_HOME/.cache/` is NOT the same as `~/.cache/`. The Hermes-internal cache may hold live OAuth tokens and auth material, not just throwaway data. Never clean it without inspecting contents first.

**Rule:** Never delete `$HERMES_HOME` wholesale. Only clean explicitly named subdirectories (logs, media, audio_cache). Everything else under `$HERMES_HOME` is either state, config, or credentials.

**Note:** `$HERMES_HOME` defaults to `~/.hermes`. If you use a non-default install or profiles, adjust paths accordingly. For OpenClaw environments, the equivalent paths under `~/.openclaw/agents/` are also protected.

## Automated maintenance (optional)

For VMs that consistently fill up, consider a disk-usage watchdog that only runs cleanup when needed:

```
# Via the cronjob tool — schedule a weekly check
schedule: "0 4 * * 0"
name: "disk-cleanup"
prompt: >
  Check disk usage with df -h /. If usage is above 80%, run the
  filesystem-maintenance skill: clean browser caches, purge package
  manager caches, vacuum journals to 200M, delete audio/media cache
  older than 7 days, prune LFS objects (dry-run first). For session
  logs older than 30 days, count and report only — do not delete
  without explicit approval. Report what was cleaned and current usage.
  If usage is below 80%, just report the current level.
```

## Pitfalls

- **Don't delete `/tmp` wholesale** — running processes (including Hermes gateway) may have active files there. Target specific stale patterns.
- **Don't blindly delete old sessions** — `$HERMES_HOME/sessions/*.jsonl` files are used by `session_search` for cross-session recall. Deleting them loses transcript history permanently. Count and review before removing; consider archiving instead.
- **Don't shrink a mounted disk** without understanding the filesystem and cloud provider. Growing a disk is safe; shrinking usually isn't.
- **Docker `prune -a`** removes images you might need to re-pull (slow on metered connections). Use plain `prune` (no `-a`) unless disk is critical.
- **`node_modules`** in active projects will need `npm install` again after deletion. Only remove from abandoned project dirs.
- **ML model caches** can be tens of GB and very slow to re-download. Only remove models you're sure you won't need.
- **Browser caches** (Playwright, Camoufox) re-download on next use — safe to delete but may delay the next browser task.
- **Always dry-run first** for destructive operations like LFS prune and bulk deletes.
