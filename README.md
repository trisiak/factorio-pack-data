# factorio-pack-data

The shared **data plane** for two sibling forks:

- [`trisiak/factorio-blueprint-editor`](https://github.com/trisiak/factorio-blueprint-editor)
  (fbe) — consumes the per-pack `editor` tier (`data.json` + `.basis` sprite
  atlas);
- [`trisiak/factorio-item-browser`](https://github.com/trisiak/factorio-item-browser)
  (FIB) — consumes the per-pack `browser/` tier (`catalog.json` +
  `icons.webp` + `icons.json`).

Design record: the FIB fork's
[`docs/data-plane.md`](https://github.com/trisiak/factorio-item-browser/blob/master/docs/data-plane.md)
(this repo is its slice 2). The pipeline that generates everything is the fbe
fork's Rust exporter (`packages/exporter/` there — see its README).

## What is committed vs. built

**Committed** (small, diffable, reviewable): `packs/packs.json` (the pack
manifest both apps read) and each pack's JSON tiers — `data.json` and
`browser/` (catalog + icon sheet).

**Never committed**: the full-resolution `.basis` sprite atlases. They are
**build products**: the deploy workflow obtains them per pack, in order of
preference —

1. **Actions cache** (from a previous run);
2. **bootstrap** from the fbe repo's committed copy (a transition path that
   goes away once fbe evicts its `data/output` textures);
3. **regeneration** — a `workflow_dispatch` with `regenerate` set runs the
   full exporter (Factorio download + dumps + basisu atlas) using
   `FACTORIO_USERNAME` / `FACTORIO_TOKEN` repo secrets. Regeneration also
   rebuilds the JSON tiers and **fails the deploy if they drift** from the
   committed ones (the regenerated JSON is uploaded as an artifact — commit
   it, then re-run). Cold SE regeneration takes hours; the cache makes every
   later deploy fast.

Every successful run publishes the whole site (manifest + JSON tiers +
textures) to GitHub Pages:
`https://trisiak.github.io/factorio-pack-data/<pack-id>/…`

## Why this shape

- Git history stays small forever; regenerating a pack churns no binaries.
- The clonable repo never distributes game-derived textures — they are
  fetched and built at deploy time with the owner's own Factorio credentials.
  No license is claimed over any game- or mod-derived content (Factorio is ©
  Wube Software; mod assets belong to their authors). The committed JSON
  tiers are factual metadata projections; the small icon sheets are kept
  versioned for reviewability and consumer stability.
- Pack-list configuration lives here, outside both apps, and the most
  privileged credential (the Factorio token) is isolated to this repo's
  secrets — the app repos never see it.

## Updating / adding a pack

1. Edit `packs/packs.json` (see the fbe exporter README for the entry format;
   third-party mods need pinned `versions`).
2. Run the deploy workflow with `regenerate: <pack-id>` (needs the secrets).
3. If the drift check fails, download the `regen-json-<pack>` artifact,
   commit its contents under `packs/<pack-id>/`, and re-run.
