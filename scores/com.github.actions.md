# GitHub Actions

GitHub is a code hosting platform. Tempo shows activity from your repos: workflow runs, pull requests, issues, releases, and security alerts.

## Helper

This source needs a **companion helper**: a relay. Tempo does not install or run it for you. You copy the files to a host you choose and configure it there by hand.

- **Type:** a relay. GitHub's servers are in the cloud and cannot reach your LAN, so the relay receives GitHub's webhooks (verifying their HMAC signature) and forwards them to Tempo.
- **Where it runs:** any host (Mac, Linux, container) that can reach GitHub's tunnel and Tempo. Secrets come from the macOS **Keychain** on a Mac, or from **environment variables / a `.env` file** elsewhere.
- **Get the files:** in the score's **Source** tab (Score Editor), the **Helper** section has **Open in Finder** and **Open README**. *Open in Finder* copies the package to `~/Library/Application Support/Tempo/Integrations/com.github.actions/` and reveals it. Copy that folder to the host that will run the relay, then follow its **Open README** (or the steps below).

## Setup

The relay listens on port `7777` and posts to Tempo's ingestion endpoint. GitHub reaches it through a public tunnel you run. The Tempo token must be **bound to `com.github.actions`** in **Settings > Ingestion**; that binding also covers any dotted sub-namespace under it. In short:

1. **Give the relay its two secrets** (the HMAC secret and the Tempo token, from **Settings > Ingestion**, bound to `com.github.actions`). The relay reads the environment first, then the macOS Keychain.
   - **On a Mac**, store them in the Keychain (never on disk):
     ```sh
     security add-generic-password -s tempo-gh-relay  -a webhook-secret     -w '<HMAC secret>'
     security add-generic-password -s tempo-ingestion -a com.github.actions  -w '<Tempo token>'
     ```
   - **On Linux / Docker / a `.env` file**, set environment variables instead:
     ```sh
     TEMPO_GH_WEBHOOK_SECRET=<HMAC secret>
     TEMPO_TOKEN=<Tempo token>
     ```
   - **To keep secrets off plaintext (recommended on Linux)**, use the `*_FILE` convention: set `TEMPO_GH_WEBHOOK_SECRET_FILE` / `TEMPO_TOKEN_FILE` to a *path* and the relay reads each secret from that file, so the value never lives in the environment or a plaintext `.env`. Point it at a **Docker secret** (`/run/secrets/...`, tmpfs), a **`systemd-creds`** encrypted credential (encrypted at rest), or any `chmod 600` file. Resolution order per secret: **`*_FILE` → env var → macOS Keychain**.
2. **Point the relay at Tempo:** set `TEMPO_URL` to `http://<mac-running-tempo>:7776/ingest` (env var, or edit the default in `relay.py`). Keep the default `127.0.0.1` only if the relay runs on the same Mac as Tempo.
3. **Run the relay** with the bundled LaunchAgent.
4. **Expose it** with a public tunnel (Cloudflare Tunnel, Tailscale Funnel, ngrok) pointing at the relay host's port `7777`.
5. **Add a GitHub webhook** (repo or org Settings > Webhooks) to your tunnel URL plus `/gh`, content type `application/json`, with the same HMAC secret.

**Firewall:** if the relay runs on a different host than Tempo, that host must reach the Mac on port **7776** (allow it in the macOS firewall or Little Snitch, and optionally restrict the token to that host's IP with its allowlist in **Settings > Ingestion**). The tunnel is the only internet-facing surface: scope it to `/gh` and keep the HMAC secret private.

## What you'll see

- Workflow runs (succeeded / failed), pull requests, reviews, issues, releases, deployments, and security alerts (Dependabot and advisories raised as critical).

Buttons open the run, the repo's Actions tab, or clone the repo. The **Open run** button renders disabled when an event has no `${metadata.runURL}` (for example a non-workflow event); the other buttons stay active.

## Known limitations

The relay needs a public tunnel to receive GitHub's webhooks: that is an internet-facing surface, guarded by the HMAC signature check on every request. Event types the relay does not handle are acknowledged to GitHub but not shown.
