# UniFi Protect

UniFi Protect is Ubiquiti's camera system. Tempo shows its camera detections and alarms.

UniFi Protect and UniFi Network appear together under one **UniFi** source group: `com.ubiquiti.unifi` is a semantic container with no direct events of its own.

## Helper

**Not required.** Protect posts to Tempo directly from an Alarm Manager webhook action.

## Setup

In **UniFi Protect**, create an alarm and add a **Webhook** action (a **Custom Webhook**):

- **Delivery URL**: `http://<your-mac-ip>:7776/ingest/unifi/protect`
- **Method**: POST
- **Authentication**: `bearer`, with the **Token** from **Settings > Ingestion** (bound to `com.ubiquiti.unifi.protect`)
- Turn on **Use Thumbnails** if you want the camera snapshot on the event (see Thumbnails below)

The `/ingest/unifi/protect` path is handled by Tempo's built-in Protect parser, which reads Protect's alarm payload (and the attached snapshot), so there is nothing to map by hand.

Scope the alarm to the cameras and detection types you care about, so the timeline stays focused. The score raises severity for `intruder` / `alarm` detections (critical) and for person / vehicle / package / face / doorbell detections (warning); plain motion stays at info.

## What you'll see

Camera events and detections, with the camera name and a link back into Protect.

Buttons open the event in Protect, or copy the camera MAC or event ID.

## Thumbnails

Tempo can show the camera **snapshot** inline on the event. For the image to travel, turn on **Use Thumbnails** on the Protect side, so Protect attaches the snapshot (a base64 JPG) to the webhook.

Thumbnails are large (roughly 10 to 60 KB each), so two things to keep in mind:

- **Retention is managed in Tempo Settings (Database / Maintenance):** thumbnails are kept for the period you choose, then stripped from the row to keep the database small. The event itself stays.
- **High-motion cameras flood faster with thumbnails on.** A camera pointed at a busy street can post constantly, and each event now also carries an image. Scope the alarm to specific detection types and cameras.

## Known limitations

Keep UniFi on the **plain** port `:7776`. Its webhook payloads aren't compatible with Tempo's TLS port (`:8776`): the request arrives empty (a known limitation it shares with Jellyfin). UniFi events on a trusted LAN are fine over plain HTTP.
