# SU1 report — cruise simulator page: shell, state, playback, the animated scene

2026-11-19 · agent SU1 · branch `cruise-sim/su1` (worktree `.claude/worktrees/cs-su1`) · last commit `e37db92`

## Summary (Українською)

Сторінка симулятора готова як оболонка: `tools/cruise-sim/ui/index.html` відкривається з диска (`file://`) або з локального сервера, без бібліотек і мережі. `app.js` тримає стан (конфіг у URL), запускає прогін (у сторінці частинами з прогрес-баром, або у воркері з відкатом на сторінку, коли браузер забороняє воркер під `file://`), відкриває файли результату / батчу / конфігу / пресетів, програє запуск (×0.25…×16, крок, скрабер з позначками попереджень і збоїв). `scene.js` малює дорогу згори: наше авто, лідера та інших, зміни смуг, справжній і виміряний розрив, зону бажаного розриву і потрібної дистанції, ціль сенсора (і «WRONG TARGET» / «no target»), попередження, дії водія, збої, профіль висоти (ухил) і панель показників. Поки рушія не було, сторінка працювала на моєму підставному результаті й пише на екрані «FAKE». Перевірено з рушієм SC1 (злито локально) і в тимчасовій збірці всіх п'яти гілок: усі 200 тестів проходять, усі чотири вигляди SU2 монтуються. Тести SU1: 36 (4 файли), разом з SC1 у моїй гілці — 106, без падінь. PNG-кадри сцени зроблено без браузера.

## What was built

