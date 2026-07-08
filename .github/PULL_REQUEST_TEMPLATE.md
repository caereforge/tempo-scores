<!-- Thanks for contributing a score! Fill this in so review is fast. Full rubric: CONTRIBUTING.md -->

## What it's for
<!-- Which tool / service / integration does this score cover? -->

## Sample event payload
<!-- The JSON you'd POST to /ingest, so a reviewer can exercise the score end to end.
     Strip real tokens, network IPs and PII first (see CONTRIBUTING > Sanitize). -->

```json

```

## Metadata fields it expects
<!-- Any non-obvious ${metadata.*} fields the score reads (e.g. "expects metadata.senderAddress"). -->

## Self-check (full rubric in CONTRIBUTING.md)
- [ ] Validated with `tempo-validate scores/<file>.json`
- [ ] No `openTerminalWith` actions (those are local drop-in only, not the reviewed catalog)
- [ ] `openURL` schemes are in the allowlist (http / https / ssh / mailto / tel / documented app schemes)
- [ ] `providerIdentifier` is reverse-DNS and stable (e.g. `com.vendor.product`)
- [ ] Sample payload exercises at least one severity rule and one action with `${metadata.*}`
- [ ] Sanitized: no real tokens, network IPs, usernames, or PII
