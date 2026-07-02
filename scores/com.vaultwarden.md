# Vaultwarden

Vaultwarden is a self-hosted password manager. Tempo shows its security-relevant authentication events, plus whether the server is reachable.

## Helper

This source needs a **companion helper**: a log watcher. Tempo does not install or run it for you. You copy the files to the right host and configure it there by hand.

- **Type:** a log watcher. Vaultwarden has no outbound webhook, so the watcher tails the container log and posts a structured event whenever it sees an auth-relevant line.
- **Where it runs:** on the **Vaultwarden container host** specifically (it reads `docker logs`), which can be any Linux or macOS host. Its secrets live in a local `.env` file.
- **Get the files:** in the score's **Source** tab (Score Editor), the **Helper** section has **Open in Finder** and **Open README**. *Open in Finder* copies the package to `~/Library/Application Support/Tempo/Integrations/com.vaultwarden/` and reveals it. Copy that folder to the Vaultwarden host, then follow its **Open README** (or the steps below).

## Setup

1. **Fill in `vaultwarden.env`**:
   - `VW_URL` = your Vaultwarden base URL (for example `https://vault.lan:8443`)
   - `TEMPO_URL` = `http://<mac-running-tempo>:7776/ingest`
   - `TEMPO_TOKEN` = a token bound to `com.vaultwarden` (from **Settings > Ingestion**)
   - `VW_CONTAINER` = the container name (default `vaultwarden`)
2. **Run `vaultwarden-run.sh`** under a flock keepalive cron so it restarts on crash or reboot (the helper's README has the exact line).

The Vaultwarden admin token is never sent to Tempo, and the watcher never reads it: it only tails `docker logs`. Its single secret is the Tempo ingest token.

### Turning signals off

The **liveness watch** (down / back up, polling `/alive`) is on by default. If you already monitor the container with another tool, silence it with `VW_EMIT_DOWN=0` (poll interval `VW_ALIVE_INTERVAL`, default 30s).

To quietly drop a specific **event type** you don't care about -- without touching the helper -- set Tempo to **dismiss it on arrival**: in the score's **Ack and dismiss** tab (Score Editor), add a rule that auto-dismisses events matching that `Event` (for example `user_login`). They still land for the record but never demand attention.

### Keeping secrets out of plaintext (`*_FILE`)

You do not have to leave the token in a plaintext `.env`. Every secret also resolves from a **file**: set `TEMPO_TOKEN_FILE` to a path and the watcher reads the secret from there, so the value never lives in the environment or a plaintext file. Point it at:

- a **Docker secret** (`/run/secrets/...`, mounted in tmpfs, never written to disk), or
- a **systemd credential** (`LoadCredentialEncrypted`, encrypted at rest), or
- any `chmod 600` file.

Resolution order: **`TEMPO_TOKEN_FILE` (file) → `TEMPO_TOKEN` (env)**.

**Firewall:** the Vaultwarden host must reach the Mac on port **7776** (allow it in the macOS firewall or Little Snitch, and optionally restrict the token to that host's IP with its allowlist in **Settings > Ingestion**).

## What you'll see

- **Failed logins**, with the source IP and account email, and a **burst** alert when 5 or more fail within 5 minutes.
- User logins, admin login success and failure, vault exports, new users, and **invitations sent**.
- **Down / back up** -- a separate liveness watch polls Vaultwarden's `/alive`, so a stopped container is noticed (the log tail alone can't see "down").

Buttons open the vault or admin, and copy the source IP or user email.

> An "invitation sent" event carries **no email** yet: the address isn't in the Vaultwarden log unless SMTP is configured (Vaultwarden then logs the recipient).

## Known limitations

The watcher reads Vaultwarden's container log, so it reports authentication events (plus `/alive` reachability), not vault contents. It must run on the container host (it uses `docker logs`).
