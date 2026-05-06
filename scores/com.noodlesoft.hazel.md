# Hazel score

Surface Hazel rule firings in Tempo's timeline with five default actions
(Open file, Open destination folder, Open source folder, Copy file path,
Copy rule name).

Hazel has **no native outbound webhook**. Instead, every Hazel rule can
**Run shell script** as one of its actions — and that's where the
integration lives. You add a small `curl` block to the rule (or a single
shared script all rules call) and Tempo gets a row whenever the rule
fires.

The integration intentionally keeps the rule logic on Hazel's side.
Tempo doesn't poll the watched folder, doesn't read Hazel's internal
state, and never sees a file that didn't already match one of your
rules. You're emitting an after-the-fact "rule fired" record.

Tested with Hazel 6.x on macOS 14+.

## Setup

### 1. Issue a Tempo ingestion token

1. Open Tempo → Settings → Ingestion
2. Add a token, label it `hazel`
3. Copy the token

### 2. Drop the score into Tempo

Either:
- Download `com.noodlesoft.hazel.json` from this repo into
  `~/Library/Application Support/Tempo/Scores/` (Tempo reloads within a second), **or**
- Wait for the V1.1 in-app catalog browser

### 3. Add the "Run shell script" action to a rule

Open Hazel → select the folder → select the rule → **Do the following**
section → click **+** → **Run shell script** → **Edit script…**

In the script editor, set **Process** to `Embedded script` and paste
this template. The two lines marked with comments are the values you
adjust for your setup — nothing else needs to change.

```sh
#!/bin/bash
# Tempo notifier for Hazel rules — POST a "rule fired" event
TEMPO_URL="http://your-mac.local:7776/ingest"   # CHANGE ME if Tempo runs elsewhere
TEMPO_TOKEN="paste-tempo-token-here"             # CHANGE ME (token from step 1)

# Hazel passes the matched file as $1 and exposes context as env vars
FILE_PATH="$1"
RULE_NAME="${HAZEL_RULE_NAME:-unknown rule}"
FOLDER_PATH="${HAZEL_FOLDER_PATH:-$(dirname "$FILE_PATH")}"
DEST_PATH="${2:-}"   # optional — see "Capturing the destination" below

json_escape() { python3 -c 'import json,sys; print(json.dumps(sys.stdin.read()))'; }

curl -sS -X POST "$TEMPO_URL" \
    -H "X-Tempo-Token: ${TEMPO_TOKEN}" \
    -H "Content-Type: application/json" \
    -d "$(cat <<EOF
{
  "providerIdentifier": "com.noodlesoft.hazel",
  "title": $(printf %s "$RULE_NAME" | json_escape),
  "startDate": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
  "eventType": "alert",
  "metadata": {
    "rule":   $(printf %s "$RULE_NAME"   | json_escape),
    "folder": $(printf %s "$FOLDER_PATH" | json_escape),
    "path":   $(printf %s "$FILE_PATH"   | json_escape),
    "dest":   $(printf %s "$DEST_PATH"   | json_escape)
  }
}
EOF
)" >/dev/null
```

> **Order matters in the rule.** Hazel runs the rule's actions
> top-to-bottom. If you `Move` a file *before* this script, then `$1`
> still resolves to the file's *original* path (Hazel passes the path as
> matched, not the post-move path). To capture both, see "Capturing the
> destination" below.

### 4. Test the rule

In Hazel, click **Preview** on the rule, or simply drop a matching file
into the watched folder. Tempo's live feed should show a new row within
a second or two with the rule name as the title.

If nothing lands, run the troubleshooting checks below.

## Capturing the destination

The default script leaves `dest` empty. If your rule includes a `Move`
or `Copy` action and you want Tempo to know where the file landed, do
one of these:

- **Hardcode it for that rule** — change `DEST_PATH=""` to the
  destination folder you wired into the Move/Copy action. Static, but
  works.
- **Pass it through `$2`** — Hazel only passes the matched file as `$1`,
  but you can chain a second "Run shell script" *after* the Move that
  re-invokes this script with the post-move path:

  ```sh
  /path/to/your-shared-tempo-notifier.sh "$NEW_PATH" "$DESTINATION_FOLDER"
  ```

  Useful if you have many rules and want a single shared notifier
  script rather than duplicating the curl block in every rule.
- **Skip it** — leave `dest` blank. The "Open destination folder"
  action becomes a no-op, but the rest of the score still works.

## Severity

