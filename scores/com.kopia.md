# Kopia Backup

Kopia is an encrypted backup tool. Tempo shows your snapshots, their sizes, and any failures.

## Helper

**Built in.** Kopia sends a raw webhook that Tempo formats. There is nothing to install or configure on the helper side; adjust presentation in the Score Editor.

## Setup

Kopia can call a webhook when a snapshot finishes. In **KopiaUI** (or `kopia notification`) add a **Webhook** profile:

- **URL**: `http://<your-mac-ip>:7776/ingest`
- **Method**: POST
- **Header**: `X-Tempo-Token: <token>` (copy it from **Settings > Ingestion**)

## What you'll see

The score raises severity by the snapshot `outcome`:

- **`ok`** (snapshot completed): the size delta shows as a headline metric, for example `+95.8 MB`.
- **`error`** (snapshot failed): surfaced as an error you can act on.
- **`warning`**: surfaced as a warning.
- **`no-change`**: an info row (nothing changed since the last snapshot).

Snapshots stack per source: the grouping key is `${metadata.repo}/${metadata.path}`, falling back to `${metadata.path}`, so repeated snapshots of the same source collapse into one stack.

The enabled buttons run a snapshot now, list snapshots, show repository status, and show maintenance info — all via the `kopia` CLI in Terminal, which works alongside the running KopiaUI server. Three more ship **disabled**: "Open KopiaUI (desktop app)", "Open Kopia server (web UI)", and "Kopia docs".

**Why "Open KopiaUI" is disabled:** since a 2026 KopiaUI update its window is a menubar-icon popover — even clicking its own Dock icon won't show it, so no external app (Tempo included) can open it. Use KopiaUI's **menubar icon** to open the window. The web-UI button needs a server URL the KopiaUI webhook doesn't send. Enable any of the three in the Score Editor's Actions tab if your setup supports them.

## Score options

In the Score Editor you can assign a **friendly name** to a backup source, so a long snapshot path becomes something recognizable at a glance ("Photos", "Trantor /etc"). Open the score's **Source** tab in the Score Editor to manage these aliases.
