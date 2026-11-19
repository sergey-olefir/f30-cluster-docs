# Brief: the vehicle ahead on the cluster — graphics for Twin Dial (design first, then a test mode)

Audience: one agent, **FV**. Base: **`main`** at `9517e89` or later (Twin Dial 0.1.4; if the lead has merged `car-issues/ct` by then, start from that). The lead merges, bumps versions and pushes; you never push, never bump a version, never connect to the car. **Not started yet: the owner asked for the brief only (2026-11-19).**

## 0. What the owner wants

The car's front camera tracks the vehicle ahead, and we can read that (below). The owner: *"if we can read the signals, we can show them on the cluster too"*. So: **our own graphic that shows the car ahead** — that there is one, how far it is, whether we are closing on it — in Twin Dial's style, with the cruise state next to it where that helps.

Two reference photos of the kind of picture meant are in `reference/vc/` (git-ignored, **never copied into the repository, never traced**; they carry another maker's marks):

- `13_facelift_assist-view_left_lane-car.png` — a small scene in the left area: our lane in perspective, a green car in it, a second car outline behind; a compact strip, not a whole page.
- `14_classic_assist-page_centre_cars-in-speedo.png` — two things at once: (a) a **centre page** "driver assistance" with a title, a line "distance 5 · Comfort", and a road in perspective with distance bars and a car at the foot; (b) a **small pictogram inside the speedometer**, under the unit: two little cars, one above the other, green, with the set speed beside them.

Read them for the idea — a perspective lane, a car that sits nearer or farther, bars for the gap, a tiny status pictogram in a dial — and draw **ours**: our geometry, our car shape (a plain generic rear view, no maker's design cues), Twin Dial's palette and type. The older references `01`–`12` in the same folder show the style Twin Dial already follows.

## 1. What is known about the data (2026-11-19)

The car: BMW G20, front camera (KAFAS), **no radar, no adaptive cruise**, plain cruise control (DCC) with a limiter, and the front collision warning. Everything below is read-only and already declared in Signal Probe (`apps/signal_probe/fields.js`, set S14; `docs/car/CP-REPORT.md` §13). `docs/cruise-set-speed-brief.md` has the full table.

| What | Where | State |
| --- | --- | --- |
| Gap to the vehicle ahead | `GapVehicleAhead.DataFrontVehicle` (event) → `frontVehicleGapX` | Seen, **raw**: 57..1736 while driving; **4093 = no target**. Unit and scale unknown |
| Its speed | same → `frontVehicleSpeedX` | Seen, raw: about 1500 when both cars hold the same speed (so probably an offset encoding of relative speed). Unknown until calibrated |
| Lateral offset, object id, qualifiers | same → `frontVehicleGapY`, `frontVehicleObjectID`, the two qualifiers | Declared, not read yet |
| Type, moving / standing, width | `GapVehicleAhead.DataFrontVehicleExtended` | Declared |
| A second gap, with the lead's type and lamps | `GapVehicleAhead.GapVehicleDrivingAhead` | Declared |
| Up to six objects with lane, distance, speed | `KeepDistanceObjectsLegacy` (events) | Declared; may not be sent on a car without adaptive cruise |
| BMW's own collision warning request | `FASuS.requestWarnBrakeCoordinatorDisplay` | Declared |
| Cruise set speed and state | `DisplayDataServer.functionalLibraryData` → `cruise`, `ldm`, `ldmColour` | Seen and already drawn by Twin Dial (`config.cruise`, the `tdcru` readout, places C1 / C2) |
| Own speed | Twin Dial has it | — |

**Consequences for the design, and they are hard requirements:**

1. **The picture must not need metres.** Until a calibrated drive, the gap is a raw number with a known "none" value and a known direction (smaller = nearer — confirm against the log; state it as an assumption if you cannot). Design the display around a **normalised gap level** (for example five steps), computed from the raw gap and our own speed through a mapping kept in `config.frontVehicle` (raw range, the "none" values, thresholds, an optional scale to metres). When the scale is known the same picture shows a time gap; when it is not, it shows near / middle / far and never prints a number.
2. **"Closing"** (we are gaining on it) comes from the change of the gap over the last second or two, not from `speedX`, until `speedX` is understood.
3. **Honest states:** no target; target seen; target near; closing fast; data stale (no event for N ms → the picture goes to "no target" by itself, never a frozen car); signal absent on this car (the page says "Немає даних", like the other pages do).
4. **It is information, not a safety system.** No red "brake" symbol of our own invention, no sound, no flashing that imitates BMW's warning. BMW's own collision warning stays BMW's and is never covered (the safety rule of `docs/twin-dial/SPEC-R3.md` §5.4 holds; the tab-2 machine is untouched). If you propose showing BMW's warning state in our picture (from `requestWarnBrakeCoordinatorDisplay`), make it an option for the owner's decision, off by default.
5. **Cruise is not adaptive.** Do not draw anything that suggests the car keeps the distance by itself (no "set distance" bars that look like a chosen gap, no steering-wheel or "assist active" symbols). Bars may show the **measured** gap.

## 2. The package

### 2.1 Design (first, and the main deliverable)

Propose **two or three placements**, each drawn as mockups with the bench renderer in every state of §1.3, day and night, and say which you recommend and why:

- **A. A centre page** (the existing one-layer `tdpage`, rect [729, 248, 1191, 595], `shared/twin_pages.js`): a title, the lane in perspective, the car ahead placed by the gap level, gap bars, the cruise set speed if cruise is on. Decide how the owner reaches it: a value of the existing "Сторінка бортового комп'ютера" / centre setting, or a page that tab 1 shows; do not add a sixth tab without the owner's word.
- **B. A small pictogram in a dial**: for example under the speedometer's unit or in the tachometer's hub area next to the drive programme — two little cars whose spacing or count of bars shows the gap level; visible on every tab. Mind what is already there: the digital speed, the unit, the limit marker, the cruise readout places C1 [1456, 500, 140, 70] and C2 [1466, 588, 188, 64], the gear, READY and the programme in the left hub.
- **C. A compact strip** in the place of the bottom row's lower line or beside the cruise readout, if you find room that the safety machine does not need.

For each: the exact geometry in panel px, the colours as codes with the night values, the fonts (only Twin Dial's own atlases; check Cyrillic coverage for every string), what animates and how fast (a car that jumps between five positions reads badly: say how it moves, and keep any motion cheap — the page is an image layer repainted in pixels, the upload budget matters), and the cost: layers per connection (today 7 / 4 / 3 of 8), pixel memory, bytes uploaded per second while a car is being followed.

Write it as `docs/twin-dial/FV-SPEC.md` (through the `docs` symlink, not committed) with the mockups in `bake/work/front-vehicle/pub/FV/` (PNG; a contact sheet per placement; file names with numbers only).

### 2.2 A test mode on the bench and for the car

Build the **recommended** placement (and B as well if it is cheap) behind a Діагностика setting **"Авто попереду (тест)"** (key `diagFrontVehicle`, off by default; off = no subscription, nothing drawn, nothing changed):

- Subscribe `DataFrontVehicle` (declare the event as `_broadcasts`, as Signal Probe does; `tools/signals/check_signals.py` must pass for `apps/twin_dial/app.json`). Add `DataFrontVehicleExtended` only if the design uses it.
- A small decoder module (`shared/front_vehicle.js`): raw → `{ present, level, closing, stale }` through `config.frontVehicle`; pure, unit-tested with the raw values above, with a jittery gap (hysteresis: the level must not flicker), with the "none" value, with silence.
- One log line per state change, numbers only (level, raw gap, raw speed), rate-limited — these lines are what the lead uses to calibrate after a drive.
- The bench gets a way to inject the event (the fake-signs pattern, `bench/fake-signs.js`) and a test that walks every state, checks the layer budget (`bench/test-twin-budget.js`), and proves the safety machine's frames are unchanged with the mode on (`bench/test-twin-cc.js` audit).
- No new layer if the budget says no; never a layer rectangle over the gear gap [1380, 660, 1460, 720], the belt group or the side lamps; every R8 texture you add has a width that is a multiple of 4 (the car shears the others); `highp` for any pixel distance over 256 px; one look per layer source (shader size).

### 2.3 f30_cluster

Out of scope for building. One paragraph in the spec: where the same information could live in the 6WB look, and what it would cost.

## 3. Rules (as every round)

- Worktree `<repo>/.claude/worktrees/fv`, branch `front-vehicle/fv` from `main`; symlink `reference`, `bake/work` and `docs` from the main checkout. Never `git add` under `docs/`, `reference/`, `bake/work/`. No big files in tracked folders (the Store refuses a repository archive over 64 MiB).
- **No other maker's name, logo, font or bitmap anywhere in the repository**, in a file name, a comment or a mockup. The reference photos stay in `reference/vc/`.
- Read-only signals; never a setter or a method. Node 8.16 syntax. Privacy as always (nothing personal in logs; this feature has no text from the car at all).
- Bench ports 8960–8969 only (8080 / 8150 are the owner's); suites one at a time; do not run `tools/twin-dial/run_all_tests.sh` (the lead does). You cannot `kill` processes; no browser automation.
- Shared files (`shared/hud.js`, `shared/twin_pages.js`, `apps/twin_dial/app.json`): marked hunks only (`// front-vehicle FV:`). Agent CT may be changing `shared/twin_pages.js` (tab 5) at the same time: keep your page in its own functions.
- Hand-in: `docs/twin-dial/FV-SPEC.md`, the mockups, `READY` in `bake/work/front-vehicle/pub/FV/`, and a final message with branch, last commit, test totals, the costs, and the decisions for the owner (which placement; where the page is reached from; whether BMW's warning state may show; the calibration drive you need: following one car at a steady speed at two or three gaps, with set S14 on).
