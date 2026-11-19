# Brief: Twin Dial, round 2 — build the app

Audience: seven build agents, packages **A1, A2, B, C, D, E, F**, working in parallel, each in its own git worktree. The specification is `docs/twin-dial/SPEC.md` (rev 2) on branch `twin-dial/r2`; **it is the authority** — read it end to end first, then §0–§4 here, then your package. The owner approved the look of the mockup (`tools/twin-dial/mockup/mockup.py`, output `bake/work/twin-dial/mockup/1_clock.png … 5_protected_zones.png`): your output must match it, and your pictures are compared against it.

The owner's instruction (2026-11-17): everything in parallel, bake everything, make a build, and push the new app. The lead (not you) merges, builds, versions and pushes. **You never push.**

## 0. The product in one paragraph

`apps/twin_dial`, id `twin_dial`, title "Twin Dial", version 0.1.0: a separate cluster app for the G20's 1920 × 720 panel, tachometer left (380, 400), speedometer right (1540, 400), R = 300 px, ±125° sweeps, two centre pages (clock, trip), a bottom row, our own check-control message plate with BMW's panel as the fallback window, day and night looks, km and km/h only. No maker name, logo, font or bitmap anywhere; free OFL fonts (Saira, Noto Sans); our own pictograms.

## 1. Rules for every package

