# Beta scores

Experimental scores not yet promoted to the bundled set or the public catalog
on `tempoapp.app/scores/`. They work, but with caveats — limited provider
coverage, no in-app configuration UI, manual activation through the
filesystem and (in some cases) the macOS Keychain.

Use these at your own pace. Feedback in
[Discord](https://discord.gg/caereforge) shapes whether and when each lands
as a first-class source.

## Current entries

### `com.caldav.fastmail.json` — Fastmail CalDAV (beta)

Tempo 1.0.3 includes a native CalDAV engine that pulls calendar events
directly from a CalDAV server. It is currently scoped to **Fastmail only**
— other CalDAV providers (iCloud, Google Calendar, Nextcloud, generic
CalDAV servers) are not supported in this beta. Promotion to a first-class
source with an in-app Settings UI is planned for a later release.

The engine has **no Settings UI in V1**. Configuration is done by hand,
through one JSON file and one Keychain item. The full walkthrough is below.

---

#### 1. Generate a Fastmail app-specific password

In your Fastmail web account, go to **Settings → Password & Security →
Manage App Passwords → New App Password**. Name it "Tempo CalDAV". Scope
it to **Calendars (CalDAV)** only. Copy the 16-character password —
Fastmail shows it once.

#### 2. Store the password in the macOS Keychain

Open Terminal and run (replace the email and password with your own):

```bash
security add-generic-password \
  -s "tempo-caldav-fastmail" \
  -a "your.email@fastmail.com" \
  -w "<16-char-app-password>"
```

Verify it landed:

```bash
security find-generic-password -s "tempo-caldav-fastmail"
```

The `-s` value (`tempo-caldav-fastmail`) is the keychain item name Tempo
will look up in step 4. You can pick a different name, as long as the JSON
in step 4 matches.

#### 3. Install the score file

The score file styles incoming Fastmail events in the Tempo timeline
(colour, default severity, three action buttons: Open in Fastmail, Copy
event link, Email organizer). Without it, events still appear, but as
unstyled cards under a generic provider row.

Download the installable wrapper from
<https://tempoapp.app/scores/beta/com.caldav.fastmail.tempo-score> and
either:

- **Double-click** the file. Tempo opens a review sheet — click
  **Install**. The score lands in
  `~/Library/Application Support/Tempo/Scores/`.

- **Or install manually** if the double-click handler isn't picking it up:

  ```bash
  mkdir -p "$HOME/Library/Application Support/Tempo/Scores"
  cp ~/Downloads/com.caldav.fastmail.tempo-score \
     "$HOME/Library/Application Support/Tempo/Scores/com.caldav.fastmail.json"
  ```

  The file watcher requires the `.json` extension to load the score.

#### 4. Create the engine config file

This is the file the CalDAV engine reads at launch to know what to pull.

```bash
mkdir -p "$HOME/Library/Application Support/Tempo"
cat > "$HOME/Library/Application Support/Tempo/external-providers.json" <<'EOF'
{
  "version": 1,
  "providers": [
    {
      "type": "caldav-fastmail",
      "username": "your.email@fastmail.com",
      "auth": { "type": "basic", "keychainItem": "tempo-caldav-fastmail" },
      "calendars": []
    }
  ]
}
EOF
```

Substitute your Fastmail email in `username`. `"calendars": []` syncs
every calendar visible to the account; to restrict, list display names
(case-insensitive): `["Personal", "Work"]`.

The engine will **reject** the entry if it sees an inlined `password` or
`token` field in `auth` — credentials must live in the Keychain, not in
plaintext on disk.

#### 5. Restart Tempo and verify

The config is read at launch. Quit Tempo (⌘Q) and relaunch. Within ~30
seconds you should see:

- A **CalDAV** umbrella row in the source panel
- **Fastmail** as a sub-source under it
- Today's events in the agenda timeline alongside Apple Calendar

If something looks off, open **Console.app** and filter on
`subsystem:app.tempoapp.Tempo category:CalDAV`. Every poll cycle logs
there with discovery, sync, and delta counts.

---

#### Caveat — duplicates with macOS Internet Accounts

If your Fastmail account is also configured under **System Settings →
Internet Accounts** (the macOS-level CalDAV), Calendar.app reads from
Fastmail and Tempo's EventKit provider sees those events under
`com.apple.calendar`. Tempo's CalDAV engine reads the **same** events
directly. The result is duplicate rows: once tagged as Apple Calendar,
once as Fastmail.

While the beta is in this shape, the cleanest path is to remove Fastmail
from Internet Accounts and let Tempo's engine be the only path. If you
want both (for Apple Mail and Calendar.app integration), a future Settings
UI will let you mark one source as authoritative.

#### Operational tips

- **Poll interval** defaults to 10 minutes. To bump it, add
  `"pollIntervalSeconds": 300` to the provider entry.
- **Sync horizon** defaults to 30 days forward and 7 days back. Tweak via
  `"syncHorizonDaysForward"` and `"syncHorizonDaysBackward"`.
- **Multiple accounts** (e.g., personal + work Fastmail): two provider
  entries in the `providers` array, each with its own
  `providerIdentifier` override. Not pretty without UI — a future release
  will polish this.
- **Bell icon** appears on any event with alarms (Apple Calendar +
  Fastmail). Tempo does not fire those alarms — the OS still does — the
  icon just surfaces that reminders are set on the event.

#### Scope of the beta

Read-only Fastmail CalDAV. Write-back, multi-provider Settings UI, OAuth
(Google, Outlook), iCloud and Nextcloud adapters are not in scope for
this beta. Bugs and rough edges welcome in
[Discord](https://discord.gg/caereforge).
