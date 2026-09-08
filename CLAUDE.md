# Atlas 3D World Explorer

Single-file CesiumJS globe explorer. The **entire app is `index.html`** — no build step,
no dependencies, no package.json. Cesium 1.119.0 loads from cdnjs at runtime (with a
jsDelivr fallback wired to the `onerror` handler on the Cesium `<script>` tag).

## Deployment

- GitHub: `eredko1/Atlas` (branch `main`)
- Vercel project `atlas`, connected via Git integration
- **Any push to `main` auto-deploys.** There is no build command — Vercel serves the
  repo as static files.
- Live: https://atlas-kappa-henna-82.vercel.app

Because it is served statically, the root file **must** stay named `index.html`.
Renaming it breaks the site with a 404 (this is exactly what happened on the first
deploy, when the file was named `atlas-3d-explorer - 3.html`).

## Working on it

Edit `index.html` directly, then commit and push to `main` to ship.

To preview locally: open `index.html` in a browser, or `python3 -m http.server 8000`
and hit http://localhost:8000. A plain `file://` open works too, since every
dependency is a remote CDN URL.

## Cesium ion token

`ION_TOKEN_DEFAULT` near the top of the script block holds the baked-in ion access
token used for world terrain, OSM buildings, and photorealistic 3D tiles. Users can
override it at runtime via the token field in the launcher UI, which persists to
`localStorage` under `atlas_ion_token` (see `getIonToken()`).

If terrain/buildings silently fail to appear but the globe still renders, a dead or
rate-limited ion token is the first thing to check — the loaders are wrapped in
`withTimeout(..., 4000, null)` and `console.warn` on failure rather than throwing, so
the app degrades quietly instead of crashing.

## Conventions

- Keep it a single self-contained file. Don't introduce a bundler or split it into
  modules — the zero-build setup is the point.
- Vanilla JS, no framework. `$(id)` is the local `getElementById` helper.
- CSS custom properties for the palette live in `:root` at the top of the `<style>` block.
