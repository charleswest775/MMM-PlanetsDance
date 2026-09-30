# MMM-PlanetsDance — context for Claude sessions

Charles's own MagicMirror² module: the planets' real orbits, from today, drawn as figures that
look like sacred geometry, for his hallway mirror, as one page (or one turn of a page) in a
rotation of pages. Split out of MMM-ChaosTheory on 2026-09-28, with its history; before that it
was the `orbits` simulation there.

## What exists (v1.0.0)

- `MMM-PlanetsDance.js` — module shell: one canvas plus an HTML caption (title, equations, note,
  live readout, updated 2×/s). A new figure on each `resume()` and every `cycleSeconds`. Loop:
  `setTimeout` until a frame is due, then one `requestAnimationFrame`. `suspend()` stops it; a
  sim with `resting = true` is polled only every 500 ms. While MagicMirror fades the module out
  (`hidden` is set at the start, `suspend()` comes after), frames draw nothing. `turns: { of, at }`
  lets modules on one MMM-pages page take turns: each counts showings from 0 and shows only when
  `showing % of === at`; off-turn it sets `display: none` and runs nothing.
- `simulations/orbits.js` — the sim (`window.PlanetsSimulations.orbits`, UMD so it runs in
  Node): a deck of eight dances (`venus`, `venus-loops`, `mercury`, `mars`, `jupiter-saturn`,
  `trigon`, `jupiter`, `saturn`), one per showing. Each builds strokes (lines, paths, dots, in
  au, with times 0…1) once, then draws them over `orbitsSeconds` (22), incrementally with
  `lighter` compositing, then rests. Sets `this.info` per instance for the caption. Options:
  `orbitsSeconds`, `orbitsDances`, and for the preview `dance` and `start`.
- `simulations/ephemeris.js` — JPL's approximate Keplerian elements (Standish): Table 1 for
  1800–2050, Table 2 outside. Global `window.PlanetsEphemeris`. **MMM-NightSky has a copy**
  (global `SkyEphemeris`, otherwise identical): a fix here belongs there too.
- `node_helper.js` — the stats panel's sampler (`/proc`, thermal zone), only between
  STATS_START and STATS_STOP.
- `tests/orbits.test.js` — `node --test`, no dependencies: conjunctions and oppositions against
  the almanacs, the two tables against each other, Kepler's third law, Venus's cycle, Mars's
  retrograde, the trigon's steps, every dance builds and finishes on time, the deck.
- `dev/preview.html` — runs the module in a desktop browser (serve the folder, e.g.
  `python3 -m http.server`), with hide/show buttons; query options override config
  (`?dance=trigon&start=1600-01-01`).

The shell (`MMM-PlanetsDance.js`, `node_helper.js`'s stats panel, `dev/preview.html`) is shared
in spirit with the sibling modules split out at the same time (MMM-ChaosTheory, MMM-Atom,
MMM-FractalZoom, MMM-Chladni, MMM-SacredGeometry, MMM-Tilings, MMM-SnowCrystal, MMM-NightSky,
MMM-PhotoDeck, all under `~/dev/mirror-modules`, MMM-StandardMap, MMM-ChaoticWaterwheel, MMM-DoubleSlit, MMM-Sandpile, MMM-Harmonograph): a fix there probably belongs in the siblings too.

## On the mirror

It shares the sacred page (`classes: "page-sacred"`, 30 s) with MMM-SacredGeometry and
MMM-Tilings, taking turns in threes: this one is `turns: { of: 3, at: 2 }`.

Measured on the Pi (700², 12 fps, seven showings): ~30% of a core while drawing, ~4% held, 33%
(23–39) over a 30 s showing, page change included. Frames change 0.1–15% of the canvas
(sacred geometry: 48%).

## Performance findings on the Pi (measured)

- Hidden: 0.3% of one core (baseline 0.2%): suspend() verified via MMM-Remote-Control hide.
- A frame that changes the canvas costs ~2%/fps fixed; beyond that, cost scales with the
  **bounding box of everything changed in the frame**. Full redraws of a 900² canvas at 20 fps
  saturate the pipeline (~150%). JS is never the bottleneck (<3 ms/frame).
- So: draw incrementally, keep each frame's changes spatially compact, and rest when the picture
  is static. Line width, opacity, `rAF` vs timer made no difference.
- Use `statsPanel: true` or `debugStats: true` to see fps and CPU on the Pi; `electronSwitches`
  are applied too late to set `remote-debugging-port`.

## Hard constraints: the target device

- **Raspberry Pi 3 B+, 905 MB RAM, 64-bit Debian 13.** Mirror runs Electron 42 in a cage
  Wayland kiosk.
- **No GPU acceleration, and it can't be enabled**: the Pi 3's VideoCore IV only does GLES 2.0,
  Chromium needs ES 3.0 (tested). All canvas drawing is CPU. **No WebGL / three.js.**
- Screen will be **portrait 1200×1920** once mounted (Dell U2413, rotated). Design for portrait.
- Electron baseline is ~0.5% of one core. A full-screen 60 fps canvas could cost a lot more.
  **Measure, don't guess**: on the Pi, `~/.cache/mm-sample.sh 60` prints Electron CPU% and RSS
  over 60 s. Record before/after numbers in the README.
- The mirror rotates pages every 15-30 s (MMM-pages, which hides/shows modules). Verify that
  `suspend()`/`resume()` actually fire on page changes. If the loop keeps running while
  hidden, it burns CPU 24/7.

## Deploying and testing

- The repo is public (github.com/charleswest775/MMM-PlanetsDance) so the Pi can `git pull`
  without credentials.
- Pi access: `ssh fatherson@raspberrypi.local` (key auth). Module path:
  `~/MagicMirror/modules/MMM-PlanetsDance`. Restart: `pm2 restart MagicMirror`
  (pm2 is in `~/.npm-global/bin`). Logs: `pm2 logs MagicMirror`.
- The mirror's **config.js lives in a separate private repo**, `charleswest775/magicmirror-setup`
  (cloned at `~/dev/magicmirror-setup`). Change the module's config block there, then
  `./deploy.sh diff` and `./deploy.sh push` (push validates config before restarting).
  Don't hand-edit config.js on the Pi without `./deploy.sh pull` afterwards.
- Iterate in `dev/preview.html` on the Mac, then confirm performance on the Pi.
- Commit as Charles's GitHub noreply address (see git config in this repo).
