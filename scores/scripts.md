# Scripts score

Generic score for **any script** (cron, launchd, manual, ad-hoc) that can
POST to Tempo's ingestion endpoint. Drives badge color, severity, and
grouping from two top-level metadata fields the script sets: `script_name`
and `keyword`.

The score ships with a vocabulary of common keywords (`OK`, `Done`,
`Warning`, `Error`, `Critical`, `Pulito`, `Down`, `Timeout`, ...). The
vocabulary is meant as a starting point — fork the score, add your own
keywords, never edit the bundled file by hand.

## Why this design

Most scripts are simple: they run, they do a thing, they want to tell
you what happened. They don't need their own dedicated source in Tempo
(would be over-engineering) but they do benefit from a consistent
visual language ("green dot for OK, red badge for Failed").

This score gives them that, with two rules:

1. **`script_name`** identifies which script ran — drives the stack
   grouping so successive runs of the same script collapse into one row.
2. **`keyword`** is a short human label the script picks for *this run* —
   drives the badge color and severity. The vocabulary is open; the score
   provides defaults for the obvious ones.

Anything more specific (custom URL, special action button, dedicated
icon) is a sign that script deserves its own score, not a `keyword` rule
extension here.

## Setup

### 1. Issue a Tempo ingestion token

1. Open Tempo → Settings → Ingestion
2. Add a token, label it `scripts` (or per-script if you want to revoke
   one without affecting the others)
3. Copy the token

### 2. Drop the score into Tempo

Either:

- Tempo already ships this score under `providerIdentifier: "scripts"`,
  so nothing to install for the default vocabulary, **or**
- Download `scripts.json` from this repo into
  `~/Library/Application Support/Tempo/Scores/` to get the catalog
  version (extended vocabulary + grouping by `script_name`). It shadows
  the bundled one.

### 3. POST from your script

Minimum payload:

```sh
curl -sS -X POST \
  -H "X-Tempo-Token: $TEMPO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Nightly cleanup finished",
    "providerIdentifier": "scripts",
    "eventType": "alert",
    "metadata": {
      "script_name": "cron_cleanup",
      "keyword": "OK"
    }
  }' \
  http://your-mac.local:7776/ingest
```

A failure run from the same script:

```sh
curl -sS -X POST \
  -H "X-Tempo-Token: $TEMPO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Nightly cleanup failed (no space left)",
    "providerIdentifier": "scripts",
    "eventType": "alert",
    "metadata": {
      "script_name": "cron_cleanup",
      "keyword": "Failed",
      "host": "nas-01"
    }
  }' \
  http://your-mac.local:7776/ingest
```

Successive runs of the same `script_name` collapse into one stack in the
timeline (1-day window). The latest event drives the badge; expanding
the stack shows the history.

## Metadata field priority

This is the contract between your script and the Scripts score. Stick to
it and the score does the right thing without per-script configuration.

| Field           | Required | Purpose                                                                    |
| --------------- | -------- | -------------------------------------------------------------------------- |
| `script_name`   | yes      | Stack grouping key. Use snake_case. Stable across runs of the same script. |
| `keyword`       | yes      | Outcome label. Drives badge + severity via the rules table below.          |
| `host`          | no       | Hostname or IP. Enables the "SSH to source host" and "Copy host" actions.  |
| `category`      | no       | Free-form tag for your own filtering. Not rendered by the score.           |

Any other metadata you pass through is preserved on the event and
visible in the details panel — useful for context (counts, paths,
durations) but the score doesn't act on it.

### Severity priority

The runtime resolves severity in this order (first match wins):

1. **Sender severity** in the payload (top-level `severity` field) when
   it is non-`info` — your script always wins if it explicitly declares
   `error` / `warning` / `critical` / `ok`. This is the "trust the sender"
   escape hatch.
2. **`keyword` rules** below — the common path for most scripts.
3. **`script_name + keyword`** combo rules — for the rare case where the
   same keyword has different meanings across scripts (see the
   `cron_cleanup` + `Skipped` example in the bundled score).
4. **Legacy `label` rules** — kept for backward compatibility with the
   original Scripts score (scripts that set `metadata.label` instead of
   `metadata.keyword`). New scripts should use `keyword`.
5. **`severityDefault`** (`info` / "Info") — fallback when none of the
   above match.

## Default keyword vocabulary

The bundled score recognizes these keywords out of the box:

