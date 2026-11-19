# FV — the vehicle ahead on Twin Dial: design, test mode, costs

Agent FV, 2026-11-19. Brief: `docs/AGENT-BRIEF-front-vehicle.md`. Branch `front-vehicle/fv` (from `main` 9517e89, Twin Dial 0.1.4).
Status words as in `SPEC-R3.md`: **verified** = measured on the bench / by a test; **hypothesis** = needs the car;
**owner** = the owner's decision, still to take (§8).

Mockups (git-ignored): `bake/work/front-vehicle/pub/FV/` — **real app on the bench** (`tools/twin-dial/fv/shots.py`, the
state gallery's own driver and moderngl painter), contact sheets by `tools/twin-dial/fv/sheets.py`:

| File | What |
|---|---|
| `010` / `011` | whole frame, A + B: day level 2 with cruise; night closing |
| `012` / `013` | whole frame: B on tab 3 (day); A + B closing fast (night) |
| `014` / `015` | whole frame: C painted over the tab-3 frame (day level 2; night closing fast) |
| `100` / `101` | **A**, the centre page: all 13 states (test off, absent, no target, levels 5..1, closing, fast, cruise standby, limiter, no cruise), day / night |
| `200` / `201` | **B**, the pictogram in the speedometer's hub: the same states + tab 3, day / night |
| `300` / `301` | **C**, the strip in the bottom row's lower line: the same states, day / night — **not built**, painted over the bench frame |
| `400` | A, the five levels side by side (the glide's stops) |

The whole run with every picture's audit: `bake/work/front-vehicle/shots/run3/index.html`.

---

## 1. The data and the decoder (`shared/front_vehicle.js`)

**Source.** `de.bmw.dass.GapVehicleAhead.DataFrontVehicle`. The brief calls it an event; the catalogue says it is a **field with a
notifier only** (id 32770, eventgroup 2, no getter, no setter) — so it is declared `"s"` exactly as Signal Probe declares it
(`_broadcasts` would be wrong). `tools/signals/check_signals.py` passes for `apps/twin_dial/app.json` (verified).
`DataFrontVehicleExtended` is not used (the design needs neither the type nor the width).

**Decoded to** `{ state, present, level 1..5, closing 0/1/2, stale, absent, word, metres, timeGap }`, with every unproven number a
key of `config.frontVehicle` (defaults in `front_vehicle.js DEFAULT`, the shipped values in `app.json`):

| Key | Default | Why |
|---|---|---|
| `none` | 4093, 4094, 4095 | 4093 seen parked; 4095 the catalogue default; 4094 the family's "error" (hypothesis) |
| `qualNone` | 2 / `EVENT_DATA_NOT_AVAILABLE` | the camera says "no data" — no target whatever the gap says. 1 (reduced) and the unfilled 15 do not veto |
| `nearer` | `smaller` | **assumption**: a smaller raw gap is nearer (57..1736 while driving fits a distance) — to confirm in the log |
| `scale` | `null` | metres per raw unit, **only after calibration**; while null **no number is ever shown** |
| `assumeScale` | 0.1 | **hypothesis** (57..1736 → 5.7..174 m, a camera's range): used only to place the gap among the five levels |
| `levels.timeGap` | 0.9 / 1.4 / 2.0 / 3.0 s | boundaries 1\|2\|3\|4\|5 by the time gap at our own speed |
| `levels.metres`, `minKmh` | 8 / 15 / 30 / 60 m, 20 km/h | below 20 km/h (or speed unknown) by distance |
| `hyst` | 0.12 | a boundary must be passed by 12 % before the level moves; plus the median of the last 3 values of one object |
| `closing` | window 1.5 s (≥ 0.6 s, ≥ 3 values); 1.0 m/s; fast ≥ 4 m/s or time-to-reach < 4 s; release at 0.6× | the least-squares slope of the gap — **never `speedX`** (not understood). A new object id (cut-in / cut-out) starts the history again |
| `staleMs` | 2000 | no value this long → no target (never a frozen car). The field is notified on change: a gap that never changes would read stale — the log's `n` (values per line) gives the native rate to set this |
| `absentMs` | 10000 | nothing at all since the start → "Немає даних" |

States (brief §1.3): **absent** (nothing since the start) · **none** (4093 / qualifier / out of range) · **stale** (= none, by
itself) · **seen** (levels 2..5) · **near** (level 1) · **closing** · **fast** (closing fast). The words on the page: level 1–2
"близько", 3 "середньо", 4–5 "далеко"; with `scale` set: "1.8 с · середньо" (the time gap). Never metres on screen.

**Log** (for the calibration): one line per change of (state, level, closing), at most one a second, plus a `hold` sample every
5 s while a car is followed; numbers and our own words only, ≤ 1800 lines a run:

```
fv: on (GapVehicleAhead.DataFrontVehicle; none 4093/4094/4095, scale none, assumed 0.1, stale 2000 ms)
fv: seen level 3 gap 420 speed 1500 obj 7 q 0/0 v 87 rate - n 6
fv: hold seen level 3 gap 412 speed 1503 obj 7 q 0/0 v 87 rate 0.2 n 48
```
(`gap`, `speed` raw; `q` the two qualifiers; `v` our km/h; `rate` the closing rate on the assumed scale, m/s; `n` values since
the previous line = the native rate.)

---

## 2. Rules every placement keeps

- **Information, not a safety system.** No red anywhere (tested: zero red pixels in every state), no flashing, no sound, no brake
  or "assist" symbol, no chosen-distance bars. "Closing fast" is **amber** (the SPEC-R3 amber 240 170 10), steady.
- **Cruise is not adaptive.** The rungs are the **measured** gap (they move with the car ahead, there is no empty "slot" to fill);
  the cruise set speed is only quoted as on the speedometer (green active / grey standby, from `twin_limit.cruiseNow`).
- **BMW's collision warning** is never read, drawn or covered by anything new: A paints into the existing page layer (which already
  covers the centre), B sits inside our own speedometer hub; neither is near BMW's panel [696, 595.75, 1226, 704], the gear gap, the
  belt group or the side lamps. The message machine (`twin_cc.js`) is untouched and proven so (§6).
- Our own car: a plain generic rear view (rounded body, narrower glasshouse, dark window and lamp slots, wheels) — no maker's cues.
  Ours = an outline at the foot; the car ahead = filled.
- Fonts: Twin Dial's own Noto atlases only (`td_title` 41, `td_body` 31, `td_val` 39); every letter of every string is in all three
  (tested). Saira is not used (Latin only).
- Night: the theme's night roles (white × 0.80, dim × 0.75, amber × 0.80, green × 0.75) — the same multipliers as every page.

---

## 3. The three placements

### A. A centre page "Авто попереду" — **recommended (built)**

The existing one-layer `tdpage` [729, 248, 462 × 347] (connection 1), painted by `shared/twin_front.js` in its own functions
(`twin_pages.js` gets two marked lines: tab 1's page is `front` while the test is on, and `front` paints itself).

| Item | Geometry (panel px) | Colour day / night | Font |
|---|---|---|---|
| Title "Авто попереду" | centred x 960, baseline 281 (the page grammar; the backdrop's title rule under it) | 255 / 204 | Noto 41 |
| Meta left: the gap word (or the time gap once calibrated) | x 739, baseline 326 | 255 / 204; amber 240 170 10 / ×0.8 when fast | Noto 31 |
| Meta right: "круїз 90 км/год" / "обмежувач 110" | right x 1181, baseline 326 (drops its unit if it would touch the word) | green 59 155 26 / ×0.75 active, 227 / ×0.75 standby | Noto 31 |
| The lane | a pinhole, horizon y 318; edges from the foot y 592 (x 810 / 1110) to y 348; 3.2 → 1.2 px, alpha 0.85 → 0.25 | 255 / 204 | — |
| Our car | outline, centre x 960, wheel line 601 (cut by the layer), 112 wide, 1.5 px | 227 × 0.7 / ×0.75 | — |
| The car ahead | wheel line at level 1..5: y 490 / 446 / 414 / 392 / 377; width 52 % of the lane there (98 → 28 px) | 255 / 204; amber when fast | — |
| Rungs (the measured gap) | one per level, at y 528 / 490 / 446 / 414 / 392, 80 % of the lane, 1.6–5 px | 255 at 0.5 (0.8 closing); amber when fast | — |
| Chevrons "we are closing" | centre (960, 510), 34 × 9 px, 3 px; two (10 px apart) when fast | 255 / amber | — |
| "Авто не видно" / "Немає даних" | centred x 960, baseline 452, on a black box (the lane stops under it) | 227 × 0.8 / ×0.75 | Noto 39 |


**Motion.** The car never jumps: its drawn position glides to the new level by 0.6 of the remaining way per page evaluation
(150 ms), snapping when under 0.1 level — 1 → 5 in 4–5 repaints (≈ 0.6 s). A new target appears at its level at once; a lost
one goes at once. Between changes nothing is repainted (a steady car costs nothing). The repaint is `_blitDiff`: only the changed
rectangle goes up.

**Reach.** Test mode: tab 1 shows the page while the test is on (it replaces the clock / trip / technical pages; off = back).
For the product, two options for the owner (§8): (i) a value **"Авто попереду"** of the existing "Сторінка бортового комп'ютера"
(`centre`) — chosen by the driver, outside the 10-s cycle; (ii) tab 1 switches to it by itself while cruise is active and a car is
followed (as the references' assistance page does). No sixth tab.

**Cost.** No layer, no pixel memory (the page layer exists). CPU: one page paint 1.9 ms on the Mac (Node 22) — on the head unit
expect 10–20× that, only during a glide (≤ 5 paints a level change). Upload: §5.

### B. A pictogram in the speedometer's hub — **recommended with A (built)**

Above the digital speed, in the free crown of the hub, on every tab: our car (outline) at the bottom, the rungs of the gap (one
per level), the car ahead (filled) right above the last rung. Nothing at all with no target, stale or absent (the hub as today).

| Item | Geometry | Colour |
|---|---|---|
| Layer `fv_icon` (type `tdfv`) | **[1504, 250, 64 × 68]**, connection 2, z 13, toggle `fv:on` | — |
| Clearances | inner hub circle r 163 (top y 237); speed digits y 374+; **KOMBI black box [1420, 320, 1611, 440]: 2 px below us**; the needle starts at ρ 0.55 R, we end at ρ 0.5 | — |
| Our car | 28 × 16, wheel line y 316 | 227 × 0.75 (night × 0.75) |
| Rungs | 18 × 2.5 px at y 294, 288, 282, 276, 270 (level 1..5) | 255 at 0.6 (0.9 closing), amber when fast |
| The car ahead | 28 × 16, wheel line 3 px above the top rung | 255 / 204, amber when fast |

No glide (the steps are 6 px). **Cost:** one image layer on connection 2 (3 → 4 of 8), 64 × 68 RGBA = 17 408 B, **+34 816 B of
pixel memory** (counted twice); a level change uploads a few hundred bytes to ~3 KB. The width 64 is a multiple of 4. No shader.
The cruise readout C1 [1456, 500, 140 × 70] and C2 are untouched; the pictogram never overlaps them.

### C. A strip in the bottom row's lower line — **not recommended (mockup only)**

[880, 664, 1040, 712]: a top view, our car left (outline), rungs, the car ahead right, chevrons ">" when closing. Only possible on
tabs 2–5 (tab 1's lower line holds the clock and the temperature), and the box lies **inside BMW's panel footprint**: it would
have to be a member of the bottom-row sheet (connection 0, toggle `cc:row`) and step aside with it whenever the window opens —
i.e. it would change the safety machine's own sheet and frames. No layer, ~0.03 MiB more in that sheet, but the risk is in the
one path that must not change. The space beside the cruise readout (C1's left) was measured too: the hub circle leaves no room
for a three-digit set speed and a pictogram (x ≥ 1439 at y 530, out of the circle at y 566).

### Not chosen

The tachometer hub (gear, READY, the programme, the unit: no free area larger than 30 px high); the speedometer's lower opening
(cruise C1 lives there); a sixth tab (the brief).

---

## 4. Budget (verified on the bench, `bench/test-front-vehicle.js` and the shots' audit)

| | Test off (= 0.1.4) | Test on |
|---|---|---|
| Layers per connection | 7 / 4 / 3 = 14 | 7 / 4 / **4** = 15 (≤ 8 each) |
| Pixel memory | 4.85 MiB | 4.88 MiB (+34 816 B) |
| Shaders | — | none new |
| Subscriptions | none of GapVehicleAhead | `DataFrontVehicle` only |

The gallery's audit flags "layers in all (round 3 budget: 14)" with the test on — the only new failing check, and it is the one
layer B costs (the owner's call: §8). Its two other failing checks ("belt group open", "side lamps: the 3 px margin") fail the same
way with the test off (pre-existing, `bake/work/front-vehicle/shots/run3`).

Switching: every change of the setting rebuilds the layout once (the pictogram's layer is made or dropped) — about 2.3 MB of
pixels, as `diagGpuText` does. Off at the start = no handler, no subscription, the layout byte for byte as 0.1.4 (plus one widget
that is not built). On while running = one late watch (Signal Probe's pattern); off again = values ignored, nothing drawn, the
proxy kept until the app restarts (a proxy shutdown has never been tried on the car).

---

## 5. Upload while a car is followed

Measured on the bench (`bench/test-front-vehicle.js` Part 3: `BENCH_TRACE`, the bytes of every pixel frame of the page layer and
the pictogram, while `DataFrontVehicle` is fed at 10 values a second; **verified** on the bench, the car's own rate unknown):

| Following | Page A | Pictogram B | Together | Share of the 2000 KB/s floor |
|---|---|---|---|---|
| a steady gap, ±4 % jitter, 8 s | 0 B | 0 B | **0 B/s** | 0 |
| drifting a level away and closing back in (40 → 60 → 40 m in 16 s: two level changes, "closing" on and off) | 346 KB | 6.6 KB | **≈ 21 KB/s** | ≈ 1 % |
| worst case: level 5 ↔ 1 every 2 s (abrupt jumps, "fast" on and off) | 1 958 KB | 22.6 KB | **≈ 242 KB/s** | ≈ 12 % while it lasts |

Per event (offline, the same painter, the single changed rectangle `_blitDiff` sends): a one-level glide ≈ 130 KB in 4 frames (the
first frame carries the meta word, about 262 × 106 px); the whole range 1 → 5 ≈ 256 KB in 6 frames; the pictogram ≤ 6 KB a change.
A car that appears or goes is one frame of up to ≈ 400 KB (the lane's middle and the meta line). For comparison the first complete
frame of Twin Dial is 2.3 MB. If the car's gap really moves a level every few seconds in town, the cheaper variants are: no glide (one
frame a change, ≈ 60–100 KB), or the meta word left out of the page (the first frame shrinks to the car's box).

---

## 6. Tests

`node bench/test-front-vehicle.js` (port 8963; `--no-bench`, `--no-cc` skip parts): **74 passed, 0 failed** (2026-11-19, 20:00; with `bench/test-twin-cc.js` 266/0 off and 267/0 on inside it).

- Part 1, the decoder: the car's raw values, none values, qualifiers; levels by time gap / distance; a ±6 % jittery gap for 30 s
  at a boundary → 0 level changes; one spike → nothing (median); a cut-in → no false "fast"; closing from the slope, not speedX;
  stale after 2 s → no target; absent after 10 s; the calibrated scale; the log (format, 1 a second, the cap); Node 8 syntax.
- Part 2, the pictures: fonts and Cyrillic, every state inside its layer, no red pixel, amber only when fast, depth by level,
  rungs = level, the glide, night, the pictogram's keep-outs.
- Part 3, the real app on a bench: off = 14 layers 7/4/3, no watch; on = late watch, 15 layers 7/4/4, +34 816 B, every state walked
  through `POST /fake {front}`, the stale state after 2.6 s of silence, the upload measured, off again = layer gone, values ignored,
  no error in the log.
- Part 4, the message machine: `bench/test-twin-cc.js` run twice, `TWIN_CC_FV` off and on (on = the page layer and the pictogram in
  the stand-in layout, a target fed at 10 Hz through every case): both pass, and the audit's frame trace (BMW's panel covered /
  row ink / our text / window open / machine state / tab, per change) is **identical**.

**A caution about Part 4 (verified, 2026-11-19 evening):** another agent's job held this Mac at a load average of 570–730 for more
than an hour. Under that load **the unchanged engine fails the audit by itself** — `bench/test-twin-cc.js` with the test OFF:
263/3 and 261/5 (V1 "panel covered 140 ms after our text first showed", V3, the 150-ms settles), and its trace varies from run to
run (117 vs 120 changes). With the machine quiet, OFF gave 266/0 three times with one identical trace (120 changes) and ON gave
267/0 with **the same** trace. So Part 4 must be run on a quiet machine; the test runs a differing pair once more before it reports a
difference.

Also touched and re-run: `bench/test-twin-layout.js` (the conn-2 "cc:" sheet check now also excludes `tdfv`).

---

## 7. f30_cluster (not built)

f30_cluster packs nothing of this today. The decoder is app-agnostic and would be packed as it is. In the 6WB look the natural place
is the info display between the dials, as a pictogram like B (our car, the rungs, the car ahead) in the 6WB greys and the 6WB
amber, in Comfort only: one own image sheet of about 64 × 64 (≈ 0.03 MiB of pixel memory, one layer on its least-used connection
— to be checked against f30's per-mode layer counts, which are tighter than Twin Dial's in Comfort) plus the same subscription.
A whole page like A has no place in the 6WB look (no centre pages). Sport / Eco Pro: none.

---

## 8. Decisions for the owner

1. **Placement**: A + B (recommended), A only, B only, or none for now. B is the only one that costs a layer (15 instead of the
   round-3 budget of 14).
2. **Where the page is reached from** (after the test): (i) a value "Авто попереду" of `centre` (recommended: the driver chooses it),
   or (ii) tab 1 switching to it by itself while cruise is active and a car is followed.
3. **BMW's own warning state in our picture** (`FASuS.requestWarnBrakeCoordinatorDisplay`, declared in Signal Probe, not read here):
   off by default; if wanted, the proposal is only to **dim our picture and write "Увага" in the meta line** while BMW's warning is
   requested — never a symbol of ours. Not built.
4. **The calibration drive** (Signal Probe set S14 on, and/or Twin Dial with "Авто попереду (тест)" on — its `fv:` lines carry the
   raw gap every 5 s):
   - standing behind a car at **5, 10 and 20 m** measured with a tape (engine on, both cars still) — the cleanest scale;
   - following **one** car at a **steady speed** (60 and 90 km/h) at **two or three gaps** (about 1, 2 and 3 s by counting), each
     held for 20–30 s, the passenger noting the time and the gap (lane markings or a phone rangefinder);
   - one approach to a slower car (closing) and one overtaking car pulling in (a cut-in: the object id).
   From these: `scale` (and whether `nearer: smaller` holds), the native rate (`n`) for `staleMs`, the no-target values, and what
   `speedX` is.

## 9. Findings outside the brief (not fixed)

- **The cruise readout clips a three-digit set speed** (Twin Dial 0.1.4, `twin_limit.js paintCruise`, place C1): the number is
  right-aligned at x 1536 in a sheet that starts at x 1456, so "110" loses its "1" and reads **"10"** (bench pictures
  `shots/run3/F/day_11_limiter.png`, `night_11_limiter.png`). Hypothesis until seen on the car, but the arithmetic is plain (80 px for
  three Saira-40 digits).
- The brief's "declare the event as `_broadcasts`": `DataFrontVehicle` is a notifier field, declared `"s"` (§1).
