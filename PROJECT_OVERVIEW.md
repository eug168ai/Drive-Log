# Drive Log — Project Overview

_Last updated: 2026-08-23 (build `2026-08-22.37`). Written for a coding-assistant handoff — treat this as a snapshot of the actual shipped code, not aspirational documentation._

## 1. High-Level Overview

Drive Log is a mobile-first Progressive Web App for tracking driving — built specifically around the **Australian learner-driver logbook** requirement (a fixed number of supervised hours, with a night-hours sub-requirement, before a provisional licence test), while also working as a general personal/business mileage and fuel-cost tracker for anyone.

Core things it does:
- Times and logs individual drives ("Quick Drive": tap Start, tap End) and multi-stop planned routes (Trip Planner, with real GPS turn-by-turn navigation).
- Tags every drive as **Personal**, **Business**, or **Supervised**, and for Supervised drives captures the conditions a logbook cares about (day/night, road type, weather, traffic, supervisor name + licence number).
- Tracks distance, time, and estimated/actual fuel cost.
- Generates a real AU-style paper-logbook-equivalent PDF, plus a general CSV/PDF report export.
- Optionally syncs trips to a personal Google Sheet and backs up route images to Google Drive, entirely opt-in and user-triggered (no server backend of its own).
- Installs as a home-screen PWA (offline shell, app icon, standalone display).

There is **no backend server**. Everything runs client-side in a single HTML file; "sync" means calling Google's own APIs directly from the browser with a user-authorized OAuth token. All data lives on-device (localStorage + IndexedDB) unless the user explicitly connects and syncs to Google.

## 2. Current Architecture

**Pattern: single-file, framework-free, full-DOM-replacement UI.**

