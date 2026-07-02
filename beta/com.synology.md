# Synology

Synology DSM is a NAS operating system. Tempo shows its notifications: storage, security and system events.

## Helper

**Not required.** DSM posts to Tempo directly through its native webhook notifications.

## Setup

In **DSM > Control Panel > Notification > Webhooks**, add a provider:

- **URL**: `http://<your-mac-ip>:7776/ingest`
- **Header**: `X-Tempo-Token: <token>` (from **Settings > Ingestion**)

DSM lets you author the JSON body sent to the webhook. The score keys on three metadata fields, so map them explicitly: severity rules glob on `metadata.subject`, grouping uses `metadata.hostname`, and the actions use `${metadata.hostname}` (DSM/SSH/ping URLs) and `${metadata.message}` (copy action). Use DSM's body editor with its `@@text@@` / `@@title@@` substitution tokens:

```json
{
  "title": "@@title@@",
  "providerIdentifier": "com.synology",
  "metadata": {
    "subject": "@@title@@",
    "message": "@@text@@",
    "hostname": "your-nas.local"
  }
}
```

Replace `your-nas.local` with the NAS hostname or IP (the action URLs build on it, e.g. `https://${metadata.hostname}:5001/`). Then enable which event categories should notify (storage, security, system).

Storage and security are the events you will usually want to act on; system notifications are chattier.

## What you'll see

DSM notifications, keyed by subject. The score raises severity by subject glob: `*Critical*` / `*Attack*` / `*Intrusion*` (critical), `*Error*` / `*Failed*` / `*Crashed*` / `*Degraded*` / `*Disabled*` (error), `*Warning*` / `*Full*` / `*SMART*` (warning), `*Completed*` (ok).

Buttons open DSM, Storage Manager, Log Center, Security Advisor, the DSM/Synology docs, or SSH/ping the NAS and copy the hostname/message.

## Known limitations

This score has not yet been tested against a live DSM, so its field mapping (including the body template above) is provisional. If events arrive but look wrong (missing subject, fields in the wrong place), check the audit (the shield icon) to see the raw payload and adjust the score's mapping.
