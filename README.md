# hermes-skills

A [Hermes Agent](https://hermes-agent.nousresearch.com) **skills tap** for Blink-provisioned agents.

Skills here are seeded automatically onto Hermes VMs by [`bot-provisioning`](https://github.com/blinkbitcoin/bot-provisioning-poc). You can also add the tap to any Hermes install yourself:

```sh
hermes skills tap add blinkbitcoin/hermes-skills
hermes skills install blinkbitcoin/hermes-skills/skills/argit-backup
hermes skills install blinkbitcoin/hermes-skills/skills/secrets-management
hermes skills update    # later: refresh installed skills
```

## Skills

| Skill | What it covers |
|-------|----------------|
| [`argit-backup`](skills/argit-backup/SKILL.md) | Operate and health-check the argit backup of `~/.hermes` (systemd timer, manifests, restore/DR) |
| [`secrets-management`](skills/secrets-management/SKILL.md) | Where Hermes keeps secrets (`~/.hermes/.env` vs the `pass` backup vault), `hermes model`, safe handling |

## Layout

Each skill is a directory under `skills/` with a `SKILL.md` (+ optional `references/`). This is the default tap path, so the tap is added with no extra config.

## Notes

- Public on purpose: these are operational instructions, **not** secret material.
- Tuned for Blink's Hermes deployment (argit DR, systemd `--user` backup timer, Slack Socket Mode).
