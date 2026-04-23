# Tempo Scores

Public catalog of scores for [Tempo](https://tempoapp.app) — the native macOS event hub.

## What's a score

A **score** is a JSON file that teaches Tempo about a source: what color to paint its events, what severity to assign, and what actions to offer when the user clicks on one. Think of it as a musical score — the instructions Tempo follows when playing a source.

One score, dropped into `~/Library/Application Support/Tempo/Scores/`, turns a generic webhook into a first-class integration with colored badges, severity rules, and one-click actions (Open dashboard, SSH, Copy IP, Run command, etc.).

## What this repo is for

- **Browse**: see every score reviewed and published by the Tempo team
- **Contribute**: submit new scores for tools you use, via pull request
- **Audit**: the full git history of every reviewed score, public and immutable

Tempo ships with a small set of bundled scores (Kopia, UniFi, Home Assistant, Uptime Kuma, GitHub Actions, Synology, Scripts). This repo is where the catalog **grows** beyond that core — anything community-contributed and reviewed lives here.

## Two channels

### Reviewed catalog (this repo)
Scores merged into `main` have passed security + functional review by the Tempo team. They carry the "checked by Tempo team" seal and are safe to install as published.

### Local drop-in
Anyone can drop any `.tempo-score` or `.json` file into their local `Scores/` folder at their own risk. No review, no warranty — responsibility of the installer. Use this for personal / work-in-progress scores you don't want to publish.

## Installing a score

Three options:

1. **Download** a `.json` file from `scores/` and drop it into `~/Library/Application Support/Tempo/Scores/`. Tempo reloads it within a second.
2. **Download** a `.tempo-score` bundle (when available) and double-click it. Tempo opens an install sheet showing the provider, actions, and any required config.
3. **In-app browser** (Tempo V1.1+): browse this catalog from Settings and install with one click.

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md) for the review rubric and PR flow.

**Short version**: fork, add your score under `scores/`, open a PR with a sample event payload that proves it works. Reviews happen as time allows — Tempo is free, and curating the catalog is a labor of love, not a full-time job.

## Responsibility

> The functional responsibility of a custom score lies with its author. Team review is a security and sanity check, not a warranty.

This applies to every score in this repo. Review reduces risk, it doesn't eliminate it.

## License

MIT. See [LICENSE](./LICENSE). Covers the score JSON files in this repo only, not the Tempo app itself.
