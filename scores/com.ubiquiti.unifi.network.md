# UniFi Network

UniFi Network is Ubiquiti's network controller. Tempo shows its client and device events.

UniFi Network and UniFi Protect appear together under one **UniFi** source group: `com.ubiquiti.unifi` is a semantic container with no direct events of its own.

## Helper

**Not required.** The controller posts to Tempo directly through alert webhooks.

## Setup

In the UniFi controller, send alert webhooks to:

- **URL**: `http://<your-mac-ip>:7776/ingest`
- **Header**: `X-Tempo-Token: <token>` (from **Settings > Ingestion**)
- Set `providerIdentifier` to `com.ubiquiti.unifi.network`

Client connect and disconnect can fire constantly on a busy network. If it gets noisy, scope the webhook to device and uplink events, or to a few specific clients, rather than every client on the LAN.

## What you'll see

Client connect and disconnect events, and device events, with the name and MAC. The score raises severity for device-down, disconnect and offline alarms (error) and for threat / intrusion / honeypot alarms (critical).

Buttons open the controller, the client, or SSH to the controller.

## Known limitations

Keep UniFi on the **plain** port `:7776`. Its webhook payloads aren't compatible with Tempo's TLS port (`:8776`): the request arrives empty (a known limitation it shares with Jellyfin). UniFi events on a trusted LAN are fine over plain HTTP.

UniFi's webhook payloads vary by controller version. If events are rejected, check the audit (the shield icon) to see the raw payload and adjust the score's field mapping.
