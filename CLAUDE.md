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

## Street & rail network

`osmNet` pulls road and rail centrelines around the player from the **Overpass
API** (OpenStreetMap, ODbL) and indexes them in a local metre frame with a grid
so nearest-segment lookups are cheap per frame. It is best-effort: two failed
rounds set `osmNet.disabled` and the sim falls back to free driving plus the
built-in `SUBWAY_LINES` station chain. Aircraft never trigger a fetch — a jet
outruns the fetch radius and would just get the app rate-limited.

Two separate things stop a car driving up a building, and they are independent
on purpose:

- **`drivePathBlocked()`** probes the surface ~2 m *ahead* of the car; a rise
  steeper than ~29° is a wall, so the car stops against it. Needs no network.
  It must stay a *lookahead* — an earlier version sampled under the car's own
  wheels and could not tell "wall ahead" from "tiles finished loading and the
  street is higher than I thought", which pinned the car in place forever.
- **`applyRoadAssist()`** pulls the car back toward the nearest OSM centreline
  once it is past the kerb. The HUD **Roads** button cycles Free / Assist /
  Locked (persisted as `atlas_roadmode`).

## Elevated track

The photoreal tileset *contains* the viaduct, so on an elevated stretch
`scene.sampleHeight` already returns the deck. `railDeckHeight()` therefore
puts the train at `surface + 0.7`, not `ground + h` — the latter is what used
to fly it ten metres over the Coney Island el. In a tunnel the same sample is
the street overhead, so it subtracts instead. The result is rate-limited so a
sample that hits a passing building cannot teleport the train.

`railRider` follows real OSM way geometry when available and hands off at way
ends to the best-aligned continuation (which is how it takes the through route
rather than a siding); it falls back to the `SUBWAY_LINES` rider otherwise.
Trailing cars ride `consistTrail`, a breadcrumb of where the lead car has been,
so they hold curves and way joints without re-walking the route backwards.

## Vehicle models

Road cars are Khronos' **Car Concept** (CC-BY-4.0) from jsDelivr, in the
Draco+KTX2 variant (3.5 MB vs 11.8 MB for the plain GLB). One download serves
all three road cars — Cesium tints the model at draw time. Measured facts baked
into `VEHICLE_MODELS`: 4.357 m long, and the nose lands on `heading - 90°`, the
same convention the box chassis uses.

`Model.readyEvent` only fires from `Model.update()`, which the scene only calls
for primitives it owns — so the model must be **added to `scene.primitives`
before** awaiting ready, or it hangs forever. It is added hidden. Any load
failure falls back to the box chassis, which is also what every vehicle without
a model entry keeps using.

## Deep links

The address bar **is** the share format. `syncDeepLink()` rewrites
`location.hash` (throttled, `replaceState`, only after 8 m of movement) so the
URL is always a link that drops someone else in the same spot. `parseDeepLink()`
/ `applyDeepLink()` read it back on load; `go=1` skips the launcher and starts
the engine straight after sign-in (wired into `unlockApplication`). The **Copy
share link** button in the pause menu adds `go=1` for you.

Params: `at=lat,lon`, `v=<vehicleId>`, `hdg=<deg>`, `n=<name>`, `go=1`.

## Recover

`recoverVehicle()` (HUD button, `B`) is the escape hatch from the climb guard,
which can otherwise wedge you in a corner with no way out. It puts a car or a
walker on the nearest OSM centreline, re-snaps a train to the rails, and levels
an aircraft off at 300 m AGL. With no OSM data it reverses 12 m along the way
you came in.

## Search anywhere

`geocodeSearch()` tries Cesium ion's geocoder first (same token as the tiles,
via `api.cesium.com/v1/geocode/search`, no `scene` needed), falls back to
Nominatim, and short-circuits when the query parses as `lat, lon`. Search boxes
live in the launcher and the pause menu; both feed `warpToPlace()`, which is
the generalised form of the old `warpToCity()` and forces an OSM refetch since
the previous city's street data is meaningless at the new one.

## On foot

`walk` is a vehicle with `kind: 'foot'`, stepped by `stepWalk()` — near-instant
acceleration, Shift to run, and a slight sine bob. `drivePathBlocked()` uses a
lower rise limit and a shorter lookahead for it, so a pedestrian can step up a
kerb but still cannot walk through a wall.

`exitVehicle()` records `parked = {vehId, lat, lon, heading, ground}` and steps
you out sideways. The key detail: **the glTF model stays with the parked car,
not with the player** — `updateVehicleModelPose()` poses it from `parked` while
on foot, and `applyVehicleVisibility()` keeps it shown. That is what makes
walking away and looking back at your own car work, and it is why the walker
needs no avatar (first person by default; foot has its own `FOOT_CAM_VIEWS`, so
read the camera through `activeCam()` / `camViews()`, never `CAM_VIEWS[curCamIdx]`).
`enterVehicle()` only works within `FOOT_REACH` (9 m); picking a vehicle from
the dropdown clears `parked` rather than leaving a phantom waypoint.

## Conventions

- Keep it a single self-contained file. Don't introduce a bundler or split it into
  modules — the zero-build setup is the point.
- Vanilla JS, no framework. `$(id)` is the local `getElementById` helper.
- CSS custom properties for the palette live in `:root` at the top of the `<style>` block.
