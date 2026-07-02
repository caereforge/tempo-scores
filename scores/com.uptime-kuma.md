# Uptime Kuma

Uptime Kuma is a self-hosted uptime monitor. Tempo shows the up/down status of every monitor you enable.

## Helper

**Not required.** Uptime Kuma posts to Tempo directly through its webhook notification.

## Setup

In Uptime Kuma go to **Settings > Notifications > Setup Notification** and choose **Webhook**:

- **Post URL**: `http://<your-mac-ip>:7776/ingest`
- **Additional Headers**: `{ "X-Tempo-Token": "<token>" }` (from **Settings > Ingestion**)

Then enable the notification on each monitor you care about.

## What you'll see

- One row per monitor (stateful), flipping between **up** and **down**.
- Repeated pings collapse into the same row instead of spamming the feed.

Buttons open the monitor URL, curl-probe it, or ping/traceroute the host.
