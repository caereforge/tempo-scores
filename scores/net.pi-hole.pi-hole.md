# Pi-hole score

Surface Pi-hole health and configuration changes in Tempo's timeline with
five default actions (open admin, open query log, open settings, copy server
URL, copy domain).

Pi-hole has no native push webhook out of the box, so this integration is
**poll-driven**: a small bash script runs on cron, checks Pi-hole's state via
its HTTP API, and POSTs an event to Tempo when something interesting changes.
The script is not bundled — pick the variant that matches your Pi-hole
version and your alerting taste.

Tested with Pi-hole **v6** (FTL HTTP API). v5 with the legacy PHP API also
works with minor URL adjustments — see the v5 note at the bottom.

## Setup

### 1. Issue a Tempo ingestion token

1. Open Tempo → Settings → Ingestion
2. Add a token, label it `pi-hole`
3. Copy the token

### 2. Drop the score into Tempo

Either:
- Download `net.pi-hole.pi-hole.json` from this repo into
  `~/Library/Application Support/Tempo/Scores/` (Tempo reloads within a second), **or**
- Wait for the V1.1 in-app catalog browser

### 3. Install the polling script (Pi-hole side or any host that can reach both Pi-hole and Tempo)

Save the script below as `pihole-tempo.sh`, edit the four config values at the
top, and run it on cron every 5–10 minutes. The script tracks state across
runs so it only POSTs to Tempo when something **changes** (no spam).

```sh
#!/usr/bin/env bash
# pihole-tempo.sh — emit Pi-hole state changes to Tempo
set -euo pipefail

# ── Config ────────────────────────────────────────────────────────────────
PIHOLE_URL="http://pi.hole"        # base URL of your Pi-hole
PIHOLE_PASS="your-admin-password"  # admin password (v6) or API token
TEMPO_URL="http://your-mac.local:7776/ingest"
TEMPO_TOKEN="paste-tempo-token-here"
STATE_DIR="${HOME}/.local/state/pihole-tempo"
# ──────────────────────────────────────────────────────────────────────────

mkdir -p "$STATE_DIR"
LAST_FILE="${STATE_DIR}/last_status"

# Auth (Pi-hole v6 — single-shot session)
SID=$(curl -s -X POST -H "Content-Type: application/json" \
    -d "{\"password\":\"${PIHOLE_PASS}\"}" \
    "${PIHOLE_URL}/api/auth" | jq -r '.session.sid // empty')

if [ -z "$SID" ]; then
    STATUS="unreachable"
else
    BLOCKING=$(curl -s -H "X-FTL-SID: $SID" "${PIHOLE_URL}/api/dns/blocking" \
               | jq -r '.blocking // "unknown"')
    case "$BLOCKING" in
        enabled)   STATUS="up" ;;
        disabled)  STATUS="disabled" ;;
        *)         STATUS="unreachable" ;;
    esac
fi

LAST=$( [ -f "$LAST_FILE" ] && cat "$LAST_FILE" || echo "")
echo "$STATUS" > "$LAST_FILE"

# Only emit on transitions
if [ "$STATUS" = "$LAST" ]; then exit 0; fi

case "$STATUS" in
    up)          ACTION="blocking_enabled";  TITLE="Pi-hole — blocking enabled" ;;
    disabled)    ACTION="blocking_disabled"; TITLE="Pi-hole — blocking disabled" ;;
    unreachable) ACTION="";                  TITLE="Pi-hole unreachable" ;;
esac

curl -sS -X POST \
    -H "X-Tempo-Token: ${TEMPO_TOKEN}" \
    -H "Content-Type: application/json" \
    -d "{
      \"providerIdentifier\": \"net.pi-hole.pi-hole\",
      \"title\": \"${TITLE}\",
      \"startDate\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\",
      \"eventType\": \"alert\",
      \"metadata\": {
        \"Status\": \"${STATUS}\",
        \"Action\": \"${ACTION}\",
        \"ServerUrl\": \"${PIHOLE_URL}\"
      }
    }" \
    "${TEMPO_URL}" >/dev/null
```

### 4. Schedule it

