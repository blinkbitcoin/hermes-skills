---
name: secrets-management
description: Handle secrets safely on a Hermes VM. Use when reading or rotating channel/LLM credentials, understanding where Hermes keeps secrets, or configuring an LLM provider. Covers ~/.hermes/.env, the hermes model command, and the GPG-encrypted pass store used for backups.
version: 1.0.0
metadata:
  hermes:
    tags: [secrets, security, credentials, ops]
    category: ops
---

# Secrets Management (Hermes)

There are **two** secret stores on this VM. Know which is which.

## 1. Runtime secrets — `~/.hermes/.env`

This is what the running gateway actually reads. It holds:

- `SLACK_BOT_TOKEN`, `SLACK_APP_TOKEN` — written by provisioning (Slack Socket Mode)
- LLM API keys (e.g. `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`) — written by `hermes model`

To configure or change your LLM provider, model, and key, **run the upstream command — don't hand-edit `config.yaml`:**

```sh
hermes model
```

`hermes model` writes the provider/model into `~/.hermes/config.yaml` and the key into `~/.hermes/.env`. Editing `config.yaml` by hand is unsupported and will be overwritten.

After changing `.env`, restart the gateway:

```sh
systemctl --user restart hermes-gateway.service
```

## 2. Backup vault — the `pass` store (`~/.password-store`)

`pass` is **not** the runtime path. argit GPG-encrypts the `kind: secret` items (`.env`, `auth.json`) into the `pass` store during backup so secrets survive a VM rebuild without ever sitting in plaintext in git. Provisioning also seeds channel tokens here.

Common paths:

- `channels/slack-bot-token`, `channels/slack-app-token` — Slack Socket Mode tokens

Read a backed-up secret if you need to inspect it:

```sh
pass show channels/slack-bot-token
```

Secrets are encrypted to the agent GPG key **and** the IT backup key, so IT can recover state in a disaster.

## Rules — always

- **Never** log, echo, `cat`, or print a secret value to stdout, a file, or a message.
- **Never** store secrets in plaintext outside `~/.hermes/.env` or `pass` (no scratch files, no committed env exports).
- To rotate **Slack** tokens, ask the operator to re-provision — provisioning is the source of truth for channel tokens.
- To rotate an **LLM** key, run `hermes model` again.
- After touching secrets, let the hourly argit backup capture them, or run `cd ~/workspace && argit backup --push`.

## How the encryption works

- `pass` encrypts each secret with GPG (agent key + IT backup key)
- Encrypted `.gpg` blobs live under `~/.password-store/` and are git-tracked for DR
- argit re-injects them into `~/.hermes` on restore
