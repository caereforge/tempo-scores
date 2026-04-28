# Proxmox score

Surface Proxmox VE cluster events in Tempo's timeline with five default
actions (Open Proxmox UI, Open Datacenter view, copy server URL, copy
node name, copy VMID).

Tested with **Proxmox VE 8.1+** which ships a native webhook notification
target — no polling script needed. Each backup, replication, alert, or
manually configured matcher fires a single HTTP POST to Tempo.

## Setup

### 1. Issue a Tempo ingestion token

1. Open Tempo → Settings → Ingestion
2. Add a token, label it `proxmox`
3. Copy the token

### 2. Drop the score into Tempo

Either:
- Download `com.proxmox.json` from this repo into
  `~/Library/Application Support/Tempo/Scores/` (Tempo reloads within a second), **or**
- Wait for the V1.1 in-app catalog browser

### 3. Configure the Proxmox webhook target

In the Proxmox web UI:

1. **Datacenter → Notifications → Add → Webhook**
2. Fill in:
   - **Endpoint Name**: `tempo`
   - **URL**: `http://your-mac.local:7776/ingest` (use the LAN address or hostname your Mac is reachable at from the Proxmox node)
   - **Method**: `POST`
   - **Headers**: add two:
     | Name              | Value                       |
     | ----------------- | --------------------------- |
     | `X-Tempo-Token`   | _the token from step 1_     |
     | `Content-Type`    | `application/json`          |
   - **Body** (paste verbatim — Tera template):

     ```text
     {
       "providerIdentifier": "com.proxmox",
       "title": {{ title|json }},
       "startDate": {{ timestamp|date(format="%Y-%m-%dT%H:%M:%SZ", timezone="UTC")|json }},
       "eventType": "alert",
       "metadata": {
         "severity": {{ severity|json }},
         "ServerUrl": "https://your-proxmox.lan:8006",
         "NodeName": {{ fields.hostname|default(value="")|json }},
         "VMID": {{ fields.vmid|default(value="")|json }},
         "TaskType": {{ fields.type|default(value="")|json }},
         "Message": {{ message|json }}
       }
     }
     ```

     > **Replace `https://your-proxmox.lan:8006`** with your cluster's actual UI URL — the score uses it for the "Open Proxmox UI" / "Open Datacenter" actions. Tera doesn't have introspection on the receiving server's identity, so it has to be hardcoded in the template.

3. Click **Add** to save the endpoint
4. **Datacenter → Notifications → Notification Matchers** — make sure the matcher routing your alerts targets this new `tempo` endpoint (default behavior is to forward everything; if you've got custom matchers, add `tempo` to the relevant ones)

### 4. Verify

Trigger a notification — easiest path is to start a backup job manually
or run a vzdump. Within a few seconds you should see the event in Tempo's
timeline, painted in Proxmox orange, with the five default actions in the
right panel. Click "Open Proxmox UI" to confirm the URL builds correctly.

## Severity rules

| Match              | Severity   | Badge    |
| ------------------ | ---------- | -------- |
| `severity:critical`| `critical` | Critical |
| `severity:error`   | `error`    | Error    |
| `severity:warning` | `warning`  | Warning  |
| `severity:notice`  | `info`     | Notice   |
| `severity:info`    | `info`     | Info     |
| _(default)_        | `info`     | Info     |

Proxmox uses five levels (`info` / `notice` / `warning` / `error` /
`critical`). The score collapses `notice` to `info` because Tempo's
visual model has only four severity bands and `notice` is closer to
informational than warning in Proxmox's own taxonomy.

## Required `metadata` fields

- **`severity`** — drives the badge. Set automatically by Proxmox via the
  template above.
- **`ServerUrl`** — base URL of the Proxmox web UI (e.g.
  `https://node.lan:8006`). Hardcoded in the template; used by the "Open
  Proxmox UI" and "Open Datacenter" actions.
- **`NodeName`** — Proxmox node hostname. Auto-populated from
  `fields.hostname` when present; used by "Copy node name".
- **`VMID`** — VM/container ID when the notification is about a specific
  guest. Auto-populated from `fields.vmid`; used by "Copy VMID". Empty
  for cluster-wide events (replication, certificate, etc.).
- **`TaskType`** / **`Message`** — informational, surfaced in the action
  panel "Details" section but no action depends on them.

## Troubleshooting

```sh
# 1. Reachability — Proxmox can reach Tempo on 7776
ssh root@your-proxmox.lan "nc -vz your-mac.local 7776"

# 2. Token + payload sanity — manual POST that mimics a Proxmox alert
curl -sv -X POST \
  -H "X-Tempo-Token: YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"providerIdentifier":"com.proxmox","title":"manual test","startDate":"2026-04-29T10:00:00Z","eventType":"alert","metadata":{"severity":"warning","ServerUrl":"https://your-proxmox.lan:8006","NodeName":"pve-01","VMID":"100","Message":"test alert"}}' \
  http://your-mac.local:7776/ingest

# 3. Watch Tempo's ingestion log on the Mac side
log stream --predicate 'subsystem == "app.tempo"' --level debug | grep -i proxmox

# 4. Test the Proxmox webhook target itself (Datacenter → Notifications → tempo → Test)
# Or via CLI on the Proxmox host:
ssh root@your-proxmox.lan "pvesh create /cluster/notifications/endpoints/webhook/tempo/test"

# 5. Inspect the actual rendered template (CRITICAL when fields don't show up)
ssh root@your-proxmox.lan "journalctl -u pve-cluster -n 200 | grep -i webhook"
```

## Sample event payload

A complete payload as Proxmox would render it for a backup completion:

```json
{
  "providerIdentifier": "com.proxmox",
  "title": "Backup completed for VM 100",
  "startDate": "2026-04-29T03:14:00Z",
  "eventType": "alert",
  "metadata": {
    "severity": "info",
    "ServerUrl": "https://pve-01.lan:8006",
    "NodeName": "pve-01",
    "VMID": "100",
    "TaskType": "vzdump",
    "Message": "Backup of VM 100 (web-srv) finished successfully (43.2 GB, 4m 17s)"
  }
}
```

## Notes

- **VE 8.0 and earlier**: no native webhook target. Use the
  `notify-script` option (Datacenter → Notifications → Add → Gotify or
  Sendmail wrapped by a custom script) to forward to Tempo. Or upgrade —
  8.1 is stable and webhooks are mature.
- **Multi-cluster**: each cluster needs its own webhook target with its
  own `ServerUrl` hardcoded in the template. Tempo lists them all under
  the same source `Proxmox` (umbrella identifier) and you can tell them
  apart by NodeName.
- **Cert validation**: if your Proxmox UI uses a self-signed certificate,
  the action's `https://` URL works in your browser as long as you've
  accepted the cert there. Tempo doesn't reach out to Proxmox itself —
  only your browser does, and only when you click an action.
- The reviewed-catalog score uses only `openURL` and `copyToClipboard`.
  Stopping/starting VMs, taking snapshots, or running backups remotely
  requires a local drop-in score with `openTerminalWith` actions plus
  whatever auth approach you trust (SSH keys, API tokens). Catalog
  scores stay observation-only.
