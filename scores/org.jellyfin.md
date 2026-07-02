# Jellyfin

Jellyfin is a self-hosted media server. Tempo shows new media and playback activity from it.

## Helper

**Not required.** Jellyfin posts to Tempo directly through its Webhook plugin.

## Setup

Install the **Webhook** plugin (Dashboard > Plugins > Catalog), then add a **Generic Destination**:

- **URL**: `http://<your-mac-ip>:7776/ingest`. Keep Jellyfin on the **plain** port. Its webhook payloads aren't compatible with Tempo's TLS port (`:8776`): the request arrives empty (a known limitation it shares with UniFi). Jellyfin's playback/library events on a trusted LAN are fine over plain HTTP.
- **Header**: `X-Tempo-Token: <token>` (from **Settings > Ingestion**)
- **Notification types**: *Item Added*, *Playback Start*, *Playback Stop*

**Turn OFF *Playback Progress*.** It fires roughly once a second and floods the timeline.

## What you'll see

- **New media added**, grouped per series, so a bulk import collapses into one stack per show.
- **Playback started / stopped**: who is watching what.

Buttons open Jellyfin, the item, or the admin dashboard.

## Score options

New-media events are grouped per series in the score's grouping rules. A bulk re-scan after reorganizing a large library collapses to a single stack per show rather than one row per episode.
