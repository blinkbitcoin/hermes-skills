# Automated Backup — how it actually runs on this VM

Backups are **not** a `hermes cron` job. They run as systemd `--user` units installed at provision time.

## Units

| Unit | Role |
|------|------|
| `argit-backup.timer` | Fires hourly (`OnCalendar=hourly`, randomized delay up to 5 min, `Persistent=true`) |
| `argit-backup.service` | Oneshot the timer triggers — runs `argit backup --push` from `~/workspace` |
| `argit-backup-shutdown.service` | Stays active only to carry an `ExecStop` that runs a final `argit backup --push` on shutdown (best-effort) |

## Inspect

```sh
systemctl --user list-timers argit-backup.timer    # last run / next run
systemctl --user status argit-backup.service       # last result
journalctl --user -u argit-backup.service -n 100   # output history
```

## Run a backup now

```sh
systemctl --user start argit-backup.service        # same path the timer uses
# or, directly:
cd ~/workspace && argit backup --push
```

## If the timer isn't running

```sh
systemctl --user enable --now argit-backup.timer
loginctl show-user "$USER" -p Linger               # should be Linger=yes so it runs without a login session
```

Do not add a `hermes cron` backup job — it would run backups twice.
