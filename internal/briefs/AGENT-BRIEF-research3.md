# Brief: research round 3 — measure the two cluster apps against each other

Audience: five research agents, packages **MP, MT, MB, MC, MK**, working in parallel, each in its own git worktree. Read §0–§2 and your own package in full; skim the others. **The car is offline for this round**: everything is measured on the Mac (code, bench, replay, GPU) and each package ends with what the lead must read or try on the car later. Nothing is pushed, no version is bumped, no shipped behaviour changes on `main`.

## 0. Why now

Since 2026-11-17 the owner's car runs two apps of ours on one engine (`shared/`), through the same cluster-render client: `apps/f30_cluster` 0.7.20 (the F-series 6WB look, three themes that follow the drive mode) and the new `apps/twin_dial` 0.1.0 (two round dials, one layout). The owner's observations on the car (2026-11-17/18), which are the evidence this round is built on:

1. **Twin Dial renders extremely fast; f30_cluster is much heavier** (start and switching).
2. **Twin Dial shows no black rectangle; f30_cluster does** (the KOMBI's box over the right dial's hub, x 1406–1597, y 342–419). The upstream issue says it cannot be investigated; now two apps on the same engine differ, so every difference between them is a candidate cause and everything they share is ruled out.
3. **Twin Dial's trip page shows only empty values.**
4. **f30_cluster 0.7.20 did not fix the messages: the see-through window over BMW's message panel stays open** ("hangs") after BMW's message is gone. It is the window, not our own text line.

The owner wants: everything measured; and for f30_cluster, if its textures really are the weight, **every possible way to make them lighter** (merge, repack, recompress, replace) explored to the end, with numbers.

## 1. Rules for every package

- **Branch and worktree:** `git worktree add <repo>/.claude/worktrees/r3-<pkg> -b research3/<pkg> main`. Symlink `reference` and `bake/work` from the main checkout (`git status --porcelain` must not list them). Commit on your branch. **No push, no version bump, no merge into `main`.**
- **No car** (no ssh). No Chrome or browser automation. No long-lived process you cannot stop yourself (agents cannot `kill`); ports 8080 and 8150 are the owner's; use 8300 + your package's number if you need one. Bench jobs one at a time inside your own worktree (they rebuild `dist/`).
- **Measure, do not guess.** Every claim carries a number, the command that produced it and its spread (run it more than once). Label each finding **verified** (measured here), **hypothesis** (needs the car), or **refuted**. A bench number is not a car number: the Mac's GPU is not the head unit's; say what transfers (bytes, layer counts, message counts, shader size) and what does not (milliseconds).
- **Experiments are branches, not changes.** Any variant of an app that the lead may later install on the car lives on your branch, behind a setting or config key that is off by default, with its tests, and with the seven f30 suites (254 / 47 / 40 / 34 / 25 / 24 / 48) and the eight twin suites still green.
- **Standing rules:** the car is read-only; never `kill` an OAP app process (only `systemctl restart oapcontroller`); never a setter, never `KombiIF.standardCCMessage` / `extendedCCMessage`, never `A19B`; no VIN, key, token, coordinates, street names or personal data in any file, log or commit; `reference/` and `bake/work/` are never committed; no maker's name, logo, font or bitmap in tracked files; BMW's symbol bitmaps are never baked, logged or committed; a BMW warning is never hidden unless our own text for that exact message is on screen in the same frame; every `r` flag needs a non-zero getter id; every sampler bound; `mediump` is fp16; the KOMBI resets on too-big effect shaders.
- **Outputs:** `docs/research3/<PKG>-REPORT.md` (method, tables, verified / hypothesis / refuted, what the car must answer), data under `design/research3/<pkg>/*.json`, tools under `tools/research3/<pkg>/`, scratch under `bake/work/research3/<PKG>/` with a `READY` note for the others in `bake/work/research3/pub/<PKG>/`. Finish with a short final message: branch, last commit, the three most important findings, what the lead must decide.

## 2. What exists (use it, do not rebuild it)

