---
name: argit-backup
description: Understand and operate argit backups of your Hermes state. Use when checking that automated backups are healthy, running a manual backup, troubleshooting backup warnings, or handling uncovered paths. Covers the systemd backup timer, manifest overlays, and GPG/pass secrets. Restore/DR is operator-only — not your job.
version: 1.0.0
metadata:
  hermes:
    tags: [backup, disaster-recovery, argit, ops]
    category: ops
---

# argit Backup (Hermes)

argit backs up your Hermes agent state (`~/.hermes`) into your workspace git repo so it survives a VM rebuild (`tofu destroy` → `tofu apply`). It classifies every path under `~/.hermes` via a manifest:

- **Secrets** (`.env`, `auth.json`) → GPG-encrypted into the `pass` store
- **Data** (`config.yaml`, `SOUL.md`, `memories/`, `hooks/`, `cron/`, `scripts/`, `channel_directory.json`) → copied as-is
- **SQLite** (`state.db`, `memory_store.db`) → safe binary snapshots (handles WAL)
- **Blobs** (`skills/`, `workspace/`) → git / git-lfs

Runtime ephemera (`sessions/`, `*_cache/`, `gateway.pid`, the `hermes-agent/` code checkout) are excluded.

## Backups are already automated — you don't set up cron

A systemd `--user` timer runs `argit backup --push` hourly, plus a best-effort flush on shutdown. **Do not create a `hermes cron` job for this** — it would double up. Your job is to keep backups *healthy*, not to schedule them. See `references/automated-backup.md`.

## Health check (do this when asked about backups)

```sh
systemctl --user status argit-backup.timer        # is the hourly timer active?
systemctl --user list-timers argit-backup.timer   # when did it last run / next run?
journalctl --user -u argit-backup.service -n 50    # recent backup output
cd ~/workspace && argit doctor                     # manifest coverage, GPG, git state
```

If the timer is inactive: `systemctl --user enable --now argit-backup.timer`.

## Commands

| Command | Purpose |
|---------|---------|
| `argit backup` | Run backup (sanitize, snapshot, stage) — no push |
| `argit backup --push` | Backup + commit + push (what the timer runs) |
| `argit review` | List uncovered paths (files in `~/.hermes` not in any manifest) |
| `argit doctor` | Read-only diagnostic report |

Run argit from the workspace repo: `cd ~/workspace`. (`argit restore` is operator-only — see below.)

## Handling backup warnings (uncovered paths)

When `argit backup` warns `! not backed up` or `argit review` lists uncovered paths:

1. `argit review` — read the report
2. Classify each path: secret / data / sqlite / blob / noise
3. Add it to the **local overlay** (never edit the bundled manifest):
   `.argit/manifest/<bundled-basename>.manifest.local.json`
   ```json
   {
     "items": [
       { "kind": "data", "source": "my-new-dir/" }
     ],
     "exclude": [
       "my-scratch.tmp"
     ]
   }
   ```
4. Re-run `argit backup` to confirm clean.

## Restore is NOT your job

`argit restore` is destructive and is performed **only by a human operator** (or automatically by provisioning on a VM rebuild). Do **not** run `argit restore` yourself, even if state looks wrong or a user asks. If you suspect lost or corrupted state, surface it to your operator with the relevant `argit doctor` output and stop there.

## Troubleshooting

- `argit doctor` — manifest coverage, GPG status, git state
- GPG errors → confirm the agent key is present and trusted: `gpg --list-keys`
- Push failures → check the workspace remote and SSH access: `cd ~/workspace && git remote -v && git push`
- Climbing failures in `journalctl --user -u argit-backup.service` → start with `argit doctor`