Hazel rule firings are routine bookkeeping events, not alerts. The
score has no severity rules and uses `info` ("Rule fired") as the
default for everything. Two reasons:

- A rule firing is a *successful* match by definition. There is no
  "rule failed" notion — the rule either matched or it didn't.
- If a specific rule's firings *should* be loud (e.g. "quarantine
  suspicious downloads"), set its severity per-rule by adding a
  `severity` field to the metadata block in the script for that rule.
  Tempo's `severityRules` will pick it up via a custom drop-in score
  variant.

## Required `metadata` fields

- **`rule`** — name of the Hazel rule that fired. Required: drives
  grouping and the title.
- **`folder`** — path of the folder Hazel was watching when the rule
  fired. Used by "Open source folder" and as the grouping fallback.
- **`path`** — full path of the file the rule processed. Used by "Open
  file" and "Copy file path".
- **`dest`** — optional destination folder for Move/Copy rules. Used
  by "Open destination folder"; the action becomes a no-op when empty.

## Stack collapsing (grouping)

The score groups by `rule` first (so all firings of "Sort PDFs into
Receipts" collapse into one stack regardless of which file each one
moved), and falls back to `folder` for cases where `rule` is somehow
empty. The window is one day — after that, a new stack starts. This
matches the typical Hazel pattern: one rule fires many times across a
day as files arrive, and you want a single line in the timeline rather
than 47 nearly-identical rows.

If you have a rule that *should* show every firing as a separate row
(e.g. one that quarantines suspicious files), remove the `grouping`
lines from the score after install — or duplicate the score with a
different `providerIdentifier` for that rule's events.

## Troubleshooting

```sh
# 1. Reachability — Mac running Hazel can reach Tempo on 7776
nc -vz your-mac.local 7776

# 2. Manual POST that mimics the rule's emit
curl -sv -X POST \
  -H "X-Tempo-Token: YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"providerIdentifier":"com.noodlesoft.hazel","title":"Sort PDFs","startDate":"2026-04-29T10:00:00Z","eventType":"alert","metadata":{"rule":"Sort PDFs","folder":"/Users/me/Downloads","path":"/Users/me/Downloads/receipt.pdf","dest":"/Users/me/Documents/Receipts"}}' \
  http://your-mac.local:7776/ingest

# 3. Watch Tempo's ingestion log
log stream --predicate 'subsystem == "app.tempo"' --level debug | grep -i hazel

# 4. Watch Hazel's own log to confirm the rule actually fired
#    Hazel → Preferences → Logging → set to "All", reproduce, view via Console.app
log stream --predicate 'subsystem == "com.noodlesoft.hazel"' --level info

# 5. Run the shell script directly outside Hazel to isolate the curl from the rule logic
HAZEL_RULE_NAME="probe rule" \
HAZEL_FOLDER_PATH="/tmp" \
/path/to/your-script.sh /tmp/probe.txt
```

## Sample event payload

```json
{
  "providerIdentifier": "com.noodlesoft.hazel",
  "title": "Sort PDFs into Receipts",
  "startDate": "2026-04-29T10:00:00Z",
  "eventType": "alert",
  "metadata": {
    "rule":   "Sort PDFs into Receipts",
    "folder": "/Users/me/Downloads",
    "path":   "/Users/me/Downloads/2026-04-29-invoice.pdf",
    "dest":   "/Users/me/Documents/Receipts/2026"
  }
}
```

## Notes

- **One shared script vs per-rule scripts** — for more than 2-3 Hazel
  rules emitting to Tempo, save the script once at e.g.
  `~/.local/bin/hazel-tempo.sh`, mark it executable, and have each rule
  call it with `~/.local/bin/hazel-tempo.sh "$1"`. Single source of
  truth for token and URL.
- **Hazel env vars** — `HAZEL_RULE_NAME` and `HAZEL_FOLDER_PATH` are
  exposed by Hazel 4+. Older versions may not set them; the script
  falls back to `dirname "$FILE_PATH"` and `"unknown rule"` so it
  doesn't crash.
- **No mutating actions in the catalog** — the reviewed-catalog score
  uses only `openURL` and `copyToClipboard`. Triggering a rerun, moving
  the file again, or quarantining via SSH are sensitive actions; if
  you want them, drop in a local score variant with `openTerminalWith`
  triggers.
- **`file://` URLs and binary files** — `Open file` opens the file in
  its default app via the `file://` scheme. For binary files Finder
  will be the fallback if no app claims the type.