- The engine: `shared/hud.js`, `shared/shaders.js`, `shared/layouts.js`, `shared/cluster_render_client.js`; f30's scene files `apps/f30_cluster/hud/scene/*.json` (about 12 MB on disk) and `hud/baked/C1…C10` (72 files); Twin Dial's `apps/twin_dial/hud/scene/` (about 1 MB: two R8 faces and a mask).
- Performance tools and earlier results: `tools/perf/` (`gpu_time.mjs`, `gpu_sweep.sh`, `switch_sweep.sh`, `overdraw.py`, `profile_top.py`, `summarise.py`), `docs/PERF-AUDIT.md`, `docs/PERF-NEEDLE-REPORT.md`; memory notes on scene-texture peak memory and the 16 MiB budget (each image counted twice).
- The bench and replay: `bench/` (the cluster-render model with layer, pixel-memory and upload accounting, `bench/twin-sim.js`, `/probe`), `tools/strip/O/` (replay and render), `tools/shots/` and `tools/shots/twin/` (state galleries without a browser), `tools/twin-dial/engine_render.py`.
- Black-box research: `docs/boxes/{BR,BS,BC}-REPORT.md` on branches `boxes/br`, `boxes/bs`, `boxes/bc` (hypotheses H-G: the KOMBI paints the box when our paint covers BMW's gear letter; H-M: it follows BMW's map; contentClass; scene-id fallback), `docs/AGENT-BRIEF-boxes.md`.
- Messages: `docs/strip/*` (branches of the strip round), commits 0.7.18–0.7.20, `apps/f30_cluster/app.json` `config.ccList` / `config.ccActive`, `shared/hud.js` (`_ccPush`, `_ccOpen`, `_ccListEval`, `_ccOnScreen`), Twin Dial's `shared/twin_cc.js` and `docs/twin-dial/R2-D-REPORT.md`; memory note `cc-source-cclist` (on this car `newCCMessage` never arrives; `ccList` lists stored messages too, id 987 stayed after BMW hid it).
- Trip computer: `shared/twin_tokens.js` (`tripValues`, `config.trip.factors`), `docs/twin-dial/VS-REPORT.md` (fields, all **unseen** on this car), `docs/twin-dial/PROBE-PROTOCOL.md`, `shared/twin_probe.js`.
- The catalogue of the car's signals: `reference/car/2026-11-15/someip/someip-interfaces.json` (git-ignored). Any car log saved earlier is under `bake/work/` if at all; search before assuming there is none, and never copy its text into a tracked file.

## 3. Packages

### MP — performance: why one app is fast and the other heavy
Goal: a side-by-side account of f30_cluster (each of its themes) and twin_dial, so the weight has a named cause.
1. **Inventory per app and layout**, from the built app and the bench's accounting: layers per connection; every asset (name, format, size in px, bytes on the wire, pixel memory counted twice); every shader (source bytes after `compact()`, samplers, texture reads per pixel, branches, uniforms); image sheets (count, size, how often re-sent); GPU text atlases.
2. **Start-up**: bytes and messages sent from "app starts" to "first complete frame", the time at the car's upload budget (floor 2000 KB/s, ceiling 5500), the order of uploads, what blocks the first frame.
3. **Switching** (f30: theme change with the drive mode, day/night, the G20 toggle; twin: page change, day/night, a message): what is destroyed and re-made, bytes re-sent, the pixel-memory peak, layers unmapped and mapped again, how long the screen is incomplete.
4. **Steady state**: messages and bytes per second while driving (needles, digits, clock), what goes as a uniform and what as an image dirty rect, fill rate and overdraw per frame (`tools/perf/overdraw.py`, `gpu_time.mjs`), shader cost per layer.
5. **Rank the causes** of f30's weight with their share (upload volume, rebuild on every mode switch, shader cost per pixel, image updates, layer count) and say which Twin Dial technique removes each (R8 luma faces coloured in the shader, one layout, uniforms for looks).
Done: `MP-REPORT.md` with the tables and a ranked list; a re-runnable `tools/research3/mp/measure.*` that prints the same numbers for any app.