All under `tools/cruise-sim/` on `cruise-sim/su1` (SC1's branch merged locally, as allowed, to run real results):

| File | What |
| --- | --- |
| `ui/index.html` | The page: header (plant badge, engine badge, threshold badge), toolbar, status line with a progress bar, Run / Batch / Report tabs, the scene canvas, the playback bar, mount points `#charts` `#settings` `#batch` `#report` and the hidden hook `#cluster-preview`. |
| `ui/app.js` | The shell. Pure half (exported for Node): config merge / diff, URL state, result normalise / serialise, file classification, `sampleAt`, `buildTimeline`, `frameAt`, `Playback`, `scriptList`, `sensorSamples`. Browser half: the `app` object, loading core + SU2 views, runs, presets, files, playback loop, keys. |
| `ui/scene.js` | The animated road. `buildScene(frame, opts)` is pure and returns a display list; `paint(ctx, ops)` plays it on a canvas; `createScene(canvas, app)` binds it to the page. Light and dark palettes. |
| `ui/worker.js` | Runs `core/sim.js` off the main thread (`init` with the page's script list, `run` with progress, result with buffers transferred). Own scope (IIFE). |
| `ui/style.css` | Tokens on `:root`, dark by `prefers-color-scheme` and `[data-theme]`. |
| `ui/fake-result.js` | SU1's FAKE stand-in result (a scripted story in the contract's shape) used only while `core/sim.js` is absent; the page and the scene say "FAKE". Also writes the fixture. |
| `tests/app.test.js` (19), `tests/scene.test.js` (8), `tests/app-dom.test.js` (7), `tests/worker.test.js` (2) | SU1's tests, node:test, run by SC1's `node tools/cruise-sim/test.js`. |
| `tests/scene-png.js` | Plays the scene's display list on `kit/runtime/canvas.js` (pure JS) and writes PNGs. |
| `tests/fixtures/su1-fake-result.json` | 87 KB result file (60 s at 0.2 s), checked against the generator. |

### The `app` object (CONTRACT "The page", plus additive members)

Contract: `app.config`, `app.result`, `app.cursorT`, `app.on('result'|'cursor'|'config', fn)`, `app.setConfig(partial)`, `app.run()`, `app.loadResultFile(json)`.
Added: `app.off`, `app.setCursor(t)` (alias `app.seek`), `app.play/pause/step(n)/setSpeed(x)`, `app.openRun(config)` (replace and run: the matrix click), `app.setCompare(result)` / `app.resultB` and event `'compare'` (A / B), `app.batch` and event `'batch'`, event `'view'` (tab shown), event `'frame'` (the sampled frame at the cursor: the cluster-preview hook), `app.sensorSamples()`, `app.frame()`, `app.isShort()`, `app.autoRun`, `app.runMode` ('auto'|'page'|'worker'), `app.defaults` (the engine's `defaultConfig()`), `app.presets`, `app.exportConfig/exportResult`, `window.app`.

- **Config.** `setConfig(partial, { replace, run })` deep-merges (arrays replace). `BASE_CONFIG` is `{ version: 1, seed: 1 }`: the page never overrides an engine default or a scenario's duration. A short run (estimated < 0.4 s from the last measured rate) re-runs 250 ms after a change when "re-run on change" is on; the same config in flight is one run; a newer run drops an older one.
- **URL.** `#c=<base64url of the JSON diff to BASE_CONFIG>&t=<cursor>&v=<tab>`. Pasting a link restores config, cursor and tab; a broken `c` is reported in the status line, never thrown.
- **Runs.** With `CS.sim.createSim`, the page advances 0.5 s per call in ~12 ms slices per frame (progress bar, no frozen tab) and stops with an error if the engine stops advancing; `runMode` worker (or auto for long runs) uses `ui/worker.js`, and falls back to the page when the browser refuses a worker (Chrome under `file://`).
- **Scripts.** Core files in `sim.LOAD_ORDER` order (identical to my list; a test checks it), `ui/scenarios.bundle.js` just before `core/scenario.js`, then SU2's `charts.js settings.js batch.js report-view.js`, each mounted with `CS.<name>.mount(el, app)` as SU2 announced; a missing file keeps a placeholder. When SU2's settings / charts mount, the shell hides its own preset group, auto-run box and Keep-as-B buttons (they are fallbacks).
- **Files.** One "Open file…" (or drop on the page) for: a result (`kind: 'cs-result'` or any `{ t, series }`, NaN as null), a batch (`kind: 'cs-batch'` or `runs: [...]` → `app.batch`, Batch tab), a config (`kind: 'cs-config'` or a bare config), presets (SU1's `cs-presets` or SU2's `cs-settings-presets`). "Save result" writes the run in that result format.

### The scene

From above, scrolling with our car, lane 0 (ours) at the bottom, + lanes to the left (= up), as SC1 / SC2 confirmed. Drawn: every vehicle from `veh.<id>.x|y|lane|lat` and `meta.vehicles` (else `lead.*` / `follower.*`), lane changes through `y`, brake lights below −1 m/s², the needed distance (red zone and dashed line), the desired gap (green ±10 % band and line), the measured gap as a teal tick across our lane plus a ray, the estimate as a violet diamond, the true and the measured gap as dimension lines with the error, a ruler, a vehicle beyond range as an arrow with its distance, brackets on the sensor's target (red + "WRONG TARGET" when it is not the true lead; "no target"), a red frame and chip while the warning is on (with its reason and TTC), chips for faults in progress (with their age), app not running, driver brake / accelerator / reaction / resume / maximum change, dropped or rejected commands, target changes, a curve's radius; a collision overlay; the elevation profile (the engine's `elevation`, else integrated `grade`; height ×1…10 to fit, dashed beyond where the run went); the readout (time, cruise state, speed, set held / commanded, driver maximum, v_ref, lead true / estimated, gaps, desired / needed, time gap and TTC in red under the thresholds, acceleration, warning, the plant "assumed" line, the last events). Range: fit to the run (40…300 m) or 50 / 100 / 200 / 300 m.

How the sensor's target is found (SC2's ids are track ids, not vehicle ids): the engine's truth link `meas.veh` when it exists, else an id that names a vehicle of the run, else the vehicle at the measured gap — the true lead whenever the measurement fits it, so two cars at a similar distance never produce a guessed "wrong target". `meas.wrong`, when recorded, decides.

PNG frames (no browser; kit's pure-JS canvas, its own font): `bake/work/cruise-sim/su1-frames/*.png` (the FAKE story: approach, cut-in warning with the driver braking, curve loss, frozen gap on a downhill, wrong target, driver override) and `su1-frames/engine/*.png` (the real engine with SC2's modules: E.cutin.slower, D.brake.hard, G.frozen, G.wrongTarget, J.appStop.B). Regenerate: `node tools/cruise-sim/tests/scene-png.js [outDir] [result.json]`.

## Tests

- SU1: 36 tests in 4 files, all pass. `tests/app-dom.test.js` boots `scene.js` + `app.js` in a vm with a minimal DOM stub, loading every script the way a browser does (one shared global scope; a script error is reported and the page goes on): first run, canvas painted, placeholders, badges never worded as a verdict, play ×16 to the end through requestAnimationFrame, step / scrub / keys, setConfig → hash → a second page from the link, replace / openRun, result / batch / config / preset files, A / B, worker refused → page fallback in createSim slices, failing run reported, run de-duplication and cancelling. `tests/worker.test.js` runs `worker.js` in a worker-like vm and compares with Node.
- Branch total (with SC1's tests): `node tools/cruise-sim/test.js` → **106 passed, 0 failed, 11 files** (~5 s).
- Integration check (scratch checkout su1 + sc1 + sc2 + sc3 + su2 with the scenario bundle generated; nothing from it committed, the worktree removed): **200 passed, 0 failed, 17 files**; the page boots with the real engine, 154 scenarios and all four SU2 views mounted.
- This DOM test found that SC1's first core files broke as browser scripts (repeated top-level `const` names across files); posted in NOTES, SC1 fixed it and added its own browser-path test.

## How the owner checks the page by eye

1. Open `tools/cruise-sim/ui/index.html` from Finder in Chrome or Safari (`file://`). Or serve: `python3 -m http.server 8980 -d tools/cruise-sim`, then http://localhost:8980/ui/ (any port 8980–8984; 8080 / 8150 are yours).
2. The header shows "assumed plant: mild / P1" (amber), the engine badge ("engine: core/sim.js", or "FAKE …" in red if the core files are missing), and a threshold badge worded "with an assumed plant: …". The status line says how long the run took and where it ran.
3. Press space: the road scrolls, the readout counts. Try ×0.25 and ×16 (`[` `]`), ← / → steps one sample (shift: ten), Home / End; drag the scrubber — red marks on it are warnings, amber are faults.
4. In Settings (SU2) pick a cut-in scenario (E.…) or G.wrongTarget / G.frozen / J.appStop.B and watch: the bracket on the target, the teal tick against the lead's rear (the sensor error), the red needed-distance zone growing on closing, the red frame and WARNING chip, the driver chips, the FAULT chip with its age, "APP NOT RUNNING".
5. Theme: the toolbar's theme menu (auto / light / dark); both must be readable.
6. Copy link, open it in a new tab: same config, same cursor. Save result, then Open file… with it: the same run comes back (source "file").
7. Under `file://` in Chrome, choose "run: in a worker": the status says the worker is unavailable and the run goes to the page — expected.

## Left out, and why

- **A live "advance in real time" mode** on `createSim`: the page runs the whole run (in slices) and then plays it; that gives the scrubber over the whole run, which a live mode cannot. The step API is used for the slices and the progress bar.
- **The cluster preview**: only the hook, as the brief says — the `'frame'` event, `app.sensorSamples()` (t, valid, gap, vrel, id, egoV per sample) and a hidden `#cluster-preview` that a module `CS.clusterPreview = { mount(el, app) }` would fill. The export of gap events for the Twin Dial bench is not built (needs the bench's replay shape and the raw-gap mapping that does not exist yet).
- **Phone layout**: not a target (brief §4); the page reflows to one column under 1100 px.
- `ui/scenarios.bundle.js` is not in my branch (no scenarios on it); see below.

## For the lead (merge seams)

1. After merging SC3: `node tools/cruise-sim/core/scenario.js --bundle > tools/cruise-sim/ui/scenarios.bundle.js` (~92 KB) and commit it — the page from the disk cannot fetch the scenario JSON. `tests/app.test.js` fails while it is missing or stale.
2. My branch contains a local merge of `cruise-sim/sc1` (up to `0927e65`); merging SC1 first, as planned, makes it a no-op.
3. SC1 wish posted (additive): record SC2's `meas._truth` as series `meas.veh` and `meas.wrong`; until then the scene infers the target from the measured gap (conservatively).

## Open decisions for the owner

- Should the scene's "wrong target" mark rely on the engine's truth series only (once SC1 records `meas.veh` / `meas.wrong`) and drop the gap-matching fallback? Recommendation: yes, once the series exist; keep the fallback for loaded logs without truth.
- The desired-gap "zone" is drawn as ±10 % of `d.des` — a display choice, not a parameter of the controller. Keep, change the width, or show only the line?
- Should the shell keep its own (fallback) presets in a separate store from SU2's settings presets, or should both use one store? Today the shell's are hidden when SU2's settings are mounted, so the owner sees one set.
- Track `ui/scenarios.bundle.js` (generated, ~92 KB, test-checked) as proposed, or generate it at release time only?
