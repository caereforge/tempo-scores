# Beszel

Beszel is a self-hosted server monitor. Tempo shows the alerts it raises as structured events, one row per host and metric.

## Helper required

Beszel's native notifications (Shoutrrr) flatten an alert to a title only, so Tempo cannot read the metric, threshold, or state from them. A small pull-helper (`beszel-tempo.py`) reads the hub's structured `alerts_history` over the PocketBase API and posts full events to Tempo instead. Open the helper package from this score's **Source** tab ("Open in Finder" / "Open README") and follow its README to deploy it on a host that can reach the Beszel hub. Disable Beszel's native Tempo webhook once the helper runs.

## Setup

1. Install this score from **Manage Sources**.
2. In **Settings → Ingestion**, create a token bound to `com.beszel`.
3. Deploy the helper (see its README): a read-only poller user on the hub, the token, and a flock keepalive cron.

## What you'll see

- One row per **host + metric**, stateful: a triggered alert and its later resolution update the same stack (flap cycles group within 6h).
- Severity: **Status** and **Disk** alerts are critical; other metrics (CPU, memory, temperature, and so on) are warning; a resolved alert turns it ok.
- Actions: **Open Beszel**, **Copy system name**.

## Notes

- Every configured Beszel alert type flows (Status, CPU, Memory, Disk, Temperature, LoadAvg, and so on); the helper does not filter by metric. Only alert transitions are pulled, not the live metric telemetry.
- Beszel is event-driven: the helper posts only on alert transitions (triggered and resolved), not on a heartbeat. A quiet timeline means no alert crossed a threshold, not that the helper is down.
- The helper refreshes its hub session automatically. A Beszel upgrade can reset the read-only user's read grants; re-apply them (see the helper README).