```sh
crontab -e
# Run every 5 minutes
*/5 * * * * /path/to/pihole-tempo.sh >> /tmp/pihole-tempo.log 2>&1
```

### 5. Verify

Disable Pi-hole blocking from the admin UI for 30s, then run the script
manually. You should see a `Pi-hole — blocking disabled` event in Tempo
within a couple of seconds, marked **warning**.

## Severity rules

| Match                              | Severity   | Badge        |
| ---------------------------------- | ---------- | ------------ |
| `Status: down`                     | `critical` | Down         |
| `Status: unreachable`              | `critical` | Unreachable  |
| `Status: high_load`                | `warning`  | High load    |
| `Status: disabled`                 | `warning`  | Disabled     |
| `Action: blocking_disabled`        | `warning`  | Blocking off |
| `Action: update_available`         | `info`     | Update       |
| `Action: gravity_update`           | `info`     | Gravity      |
| `Action: blocking_enabled`         | `info`     | Blocking on  |
| _(default)_                        | `info`     | Info         |

## Required `metadata` fields

- **`ServerUrl`** — base URL of the Pi-hole (e.g. `http://pi.hole`). Used by
  every action.
- **`Status`** or **`Action`** — drives severity. At least one should be
  present. The reference script above sets `Status` for liveness transitions
  and `Action` for configuration changes.
- **`Domain`** — only when the event is about a specific domain (e.g. an
  unblock action). Used by the "Copy domain" action; optional otherwise.

## Pi-hole v5 note

For v5, replace the auth/blocking calls with the legacy PHP API:

```sh
# v5 auth: API token (web admin → Settings → API → "Show API token")
PIHOLE_TOKEN="your-api-token-here"

BLOCKING=$(curl -s "${PIHOLE_URL}/admin/api.php?status&auth=${PIHOLE_TOKEN}" \
           | jq -r '.status // "unknown"')
case "$BLOCKING" in
    enabled)  STATUS="up" ;;
    disabled) STATUS="disabled" ;;
    *)        STATUS="unreachable" ;;
esac
```

The rest of the script is identical.

## Troubleshooting

```sh
# 1. Reachability — can your script's host reach Tempo?
nc -vz your-mac.local 7776

# 2. Pi-hole API auth (v6) — does the password work?
curl -sS -X POST -H "Content-Type: application/json" \
  -d '{"password":"your-pass"}' http://pi.hole/api/auth

# 3. Tempo manual POST (mimics the script)
curl -sv -X POST \
  -H "X-Tempo-Token: YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"providerIdentifier":"net.pi-hole.pi-hole","title":"manual test","startDate":"2026-04-29T10:00:00Z","eventType":"alert","metadata":{"Status":"disabled","ServerUrl":"http://pi.hole"}}' \
  http://your-mac.local:7776/ingest

# 4. Watch Tempo's ingestion log
log stream --predicate 'subsystem == "app.tempo"' --level debug | grep -i pi-hole

# 5. Confirm the script's cron output
tail -f /tmp/pihole-tempo.log
```

## Sample event payload

```json
{
  "providerIdentifier": "net.pi-hole.pi-hole",
  "title": "Pi-hole — blocking disabled",
  "startDate": "2026-04-29T10:00:00Z",
  "eventType": "alert",
  "metadata": {
    "Status": "disabled",
    "Action": "blocking_disabled",
    "ServerUrl": "http://pi.hole"
  }
}
```

## Notes

- The polling script is intentionally minimal — track only Status and Action
  transitions. Extend it as you wish (gravity update detection via
  `/api/dns/blocking` query rate, `/api/info` for version/update checks)
  using the same `metadata` keys the score expects.
- The reviewed-catalog score uses only `openURL` and `copyToClipboard`.
  Terminal-based actions (e.g. `pihole disable 30m`) require a local
  drop-in score — explicitly trusted by the user, not the catalog.
- For multi-instance setups (primary + secondary Pi-hole), run one script
  per instance with its own `ServerUrl` and `PIHOLE_PASS` — Tempo will list
  them as the same source but each event carries its own URL.
