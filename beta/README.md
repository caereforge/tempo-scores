# Beta scores

Experimental scores not yet promoted to the bundled set or the public catalog
on `tempoapp.app/scores/`. They work, but with caveats — limited provider
coverage, no in-app configuration UI, manual activation.

Use these at your own pace. Feedback in
[Discord](https://discord.gg/caereforge) shapes whether and when each lands
as a first-class source.

## Current files

### `com.caldav.fastmail.json` — Fastmail CalDAV (beta)

Tempo's CalDAV engine ships in V1.0.3 but is currently scoped to **Fastmail
only**. The score in this directory styles incoming Fastmail CalDAV events
(colour, default severity, action buttons) in the timeline.

To activate:

1. Download the installable wrapper from
   <https://tempoapp.app/scores/beta/com.caldav.fastmail.tempo-score>
2. Double-click. Tempo opens a review sheet — click **Install**. The score
   lands in `~/Library/Application Support/Tempo/Scores/`.
3. Configure your Fastmail CalDAV credentials in Tempo Settings (the engine
   uses them to pull events directly from the server — no ingestion token
   needed, CalDAV is pull-based not push-based).

Other CalDAV providers (iCloud, Google, Nextcloud, generic CalDAV) are not
supported in this beta. Promotion of CalDAV to a first-class source with a
public Settings UI is planned for a later release.
