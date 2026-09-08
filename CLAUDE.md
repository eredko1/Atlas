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

## Render quality

`QUALITY_TIERS` near the sign-in code defines four tiers (Performance → Ultra). Each
pairs a target device pixel ratio with a `maximumScreenSpaceError` for the
photorealistic tileset, since those are the two knobs that actually move image
quality and load time. The HUD's **Quality** button cycles Auto → Performance →
Balanced → High → Ultra → Auto and persists to `localStorage` under
`atlas_quality`.

The important non-obvious bit: Cesium's default `useBrowserRecommendedResolution`
pins the drawing buffer to CSS pixels and ignores `devicePixelRatio`, so on a DPR-3
phone it renders a third of the linear resolution and upscales. `applyQuality()`
turns that off and sets `resolutionScale` itself. If the app ever looks soft on a
retina screen again, check that first.

In Auto, `updateAutoQuality()` walks the ladder from measured FPS over 1.5s windows,
with a 10s warmup after launch so heavy initial tile decoding doesn't drag the tier
down. Auto is capped at High on handhelds; Ultra there is manual only.

## Conventions

- Keep it a single self-contained file. Don't introduce a bundler or split it into
  modules — the zero-build setup is the point.
- Vanilla JS, no framework. `$(id)` is the local `getElementById` helper.
- CSS custom properties for the palette live in `:root` at the top of the `<style>` block.
