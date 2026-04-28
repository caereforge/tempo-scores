# Jellyfin score

Surface Jellyfin server events in Tempo's timeline with five default actions
(open Jellyfin web, open item details, open admin dashboard, copy server URL,
copy item ID).

Tested with Jellyfin 10.9 + the official **Webhook plugin**.

## Setup

### 1. Install the Webhook plugin in Jellyfin

1. Open the Jellyfin web UI as an administrator
2. Dashboard → Plugins → **Catalog**
3. Search for **Webhook**, click Install
4. **Restart** the Jellyfin server (Dashboard → Restart Server)

### 2. Issue a Tempo ingestion token

1. Open Tempo → Settings → Ingestion
2. Add a new token, label it `jellyfin`
3. Copy the token (you'll need it next)

### 3. Configure the webhook destination in Jellyfin

1. Dashboard → Plugins → **Webhook**
2. Click **Add Generic Destination**
3. Fill in the form:
   - **Webhook Name**: `Tempo`
   - **Webhook URL**: `http://your-mac.local:7776/ingest`
     (replace `your-mac.local` with your Mac's hostname or LAN IP)
   - **Headers** — add two:
     | Key                | Value                       |
     | ------------------ | --------------------------- |
     | `X-Tempo-Token`    | _the token from step 2_     |
     | `Content-Type`     | `application/json`          |
   - **Notification Types**: tick the events you want surfaced. Suggested set:
     - `Authentication Failure`
     - `Application Error`
     - `Plugin Installation Failed`
     - `Plugin Update Failed`
     - `Scheduled Task Failed`
     - `Item Added`
     - `Playback Start` / `Playback Stop` _(optional — high volume)_
   - **Template** (paste verbatim into the Template field):

     ```handlebars
     {
       "providerIdentifier": "org.jellyfin",
       "title": "{{NotificationType}}{{#if ItemName}} — {{ItemName}}{{/if}}",
       "startDate": "{{Timestamp}}",
       "eventType": "alert",
       "metadata": {
         "NotificationType": "{{NotificationType}}",
         "ServerName":       "{{ServerName}}",
         "ServerUrl":        "{{ServerUrl}}",
         "ServerVersion":    "{{ServerVersion}}",
         "Username":         "{{NotificationUsername}}",
         "ItemName":         "{{ItemName}}",
         "ItemId":           "{{ItemId}}",
         "ItemType":         "{{ItemType}}",
         "DeviceName":       "{{DeviceName}}",
         "ClientName":       "{{ClientName}}"
       }
     }
     ```

4. Click **Save**

### 4. Drop the score into Tempo

Either:
- Download `org.jellyfin.json` from this repo into
  `~/Library/Application Support/Tempo/Scores/` (Tempo reloads within a second), **or**
- Wait for the V1.1 in-app catalog browser, then install with one click

### 5. Verify

Trigger any of the configured events in Jellyfin (e.g. add a movie, fail a
login). The event should appear in Tempo's timeline within a couple of
seconds, painted in Jellyfin purple, with the five default actions in the
right panel.

## Severity rules

| Notification type           | Severity   | Badge       |
| --------------------------- | ---------- | ----------- |
| `AuthenticationFailure`     | `error`    | Auth fail   |
| `ApplicationError`          | `error`    | Error       |
| `*Failed` (any)             | `error`    | Failed      |
| `ScheduledTaskFailed`       | `warning`  | Task failed |
| `PluginInstalled`           | `info`     | Plugin      |
| `PluginUninstalled`         | `info`     | Plugin      |
| `PluginUpdated`             | `info`     | Plugin      |
| `ItemAdded`                 | `info`     | New item    |
| `PlaybackStart`             | `info`     | Started     |
| `PlaybackStop`              | `info`     | Stopped     |
| _(default)_                 | `info`     | Info        |

## Required `metadata` fields

Most actions need `ServerUrl`. The "Open item" action and "Copy item ID"
need `ItemId` (only set on item-related notifications — clicking them on a
non-item event opens a malformed URL, harmless).

## Troubleshooting

```sh
# 1. Reachability — can Jellyfin reach the Mac on 7776?
nc -vz your-mac.local 7776

# 2. Token + payload sanity — manual POST that mimics a Jellyfin event
curl -sv -X POST \
  -H "X-Tempo-Token: YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"providerIdentifier":"org.jellyfin","title":"manual test","startDate":"2026-04-29T10:00:00Z","eventType":"alert","metadata":{"NotificationType":"PlaybackStart","ServerUrl":"http://media.lan:8096","ItemId":"abc123","ItemName":"Test"}}' \
  http://your-mac.local:7776/ingest

# 3. Watch Tempo's ingestion log (should show the incoming event)
log stream --predicate 'subsystem == "app.tempo"' --level debug | grep -i jellyfin

# 4. Watch Jellyfin's webhook plugin log (server-side)
tail -f /var/log/jellyfin/log_*.log | grep -i webhook

# 5. Confirm via Tempo search — open Tempo, type "jellyfin"
```

## Sample event payload

A complete payload exercising the score's severity rules and metadata
interpolation:

```json
{
  "providerIdentifier": "org.jellyfin",
  "title": "PlaybackStart — The Office S03E12",
  "startDate": "2026-04-29T22:14:00Z",
  "eventType": "alert",
  "metadata": {
    "NotificationType": "PlaybackStart",
    "ServerName":       "media-server",
    "ServerUrl":        "http://media.lan:8096",
    "ServerVersion":    "10.9.6",
    "Username":         "alice",
    "ItemName":         "The Office S03E12",
    "ItemId":           "abc123def456",
    "ItemType":         "Episode",
    "DeviceName":       "Apple TV",
    "ClientName":       "Jellyfin Apple TV"
  }
}
```

## Notes

- This score lives in the reviewed catalog and uses only `openURL` and
  `copyToClipboard` actions. For Terminal-based actions (SSH, log tail,
  service restart), use a local drop-in score — those require the user's
  explicit trust and can't ship in the catalog.
- `ServerUrl` should be reachable from your Mac (LAN URL is fine, or a
  Tailscale/VPN address). Tempo doesn't care — it just builds the link for
  your browser to follow.
