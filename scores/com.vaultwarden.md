# Vaultwarden score

Surface Vaultwarden auth events, admin actions, and liveness in Tempo with
four default actions (open vault, open admin, copy server URL, copy user
email).

Vaultwarden has **no native outbound webhook**, so this integration is
**poll- and log-driven**: a small bash script either polls Vaultwarden's
admin diagnostics endpoint (liveness) and/or tails its log file (auth
events), and POSTs to Tempo when something changes.

The integration intentionally keeps the source's secrets local — your admin
token never leaves the machine running the script.

Tested with Vaultwarden 1.32+ in a standard Docker setup.

## Setup

### 1. Issue a Tempo ingestion token

1. Open Tempo → Settings → Ingestion
2. Add a token, label it `vaultwarden`
3. Copy the token

### 2. Drop the score into Tempo

Either:
- Download `com.vaultwarden.json` from this repo into
  `~/Library/Application Support/Tempo/Scores/` (Tempo reloads within a second), **or**
- Wait for the V1.1 in-app catalog browser

### 3. Install the polling + log-tail script

The script does two things on each run:
- **Liveness probe** via `${VW_URL}/alive` (public, no auth) — emits a
  `Status` transition when the server stops/starts responding
- **Auth event tail** via the Vaultwarden log file — emits `Event` entries
  for failed/successful logins, admin actions, vault exports, etc.

Save as `vaultwarden-tempo.sh`, edit the config, run on cron (e.g. every
2 minutes for liveness; the log tail does its own state-tracking).

```sh
#!/usr/bin/env bash
# vaultwarden-tempo.sh — emit Vaultwarden state + auth events to Tempo
set -euo pipefail

# ── Config ────────────────────────────────────────────────────────────────
VW_URL="https://vault.example.lan"
VW_LOG="/var/log/vaultwarden/access.log"   # or wherever yours lives
TEMPO_URL="http://your-mac.local:7776/ingest"
TEMPO_TOKEN="paste-tempo-token-here"
STATE_DIR="${HOME}/.local/state/vaultwarden-tempo"
# ──────────────────────────────────────────────────────────────────────────

mkdir -p "$STATE_DIR"
LAST_STATUS_FILE="${STATE_DIR}/last_status"
LAST_LOG_OFFSET="${STATE_DIR}/last_log_offset"

emit_event() {
    local title=$1 event=$2 status=${3:-} email=${4:-}
    curl -sS -X POST \
        -H "X-Tempo-Token: ${TEMPO_TOKEN}" \
        -H "Content-Type: application/json" \
        -d "{
          \"providerIdentifier\": \"com.vaultwarden\",
          \"title\": \"${title}\",
          \"startDate\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\",
          \"eventType\": \"alert\",
          \"metadata\": {
            \"Event\": \"${event}\",
            \"Status\": \"${status}\",
            \"ServerUrl\": \"${VW_URL}\",
            \"UserEmail\": \"${email}\"
          }
        }" \
        "${TEMPO_URL}" >/dev/null
}

# ── 1. Liveness probe ────────────────────────────────────────────────────
if curl -fsS --max-time 5 "${VW_URL}/alive" >/dev/null 2>&1; then
    STATUS="up"
else
    STATUS="unreachable"
fi
LAST=$( [ -f "$LAST_STATUS_FILE" ] && cat "$LAST_STATUS_FILE" || echo "")
echo "$STATUS" > "$LAST_STATUS_FILE"
if [ "$STATUS" != "$LAST" ]; then
    case "$STATUS" in
        up)          emit_event "Vaultwarden up"              "alive_recovered" "up" ;;
        unreachable) emit_event "Vaultwarden unreachable"     "alive_failed"    "unreachable" ;;
    esac
fi

# ── 2. Auth event tail ───────────────────────────────────────────────────
[ -f "$VW_LOG" ] || exit 0
LAST_OFFSET=$( [ -f "$LAST_LOG_OFFSET" ] && cat "$LAST_LOG_OFFSET" || echo 0)
CURRENT_SIZE=$(stat -f%z "$VW_LOG" 2>/dev/null || stat -c%s "$VW_LOG")
# Log rotation handling
[ "$CURRENT_SIZE" -lt "$LAST_OFFSET" ] && LAST_OFFSET=0

tail -c +$((LAST_OFFSET + 1)) "$VW_LOG" | while IFS= read -r line; do
    case "$line" in
        *"Username or password is incorrect"*|*"login failed"*)
            email=$(echo "$line" | grep -oE '[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+' | head -n 1)
            emit_event "Login failed" "login_failed" "" "$email"
            ;;
        *"User logged in successfully"*|*"Logged in"*)
            email=$(echo "$line" | grep -oE '[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+' | head -n 1)
            emit_event "User logged in" "user_login" "" "$email"
            ;;
        *"Admin authenticated"*)
            emit_event "Admin login"          "admin_login"        "" ""
            ;;
        *"Admin login failed"*)
            emit_event "Admin login failed"   "admin_login_failed" "" ""
            ;;
        *"User registered"*|*"User created"*)
            email=$(echo "$line" | grep -oE '[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+' | head -n 1)
            emit_event "User created" "user_created" "" "$email"
            ;;
        *"Vault exported"*|*"Exported vault"*)
            email=$(echo "$line" | grep -oE '[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+' | head -n 1)
            emit_event "Vault exported" "vault_exported" "" "$email"
            ;;
    esac
done

echo "$CURRENT_SIZE" > "$LAST_LOG_OFFSET"
```

