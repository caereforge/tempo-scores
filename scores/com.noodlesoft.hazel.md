# Hazel

Hazel is a macOS file automation tool. Tempo shows a row for each file your rules act on.

## Helper

**Not required.** Hazel posts to Tempo directly from a rule's shell-script action.

## Setup

Hazel runs only on the Mac, so keep the token in the **macOS Keychain**, never inline in the rule. Store it once (from a Terminal):

```sh
security add-generic-password -s tempo-ingestion -a com.noodlesoft.hazel -w '<token>'
```

Copy `<token>` from **Settings > Ingestion**. Then, in a Hazel rule, add a **Run shell script** action (embedded) that reads the token from the Keychain at run time and posts:

```sh
curl -s -X POST http://127.0.0.1:7776/ingest \
  -H "X-Tempo-Token: $(security find-generic-password -s tempo-ingestion -a com.noodlesoft.hazel -w)" \
  -H "Content-Type: application/json" \
  --data "{\"title\":\"$1 matched <rule>\",\"providerIdentifier\":\"com.noodlesoft.hazel\",\"metadata\":{\"path\":\"$1\",\"rule\":\"<rule>\"}}"
```

Hazel passes the matched file path as `$1`. The bundled `tempo-post` tool does the same Keychain lookup for you if you prefer it to raw curl.

## Splitting into sub-sources

You can divide Hazel events into logical sub-sources in the source panel (for example **Email rules**, **Scanner rules**, **Downloads**) without writing more than one score. Post each kind with a dotted `providerIdentifier` under `com.noodlesoft.hazel`:

- `com.noodlesoft.hazel.email`
- `com.noodlesoft.hazel.scanner`
- `com.noodlesoft.hazel.<anything>`

Each one shows as its own row grouped under **Hazel** in the source panel, so you can hide, focus, or auto-dismiss them independently. The **single Hazel score still styles all of them**: it covers every `com.noodlesoft.hazel.*` sub-source, so there is nothing extra to install. The token bound to `com.noodlesoft.hazel` already accepts every sub-source too, so nothing else to set up.

**Use one level only.** The sub-source is the first segment after `com.noodlesoft.hazel` — `…hazel.scanner` is **Scanner**. Anything deeper rolls up into it: `…hazel.scanner.invoices` still shows under **Scanner**, with the deeper name kept for the action panel. That cap is deliberate — it lets you split Hazel logically without a deep or auto-generated identifier sprouting a tree of rows the panel can't manage. Make as many first-level sub-sources as you like; just don't nest past one.

## What you'll see

- A row per file a rule touched, with the rule name and destination.

Buttons open the file, the source or destination folders, or copy the path.

## Score options

Hazel lets you add a script at **any point** of a rule's execution. A script can post at the end of a rule or at intermediate steps.

When a single rule run posts several events, use the score's **grouping** so they collapse into one compact stack instead of scattering down the timeline. By default Hazel events group by **rule**, then **folder**.

## Run-bound stacking (one stack per rule run)

To collapse a multi-step rule into a single stack (start, step 1, step 2, ..., done), generate **one run id at the start of the rule** and pass it on every post. Tempo stacks every event that shares the same `runID`:

```sh
RUN="$(uuidgen | cut -c1-12)"
TOK="$(security find-generic-password -s tempo-ingestion -a com.noodlesoft.hazel -w)"
post() { tempo-post --provider com.noodlesoft.hazel --token "$TOK" --run-id "$RUN" "$@"; }

post --title "Backup started"  --run-total 5 --status "start"
post --title "Compressing"     --status "step 1"
post --title "Encrypting"      --status "step 2"
post --title "Uploading"       --status "step 3"
post --title "Backup complete" --status "done" --final
```

`--run-id` puts the id in `metadata.runID`, `--status` labels each step, `--run-total` shows progress, and `--final` marks completion. The score keys on `${metadata.runID}` first, so each rule **execution** is its own stack and a second run of the same rule starts a fresh one. Without a run id, events fall back to grouping by rule, then folder.