| Keyword    | Severity   | Badge      |
| ---------- | ---------- | ---------- |
| `OK`       | `ok`       | OK         |
| `Done`     | `ok`       | Done       |
| `Success`  | `ok`       | Success    |
| `Pulito`   | `ok`       | Pulito     |
| `Warning`  | `warning`  | Warning    |
| `Slow`     | `warning`  | Slow       |
| `Retried`  | `warning`  | Retried    |
| `Error`    | `error`    | Error      |
| `Failed`   | `error`    | Failed     |
| `Timeout`  | `error`    | Timeout    |
| `Critical` | `critical` | Critical   |
| `Down`     | `critical` | Down       |
| _(other)_  | `info`     | Info       |

Any keyword not in the table falls through to `info` / Info. Extend the
vocabulary by editing your local copy of `scripts.json` and adding rules
in the same shape:

```json
{ "match": { "keyword": "Quarantined" }, "severity": "warning", "label": "Quarantined" }
```

Reload time is sub-second — Tempo watches the Scores folder and re-applies.

### Combining `script_name` + `keyword`

When the same keyword has different meanings depending on which script
produced it, add a more specific rule. The matcher requires **all** keys
in `match` to be present and equal, so combo rules win over single-key
rules naturally:

```json
{ "match": { "script_name": "cron_cleanup", "keyword": "Skipped" },
  "severity": "info", "label": "Skipped", "color": "#6E7681" }
```

This is the same `keyword: "Skipped"` that for another script might
mean something more interesting — but for `cron_cleanup` it just means
"nothing to do this run", so the rule paints it muted gray.

### Wildcards

String rule values support shell-style globs:

```json
{ "match": { "keyword": "Failed*" },  "severity": "error", "label": "Failed" }
```

Matches `Failed`, `Failed (DB)`, `Failed: timeout` — useful when the
script appends context to the keyword.

## Default actions

| Action            | Trigger                                          |
| ----------------- | ------------------------------------------------ |
| SSH to source host | `ssh://${metadata.host}` (disabled if `host` empty) |
| Copy host         | `${metadata.host}` to clipboard                  |
| Copy script name  | `${metadata.script_name}` to clipboard           |
| Copy title        | the event's title to clipboard                   |

Add your own per-script actions by forking the score and appending to
`defaultActions`. Reviewed-catalog actions stay observation-only
(`openURL` + `copyToClipboard`); destructive actions
(`openTerminalWith`) belong in your local drop-in.

## Sample event payloads

A daily backup script reporting success:

```json
{
  "title": "Kopia snapshot finished — 4.2 GB in 38s",
  "providerIdentifier": "scripts",
  "eventType": "alert",
  "metadata": {
    "script_name": "kopia_nightly",
    "keyword": "OK",
    "host": "nas-01",
    "snapshot_id": "k1234abcd"
  }
}
```

A monitoring loop that detected a high-latency condition:

```json
{
  "title": "DNS resolution > 800ms on resolver pool",
  "providerIdentifier": "scripts",
  "eventType": "alert",
  "metadata": {
    "script_name": "dns_watchdog",
    "keyword": "Slow",
    "host": "ops-mac.lan",
    "p99_ms": 812
  }
}
```

A cleanup script with a quiet "nothing to do" outcome (combo rule):

```json
{
  "title": "Cleanup: no stale files found",
  "providerIdentifier": "scripts",
  "eventType": "alert",
  "metadata": {
    "script_name": "cron_cleanup",
    "keyword": "Skipped"
  }
}
```

## Notes

- **Naming**: `script_name` should be stable across runs. Renaming a
  script breaks the stack continuity. snake_case is the convention, but
  the score doesn't enforce it.
- **Title vs keyword**: the title is the human sentence ("Backup finished
  at 04:12 in 38s"); the keyword is the *outcome class* ("OK"). Don't
  put the whole sentence in the keyword.
- **Stack window**: 1 day. Runs older than 24h start a new stack, even
  for the same `script_name`. Adjust `groupingWindow` in your local copy
  if you want longer continuity (e.g., `"7d"` for weekly scripts).
- **No `host`**: actions that depend on `host` show up disabled, with a
  tooltip pointing at the missing field. That's intentional — the score
  doesn't disappear; the user sees *why* the button is inactive.
- **Bundled vs catalog**: Tempo ships a smaller bundled `scripts.json`
  with just the `label`-based vocabulary, for backward compat. This
  catalog version supersedes it when dropped into the user's Scores
  folder.