### MT — f30_cluster textures: every way to make them lighter
Goal: all options, each with its saving, its cost in picture quality and its work, so the owner can choose. Works from MP's inventory as soon as it is published (start with your own listing of `apps/f30_cluster/hud/scene` and `hud/baked`).
For every texture f30 uses: what it is, where it is sampled, its format, size, bytes, how much of it is non-trivial (entropy, constant or symmetric regions, channels that carry nothing). Then measure each option on the real assets:
1. **Format**: RGBA8 → R8 (luma + tint in the shader, as Twin Dial), RG8 (luma + alpha), fewer bits; which textures qualify.
2. **Channel packing / merging**: several R8 maps (albedo, shadow, glow, highlight, mask) into the channels of one RGBA texture; several small textures into one atlas; what it does to the sampler count and to the "every sampler bound" rule.
3. **Resolution**: which textures survive ½ or ¼ scale (many are already `@0.5x`), by pixel difference in the final rendered frame, not in the texture.
4. **Symmetry and reuse**: mirrored halves, radial symmetry (a 1-D radial profile instead of a 2-D face), sharing one texture between dials, between the three themes, and between a theme and its G20 variant; keeping shared textures across a theme switch instead of dropping and re-sending them.
5. **Procedural replacement**: what a shader can draw exactly (gradients, rings, ticks, vignettes) and what it costs in shader bytes against the KOMBI's size limit.
6. **Compression**: whether cluster-render's protocol accepts any compressed or palettised format, or a compressed transfer (read its PROTOCOL and client code, and the upstream repo's source if needed with generic web requests only); what the wire format costs today (JSON, base64?) and whether a denser encoding of the same pixels is possible on our side.
7. **Upload strategy**: order (what the driver needs first), lazy upload of what is not visible, preloading the next theme while idle, dirty-rect instead of whole re-sends, caching across layout switches within the 16 MiB budget and its peak.
For each: bytes before and after, pixel memory before and after, the difference image and its numbers against today's render (use `tools/shots` states), the risk, and an estimate of the work. End with **three packages of options**: "safe and cheap", "the best trade", "everything", each with its total saving in upload bytes, pixel memory and switch time.
Done: `MT-REPORT.md`, `design/research3/mt/textures.json` (the inventory with every measurement), and a prototype for the two most promising options on your branch, behind a config key, with its render diff.

### MB — the black rectangle: what differs between the app that has it and the app that does not
Goal: turn "f30 has it, Twin Dial does not" into the shortest list of candidate causes and the experiments that separate them.
1. **Trace both apps on the bench**: every message to cluster-render (connect, slot, content id and class, client id, layer create/update/destroy with rectangles, map/unmap, asset uploads, timing) at start, at a drive-mode switch, at day/night, at a message. Diff the two traces: what f30 does that Twin Dial never does.
2. **Geometry**: for f30's themes and Twin Dial, every layer rectangle and every opaque pixel against BMW's gear letter [1398, 668, 1424, 703], the gap [1380, 660, 1460, 720], the box's own area [1406, 342, 1597, 419], BMW's map slot and message panel — in each state. Twin Dial keeps the gap as a gap in layer rectangles; say exactly what f30 does there in each theme and state (rectangle over it, transparent pixels, opaque pixels).
3. **Theories to rank with the evidence**: (a) a layer rectangle (not pixels) over the gear letter; (b) the rebuild itself — f30 re-makes its layers on a drive-mode switch, and the owner saw the box appear after switching, while Twin Dial never rebuilds on a mode switch; (c) layer count or pixel memory near a limit; (d) the map slot or a content id used only by f30; (e) timing between unmap and map. For each: supported, weakened or refuted by the diff and by `docs/boxes/*`.
4. **Experiments, ready for the car** (branches, off by default, each a single setting): f30 with the gap cut out of its layer rectangles; f30 that does not rebuild on a mode switch (or rebuilds in a different order); Twin Dial with the gear letter covered (`gearCover`); Twin Dial forced to rebuild repeatedly. For each: the exact steps for the owner, what to photograph, and what each outcome proves.
5. **A draft for the upstream issue** (`docs/research3/MB-UPSTREAM-DRAFT.md`): the evidence in neutral terms, no personal data, no car identity; the lead decides whether it is posted.
Done: `MB-REPORT.md` with the trace diff and the ranked theories, the experiment branches, the owner's protocol.

