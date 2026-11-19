# Brief: Twin Dial, round 3 — live tabs, the pages behind them, the dials re-measured

Audience: ten agents, packages **SD, SF, SB, SM, SP, SN, SV, SC, K1, K2**, working in parallel, each in its own git worktree. Read §0–§3 and your own package in full; skim the others. This round is **analysis, design and mockups, plus one prototype** (SB's tab setting): its product is `docs/twin-dial/SPEC-R3.md` (written by the lead from your reports) and a set of mockups the owner chooses from. The accuracy of the analysis matters more than its speed: every number measured, every signal read from the catalogue member by member, every claim labelled.

## 0. What the owner said (2026-11-18)

Twin Dial 0.1.0 is on the car. Looking at it against the references:

1. **The tab bar is dead.** The five tabs never light. They must be **live tabs**: the selected tab is highlighted and the centre shows that tab's content.
   - Tab 1 (pump + range): **the on-board computer** (today's clock and trip pages).
   - Tab 2 (triangle): **the messages, in the centre.** Decided: our message moves from the bottom plate to the centre page of tab 2; the bottom plate goes.
   - Tab 3 (radio/media) and tab 4 (phone): **open.** "Search the signals for phone and radio, the deepest possible analysis"; then evaluate, and **draw several mockups** of what could be there — including what the references' maker draws there and alternatives from data we already have. Either tab may be removed or replaced.
   - Tab 5 (arrow): **navigation: the map in the centre.**
2. **Switching tabs:** find the button's signal (steering wheel, stalk, controller — whatever the car can tell us) **and** add a setting in the app.
3. **The dials need another look**, with two findings of the owner's own:
   - the tachometer's **red zone is not drawn to the end** (ours stops at "8", grey band follows);
   - each dial has **three rings — two thin ones and the thick one between them; the innermost thin ring has a longer arc** in the references than ours.
   Assess geometry, gradients and highlights again, with the new references.
4. **Speed limit:** find where it can be drawn.
5. **The small speedometer numerals stay as they are.**
6. **Deferred, do not design now:** drawing BMW's telltales ourselves at the top and making the whole bottom ours. Record what the references do with telltales, nothing more.

The lead's own measurements so far (hypotheses for you to confirm or correct, not facts): on reference 06 the tachometer's bezel fits centre (340.5, 410.0), R 152.2 px; the inner rim is at 0.540 R (ours 0.545) but about half as thick as ours; the band runs 0.575–0.940 R with the tick strip at 0.89–0.94 and a brighter line at its outer edge; band colours on 06 (its black is 20, not 0): inner top (41, 45, 46), mid (50, 54, 59), outer (68, 77, 81), strip (98, 107, 110) — flatter, lighter and cooler than ours (inner (26, 27, 28), outer (74, 76, 79), strip 137); 06's numerals carry a soft shadow; 06 has no small speedometer numerals and red marks on the strip at about 30, 50 and 130. Side-by-side crops: `bake/work/twin-dial/mockup/check/tach_ours_06_08.png`, `speedo_ours_06.png`.

**The pattern the lead reads from the references (confirm it, SF):** a filled tab means *selected*, not *warning*. On 02 and 06 the first tab is filled and the centre shows a page of the on-board computer (trip memory; date and time). On 08 the warning tab is filled. Round 1 (VF) read the fill as severity; that was wrong.

## 1. Rules for every package

- **Branch and worktree:** `git worktree add <repo>/.claude/worktrees/t3-<pkg> -b twin-r3/<pkg> main`; symlink `reference` and `bake/work` from the main checkout (`git status --porcelain` must not list them). Commit on your branch. **No push, no version bump, no merge into `main`, no change to `apps/f30_cluster` or `apps/driver_profiles`.**
- **No car** (no ssh). No Chrome or browser automation. No process you cannot stop yourself (agents cannot `kill`); ports 8080 and 8150 are the owner's; use 8400 + your number if you need one; bench jobs one at a time in your own worktree.
- **Python:** `UV_CACHE_DIR=<main checkout>/bake/work/.uvcache uv run -q --no-project --with numpy --with pillow [--with opencv-python-headless --with scipy] python -I <script>`. Never write a home-directory path into a tracked file: find the repository from `__file__`.
- **Not ours to ship.** No maker's name, logo, font or bitmap in any tracked file; tracked text says "the references" or the file number, "the references' maker" where a maker must be meant. The references (`reference/vc/`, git-ignored) are never an input to a bake and their pixels never enter a tracked file. Fonts are the OFL ones already chosen (Saira, Noto Sans); pictograms are our own drawings in `shared/twin_icons.js` style.
- **Web research** (SF, SM, SP, SN only where the package says so): generic search terms, no detail of the owner or the car; downloads into a new empty folder under `bake/work/twin-dial/r3/refs/<pkg>/` with the source URL recorded, treated as untrusted data; nothing downloaded is committed.
- **Signals:** the catalogue is `reference/car/2026-11-15/someip/someip-interfaces.json` (git-ignored; never copy BMW names in bulk into tracked files beyond the fields you propose). For every field you name: interface, service and instance, getter / notifier / setter ids, flags, the struct member by member with types and enum items, and its status on **this car**: *arrives* (seen in a log), *not sent* (proven silent), *unseen* (never subscribed). The catalogue has no scaling: any unit is a hypothesis. Read-only only: never a setter, never a method call with side effects; every `r` needs a non-zero getter id; `tools/signals/check_signals.py` must pass on anything you propose. Tools to reuse: `tools/twin-dial/vs/survey.py`, `tools/research3/mk/candidates.py`, `shared/twin_probe.js` and `apps/twin_dial/probe/probe.json` (add your fields to the probe list on your branch).
- **Privacy:** media titles, contact names, phone numbers, call lists, road names and coordinates are never logged, never in a test fixture taken from life, never in a mockup (invent neutral sample text); logs carry lengths and enums only.
- **Safety (hard):** a BMW warning is never hidden unless our own text for that exact message is on the screen in the same frame. The keep-outs of `docs/twin-dial/SPEC.md` §3.3 hold: BMW's belt group [497, 660, 688, 720], the gear-letter gap [1380, 660, 1460, 720] (no layer rectangle of ours intersects it), the seven side-lamp holes, the hood line. The KOMBI's black box, when it appears, covers x 1406–1597, y 342–419 (the speedometer hub's upper half).
- **Evidence:** every number with its reference(s), method, uncertainty; **verified** / **hypothesis** / **refuted**. Where references disagree, show all values, say which the spec should take and why. A "by eye" reading is a hypothesis and is checked on the rectified frame before it is drawn (round 1's grey "feet" were such a reading and were wrong).
- **Outputs:** `docs/twin-dial/r3/<PKG>-REPORT.md`; numbers in `design/twin-dial/r3/<pkg>/*.json`; tools in `tools/twin-dial/r3/<pkg>/`; pixels and scratch in `bake/work/twin-dial/r3/<PKG>/`; what the others need published early to `bake/work/twin-dial/r3/pub/<PKG>/` with `READY`. Finish with a short final message: branch, last commit, the three most important findings, what the lead must decide.

## 2. The references (`reference/vc/`, git-ignored)

| # | File | What it is | Good for |
|---|---|---|---|
| 01 | `01_classic_map_tacho-left_mph.png` | classic generation, map | cross-check only |
| 02 | `02_facelift_hybrid_powermeter_trip.png` | facelift, real car, phone photo; first tab filled, trip page | geometry master of round 1; tab bar; bottom row |
| 03 | `03_classic_compact_date-panel_led-bars.png` | classic | cross-check only |
| 04 | `04_facelift_bench_red-wedge_car.png` | facelift-style bench unit; warning tab filled, car picture | red wedge, bars under the dials |
| 05 | `05_facelift_render_settings.png` | facelift render | levels |
| **06** | `06_facelift_render_date_side-gauges.png` | **facelift, a clean frontal render, 1280 × 720**: tachometer with gear in the hub centre and the program under it, red arc on the bezel, side gauges, first tab filled with "date and time" in the centre, telltales in the bottom row | **the best new frame: rings, gradients, numerals, tab fill, page grammar** |
| 07 | `07_classic_map_nav-tab_turn-hub.png` | classic, 3750 × 2151: navigation tab selected (outlined), map between the dials, turn arrow and lanes in the speedometer's hub, gear in the tacho hub centre | navigation content, hub use |
| 08 | `08_facelift_bench_red-wedge_car_v2.png` | the bench unit again, sharper: six tabs, warning tab filled red, wedge to the band's end, car picture, voltage | wedge extent, tab count, car page |
| 09 | `09_facelift_render_map_unit.png` | a seller's render: map in the centre, bottom row with trip, clock, temperature | map framing (low trust) |
| 10 | `10_facelift_reduced_nav-menu.png` | facelift, the **reduced view**: small dials in the lower corners, a navigation menu list in the centre with a selection bar, car picture with an open door, speed-limit sign at the bottom right | menu page grammar, reduced view, sign placement, car picture |
| 11 | `11_classic_map_comfort_sign.png` | classic: map, program under the gear, sign at the map's lower right | sign placement, hub labels |

Round 1's rectified frames and tools: `bake/work/twin-dial/pub/VR/`, `tools/twin-dial/vr/`, `tools/twin-dial/vd/`, `tools/twin-dial/vf/`, `tools/twin-dial/vc/`. The current spec: `docs/twin-dial/SPEC.md`; the built app: `apps/twin_dial`, `design/gen_twin.py`, `tools/twin-dial/bake_face.py`, `shared/twin_*.js`; the mockup renderer: `tools/twin-dial/mockup/mockup.py`; the real app's state gallery: `tools/shots/twin/` (output `bake/work/twin-dial/r2/E/gallery/real-full/`).

## 3. What exists on the platform side (do not rediscover)

- Twin Dial's engine contract: `docs/twin-dial/ENGINE-CONTRACT.md`; budget today 14 layers (6 / 6 / 2 per connection, limit 8 each), 4.7 MiB steady; the centre is repainted as a sheet; changing the `centre` setting rebuilds the layout today (research3/MP: it should be applied in place).
- Messages: `shared/twin_cc.js` (states none / window / ours, strict D7 latch, plate painter), `docs/twin-dial/R2-D-REPORT.md`, `docs/research3/MC-REPORT.md` (why the window hangs: the "shown now" number 65535 or a number the list no longer carries; the `release` fix).
- Signals known to arrive on this car: speed, rpm, gear, drive program, vehicle condition, fuel / range / reserve, coolant, oil, ambient, the clock, day/night, `ccList`, road label and `guidingState`, the speed-limit struct (f30_cluster's `signOf`, `config.sign*`). Unseen: trip computer, odometer member, media, phone, doors, voltage, the cluster view / stalk button (`HMIContext.currentICWidget`, `statusKombiSelection`), the next turn. `docs/twin-dial/VS-REPORT.md`, `docs/research3/MK-REPORT.md`, `docs/twin-dial/PROBE-PROTOCOL.md`.
- BMW's map: a separate slot of about 1088 × 480 under us; f30_cluster's `stockMap` hole; `docs/boxes/*` and `docs/research3/MB-REPORT.md` (the black box; one earlier finding says mapping the panel slot makes the KOMBI disable the map slot — read it before promising a map).

## 4. Packages

### SD — the dials, measured again
The owner's two findings first, then everything else, on **every** reference that shows it (02, 04, 05, 06, 08; 07 and 11 for the hub; 10 for the reduced dials), rectified as round 1 did (rectify 06–11 with `tools/twin-dial/vr/`; publish the frames to `pub/SD/`).
1. **The three rings and their arcs.** For the outer thin ring (bezel), the thick band and the inner thin ring (rim): radius, width, and **the start and end angle of each arc** on both dials, with how each end is finished (square cut, radial cut, fade, a foot). State exactly how the inner ring's arc compares with the band's and the bezel's, per reference, and how ours differs (`tools/twin-dial/bake_face.py`, `RIM_OPEN`, the baked faces). This is the owner's finding: get it right to half a degree.
2. **The red zone.** Start (per reference: value and angle), end (the scale's last mark, the band's end, the bezel's end?), inner and outer radius, wedge fill (04, 08) against arc on the bezel (06, 05), how the numerals 7 and 8, their ticks and the radial lines sit on it, where "8" sits relative to the zone's end. Ours stops at 125° with the band going on to 144°.
3. **The scale laws again** with 06: tachometer 0 and 8 angles, where the needle rests ("OFF"), speedometer law, what is at the band's ends.
4. **The band's look:** radial and angular gradient (linear light, per reference, after its own black level), the tint (06 is cool: quantify per channel), the tick strip (level, the brighter line at its outer edge, the dark gap to the bezel), the major lines across the band, any sheen or highlight (a lighter sector, the lit lower ends), the hub disc's level against the surround.
5. **Numerals and ticks:** size, weight, the soft shadow under 06's numerals (offset, blur, strength), outlined digits in the hub, the red marks on the speedometer strip (values and meaning), minor/major tick counts per reference.
6. **The hub's content:** gear in the centre with the program under it (06, 07, 11) against gear low in the hub (02): positions and sizes in R; READY, OFF, the unit label (curved outside the lower right on 06, 08, 02); the needle (colour per reference, root, length, whether it crosses the rim).
7. **What is under and beside the dials:** bars under the dials (04, 08), side gauges (06), the reduced view's dials (10: size, position, what remains on them).
8. **Our render against it:** the same measurements on the real app's picture (`bake/work/twin-dial/r2/E/gallery/real-full/A/day_none_clock_driving.ours.png`), a table "element — references — ours — change proposed", and `design/twin-dial/r3/sd/dials_v2.json` in the format of `design/twin-dial/vd/dials.json`.
Publish the ring and red-zone numbers to `pub/SD/` as soon as you have them: K1 draws from them.

### SF — the frame, the tabs, the page grammar (with web research)
1. **Tab semantics.** From every reference: which tab is filled or outlined, what the centre shows then, what each tab carries (icon, text, value), how many tabs (five on 02 and 06, six on 08), which appear only sometimes (the warning tab), the fill colours (red on 02 / 06 / 08, another on 05, the outlined trapezoid of the classic 07 / 11), whether the rule under a filled tab changes. Confirm or refute "filled = selected".
2. **What the references' maker draws behind each tab** — web research (owner's manuals, press material, videos' stills; generic terms): the page tree of each tab (on-board computer pages; warnings list; radio and media lists, the now-playing view, cover art; phone: call list, contacts, an incoming call; navigation: map, menu), what a scroll or selection looks like (10's selection bar, 05's scroll bar), what happens on an incoming call or a new warning (does the view switch by itself?), how the reduced view changes it. Record sources; describe in words and measured numbers, never copy artwork.
3. **Page grammar, measured** (rectified 02, 05, 06, 08, 10): title and its rule, body rows, icons in rows, the selection bar, the scroll bar, alignment and spacing rules, in the frame units of `docs/twin-dial/VF-REPORT.md` and in panel px for our layout (`design/twin-dial/vg/layout.json`).
4. **The tab bar re-measured on 06** against VF's numbers from 02 (cells, separators, ramp, the fill's colour and extent, text and icon sizes); what our bar gets wrong today (compare the real app's picture and the owner's crop `bake/work/twin-dial/mockup/check/owner_tabbar_crop.png`).
5. **Telltales and the bottom row** on each reference (where lamps sit: above the bar on 02, in the bottom row on 06 and 10): **record only** (deferred by the owner).
6. **Speed-limit sign** on 10 and 11: position, size, relation to the bottom row and the map.
Output: `SF-REPORT.md`, `design/twin-dial/r3/sf/{tabs,pages}.json`, published early for K2 and SC.

### SB — the button: how the driver switches tabs (and the setting)
1. **Catalogue search, exhaustive**, for anything that changes when the driver presses a button that could select a view: the turn-signal stalk's on-board-computer button, steering-wheel buttons (multifunction wheel: up / down / OK / menu wheel, voice, phone, media skip), the controller and its buttons, the cluster's own selection state (`statusKombiSelection*`, `statusConfigurationKombi*`, `statusDisplayKombi`), the head unit's idea of the cluster (`HMIContext` / `HMIContext2` `currentICWidget`, `currentICGraphicalContext`), entertainment-list and telephone-list state in the cluster, anything named operation / key / button / MFL / steering wheel / BC / list / scroll / selection. For each candidate: the full member table, the reasons it might or might not move on a press, and whether reading it could have a side effect (none may).
2. **Other evidence:** the 2026-11-15 car snapshot logs (`reference/car/`), every upstream app's source for how it reads a button (`kit/`, the bimmer-lab engine in `shared/`, and the public repos `cauliflowermossgrape/idrive7-apps`, `bimmer-lab/apps`, `robertnisipeanu/idrive-apps` — read what is vendored first; fetch only public files, generic requests), the RHMI side (can a centre-screen action or the kit's `ctx` give an app a key event while its screen is closed?), f30_cluster's own notes (`docs/FEATURE-AUDIT.md` §6.1 says nothing in the catalogue is the button itself — verify, don't repeat).
3. **Ranked candidates and the probe:** which fields to log, the exact presses for the owner (each button, short and long, with and without a message, with the stock cluster's menus open), the log lines, and the decision table "if X moves on press Y → it is our tab key". Add the fields to the probe build on your branch (`probe.json`, `twin_probe.js`), numbers only.
4. **The tab model and the setting — a prototype on your branch**, off the car: a `tab` state in the engine (1 computer, 2 messages, 3, 4, 5 navigation), the filled tab drawn by `tdback` (the highlight exists in the contract: reuse it for selection), the centre following it, and the app setting **"Tab" = auto / 1 / 2 / 5** (3 and 4 once they exist) applied in place (no layout rebuild). **Auto rules** to propose and implement: a new message → tab 2 until it ends, then back; guidance active → tab 5; otherwise the driver's last choice; never a tab change above a speed the owner can set, except for a message. A key source is pluggable (`config.tabKey`: null until the probe finds one). Tests: `bench/test-twin-tabs.js`, the twin and f30 suites unchanged.
Output: `SB-REPORT.md`, `design/twin-dial/r3/sb/candidates.json`, the probe additions, the prototype.

### SM — radio and media: the deepest possible signal analysis
1. **Every media field in the catalogue**, member by member: the head unit's media state (`HmiMediaState.playerMetadata` and its siblings, instance ids), `MediaSource` (active source, the sources list, player types), tuner / radio (station name, frequency, band, RDS / DAB text, presets, station lists), online and projected audio (CarPlay / Android Auto now-playing, Bluetooth audio metadata), playback state and position, track number and counts, cover art (is there any image field, its format and size — and whether using it is ours to draw), entertainment lists the cluster shows (the stock "entertainment list" in the cluster: which interface feeds it), volume / mute.
2. **Which of them the stock cluster itself uses** (the G20's own media line and list in the cluster): find the interface that feeds the KOMBI, since that one certainly carries data on this car.
3. **Evidence on this car:** the snapshot logs, other installed apps' declarations (does any app on the car already read media?), instance pitfalls (`HMI_CID`), text encodings, update rates.
4. **Ranked plan:** the fields for a now-playing line (tab text), a now-playing page (source, title, artist, album, progress), a station / track list (if a list field exists), with fallbacks per source type; what is impossible. Probe additions with the owner's steps (FM, DAB, USB, Bluetooth, CarPlay; play, pause, skip, change source) and the decision table.
5. **What the references' maker shows in the media tab** (with SF's research): the content to imitate with our data.
Privacy: titles never logged — lengths, enums and source types only.
Output: `SM-REPORT.md`, `design/twin-dial/r3/sm/fields.json`, the data model for K2 (`pub/SM/model.json`: what a page may show and which parts are certain).

### SP — phone: the deepest possible signal analysis
The same method as SM for telephony: `ConnectivityAggregator`, `Telephone` (both instances), CarPlay / projection telephony, Bluetooth device state — connected device (name is private: length only), signal strength, phone battery, roaming, network name, call state (incoming, active, held, dialling, ended), call duration, caller identity fields (never logged, and decide whether showing a name is acceptable to draw at all), missed calls / call lists, the microphone-mute state, the stock cluster's telephone list feed. Which the stock cluster uses; what arrives on this car; the probe steps (connect, ring the car, answer, hang up, miss a call, CarPlay on); the decision table; the data model for a phone tab (`pub/SP/model.json`), and an honest verdict whether a phone tab is worth having with what is certain. With SF: what the references' maker shows there.
Output: `SP-REPORT.md`, `design/twin-dial/r3/sp/fields.json`.

### SN — navigation: BMW's map in the centre, and the guidance content
1. **Can BMW's map show in our centre at all?** Read `docs/boxes/*`, `docs/research3/MB-REPORT.md`, the cluster-render protocol and client, f30_cluster's `stockMap` hole and the photos' records: does the KOMBI draw its map while the panel slot is ours (one finding says it disables the map slot when the panel slot is mapped — establish exactly what was observed and on which setup); where BMW's map lands on the G20 panel (its rectangle against our centre window [729, 172, 1191, 613] and the dials); which stock cluster view must be active for the map to exist (the driver's cluster setting — a link to SB); what else BMW draws with it (its own frame, a road bar, a compass, lane guidance) and whether those would show through.
2. **The window:** the exact hole a `tdback` needs (shape, per-state), what stays ours around it (title, rule, bottom row), what happens when the map is not there (fallback page), the relation to the black box (theory H-M: the box follows BMW's map).
3. **Guidance content from signals** (for a fallback page and for the tab's text): `guidingState`, `currentRoad`, `Guidance.nextIntersection` and every sibling (distance, direction / manoeuvre type, lanes, road names, arrival time, remaining distance), member by member; what arrives; the probe steps; what a turn arrow needs (our own pictograms).
4. **What the references show** (01, 07, 09, 11): the map's extent between the dials, what sits in the speedometer's hub on 07 (turn arrow, lanes), road label bars.
5. **Experiments for the car**, each a config key, off by default: the window with the stock map; each with what to photograph and what each outcome proves.
Output: `SN-REPORT.md`, `design/twin-dial/r3/sn/{map,guidance}.json`, the model for K2.

### SV — other content for tabs 3 and 4, and the speed limit
Candidates the owner will choose between, each with its signals proven or not:
- **Vehicle / technical page:** oil and coolant temperature, boost, battery voltage, tyre pressures and temperatures, as rows with bars (what f30_cluster's engine already reads and what arrives; `shared/hud.js` tokens).
- **Car picture with doors and lids** (references 04, 08, 10): door and lid contacts, our own top-view drawing, what it costs (texture, layers).
- **Driver assistance page:** the speed-limit sign large, no-overtaking, cruise / limiter set speed, distance warnings — what the car sends (`signOf`, f30's sign config, anything about set speeds).
- **Consumption / efficiency page:** instantaneous and average consumption, a history we compute ourselves, range.
- **Anything else the catalogue makes easy** (compass, altitude, date, service info) — list with a one-line verdict.
For each: fields member by member, status on this car, the probe steps where unseen, the data model (`pub/SV/model.json`), cost in layers and pixel memory.
**Speed limit placement**, measured against our layout and the keep-outs: (a) a sign in the bottom row's right end (the references' place: 10, 11) — the free rectangle between the bottom row and the gear gap, in px; (b) a marker on the speedometer's scale at the limit (shader); (c) under the unit in the speedometer's hub (clear of the black box's area); (d) large on an assistance page. For each: the rectangle, clearances, what it needs from the engine, how it behaves with no limit / unknown / conditional limits.
Output: `SV-REPORT.md`, `design/twin-dial/r3/sv/*.json`.

### SC — messages in the centre (tab 2): design and safety
The owner's decision: the message leaves the bottom plate and lives in the centre, on tab 2.
1. **Design** (with SF's page grammar and 08's warning state): the message card in the centre window (symbol, headline up to how many lines at which size, the counter, severity colour — and where severity shows now that a filled tab means "selected"), the **list of messages** (the car's list carries stored entries too: which to list, in what order, how a stored one differs from the one shown now), the empty state ("no messages"), what the warning tab looks like with no message (hidden, dim, or always there).
2. **The safety machine re-derived.** BMW's panel is at the bottom [696, 595.75, 1226, 704]; our bottom row covers it; our text is now in the centre. State precisely, frame by frame: a message appears → the fallback window opens (bottom row steps aside) → tab 2 is selected and our card is painted → only after our card for that exact message has been on screen `CC_SETTLE_MS` may the bottom row return and cover BMW's panel. If the driver or an auto rule leaves tab 2 while the message still shows: the window must open again (our text is no longer on screen) — or the tab may not be left; choose and argue. D7 (never cover until the car has confirmed "shown now"), the eight keep-open conditions of `SPEC.md` §8.4 and research3/MC's `release` cases all re-stated for the centre. Give the state table and the tests (`bench/test-twin-cc.js` cases re-written for the centre; the per-frame pixel audit: BMW's panel is covered only while our card for the shown number is fully on screen).
3. **Cost:** what the plate's removal frees (a layer, pixel memory) and what the card needs.
No engine code this round beyond what a test harness needs; the state machine as a table plus a model in `tools/twin-dial/r3/sc/` that the tests of the build round will reuse.
Output: `SC-REPORT.md`, `design/twin-dial/r3/sc/{card,states}.json`, the model for K2.

### K1 — mockups: the dials
Extend `tools/twin-dial/mockup/mockup.py` (keep the old pictures reproducible; new output under `bake/work/twin-dial/mockup/r3/`). Start at once from the lead's numbers in §0 and re-render when SD publishes (`pub/SD/`).
- **Fixes:** the red zone to the band's end with "8" wholly inside; the three rings with their measured arcs; the inner ring's width.
- **Variants, each a full 1920 × 720 picture and a dial close-up, side by side with the reference crop:** band look A (today) / B (06: flatter, lighter, cool) / C (between); red zone as a wedge (04 / 08) / as an arc on the bezel (06) / both; numerals flat / with 06's soft shadow; gear low in the hub (today) / in the hub's centre with the program under it (06); the tick strip with its bright outer line; needle white (decided) with and without the shadow.
- A contact sheet the owner can choose from (`r3/dials_sheet.png`) with each variant labelled, and a short `K1-REPORT.md` naming what each variant costs to build (face re-bake, shader bytes).
The small speedometer numerals stay. Every picture respects the keep-outs (draw the protected zones on one annotated copy).

### K2 — mockups: the tabs and their pages
With `mockup.py` (shared with K1: put your code in `tools/twin-dial/mockup/pages.py` and import the dial drawing; agree nothing else). Start from SF's pattern in §0 and today's layout; re-render as SF, SC, SM, SP, SN, SV publish their models (`pub/<PKG>/`).
- **The live tab bar:** each of the five tabs selected in turn (the fill, the rule under it, icon and text colours), the warning tab with and without a message, a five-tab and a four-tab bar (one of 3 / 4 removed), a bar where tab 3 or 4 carries another icon.
- **Tab 1:** date and time; trip computer (two pages) — as today, in the new grammar.
- **Tab 2:** the message card (info, yellow, red; one line and two lines), the list with several messages, the empty state; the same moment with the fallback window open at the bottom (BMW's panel drawn as the bench draws it).
- **Tab 3 / 4 — at least two variants each from the signal packages, more if the data supports it:** media now-playing (radio: station; track: title, artist, source, progress), media with only what is certain; phone (idle, incoming, active call); and the alternatives: technical page, car picture with a door open, driver assistance with the limit, consumption. Label each with its data status (arrives / unseen) from the models.
- **Tab 5:** the centre as a window on BMW's map (use a neutral drawn map stand-in, clearly not a real place), with our title and rule or without; the fallback guidance page (arrow, distance, road label placeholder).
- **Speed limit:** SV's placements (a)–(d), each on the driving picture, alone and combined.
- Sample text is invented and neutral (no real station, person, number or place); Ukrainian labels as today.
- A contact sheet per tab and one overview (`r3/pages_sheet.png`), and `K2-REPORT.md` listing every picture with its data status.

## 5. Order and hand-offs

Everything starts at once. Early publications that unblock others: **SD → K1** (rings, red zone: within the first hour), **SF → K2, SC** (tab semantics and page grammar), **SM, SP, SN, SV, SC → K2** (models). K1 and K2 render first from §0 and today's spec and re-render on each `READY`; they finish last.

## 6. What the lead does with it

Reads the ten reports, writes `docs/twin-dial/SPEC-R3.md`, shows the owner the contact sheets with the open choices (band look, red zone, gear position, tabs 3 and 4, speed-limit place, leaving tab 2 during a message), writes one probe protocol for the next car session (button, media, phone, guidance, map window), and only then plans the build round.
