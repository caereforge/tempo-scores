# Fastmail (experimental)

Fastmail is an email and calendar service. Tempo shows today's Fastmail calendar events alongside your other agenda items. Agenda events are read-only in v1.1.

## Helper

**Built in.** Tempo has its own CalDAV client, so there is nothing external to install or download. The **account** is configured from this score's **Source** tab in the Score Editor. Once the account is configured, choose *which* calendars to sync in **Settings → Agenda**.

## Setup

> **Experimental.** CalDAV is still being hardened and is aimed at technical users for now.

**1. Edit the config file.** In the Source tab, click **Open configuration** to open (or create) it:

```
~/Library/Application Support/Tempo/external-providers.json
```

Set `username` to your full Fastmail address. You can leave `calendars` empty here: once Tempo connects it **discovers your calendars from the server** and lets you check/uncheck which to sync in **Settings → Agenda → External Providers**, saving your choices back to this file automatically. The `auth.keychainItem` value is a name you choose for the Keychain entry (for example `tempo-caldav-fastmail`).

**2. Store the password in the macOS Keychain.** The password is **never** written to the config file: Tempo refuses to load a provider with an inline password. Create a Fastmail **app-specific password** (Fastmail: Settings → Privacy & Security → App passwords), then in Terminal:

```sh
security add-generic-password \
  -s tempo-caldav-fastmail \
  -a you@fastmail.com \
  -w 'your-app-specific-password'
```

The `-s` value must match `auth.keychainItem` in the config, and `-a` must match `username`.

**3. Restart Tempo.** It loads the config at launch and watches it for changes.

The full walkthrough is the **Setup README** button in the Source tab (bundled with Tempo). If something does not appear, check `Console.app` filtered by `Tempo`: the CalDAV engine logs what it loaded and any auth errors.

## What you'll see

- Today's Fastmail calendar events, with the organizer.

Buttons open the event in Fastmail, copy its link, or email the organizer.

## Known limitations

Experimental and read-only in v1.1: events are display-only; manage them in Fastmail. **Account** setup is manual (a config file plus a Keychain entry). Choosing **which calendars** to sync is done from **Settings → Agenda**, with the calendar list discovered automatically.