- One global mutable `state` object (`index.html:1470`) holds the entire application's UI and data state.
- A `render()` function (`index.html:4714`) calls `renderApp()` (or `renderLogin()` pre-auth / `renderErrorFallback()` on a caught crash), which returns one big HTML string for the whole app body. That string is assigned via `app.innerHTML = ...`.
- Immediately after, `attachAppHandlers()` (a large event-delegation function spanning roughly `index.html:8289`–`9930`) re-queries the fresh DOM and re-binds every button/input/select's event listener from scratch, because the previous DOM (and its listeners) was just destroyed.
- There is no virtual DOM, no diffing, no component tree, no reactivity system. Any state mutation that should be visible is followed by an explicit `render()` call. This is simple and predictable but means **every interaction re-renders the entire app** (mitigated by the app being simple enough that this hasn't caused a real performance problem, and by a couple of hand-rolled fast paths — see `renderTimerOnly()` for the once-a-second driving-timer tick, which intentionally does NOT do a full render).
- Because a full `innerHTML` replace destroys any live library objects living in that DOM (e.g. a Google `Map` instance, a `DirectionsResult`), those objects are deliberately kept in bare module-level `let` variables **outside** `state` rather than serialized into it — see `plannerDirectionsResult` (`index.html:2071`) and the live-drive tracking object `liveDrive` (`index.html:~3041`). Only plain-data snapshots of these are ever persisted.

**Persistence is piecemeal, not one big store.** A small abstraction (`storageGet`/`storageSet`/`storageDelete`, `index.html:4675`–`4696`) prefers a `window.storage` API when the app happens to be running inside a Claude/Cowork host environment, and falls back to plain `localStorage` otherwise. Each concern (trips, settings, saved addresses, fuel fill-ups, etc.) is saved under its own key, independently, at the point where it changes — there is no central "save the whole state" call.

**Two storage tiers, split by size, not by meaning:**
- `localStorage` (via the abstraction above) holds everything structured: trips, settings, profile, supervisors, saved addresses, fuel fill-ups, planned routes, recent locations, the live-drive resume snapshot, and the cached Google OAuth token.
- **IndexedDB** (a separate, larger-quota store, DB name `driving-log-route-images`) holds one full-resolution route-map image per trip, fetched lazily only when the Edit Trip modal is actually opened. Every trip's small thumbnail still lives inline in the `localStorage` trips blob; only the "zoom in" detail image is offloaded to IndexedDB, specifically because IndexedDB's quota is far more generous than localStorage's and these images would otherwise risk blowing the localStorage budget.

## 3. Folder Structure

```
Drive-Log/
├── index.html          # The entire application: markup + <style> + <script>. ~9,970 lines.
├── manifest.json        # PWA manifest (name, icons, standalone display, theme color).
├── sw.js                 # Service worker: network-first fetch with cache fallback.
├── icons/
│   ├── icon-192.png / icon-512.png                  # Standard app icons.
│   ├── icon-192-monochrome.png / icon-512-monochrome.png  # "Maskable/monochrome" purpose icons.
│   ├── apple-touch-icon.png
│   └── favicon-32.png
└── .github/
    └── workflows/
        └── deploy-pages.yml   # Deploys to GitHub Pages on push to main or staging.
```

No `package.json`, no `node_modules`, no build step, no bundler, no test directory. What you see in `index.html` is exactly what ships.

### `index.html` internal layout (line ranges are approximate but current as of this doc)

| Lines | Contents |
|---|---|
| 1–15 | `<head>`, meta tags, manifest/icon links, eager `<script>` load of Google Identity Services |
| 16–665 | `<style>` — CSS custom properties (theme colors, light/dark), all component styles |
| 671–700 | `APP_BUILD`, all storage-key constants, `GOOGLE_CLIENT_ID`, `GOOGLE_MAPS_API_KEY`, `MAP_ID` |
| 702–825 | Lazy script loaders (Maps, jsPDF), region/locale/unit detection helpers |
| 826–963 | Currency formatting, geometry/astronomy helpers (haversine distance, sunrise/sunset) |
| 952–1213 | Google Maps Timeline JSON import/parsing helpers |
| 1117–1213 | Places Autocomplete wiring |
| 1213–1470 | Google OAuth token client, Sheets/Drive API calls |
| **1470–1559** | **The `state` object** |
| 1561–1614 | `icons` — inline SVG string constants |
| 1615–1751 | `CAR_FUEL_DATA` — hardcoded UK/AU vehicle fuel-consumption table |
| 1751–1815 | Region detection, MPG↔L/100km conversion, fueleconomy.gov API wrappers |
| 1795–1965 | Date/time helpers, geolocation, reverse geocoding, saved-address matching |
| 1966–2063 | Distance/duration estimation, Quick Drive end-of-trip flow |
| **2064–2660** | **Trip Planner** logic: stops/legs, saved routes, saved addresses, planner "Log Drive" |
| **2661–4670** | **Live GPS driving/navigation**: turn-by-turn logic, off-route detection/reroute, vector-map camera, ETA bar, breadcrumb path, resume-after-refresh |
| 4673–4713 | Storage abstraction, `THEME_COLORS`, `applyTheme()` |
| **4714–4956** | Core render dispatch, login screen, app boot/hydration (`loadData()`) |
| **5025–5420** | Log-screen dashboard: chart primitives, Supervised Driving card, Driving Metrics card |
| **5587–6125** | Trip Planner tab rendering (field blocks, leg rows, saved routes, Quick Drive, ETA bar, planner modals) |
| **6126–6584** | `renderApp()` shell/nav, `renderHistoryTab()` (Log tab), `renderSettingsTab()` |
| **6601–7419** | Remaining modal render functions (21 total — see §5) |
| **7419–7906** | Official AU Log Book PDF generator (jsPDF) |
| **7906–7957** | IndexedDB route-image storage |
| 7959–8125 | Detailed route-image generation, Google Drive image upload |
| 8125–8289 | DOM utilities (long-press, marquee auto-scroll for overflowing text) |
| **8289–9930** | `attachAppHandlers()` — every event listener in the app, wired by delegation |
| 9930–9966 | Boot IIFE (load profile, PIN gate, first render), service worker registration |

## 4. Technologies, Frameworks, Libraries, APIs

**Runtime**: vanilla JavaScript (ES2017+ features: async/await, template literals, optional chaining), vanilla CSS (custom properties for theming), no framework of any kind (no React/Vue/etc.), no TypeScript, no JSX.

**Third-party libraries** (both loaded lazily, from CDN, only when actually needed — never bundled):
- **jsPDF** (`v2.5.1`, from `cdnjs.cloudflare.com`) — generates both the general Report export and the Official Log Book PDF.
- (Google Maps' own JS SDK, see below, is the only other external script, and it's also lazy-loaded.)

**Google APIs/services in use:**

| Service | Used for | Auth |
|---|---|---|
| **Google Maps JavaScript API** (Places, Geocoding, Directions, Distance Matrix, vector-rendering `Map` via a Map ID) | Address autocomplete, forward/reverse geocoding, route distance/duration estimates, live turn-by-turn directions, the in-app map itself | API key, hardcoded in source (see §7) |
| **Google Maps Static Maps API** | Small thumbnail + larger detail route images saved with each trip | Same API key |
| **Google Identity Services** (`accounts.google.com/gsi/client`) | OAuth2 token client for Sheets/Drive access | `GOOGLE_CLIENT_ID`, hardcoded in source |
| **Google Sheets API** | Appends each synced trip as a row to a personal "Drive Log" spreadsheet; reads rows back for import; a hidden "AppData" sheet is used as a cross-device settings blob | OAuth token, scopes: `spreadsheets`, `drive.metadata.readonly`, `drive.file` |
| **Google Drive API** | Finds/creates the "Drive Log" spreadsheet file; uploads detailed route images | Same OAuth token |
| **`www.google.com/maps/dir/?...`** (deep link, not an API) | Optional hand-off to the real Google Maps app for voice-guided navigation alongside this app's own in-app tracking | None (plain URL) |

**Other external API:**
- **fueleconomy.gov REST API** (US government, public, no key) — vehicle fuel-economy lookup (year → make → model → trim → combined MPG) for US-region users. For non-US regions, a hardcoded fallback table (`CAR_FUEL_DATA`, UK + AU vehicles) is used instead, since no equivalent free live database exists for those markets.

**No weather API** is used anywhere. The "Weather" field in Supervised-drive logging (dry/wet) is a manual user selection. Day/Night classification is computed locally via a sunrise/sunset astronomical formula, not fetched.

**PWA**: standard `manifest.json` + a hand-written service worker (`sw.js`) — no Workbox or similar tooling.

## 5. All Implemented Features

### Navigation
Two visible tabs: **Drive** and **Log**. A third screen, **Settings**, is reached via the gear icon in the header (not a nav-bar tab). The "Drive" tab is simultaneously the dashboard home screen, the Quick Drive control, and the entry point into the Trip Planner.

### Drive logging
- **Quick Drive**: tap Start, drive, tap End — duration is timed live; end location/distance is auto-filled from GPS with a manual-entry fallback.
- **Trip Planner**: build a multi-stop route (each stop geocoded via Places/Geocoding), tag each leg (Personal/Business/Supervised) with proximity-based auto-suggestion from saved addresses and drive history, preview the route via Directions, then either:
  - **"Log Drive"** — commit the trip(s) using the estimated distance/duration, without live tracking, or
  - **"Start Drive"** — real GPS-tracked live navigation (see below).
- **Saved Routes** and **Saved Addresses** (Home/Work/Other) for fast re-entry.
- **Import historical drives** from an exported Google Maps Timeline JSON file (parses location history, detects likely driving segments, lets the user select which to import) — `renderAddDrivesModal`.

### Live GPS driving/navigation
- Real `watchPosition`-based tracking once "Start Drive" is tapped.
- Turn-by-turn instruction banner with maneuver icons (turn left/right, roundabout, U-turn, merge, fork, etc.), centered distance-then-instruction layout.
- ETA bar, breadcrumb path recording (used later to draw the actual driven route, not just a straight line, in the trip's detail image).
- Off-route detection with a hysteresis/confirm-fix count before triggering a reroute (avoids false positives from a single noisy GPS fix), plus a reroute cooldown.
- A custom vector-map camera (real heading/tilt/zoom, via a Google Maps **Map ID**) rather than a CSS-transform tilt illusion — the CSS-hack approach was tried first and explicitly replaced.
- Angle/zoom-cycling FABs, north-locked compass.
- Auto-completes the drive when GPS dwells near the final planned stop for a sustained period (not on some arbitrary distance alone), to avoid false completions.
- Optional **"Launch Google Maps Navigation on Start Drive"** setting: when on, tapping Start Drive also opens the real Google Maps app (deep link) for voice-guided turn-by-turn, running alongside this app's own in-app tracking (the in-app tracking is what actually gets logged; Google Maps is just for the driver's own turn-by-turn guidance).
- Resume-after-refresh: a snapshot of the in-progress live drive is persisted so a mid-drive page reload doesn't lose it.

### Drive categorization & supervised-hours tracking
- Every drive is tagged **Personal**, **Business**, or **Supervised**.
- Supervised drives capture: time of day (day/night, computed from real sunrise/sunset for the trip's location and date), road type, weather, traffic level, and the supervising driver (name + licence number, picked from a saved Supervisors list or added inline).
- Per-Australian-state/territory default hour targets (total hours + night-hours sub-target) are auto-detected from the device's IANA timezone (e.g. NSW/VIC 120h total/20h night, WA 50h/5h, etc.) and used as sensible defaults, fully user-editable.
- A completion modal congratulates the user once both targets are met.
- Business drives get an extra "purpose/notes" step.

### Log / History tab
- Sortable, filterable trip list (by tag, date range, minimum distance, free-text search).
- Edit any logged trip (all fields, including its route image and supervised conditions); split one trip into two at a chosen time (proportionally splits the distance too); bulk-select and delete; clear all history (with confirmation).
- **Report export**: CSV/PDF with a configurable field picker and date range/type filter.
- **Official Log Book export**: a separate PDF generator that reproduces the actual AU state learner-driver paper logbook layout (day/night hour tables per drive, a declaration-of-hours summary, foreign/interstate licence fields).

### Dashboards (on the Drive tab)
- Supervised Driving card: progress toward total/night hour targets, with a genuinely-stacked day/night bar chart and pace-to-target math (per day/week/etc.).
- Driving Metrics card: a single configurable card that can show Distance, Time, and/or Fuel Cost, each as its own toggle.
- Bar charts are hand-rolled (no charting library) with custom tooltip and label rendering.

### Fuel tracking
- Two costing modes: **Estimated** (a flat rate the user enters) and **Accurate (Beta)** (derived from actual logged fill-ups: date, odometer, volume, price).
- **Vehicle fuel-usage lookup**: US users query fueleconomy.gov live; everyone else gets a curated static table of common UK/AU vehicles by make/model/year (with an explicit disclaimer that these are approximate, not pulled from an official live database).

### Odometer tracking
- Configurable reminder frequency (e.g. quarterly) with a periodic prompt modal to log the current reading.

### Settings
- **Google Connector**: connect/disconnect Google account, choose an existing "Drive Log" spreadsheet or create a new one, manual "Sync Now", periodic sync reminders (also configurable frequency), CSV/xlsx export link for the connected sheet.
- **Dark Mode** switch (two-state: every session resolves to explicitly "dark" or "light", never left to just follow the OS with no attribute — see §7).
- **Theme Color**: 7 accent colors (amber/green/teal/blue/purple/pink/red), circular checkmark-style picker, each with its own derived "Cost" (darker) and "Time" (lighter) chart-series shades.
- Dashboard show/hide toggles (per-card and per-stat).
- Drive Type defaults, per-state supervised-hour targets and target date.
- **"Learn Routes (Beta)"**: auto-suggests a drive tag based on proximity to previously-driven or saved routes, with a configurable match-radius.
- Saved Addresses management (Home/Work/Other).
- Fuel Cost Tracking toggle and mode.
- Odometer reminder settings.
- **Security**: optional PIN-on-open.
- Clear History (destructive, confirmed).
- A hidden debug reveal: tapping the "Google Connector" section header toggles a small "Build `<APP_BUILD>`" line at the bottom of Settings — not documented anywhere in the UI itself.

### PWA / installability
- Installable to a home screen (manifest + icons, standalone display, portrait orientation).
- Offline app-shell caching via the service worker (see §7 for the exact caching strategy — it is network-first, not the more common cache-first).
- Forces a full page reload the moment a new service worker takes control, so a stale in-memory copy of the app never keeps running silently after an update.

## 6. Partially Implemented / Beta / Incomplete Features

- **"Learn Routes (Beta)"** — shipped and functional, but explicitly labeled Beta in the UI (auto-tagging confidence not fully trusted yet).
- **"Fuel Cost Tracking (Beta)"**, specifically its **"Accurate (Beta)"** costing sub-mode — shipped and functional, same caveat.
- **`state.showBuildInfo`** — functional but genuinely undiscoverable: there is no visible control for it; it only toggles via tapping the Google Connector section header. Fine as a developer convenience, but worth deciding whether to formalize (a real "About" row) or remove.
- No other stubs, feature flags, or commented-out "future work" blocks exist in the codebase — everything else that's reachable in the UI is fully implemented.

## 7. Design Decisions and Reasoning

- **Single file, no build step.** The whole app is one `index.html` so it can be hosted from GitHub Pages with zero tooling and edited/reasoned about without a build pipeline. The tradeoff (a ~10,000-line file) is accepted deliberately; see §8/§9 for the practical cost of this.
- **Full-DOM-replacement rendering**, not incremental DOM diffing. Simpler to reason about (render is a pure function of `state`, always), at the cost of having to explicitly keep non-serializable objects (live Maps instances, `DirectionsResult`) outside `state` so they survive re-renders correctly, and of re-binding all event listeners on every render via `attachAppHandlers()`.
- **Piecemeal persistence, not one big JSON blob.** Each concern (trips, settings, saved addresses, etc.) is saved to its own storage key independently, at the point of mutation. This means a crash mid-flow only risks losing the one thing that was being edited, not the whole app's data.
- **IndexedDB only for the one thing that needed a bigger quota** (full-resolution route images), while everything else — including each trip's own small thumbnail — stays in localStorage. A deliberate size/latency tradeoff: the small thumbnail always renders instantly with the trip list; the big detail image is fetched lazily, only when the user actually opens a trip.
- **Sync is always manual, never automatic.** New trips are explicitly marked `synced: false` and stay local until the user taps "Sync Now" (or dismisses/acts on a periodic reminder). This is a deliberate privacy/control choice, not a missing feature — logged drives never leave the device silently.
- **Two-state theming, not three.** Even though the OS exposes a "system" preference, this app always resolves `applyTheme()` to an explicit `data-theme="dark"` or `="light"` on `<html>` — there's no code path that leaves theme unset to passively follow `prefers-color-scheme` at runtime. This was confirmed intentional during earlier work on this codebase (see git history around the theme-color redesign) and deliberately not changed.
- **Region-aware defaults, not one-size-fits-all.** Supervised-hour targets default per Australian state/territory (detected via IANA timezone); the fuel-economy lookup switches between a live US government API and a curated static table depending on detected region; distance units (km vs. mi) default from the browser locale. All of these remain fully user-editable — they are starting points, not hard constraints.
- **Vector-map camera over a CSS hack.** The live-driving map's heading/tilt/zoom camera was originally simulated with CSS 3D transforms; this was replaced with a real Google Maps vector-rendering camera (via a Map ID), and the old approach's containing-block quirks are explicitly still guarded against in a couple of places even though the original cause is gone, as cheap insurance against the same class of bug recurring.
- **`window.storage` abstraction over `localStorage`.** The storage helpers transparently prefer a `window.storage` API when present (i.e. when hosted inside certain Claude/Cowork environments) and fall back to `localStorage` otherwise — the rest of the app never needs to know which backend it's actually using.

## 8. Known Bugs and Technical Debt

- **API keys are committed in plaintext client source.** `GOOGLE_MAPS_API_KEY` and `GOOGLE_CLIENT_ID` (`index.html:693`–`694`) are real, live-looking values checked directly into the repository. Because this is a static client-side app, the key is necessarily visible to anyone regardless of where it's stored — but it should still be locked down with HTTP-referrer/API restrictions in the Google Cloud Console (and the OAuth client's authorized origins kept tight) so it can't be abused if scraped from the deployed site. This is the single highest-priority item to verify/tighten before treating this as production-hardened.
- **The GitHub Pages workflow references a `staging` branch that doesn't currently exist** (`.github/workflows/deploy-pages.yml` checks out `main` and `staging` and combines them into one Pages site). Only `main` and two `claude/*` feature branches exist in this repo right now — the `staging` checkout step will fail (or silently produce an empty `/staging/` folder, depending on the checkout action's behavior on a missing ref) until that branch is created or the workflow is simplified to drop it.
- **A permanent migration patch for an old data bug.** `loadData()` backfills a fresh id onto any trip missing one, because an earlier buggy save path could produce trips with no `id` — and since the Log tab's multi-select checkboxes match trips by reading `id` off a DOM `data-` attribute, a real `undefined` id stringifies to the literal text `"undefined"`, which never matches correctly on re-render and permanently soft-locks that trip's selection checkbox. The migration works, but it's a sign the underlying save path bug should be double-checked for any other place it could still occur.
- **No automated tests exist.** There is no test runner, no CI check beyond the Pages deploy, and no linter configured. Verification during development has been entirely manual — visually, plus ad hoc headless-Playwright scripts written and run during sessions (never committed to the repo) and a plain `node --check` syntax pass on the extracted inline `<script>` content. Any future change should assume **zero** regression-safety net beyond what a human or an agent manually re-checks.
- **Single ~10,000-line file.** There's no technical reason it couldn't be split (e.g. by feature area) even while keeping a no-build-step deploy, but right now navigating it requires targeted `grep`/search rather than being able to open "the fuel module" as its own file.
- **Two "(Beta)" features** (Learn Routes, Accurate Fuel Cost) have been shipped for a while without graduating — worth a deliberate decision either to remove the label (if they're trusted now) or to document their known limitations somewhere the user can see.

## 9. Setup Instructions

This is a static site with no build step.

**To run locally:**
```bash
cd Drive-Log
python3 -m http.server 8080
# then open http://localhost:8080/index.html
```
Any static file server works — the only requirement is serving over HTTP (not `file://`), since the service worker, geolocation, and PWA install prompts all require a real origin (`http://localhost` is treated as secure for this purpose; a plain `file://` path is not).

**To deploy:** push to `main` (or `staging`, once that branch exists/is wired correctly — see §8) on GitHub; `.github/workflows/deploy-pages.yml` builds and publishes to GitHub Pages automatically. No manual deploy step, no secrets to configure for the deploy itself.

**Google integration (optional, for Sheets sync / Maps features):** the app ships with working `GOOGLE_MAPS_API_KEY` and `GOOGLE_CLIENT_ID` values already filled in (see §8 for the security caveat). To point the app at a different Google Cloud project instead:
1. Create/select a project in Google Cloud Console.
2. Enable the **Maps JavaScript API**, **Places API**, **Geocoding API**, **Directions API**, **Distance Matrix API**, **Sheets API**, and **Drive API**.
3. Create an API key, restrict it (HTTP referrers = your deployed origin(s)), and set it as `GOOGLE_MAPS_API_KEY` (`index.html:694`).
4. Create an OAuth 2.0 Client ID (Web application), add your deployed origin(s) as authorized JavaScript origins, and set it as `GOOGLE_CLIENT_ID` (`index.html:693`).
5. `mapsConfigured()`/`googleConfigured()` (`index.html:700`, `1218`) simply check the values aren't still the placeholder text `"PASTE_..."` — they don't validate the key actually works, so a bad key will fail at the point of first real use (e.g. the map won't render) rather than at startup.

**No environment variables, no `.env` file, no secrets manager** — configuration is just the two constants above, directly in source.

## 10. How the Main Components Interact

1. **Boot**: the bottom-of-file IIFE runs on page load, calls `loadData()` to hydrate `state` from `localStorage`/`window.storage` (profile, trips, settings, supervisors, saved addresses, fuel fill-ups, planned routes, recent locations, cached Google token), checks whether a PIN gate (`state.settings.requirePin`) should block entry, then calls `render()`.
2. **Render dispatch**: `render()` picks `renderLogin()` (no profile yet, or PIN required and not yet entered) or `renderApp()`. `renderApp()` always renders the header/nav shell, then delegates the tab body to `renderPlanTab()` (Drive), `renderHistoryTab()` (Log), or `renderSettingsTab()` (Settings) based on `state.tab`, and layers any active modal on top based on which modal-triggering state field is currently set (`state.editingTrip`, `state.pendingTrip`, `state.planner.confirmingDiscard`, etc.).
3. **Event wiring**: immediately after every render, `attachAppHandlers()` re-queries the DOM by id/class and attaches fresh `onclick`/`onchange`/`oninput` handlers. A handler typically mutates one or more `state` fields directly, optionally awaits an async operation (geocoding, a Directions call, a storage write, a Google API call), and then calls `render()` again to reflect the change.
4. **Persistence** happens inline, at the point of mutation — e.g. saving a trip calls `storageSet(TRIPS_KEY, state.trips)` right after pushing onto the array, rather than some later centralized "flush" step. This means the on-screen state and the persisted state are kept in lockstep by convention (every mutating handler that matters is expected to persist before or right after re-rendering), not enforced by any framework.
5. **Live driving** runs somewhat outside the normal render loop for performance: `navigator.geolocation.watchPosition` delivers GPS fixes to `handleLiveFix()`, which updates the live-drive tracking object and, for the parts of the UI that must update every second regardless of a full re-render (the driving timer, the live map camera), pushes DOM updates directly (`updateLiveDriveDOM()`, `renderTimerOnly()`) rather than going through the full `render()` pipeline — a deliberate escape hatch from the "always full re-render" rule for the one place where that would be too slow/jittery.
6. **Google Sync** is entirely separate from local logging: trips are created and fully usable offline; only when the user taps "Sync Now" (or acts on a sync reminder) does the app request/reuse an OAuth token, find-or-create a spreadsheet, and append rows for any trip not yet marked `synced`. A failure at any step in that chain (e.g. a route-image Drive upload) is treated as best-effort and doesn't block the rest of the sync.
7. **Service worker** operates independently of all the above: it intercepts every same-origin GET, tries the network first (so the live app is never stuck on stale cached JS while online), and only falls back to the cache when offline. It has no communication channel with the app's own `state` beyond forcing a page reload when a new version takes over.

## 11. Recommended Next Development Tasks

1. **Lock down the exposed Google API key/OAuth client** with proper HTTP-referrer/origin restrictions in the Google Cloud Console, if that hasn't already been done outside this repo — this is the most consequential open item.
2. **Fix the `staging` branch reference** in `.github/workflows/deploy-pages.yml` — either create a real `staging` branch to serve as the pre-production preview it implies, or simplify the workflow to just deploy `main` until staging is actually wanted.
3. **Add a minimal CI check**: even just running the `node --check` syntax validation (already used ad hoc during manual development sessions) on every push/PR would catch a broken commit before it reaches `main`/Pages.
4. **Decide the fate of the two Beta features** (Learn Routes, Accurate Fuel Cost) — graduate them or document their known limitations somewhere user-visible.
5. **Re-audit the old missing-`id` trip bug** — the migration in `loadData()` handles existing bad data, but it's worth confirming no current save path can still produce a trip without an `id`.
6. **Consider splitting `index.html`** into a few logical files (e.g. by feature area) if it keeps growing — still deployable with zero build step via plain `<script src>` tags, while making the codebase easier for a new contributor (human or agent) to navigate without relying entirely on full-text search.
7. **Formalize or remove `state.showBuildInfo`** — either give it a discoverable "About" entry point in Settings, or drop the hidden gesture if it's no longer useful.