- **Branches and worktrees.** Base is `twin-dial/r2`. `git worktree add <repo>/.claude/worktrees/r2-<pkg> -b twin-dial/r2-<pkg> twin-dial/r2`, work there. Symlink `reference` and `bake/work` from `<repo>` (`.gitignore` covers symlinks; `git status --porcelain` must not list them). Commit often on your branch, with the trailer the harness gives you. **No push. No version bump. No change to `apps/f30_cluster/` or `apps/driver_profiles/`** (their app.json, main.js, layouts, baked files); `shared/` and `tools/` edits only in the files your package owns (§3), and any edit to a file another package owns is a small hook block marked `// twin_dial:` — the lead merges.
- **f30_cluster must stay exactly as it is.** Run before your last commit: `node bench/test-bottom-row.js`, `test-native-ui.js`, `test-strip-mask.js`, `test-fonts.js`, `test-probes.js`, `test-daynight.js`, `test-night.js` (all as on `main`: 254 / 47 / 40 / 34 / 25 / 24 / 48 checks) and report any change. Run bench jobs one at a time inside your worktree (they rebuild `dist/`), headless only (no Chrome, no browser automation), no long-lived process (agents cannot `kill`; ports 8080 and 8150 belong to the owner — do not use them; use 8200 + the package's number if a port is unavoidable and stop it yourself).
- **No car** (no ssh, no reading the car). Never write to anything outside your worktree and `bake/work/twin-dial/`.
- **Python:** `UV_CACHE_DIR=<repo>/bake/work/.uvcache uv run -q --no-project --with numpy --with pillow [--with fonttools] python -I <script>`.
- **Not ours to ship.** The public repo gets our own drawings and numbers only: no maker's name, logo, font file or bitmap; the references (`reference/`, git-ignored) are never an input to a bake and never copied into a tracked file; no BMW symbol bitmap (`requestDefaultSymbols`) baked, logged or committed; never `KombiIF.standardCCMessage` / `extendedCCMessage` or any setter; no VIN, key, token, coordinates, street names or personal data in any file, log text or commit. Tracked text says "the references" or a file number, never a maker.
- **Safety rule (hard):** a BMW warning is never hidden unless our own text for that exact message is on screen in the same frame (SPEC §8.4). BMW's belt group, gear-letter gap, side lamps and message panel keep-outs (SPEC §3.3) are never violated.
- **Platform rules** (memory notes, learned the hard way): every `r` flag in app.json needs a non-zero getter id (`tools/signals/check_signals.py` must pass); every sampler2D needs a bound asset or the layer is not drawn; KOMBI resets on big effect shaders (targets in SPEC §13.2); `mediump` is fp16, use `highp` for panel-px lengths > 256; drop unused scene textures before making new ones; 8 layers per connection, 16 MiB pixel memory counted twice.
- **Evidence:** write what you verified (test output, picture diff) and what you did not. Anything not proven on a car is a hypothesis; say so.
- **Outputs:** code and tests on your branch, your report `docs/twin-dial/R2-<PKG>-REPORT.md` (what was built, files, how to run, test results, open problems, anything you changed outside your files), and a published note for the others in `bake/work/twin-dial/pub/R2/<PKG>/` with `READY` touched when it is usable. Finish with a short final message: branch, last commit, tests, what the lead must merge or decide.

## 2. The engine contract (B owns it; others build to it)

Published first, by B, as `docs/twin-dial/ENGINE-CONTRACT.md` and copied to `bake/work/twin-dial/pub/R2/B/CONTRACT.md` with `CONTRACT.READY`. B has **30 minutes** to publish it; until then the names below are the initial contract. Later changes are additive only; if something must break, B writes it in the file's changelog and messages the lead.

Two new layout widget types, in `shared/hud.js` (widget path) and `shared/shaders.js` (shaders):

- `tdback` — effect layer, black background with: see-through `holes` (list of `[x0, y0, x1, y1]` panel px), the tab bar (`tab`: rect, `seps` x list, ramp, rule, highlight cell x-range driven by the message severity), the centre-title rule and bottom rule (`rules`: `[{y, x0, x1, role}]`), and the **fallback window** sampled from an R8 mask asset (`panelMask`, `panelMaskRect`) opened by the cc state. Three instances (`part: "L" | "RT" | "RB"`, SPEC §13.2); only `L` carries tab bar, rules, mask.
- `tdial` — effect layer, one per dial: `kind: "tach" | "speed"`, `centre`, `R`, `rect`, `face` (baked R8 asset name), value `signal`, `sweep` (125), `redline` (setting), `bar` (`{y, x0, x1, segs, signal, reserve}` fuel or coolant), colour roles, night uniforms. The R8 face carries luma only; region tint (white / band / red) comes from ρ, θ ranges in the shader.

Widgets that already exist and are reused unchanged: `text` (image sheets, Noto/Saira atlases), `atext` (GPU digits: speed and gear), `icon`. New text tokens (odometer, trip values, weekday, range, tab media text …) are added by **C** in `shared/hud_text.js` / `shared/hud_data.js`.

Message state (D owns, `shared/twin_cc.js`): the hud engine exposes `cc.state` ∈ `none | window | ours` and `cc.severity`; `tdback` reads the window flag, the plate sheet and bottom-row sheet read `ours`, the tab highlight reads `severity`.

## 3. Ownership (who edits what)

| Package | Owns |
|---|---|
| A1 fonts, atlases, pictograms | `tools/fonts-src/` (Saira, Noto Sans subset + OFL), `tools/bake_fonts.py` (`twin_dial` list, `stretch`), `apps/twin_dial/hud/fonts/`, `apps/twin_dial/hud/icons/` or the icon bake the engine reads, `design/strip/pictograms/pictograms.js` `stepDist` export, `tools/twin-dial/vt/` bake glue |
| A2 faces and masks | `tools/twin-dial/bake_face.py`, `tools/twin-dial/bake_mask.py`, `apps/twin_dial/hud/scene/*`, `bake/manifest`-style record, `test`-scripts for the bake |
| B engine | `shared/shaders.js` (`tdial`, `tdback`), `shared/hud.js` widget paths + night + uniforms, `docs/twin-dial/ENGINE-CONTRACT.md`, `bench/test-twin-budget.js` |
| C app and layout | `design/gen_twin.py`, `apps/twin_dial/{app.json,main.js,NOTICE,hud/layouts/twin.json}`, settings + menu items, pages and cycle logic, new tokens in `shared/hud_text.js` / `shared/hud_data.js`, `mgu-repo.json` entry, `bench/test-twin-layout.js`, `bench/test-twin-pages.js` |
| D messages | `shared/twin_cc.js` (new) + hook blocks in `shared/hud.js`, `bench/test-twin-cc.js`, the plate/bottom-row/tab-highlight logic, strict D7 latch, logging |
| E bench and gallery | `bench/` G20 simulation additions for `twin_dial`, `tools/shots/` matrix and gallery, `tools/strip/O` pixel audit hooks, a mockup-vs-render comparison tool, `docs/twin-dial/R2-GALLERY.md` |
| F probe | `shared/twin_probe.js` (new) + one hook, `apps/twin_dial` probe config (app.json `config.probe`, signals), `bench/test-twin-probe.js`, `docs/twin-dial/PROBE-PROTOCOL.md` (what the owner does on the car for P1–P12) |

Where two packages need the same file (`app.json` signals: C and F; `shared/hud.js`: B and D and F), the first writer's lines win; the others add a marked block, nothing else.

## 4. Packages

### A1 — fonts, atlases, pictograms (SPEC §9, §10)
1. Take Saira and Noto Sans from `bake/work/twin-dial/refs/vt-fonts/` (VT downloaded them; OFL only; record the source URL and licence in `tools/fonts-src/SOURCES.json`). Add `Saira[wdth,wght].ttf`, a **subset of Noto Sans** (full `CAR_TEXT` + Latin: Ukrainian, Russian, Belarusian, Latin accents, № « » „ " " – — … U+02BC) and the two `OFL.txt` files. Keep the repo small (subset, drop what is not used).
2. `tools/bake_fonts.py`: a `twin_dial` list with every slot of SPEC §9's table (sizes as listed; Saira slots with a **1.06 horizontal stretch**, new `stretch` option in `bake()`; one atlas per face + size, digit-only for Saira, full text for Noto). A test that no Cyrillic string can reach a Saira atlas. Atlas budget in the report (target ≤ 0.8 MiB per full-text Noto size).
3. Pictograms: bake VT's 25 as the engine reads them (check `shared/icons.js`, `tools/twin-dial/vt/`, `design/twin-dial/vt/pictograms/baked/`); export `stepDist` from `design/strip/pictograms/pictograms.js` and drop the duplicate in `design/twin-dial/vt/pictograms.js`; icon sizes per SPEC §10.
4. A comparison sheet (git-ignored, `bake/work/twin-dial/r2/A1/`) of Saira 1.00 vs 1.06 against the mockup crops for the owner.
Done: the atlases and icons exist under `apps/twin_dial/hud/`, `node bench/test-fonts.js` still 34, your own font test green.

### A2 — baked faces and the panel mask (SPEC §4, §5, §13.3)
1. `tools/twin-dial/bake_face.py`: the two R8 luma faces (616 × 560 each, drawn at 3× and reduced, from SPEC numbers only — **never from a reference image**): scale band with the two-way shading and cool tint **in luma** (document how the shader will apply the tint), tick strip, major lines, ticks, bezel ring open at the bottom, inner rim (0.545 R, width 0.016), numerals from Saira (tach 1–8 at 57 px, speedometer large 28 px and small 18 px, the set of SPEC §5.3 with 90 and 120 dropped and the ≥ 4 px clearance check), "OFF" label, bar frames and end labels ("0" "1/1", "C" "H"), unit labels, icons under the dials, band ends in a radial cut (no feet). Region ranges for the shader (ρ, θ → white / band / red) go in a JSON beside each face.
2. `tools/twin-dial/bake_mask.py`: the fallback-window mask — `zc.panel` dilated 3 px, R8 ≈ 536 × 112 (use `tools/strip/bake_mask.py`'s "msg" lane code and VF's `message_panel.json` `fallback_window.polygon_px`); plus a check that the plate rectangle [694.5, 592, 1230, 720] covers it.
3. Output to `apps/twin_dial/hud/scene/` in the format the engine loads scenes in (look at `apps/f30_cluster/hud/scene/*.json` and `tools/bake_scene.py` / `tools/strip/bake_mask.py` first); every baked file reproducible from the repo with one command; a `bake/README`-style note.
4. A pixel comparison against the mockup's dial crops (`bake/work/twin-dial/mockup/`): geometry within 2 px, report the numbers.
Done: faces + mask + region JSON on your branch, comparison table in the report.

### B — engine: `tdial`, `tdback`, night (SPEC §4, §5, §7, §13)
1. Publish the contract (§2) within 30 minutes, then implement it.
2. `shared/shaders.js`: `tdial` (face sample, region tint, band tint and `bandGain`, red wedge from `redline`, needle with the SPEC profile and optional shadow, fuel/coolant bar with 5 segments and reserve, night uniforms; no glow; day/night by uniforms or a resolved `#if`, **never both looks in one source**; ≤ 8 KB after `compact()`) and `tdback` (≤ 6 KB; tab ramp, separators, highlight, rules, holes, mask-sampled window). `highp` for lengths > 256 px; ≤ 32 uniforms; every sampler bound.
3. `shared/hud.js`: widget paths for both types (scene needs/drops like `wbback`/`dial look:wb`, face and mask asset loading, uniform updates for needle angle with the 0.7.14 smoothing, redline, fuel, reserve, coolant, severity, window flag, night), the speedometer layer's rect ending at y 660, and the three-part background split. Keep f30_cluster's paths untouched.
4. `bench/test-twin-budget.js`: layers per connection ≤ 8, steady ≤ 6 MiB, peak ≤ 8 MiB with the bench's 16 MiB enforcement, shader source sizes ≤ targets, every sampler bound, no layer rect intersects [1380, 660, 1460, 720]. Use stand-in faces until A2's land.
Done: both widget types render on the bench from a hand-written test layout, budget test green, f30 suites unchanged.

### C — the app, the layout, the pages (SPEC §5.1–§6, §12, §14)
1. `design/gen_twin.py` → `apps/twin_dial/hud/layouts/twin.json` from `design/twin-dial/vg/layout.json` + VD + VF + SPEC (gen_6wb pattern, baseline placement): both `tdial`, three `tdback`, digital speed and gear (`atext`), hub labels (READY, unit, program, OFF is in the face), tab bar contents (SPEC §6.1 table), both centre pages, bottom row, bar labels/icons — everything with the sizes, positions and colours of the SPEC.
2. `apps/twin_dial/app.json` (id, title, version 0.1.0, `loggingId` "OATD", `iconGlyph` "TD", `shared` list per SPEC §14.1 plus `twin_cc.js` and `twin_probe.js`, signals per SPEC §11 — `tools/signals/check_signals.py` against the catalogue copy at `reference/car/2026-11-15/someip/someip-interfaces.json` must pass), `main.js` (kit `mgu.run`, menu and settings per SPEC §14.3: `centre`, `redline`, `nightLook`, `bandGain`, `demo`, plus an `enabled` on/off if f30_cluster has the equivalent), `NOTICE` (Saira, Noto Sans OFL, kit), a clean `settings` screen via `shared/menu.js`.
3. Tokens in `shared/hud_text.js` / `shared/hud_data.js`: odometer (`mileageKilometre`, factor 1 km, invalid → "–"), range, trip rows (averages, distance, duration, departure; "–" until P4 factors are set in config), weekday (Ukrainian, lower case), local date and clock (reuse f30_cluster's `config.clock` / `clockLocal` path), outside temperature (sign, one decimal), fuel and coolant bar values.
4. Pages: `centre` = clock | trip | cycle (SPEC §6.2: 10 s per page while standing, held above 5 km/h), page change as a repaint of one 540 × 400 sheet.
5. `mgu-repo.json`: add twin_dial (look at how f30_cluster is listed). `bench/test-twin-layout.js` (every widget clears `visible.check`, keep-outs of SPEC §3.3, furniture rule, numeral clearance, no layer rect in the gear gap, no Cyrillic in a Saira widget), `bench/test-twin-pages.js` (switching, no cycle while moving, "–" fallbacks).
Done: the app builds with the kit (`node bench/server.js twin_dial` headless build or the kit build; no `BENCH_STUB` artefact may be committed), tests green, `check_signals.py` passes.

### D — messages (SPEC §8)
`shared/twin_cc.js` (the cc state machine for twin_dial; reuse f30_cluster's `_ccPush`/`_ccOpen`/`_ccListEval`/`_ccOnScreen`/`CC_SETTLE_MS` logic by calling into hud.js, not by copying it where avoidable) and small marked hooks in `shared/hud.js`: severity from `cCColor`, our plate sheet (symbol cell, headline in Noto 42, wrap to 441 px, two lines, "…", counter), the bottom row stepping aside, the tab highlight and rule colour, the generic marker for unmapped ids, the **strict D7 latch** (`config.ccActive.strict`: until `statusActiveCCMessage` has once matched a ccList entry, the plate is never drawn over BMW's panel and the window opens whenever the heuristic says a message may show), all eight keep-open conditions of SPEC §8.4, the log lines of SPEC §14.4 (numbers and lengths only; text never logged). `bench/test-twin-cc.js` = VG report §5.4 cases 1–9 including a frame-by-frame pixel audit on the bench's cluster-render model (at no frame is BMW's panel covered while our text for the shown number is absent). Build the state machine and its tests against stand-in widgets until B's and C's land; then integrate in your branch.
Done: `test-twin-cc.js` green (all nine cases), f30 suites unchanged (the shared hooks must be inert for f30_cluster).

### E — bench simulation and gallery (SPEC §16)
1. Make the bench run `twin_dial`: the G20 simulation draws BMW's lamps, belt group, gear letter and message panel beneath/over us; signal injection for every input of SPEC §11 (speed, rpm, gear, program, ready, fuel, reserve, coolant, ambient, clock, day/night, ccList + active number + the D7 confirmation, trip fields, media text); a demo sweep.
2. `tools/shots/` matrix for twin_dial: {day, night} × {no message, ours, window, blind} × {clock, trip} × {engine off, ready, driving} + demo, rendered replay-based (no browser); run it on whatever of B/C/D has landed (use `twin-dial/r2` plus stand-ins), re-run at the end on the merge candidate the lead gives you.
3. A comparison tool: each rendered state against the mockup (side by side + difference), numbers per region; `docs/twin-dial/R2-GALLERY.md`.
Done: a gallery of every state exists under `bake/work/twin-dial/r2/E/` (git-ignored) and the comparison runs.

### F — probe mode (SPEC §17.1)
`shared/twin_probe.js` + a hook: with `config.probe` on, subscribe VS's P1–P12 fields read-only and log raw values once per change — numbers, enums, lengths of text, **never text, never coordinates or road names** — in the app's own log. The probe fields are declared in a separate `signals` block selected by the probe flag (the normal build declares only SPEC §11), and `check_signals.py` passes for both blocks. `docs/twin-dial/PROBE-PROTOCOL.md`: step-by-step actions for the owner at the car for each P-item, the exact log lines to read, and how each result becomes a config value (trip factors, odometer, units, stalk-button signal, media/phone fields). `bench/test-twin-probe.js` with injected values. No write of any kind to the car.
Done: probe mode on the bench logs every P-item's line from injected values; both signal blocks pass the check.

## 5. Integration (lead only)

The lead merges A1, A2, B, C, D, E, F into `twin-dial/r2`, resolves hook conflicts, runs every suite (new and f30's), runs E's gallery and comparison, builds with the kit, sets version 0.1.0, and pushes after the final checks. Package agents: keep your hooks small and your tests honest; the lead trusts a green test only if it fails when the feature is broken.
