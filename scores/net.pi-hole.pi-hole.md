# Pi-hole

Pi-hole is a network-wide ad blocker. Tempo shows when its DNS blocking is turned on or off and when it becomes unreachable, and -- optionally -- when an update is available, the blocklist ("gravity") is refreshed, or the host is under high load.

## Helper

This source needs a **companion helper**: a poller. Tempo does not install or run it for you. You copy the files to a host you choose and configure it there by hand.

- **Type:** a poller. Pi-hole has no outbound webhook, so the poller asks its API (v6) for the blocking state and posts to Tempo only when the state changes.
- **Where it runs:** any host (Linux or macOS) that can reach both Pi-hole and Tempo, typically near Pi-hole. Its secrets live in a local `.env` file, so it is not tied to a Mac.
- **Get the files:** in the score's **Source** tab (Score Editor), the **Helper** section has **Open in Finder** and **Open README**. *Open in Finder* copies the package to `~/Library/Application Support/Tempo/Integrations/net.pi-hole.pi-hole/` and reveals it. Copy that folder to the host that will run the poller, then follow its **Open README** (or the steps below).

## Setup

1. **Fill in `pihole.env`** (beside the script):
   - `TEMPO_URL` = `http://<mac-running-tempo>:7776/ingest`
   - `TEMPO_TOKEN` = a token bound to `net.pi-hole.pi-hole` (from **Settings > Ingestion**)
   - `PIHOLE_URL` = `http://<pihole-host>`
   - `PIHOLE_PASS` = your Pi-hole web/app password
2. **Schedule `pihole-run.sh`** on a roughly 15-second loop (cron or launchd).

Your Pi-hole password is used only against Pi-hole to get a session; it is never sent to Tempo.

### Optional signals

Beyond blocking state (always on), the poller can also report, each independently toggleable:

| Signal | Env flag (default on) | What triggers it |
|---|---|---|
| Update available | `PIHOLE_EMIT_UPDATE` | any component (core / web / FTL / docker) `local != remote` |
| Blocklist (gravity) updated | `PIHOLE_EMIT_GRAVITY` | gravity's blocked-domain count changes |
| High load | `PIHOLE_EMIT_LOAD` | 15-min CPU load over `PIHOLE_LOAD_THRESHOLD` (percent of one core, default `200`) |

Set a flag to `0` to silence that signal -- for example if you already watch host load with another tool such as Beszel: `PIHOLE_EMIT_LOAD=0`.

### Keeping secrets out of plaintext (`*_FILE`)

You do not have to leave the token or the Pi-hole password in a plaintext `.env`. Each secret also resolves from a **file**: set `TEMPO_TOKEN_FILE` and `PIHOLE_PASS_FILE` to paths and the poller reads them from there, so the values never live in the environment or a plaintext file. Point them at:

- a **Docker secret** (`/run/secrets/...`, mounted in tmpfs, never written to disk), or
- a **systemd credential** (`LoadCredentialEncrypted`, encrypted at rest), or
- any `chmod 600` file.

Resolution order per secret: **`*_FILE` (file) → env var**.

**Firewall:** the host running the poller must reach the Mac on port **7776** (allow it in the macOS firewall or Little Snitch, and optionally restrict the token to that host's IP with its allowlist in **Settings > Ingestion**).

## What you'll see

- One row (stateful) that flips between blocking **enabled**, **disabled**, and **unreachable**.
- **Update available** when a Pi-hole component is behind its latest release.
- **Blocklist updated** when gravity's domain count changes (a blocklist refresh).
- **High load** when the host's 15-minute CPU load crosses the threshold.

Buttons open the admin, the query log, or settings.

## Known limitations

The poller reports blocking state, reachability, updates, gravity refreshes and host load. It does not report individual DNS queries.
