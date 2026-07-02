# Todoist

Todoist is a task manager. Tempo shows today's and overdue tasks alongside the other agenda items. Todoist is agenda-class: it appears in the day view only, never rings the bell, and never counts toward the badge. It is read-only in v1.1; manage tasks in Todoist.

## Helper

This source needs a **companion helper**: a poller. Tempo does not install or run it. Copy the files to a host you choose and configure it there.

- **Type:** a poller. Todoist cannot reach your LAN, so the poller asks the Todoist API for today's tasks on a schedule and posts them to Tempo.
- **Where it runs:** any host (Mac, Linux, container) that can reach the internet and Tempo. Secrets come from the macOS **Keychain** on a Mac, or from **environment variables / a `.env` file** elsewhere.
- **Get the files:** in the score's **Source** tab (Score Editor), the **Helper** section has **Open in Finder** and **Open README**. *Open in Finder* copies the package to `~/Library/Application Support/Tempo/Integrations/com.todoist/` and reveals it. Copy that folder to the host that will run the poller, then follow its **Open README** (or the steps below).

## Setup

1. **Give the poller its two secrets** (your Todoist API token, from Todoist Settings > Integrations > Developer; and the Tempo token, from **Settings > Ingestion**, bound to `com.todoist`). It reads the environment first, then the macOS Keychain.
   - **On a Mac**, store them in the Keychain (account `todoist-poll`):
     ```sh
     security add-generic-password -a todoist-poll -s todoist-api-token -w '<Todoist API token>'
     security add-generic-password -a todoist-poll -s tempo-token       -w '<Tempo token>'
     ```
   - **On Linux / Docker / a `.env` file**, set environment variables instead:
     ```sh
     TODOIST_TOKEN=<Todoist API token>
     TEMPO_TOKEN=<Tempo token>
     ```
   - **To keep secrets off plaintext (recommended on Linux)**, use the `*_FILE` convention: set `TODOIST_TOKEN_FILE` / `TEMPO_TOKEN_FILE` to a *path* and the poller reads each secret from that file, so the value never lives in the environment or a plaintext `.env`. Point it at a **Docker secret** (`/run/secrets/...`, tmpfs), a **`systemd-creds`** encrypted credential (encrypted at rest), or any `chmod 600` file. Resolution order per secret: **`*_FILE` → env var → macOS Keychain**.
2. **Point the poller at Tempo:** set `TEMPO_URL` to `http://<mac-running-tempo>:7776/ingest` (the default `127.0.0.1` is right only if it runs on the same Mac as Tempo). By default it polls the `(today | overdue)` filter.
3. **Schedule it** with the bundled launchd template.

**Firewall:** if the poller runs on a different host than Tempo, that host must reach the Mac on port **7776** (allow it in the macOS firewall or Little Snitch, and optionally restrict the token to that host's IP with its allowlist in **Settings > Ingestion**).

## What you'll see

- Today's and overdue tasks, with the project name.

Buttons open the task or the Todoist app.

## Known limitations

Read-only in v1.1: completing a task from Tempo is not wired (the poller only reads).
