# Contributing a score

Thanks for considering a contribution. This document explains how to submit a score and what the review looks for.

## The flow

1. Fork this repo.
2. Add your score under `scores/` as `<providerIdentifier>.json` (e.g. `com.example.tool.json`).
3. Validate locally with `tempo-validate scores/<yourfile>.json`. Two ways to get it:
   - **From the Tempo DMG**: starting with v1.0.5, `tempo-validate` ships alongside `Tempo.app` on the install disk image. Drag it to anywhere on your `PATH` (e.g. `/usr/local/bin/`).
   - **Standalone download**: `curl -o /usr/local/bin/tempo-validate https://tempoapp.app/utilities/tempo-validate && chmod +x /usr/local/bin/tempo-validate`. Single Python 3 file, no dependencies, schema baked in.
4. Open a pull request. In the PR body include:
   - **What it's for**: which tool / service / integration
   - **A sample event payload** (the JSON you'd POST to `/ingest`) — so the reviewer can exercise the score end-to-end
   - **Notes** on any non-obvious `metadata.*` fields the score expects
5. A reviewer will run the rubric below and either merge or leave comments with concrete criteria to address.

## Review rubric

Every score is checked against these criteria before merge. The list is public so you can self-check before submitting.

### Security

- [ ] No `openTerminalWith` actions. Terminal commands are a privilege of the end user, not of a distributed score. Local drop-in scores can use them; the reviewed catalog cannot.
- [ ] All `openURL` triggers use schemes in the public allowlist: `http`, `https`, `ssh`, `mailto`, `tel`, or app schemes documented in the PR description.
- [ ] No hidden network side effects. Actions must do what their label says, nothing more.
- [ ] Variable interpolation (`${metadata.xxx}`) only reads fields your score documents. No scraping unrelated metadata.

### Functional

- [ ] The `providerIdentifier` is namespaced and stable (reverse-DNS style: `com.vendor.product` or `scripts.<lang>.<name>`).
- [ ] `displayName` is human-readable, no version numbers, no ALL CAPS.
- [ ] `color` is a valid hex (`#RRGGBB`) and visually distinct from existing scores where possible.
- [ ] At least one action is defined, and every action has a clear `label` + a matching SF Symbol `systemIcon`.
- [ ] If `severityRules` are used, the `match` patterns are sensible and the resulting severities (`info` / `warning` / `error` / `critical`) match common user expectations for the tool.
- [ ] A sample event payload is provided that exercises at least one severity rule and one action with variable interpolation.

### Documentation

- [ ] PR description names the tool/service clearly so users searching the catalog can find it.
- [ ] Any required `metadata.*` fields are listed in the PR body (e.g. "this score expects `metadata.senderAddress` and `metadata.deviceMac`").

## Responsibility

> The functional responsibility of a custom score lies with its author. Team review is a security and sanity check, not a warranty.

If a merged score later turns out to break in some corner case, open an issue or a follow-up PR — the author is the natural maintainer of the integration they contributed.

## Review timing

Reviews happen as time allows. Tempo is free, and curating this catalog is a labor of love — not a full-time job. If you need a score urgently, use the local drop-in channel (just put the JSON in your own `Scores/` folder — it loads immediately, no review needed).

## Sanitize before submitting

A contributed score becomes public the moment the PR opens. Before pushing, check the JSON for:

- **Hardcoded tokens** in `triggerPayload` or sample payloads — strip them, leave a placeholder like `<your-token>` and document the field
- **Hostnames or IPs from your own network** in URL triggers or examples — use `your-host.local` / `192.0.2.0/24` (TEST-NET-1, RFC 5737) as documentation placeholders
- **Filesystem paths that reveal your username or directory structure** — use `~/Documents/example/` style instead
- **PII** in sample payloads — invent realistic-but-fake values

Sanitization is a security ask but also a practical one: a score is meant to be reusable by anyone, so it shouldn't carry your specific environment.

## Questions

Open a GitHub Discussion or drop a message in the Tempo Discord — those are the channels where contributor threads live and stay searchable.