> **Tuning the script.** The patterns above match the default Vaultwarden
> log format. Confirm yours with `tail vaultwarden.log` and tweak the `case`
> branches if the strings differ. Vaultwarden's wording has changed across
> versions — adapt rather than blindly trust.

### 4. Schedule it

```sh
crontab -e
# Every 2 minutes — covers liveness within ~2min, log tail catches up
*/2 * * * * /path/to/vaultwarden-tempo.sh >> /tmp/vaultwarden-tempo.log 2>&1
```

### 5. Optional: brute-force burst detection

Add this to the script after the log tail to flag suspicious patterns:

```sh
# If 5+ login_failed events appeared in the last 5 minutes, escalate
RECENT_FAILS=$(grep -c "login_failed" "${STATE_DIR}/recent.log" 2>/dev/null || echo 0)
if [ "$RECENT_FAILS" -ge 5 ]; then
    emit_event "Login failures burst (${RECENT_FAILS}/5min)" "login_failed_burst" "" ""
fi
```

This triggers the `warning` severity rule with a "Brute-force?" badge.

## Severity rules

| Match                              | Severity   | Badge          |
| ---------------------------------- | ---------- | -------------- |
| `Status: down`                     | `critical` | Down           |
| `Status: unreachable`              | `critical` | Unreachable    |
| `Event: login_failed_burst`        | `warning`  | Brute-force?   |
| `Event: admin_login_failed`        | `warning`  | Admin fail     |
| `Event: vault_exported`            | `warning`  | Vault export   |
| `Event: login_failed`              | `info`     | Login fail     |
| `Event: user_login`                | `info`     | Login          |
| `Event: user_created`              | `info`     | New user       |
| `Event: user_invited`              | `info`     | Invite         |
| `Event: admin_login`               | `info`     | Admin          |
| `Event: backup_completed`          | `info`     | Backup         |
| _(default)_                        | `info`     | Info           |

`vault_exported` is escalated to **warning** intentionally: even when
legitimate, an export is a sensitive action worth surfacing in case it was
not done by the account holder.

## Required `metadata` fields

- **`ServerUrl`** — base URL of Vaultwarden (e.g. `https://vault.example.lan`).
- **`Event`** — drives severity. Required for any non-liveness event.
- **`Status`** — used only for liveness transitions (`up` / `unreachable` /
  `down`).
- **`UserEmail`** — set when the event involves a user (login, export,
  registration). The "Copy user email" action reads it.

## Troubleshooting

```sh
# 1. Reachability — Mac → Vaultwarden, and script host → Tempo
nc -vz vault.example.lan 443
nc -vz your-mac.local 7776

# 2. Vaultwarden alive endpoint
curl -fsS https://vault.example.lan/alive && echo " ALIVE OK"

# 3. Tempo manual POST (mimics the script's auth event)
curl -sv -X POST \
  -H "X-Tempo-Token: YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"providerIdentifier":"com.vaultwarden","title":"manual test","startDate":"2026-04-29T10:00:00Z","eventType":"alert","metadata":{"Event":"login_failed","ServerUrl":"https://vault.example.lan","UserEmail":"alice@example.com"}}' \
  http://your-mac.local:7776/ingest

# 4. Watch Tempo's ingestion log
log stream --predicate 'subsystem == "app.tempo"' --level debug | grep -i vaultwarden

# 5. Confirm the script's cron output + state files
ls -la ~/.local/state/vaultwarden-tempo/
tail -f /tmp/vaultwarden-tempo.log
```

## Sample event payload

```json
{
  "providerIdentifier": "com.vaultwarden",
  "title": "Login failed",
  "startDate": "2026-04-29T10:00:00Z",
  "eventType": "alert",
  "metadata": {
    "Event": "login_failed",
    "Status": "",
    "ServerUrl": "https://vault.example.lan",
    "UserEmail": "alice@example.com"
  }
}
```

## Notes

- The reviewed-catalog score uses only `openURL` and `copyToClipboard`.
  Disabling Vaultwarden, kicking active sessions, or rotating the admin
  token are all sensitive actions and require a local drop-in score —
  explicitly trusted by you, not the catalog.
- For multi-instance setups, run one script per Vaultwarden with its own
  `VW_URL` and state directory; events flow to the same Tempo source.
- The Vaultwarden admin token is **never** sent to Tempo. The script reads
  the local log; the token only lives on the machine running the script.