### MC — messages: why f30_cluster's window over BMW's panel stays open
Goal: every path in 0.7.20 that can leave the window open after BMW's message is gone, each tied to what the car's log would show.
1. **Enumerate from the code** every state in which the window is open and what closes it: `statusActiveCCMessage` never notified (the heuristic: 20 s, and red entries for as long as listed); it notifies with a resting value that is not in `noneValues` (the catalogue default 65535, or the last number kept); its number is not a `ccList` id (no entry → open); a stored entry of a kept colour; the latches (`_rowFail`, missing glyph); the opaque themes' blind case. Reproduce each on the bench with injected signals and record the log lines it produces.
2. **Compare with Twin Dial's machine** (`shared/twin_cc.js`, strict D7, two witnesses): the same inputs through both, where they disagree and which is right.
3. **Search `bake/work/` for a saved car log** of 2026-11-17 and, if one exists, match its `active cc:` / `ccList` / `no ccList entry` lines to the cases (never copy message text; numbers and lengths only). If none exists, say so.
4. **A decision table for the lead's next log read**: "if the log shows X, the cause is Y, the fix is Z" for every case, and the candidate fixes implemented on your branch behind config (for example `noneValues` with 65535, a time limit on a number with no entry, the second witness `Vehicle2.statusDisplayedCCMessage` for f30, closing when `ccList` returns to its resting entry) — each with a test that fails without it, and none that could hide a BMW warning without our text on screen.
Done: `MC-REPORT.md`, the decision table, the fixes on your branch, off by default.

### MK — trip computer: empty values on Twin Dial
Goal: say whether it is "by design until the factors are set" or "the car does not send it", and prepare both ways out.
1. **Trace the path** in `shared/twin_tokens.js` from the three fields (`statusBordcomputerAverage`, `statusBordcomputerTravelData`, `statusBordcomputerTravelStart`) to the page: what shows with `config.trip.factors` null (the shipped state), what the normal build logs when a field arrives, and whether that log line exists at all. If the shipped build cannot tell "silent" from "unscaled", add the missing log line (numbers only) on your branch.
2. **Candidates from the catalogue** for every trip value (average consumption, average speed, distance, duration, departure time): the KombiIF fields, `ResetLogic3.bcDataAll`, and anything else the catalogue's search finds; getter and notifier ids, member types, the plausible scalings and how to tell them apart from one reading against the iDrive trip page.
3. **The owner's decision (2026-11-18): no more dashes by design.** The signals ARE the trip computer; a "factor" is only the unit of the raw integer, which the catalogue does not state. So on your branch: (a) **the most likely factors on by default** (`config.trip.factors` = speed 0.1, cons 0.1, way 0.1, bc 1, or better ones if your catalogue work shows them), each value marked "unverified" once in the log, so the first drive compares them with the car's own trip page and a wrong one is a one-line config fix; (b) **one log line per field on first arrival and a summary after 2 minutes that names every silent field** (numbers only), so the shipped build itself tells "the car does not send it" from "the unit is wrong"; (c) a fallback chain (the first field that delivers a valid value wins, e.g. KombiIF fields → `ResetLogic3.bcDataAll`). A dash stays only for a field that is silent or carries its invalid marker. Tests with injected values, including the invalid markers and a silent field.
4. **The owner's protocol**: what to read on the car's own trip page and what to compare, in five minutes.
Also: the odometer in the bottom row (`mileageKilometre`, unread until now) and the range/trip slot R1 — the same question, the same treatment.
Done: `MK-REPORT.md`, the two builds, the protocol.

## 4. What the lead does with it

The lead merges nothing automatically: reads the five reports, writes one car-session plan (log read first, then the experiments in an order that separates the theories), and brings the choices to the owner — above all MT's three option packages for f30's textures.
