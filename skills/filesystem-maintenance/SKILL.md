---
name: filesystem-maintenance
description: Keep disk usage under control on a Hermes VM. Use when disk is filling up, the agent has accumulated stale artifacts, or as a periodic housekeeping check. Covers safe cleanup targets, commands, and things to never delete.
version: 1.0.0
metadata:
  hermes:
    tags: [disk, cleanup, maintenance, ops]
    category: ops
---

# Filesystem Maintenance (Hermes)

Agents generate a surprising amount of disk waste — build artifacts, downloaded models, orphaned containers, pip/uv caches, and session logs add up fast. This skill tells you what to clean, what to leave alone, and how to stay on top of it.

## Quick health check

```sh
df -h /                          # overall disk usage
du -sh ~/.hermes/*/ | sort -h    # Hermes directory breakdown
du -sh /tmp/* 2>/dev/null | sort -h   # temp file accumulation
```

If `/` is above 80%, start cleaning. Above 90% is urgent.

## Safe cleanup targets (biggest wins first)

### 1. Temp files and working copies

Agents clone repos, download files, and extract archives into `/tmp` and working directories, then forget about them.

```sh
# Review what's in /tmp before removing
du -sh /tmp/* 2>/dev/null | sort -h
# Remove stale working copies (interactive — review first)
find /tmp -maxdepth 1 -type d -mtime +1 -name "*.tmp" -o -name "hermes-*" | xargs -r du -sh
# Clean up after confirming
rm -rf /tmp/hermes-skills-* /tmp/repo-* /tmp/*.tmp
```

**Rule:** Always review before bulk-deleting from `/tmp`. Other processes may have active files there.

### 2. Package manager caches

```sh
# Python (uv / pip)
du -sh ~/.cache/uv 2>/dev/null
uv cache clean                   # safe — packages re-download on demand

# pip (if used)
du -sh ~/.cache/pip 2>/dev/null
pip cache purge 2>/dev/null

# npm / node
du -sh ~/.npm/_cacache 2>/dev/null
npm cache clean --force 2>/dev/null

# apt
sudo apt clean 2>/dev/null       # removes cached .deb packages
```

### 3. Container and image sprawl

```sh
# Docker (if present)
docker system df 2>/dev/null     # see what Docker is using
docker system prune -f           # remove stopped containers, dangling images, unused networks
docker builder prune -f          # remove build cache
# For aggressive cleanup (removes ALL unused images, not just dangling):
# docker system prune -a -f     # ← ask before running this
```

### 4. Old log files

```sh
# Journal logs (systemd)
journalctl --disk-usage
sudo journalctl --vacuum-size=200M   # keep only 200 MB of logs

# Hermes session logs — old sessions accumulate
du -sh ~/.hermes/sessions/
# Sessions older than 30 days are usually safe to remove
find ~/.hermes/sessions/ -name "*.jsonl" -mtime +30 | wc -l
find ~/.hermes/sessions/ -name "*.jsonl" -mtime +30 -delete
```

### 5. Python virtual environments

Agents create venvs for one-off tasks and never clean them up.

```sh
# Find venvs outside of active projects
find ~/ -maxdepth 4 -name "pyvenv.cfg" -exec dirname {} \; 2>/dev/null | while read d; do
  echo "$(du -sh "$d") — last used $(stat -c %y "$d/pyvenv.cfg" | cut -d' ' -f1)"
done
# Delete the ones you don't recognise or haven't touched in weeks
```

### 6. Git and build artifacts

```sh
# Dangling git objects in workspace repos
cd ~/workspace && git gc --aggressive --prune=now
# Stale worktrees
git worktree list
git worktree prune

# Build outputs in project dirs
find ~/ -maxdepth 4 -type d \( -name "node_modules" -o -name "__pycache__" -o -name ".mypy_cache" -o -name "dist" -o -name "build" \) 2>/dev/null | xargs -r du -sh | sort -h
```

### 7. Git LFS objects

LFS keeps old versions of large files locally even after they've been pushed. Across multiple repos this adds up quickly.

```sh
# Find all git repos under home and prune LFS objects in each
find ~/ -maxdepth 4 -name ".git" -type d 2>/dev/null | while read gitdir; do
  repo="$(dirname "$gitdir")"
  # Skip bare repos
  [ -f "$repo/.git" ] || [ -d "$repo/.git" ] || continue
  size_before=$(du -sh "$repo/.git/lfs" 2>/dev/null | cut -f1)
  [ -z "$size_before" ] && continue
  echo "=== $repo (LFS: $size_before) ==="
  git -C "$repo" lfs prune
done
```

`git lfs prune` deletes local LFS objects that have already been pushed and are not referenced by the current checkout. It's always safe — the objects remain on the remote and will re-download on demand.

### 8. Audio / media cache

```sh
du -sh ~/.hermes/audio_cache 2>/dev/null
# TTS audio files older than 7 days
find ~/.hermes/audio_cache -type f -mtime +7 -delete 2>/dev/null
```

## Things to NEVER delete

| Path | Why |
|------|-----|
| `~/.hermes/.env` | Runtime secrets — gateway won't start |
| `~/.hermes/config.yaml` | Agent configuration |
| `~/.hermes/state.db` | Session state and memory |
| `~/.hermes/memory_store.db` | Persistent memory |
| `~/.hermes/skills/` | Your skill library |
| `~/.hermes/workspace/` | Your workspace (this repo, case files, etc.) |
| `~/.hermes/cron/` | Scheduled job definitions |
| `~/.hermes/hooks/` | Event hooks |
| `~/.password-store/` | GPG-encrypted backup vault |
| `~/workspace/.git/` | Git history — `git gc` is fine, deleting `.git` is not |

## Automated maintenance (optional)

For VMs that consistently fill up, add a lightweight cron job:

```sh
# Example: weekly cleanup of safe targets
hermes cron create --schedule "0 4 * * 0" --name "disk-cleanup" \
  --prompt "Run filesystem-maintenance skill: clean /tmp files older than 3 days, purge uv/pip cache, vacuum journals to 200M, delete session logs older than 30 days, delete audio cache older than 7 days. Report what was cleaned and current disk usage."
```

## Pitfalls

- **Don't delete `/tmp` wholesale** — running processes (including Hermes gateway) may have active files there. Target specific stale patterns.
- **Don't `rm -rf ~/.hermes/sessions/`** — delete old files only. The current session is in there.
- **Don't shrink a mounted disk** without understanding the filesystem and cloud provider. Growing a disk is safe; shrinking usually isn't.
- **Docker `prune -a`** removes images you might need to re-pull (slow on metered connections). Use plain `prune` (no `-a`) unless disk is critical.
- **`node_modules`** in active projects will need `npm install` again after deletion. Only remove from abandoned project dirs.
