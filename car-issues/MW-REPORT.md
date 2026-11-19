# MW: the map window takes the whole free space between the dials (Twin Dial)

Agent MW, round "car issues" of 2026-11-19. Branch `car-issues/mw` (worktree `.claude/worktrees/ci-mw`), from `main` at
`8421a64`. Commits: `4e8f565` (W3), `d0b8c8a` (a fix found on the bench), `d9d1f15` (the mockup tool), `9349172` (W3's arc
radius). Not pushed, no version bump, the car not touched. Mockups: `bake/work/car-issues/pub/MW/`.

**Summary.** W3 is built as a third choice of Діагностика "Вікно карти" (off / W2 / **W3** / W1). Nothing changes by default.
W3 is the box [618, 173, 1302, 588] minus two discs of radius 324 round the dials' centres. Its sides are arcs that follow the
bezels, its top is the tab rule and its foot is y 588. W3 is 684 px wide under the tab rule, 586 at y 250 and 512 at mid-height.
W2 is 520 everywhere. The shader grows from 2 672 to 2 950 bytes. No new layer, no new sampler. W3 still needs the car: whether
it shows more map depends on what the cluster draws in the new corners, and two photos disagree on that (§1). The car test in
§5 settles it. On the way I found and fixed one bug, proven on the bench: switching the variant while the window was open did
nothing until the window closed and opened again (§4).

Status words below: **proven** (by code, a test or a measurement whose source is given), **hypothesis** (not confirmed by the
car).

## 1. What there is to show: where BMW's map lies

| Source | What it shows | Status |
|---|---|---|
| Photo A (Twin Dial, W2, CarPlay's map with its orange position arrow, 2026-11-19 14:18 car clock). I rectified it with a 4-point homography from our own drawn features: the tab bar's top corners (618, 127) and (1302, 127), and the bottom rule's ends (629, 613.5) and (1292, 613.5). Error about ±3 px in the middle. | **Left edge.** Rows 380–440: code 24–30 at x 688–700 (our black plus glare). From x 704 to 776 the code is a flat 46–50. Rows 200–260: 33–37, then a flat 45–55 from x 704. **Top**, x 1000–1100: code 87 right under the rule (y 174–178). There is no ramp. BMW's own map box (photo 41) rises from 10 % to 50 % over x 695 → 724, a 30 px gradient. Photo A shows a 4 px step at W2's edge. So the map in photo A is at full level at least out to W2's left and top edges: the "rectangle" the owner sees is **W2's own cut**. The right edge is lost in sun glare (codes 65–88 both inside and outside). | measured (proven for the left and top edges, within the rectification error) |
| Photo 41 (our 6WB Dials 0.7.7 over BMW's own navigation map, cluster-render 0.5.8; CAR3-H) | A box between the dials: 10 % at x 695–1216, y 137–592; 50 % at 724–1192, 162–575. It has a fall-off of 30–50 px of its own. | measured earlier (CAR3-H) |
| Rectified stock photos with no theme: 08, 09, 39, 61, 64, 68 (BMW's blue map and CarPlay's grey map) | The map covers all of W3, the new corners included. It reaches about x 540–1400 and fades under BMW's own dial frames. Overlay checked visually; scratch images only. | measured |
| Photo 14 (our 0.7.2 text over the stock cluster: "stock under unbound theme") | **BMW keeps drawing its stock layout under a theme.** The map is wide; the stock digital speed, its rule and the drive-mode label are all there. | seen |
| cluster-render (skill reference `cluster-and-hud.md`) | The map slot 0x100 is 1088 × 480. Placed in the middle that is x 416–1504: wider than W3 (612–1308 at most). | documented |

**Does the map exist behind our dial layers?** In the stock view the slot's picture reaches under the inner parts of our dials
(about x 540 / 1400). That does not matter: our dials are opaque inside 304 px. **Between the dials it fills W3 in every stock
photo.** With a theme up, the two photos disagree. Photo 41 (BMW's own map, cluster-render 0.5.8) shows only the box. Photo A
(CarPlay's map, today's cluster-render) shows full level at W2's edges. Ranked hypotheses (none confirmed by the car):

1. **H1:** on today's cluster-render the map is drawn as in the stock view, so W3's corners show map. Photo A favours this.
2. **H2:** the box depends on the map's source: BMW's navigation map gets the box, CarPlay's map does not. Photo 41 was BMW's
   map, photo A CarPlay's.
3. **H3:** the box depends on BMW's view or drive mode at the time.

**Does BMW fade the map at its own edges?** Its own map with a theme up: yes, 30–50 px (photo 41). The stock view: an irregular
vignette under its dial frames. CarPlay's map in photo A: no fade inside W2.

## 2. The shape, and why

Figure "geometry" (`bake/work/car-issues/pub/MW/mw_geometry.png`, a local working file of the code repository, not published) — `bake/work/car-issues/pub/MW/mw_geometry.png`.

| Width at y | 174 | 200 | 250 | 300 | 349 | 400 | 451 | 500 | 550 | 587 |
|---|---|---|---|---|---|---|---|---|---|---|
| W3 open x | 618–1302 | 635–1285 | 667–1253 | 688–1232 | 700–1220 | 704–1216 | 700–1220 | 688–1232 | 667–1253 | 644–1276 |
| W3 width (W2 = 520) | 684 | 650 | 586 | 544 | 520 | 512 | 520 | 544 | 586 | 632 |

W3 is 232 792 px against W2's 222 040: it gains 17 536 px and loses 6 784. The losses are the strips of 4 px either side at
mid-height and W2's bottom 12 rows (y 588–600).

- **The sides: arcs of 324 px round the dials' own centres** (380, 400) and (1540, 400). Our dial layers are see-through outside
  their bezel: `tdial`'s alpha is 1 inside 1.0128 R and 0 from 1.0128 R + 0.5 = **304.3 px** (`shaders.js` TDIAL, the `cv` line).
  So the dial layers' rectangular corners are **already transparent**. The black there is the backdrop's (`bg_l`), and the hole
  in the backdrop is enough: no dial layer changes. The brief assumed the corners were ours to clear; they are not.
- **The ring between bezel and map is 20 px, not 6.** The first cut used 310 px (a 6 px ring) and was too wide. BMW draws its
  stock layout under a theme (photo 14). Its items nearest the gap, measured on the rectified stock photos 08 / 09 / 39 / 61 / 68
  (±1 px):
  - the rule under BMW's digital speed ends at **x 694** (y 412–415);
  - BMW's drive-mode label ("COMFORT PLUS") and the rule under it start at **x 1223** (y 337–415).

  With 310 px, W3 opened x 690–1230 at y 400 and would have shown 2–6 px slivers of both. With 324 px both stay under our black
  by 1.5 px. W2 (700 / 1220) kept them covered by 6 / 3 px, so this is also why W2 never showed them. Cost: at the exact middle
  W3 is 4 px narrower than W2 on each side (704–1216). Above y 349 and below y 451 it is wider, up to 82 px each side at the
  top. Only Comfort-mode stock photos exist: the label's place in Sport and Eco Pro is **not measured** (car test, §5).
- **The top is the tab rule** (rows 170–172 stay drawn and opaque, open from y 173–174, as W2). The sides are clipped at the
  tab bar's ends, x 618 / 1302. The arcs would allow 612 / 1308 at y 174; the clip keeps the map from reaching out past the bar.
- **The foot is y 588, not 600.** W3's lower right corner reaches x 1276 at y 587. With a foot at 600 it would have shown the
  top 9 px of **BMW's stored-message marker** (the "i" or the warning triangle, [1243, 591, 1285, 628]; G20-DESIGN-SPEC). W2 never
  reached the marker because it ends at x 1220. A side effect: W3 no longer overlaps BMW's message panel (top 595.75), which W2
  overlaps by its rows 595–599.
- **The road bar stays as it is**: [760, 556, 1160, 594], 60 % black, Noto 31, max width 380. Its bottom 6 px now fall on our
  black, which looks the same as before because the bar is nearly black. The text keeps baseline 588 (mockups
  `mw_*_road_*.png`). Raising it 6 px is a decision (§6).
- **Nothing of ours sits over the new part of the map.** The layer plan (the engine's own planner) puts only `bg_l` (whose hole
  this is) and the two dials (transparent there) where W3 opens beyond W2. The limit sign [1252, 628, 1348, 716], the cruise
  readouts (inside the speedometer) and the bottom row (y ≥ 613) are all outside W3. The title rule is already dropped while the
  map shows.
- **Safety.** W3 never meets the belt group, the gear gap, the seven side lamps, the lamp strip, the stored-message marker or
  BMW's message panel. The tab bar and the bottom row stay opaque. W3 opens only under W2's rule: tab 5 selected and the map
  believed present, shut at once on any doubt. `shared/twin_cc.js` is untouched.

## 3. Built: W3 on the bench

| Where | What |
|---|---|
| `shared/shaders.js` TDBACK (marked `car-issues MW`) | One uniform, `u_mapA` (the centres' x, their y, the radius; 0 = no arcs). Four lines in the existing map block: each disc multiplies the window's openness by `clamp((sqrt(dot(l, l)) - 1) * R + 0.5)`, with `l = (p - centre) / R`, so the offset is divided by the radius **before** squaring. The radius is 0 for W1, W2 and a shut window, which then draw exactly as before (GPU test: identical alpha). |
| `shared/hud.js` (marked) | `TW_MAP.W3 = { box: [618, 173, 1302, 588], fade: [1, 1, 1], r: 1, arc: [380, 1540, 400, 324] }`. `_twinBackSend` sends `u_mapA` with `u_map`. `HudApp.TW_MAP` is exported for the tests. |
| `shared/twin_pages.js` (marked) | `mapVariant` accepts "W3". The tick fix of §4. |
| `apps/twin_dial/app.json` | The "W3" option after W2. `config.diag.diagMapWindow.options.W3`. The two `_about` texts. `config.mapWindow` stays `null`. `check_signals.py` passes. |

**The shader's size against what the car has accepted**

- tdback part L: **2 672 → 2 950 bytes** after `compact()`, +278 bytes (+10 %). Uniforms 17 → 18 (the limit is 32). Samplers
  unchanged (`u_mask` only). Parts RT / RB unchanged at 537 bytes. `tdial` unchanged at 5 763 bytes.
- Already drawn on the car: tdback L at 2 672 bytes (Twin Dial 0.1.x); f30's WBBACK at about 9.0 KB and WBDIAL Comfort at
  12.4–13.3 KB (0.7.13 on; MP-REPORT; memory note `kombi-shader-size`).
- Resets seen: one 6WB source of 26.7 KB (25.4 KB worked).
- So 2 950 bytes is far below anything the KOMBI has refused.
- "One look per layer source" holds: W3 lives in part L's existing source (STYLE 0) and is chosen by uniforms, exactly as W1 and
  W2 are. No new look and no new branch kind.

**Layers and memory, unchanged**: per connection 7 / 4 / 3 (layout plan) and 5 / 3 / 2 (budget fixture); 4.31 MiB steady and
at the peak (`bench/test-twin-budget.js`).

**fp16.** The arcs are circle distances of 300–700 px.

- The shader asks for `highp` (unchanged), and divides by the radius before squaring in case only `mediump` (fp16) exists.
- The budget test models the arc term **op by op at fp16** (IEEE binary16 rounding of every intermediate), over every pixel of
  W3's box plus 4 px round it, for both discs: 585 432 pairs. Result: no NaN or ∞; no pixel more than 1.5 px inside a disc
  opens; the edge's coverage stays within 0.66 of exact. The 0.66 is because x above 1024 is itself a whole pixel at fp16.
- The same model with the distance measured in px (an unnormalised length) opens **46 974 pixels inside the discs**: the
  squares pass 65 504. So the test notices the mistake that cost the 6WB dials in 0.7.1–0.7.4.

## 4. Found on the bench and fixed: a variant chosen while the window is open did nothing

- **Symptom** (proven: the mockup run's snapshot). With W2 open on tab 5, choosing W3 in Діагностика left `u_map` at
  [700, 173, 1220, 600] and `u_mapA` at zeros.
- **Cause** (`shared/twin_pages.js` `update`): `hud.twinMap` was sent only when the window opened or shut. The Діагностика change
  applies in place (no rebuild), so the old variant stayed until the window shut and opened again. The same was true of
  W2 ↔ W1 before this round.
- **Effect on the car:** the owner's W2-vs-W3 comparison on tab 5 would have shown no difference.
- **Fix:** the tick resends when the variant differs from what was sent.
- **Test:** `bench/test-twin-centre.js`. Two checks (W2 → W3, then → W1, while open) **fail without the fix** and pass with it.

## 5. The car test (for the owner; parked, engine off or idling, belt lamp check first)

1. Twin Dial up. A map in the cluster: a CarPlay route, with the cluster's map view on. Tab 5. Діагностика → "Вікно карти" =
   **W2**. If the window stays shut, set "Примусове вікно карти" on. Wait 10 s (the window stays shut 8 s after a build).
   **Photograph** the whole panel, straight on, as photo A.
2. Same place: "Вікно карти" = **W3**. Within a second the window should change shape (the §4 fix). Wait 2 s. **Photograph.**
3. Repeat 1–2 in **Sport** and **Eco Pro**: BMW's stock layout under us changes with the mode.
4. Repeat 1–2 with **BMW's own navigation map** in the cluster instead of CarPlay's. This decides H1 / H2 of §1.
5. Leave "Вікно карти" on what you prefer, or **off**.

**What to look at in the W3 photos:**

- The four new corners (left and right of W2's old sides, above y ~350 and below y ~450): map, or black, or the soft edge of
  BMW's own box.
- Next to each bezel at mid-height: any sliver of a BMW label or line. There should be none.
- The lower right corner, near x 1250 just above the bottom rule: no piece of BMW's "i" or triangle.

**Which photo answers what:**

- Map in the corners of 2 and 4: H1 (W3 shows the free space whatever the source).
- Map in 2 but black or a box edge in 4: H2 (W3 helps only with CarPlay's map).
- Black in both: the box is the theme's doing (CAR3-H's reading). W3 brings nothing; keep W2.

**Logs** (the lead, after): the twin log's "map window open (W2)" and "map window open (W3)" lines, at the times of the photos.
With "Журнал слотів" on: the slot 0x100 rows. **No log shows the map's extent** (pixels are not logged), so the 2026-11-19 logs
cannot settle §1. They would only confirm which variant was sent when. MW did not wait for them.

## 6. Decisions for the owner

1. **W2 or W3**, after the car test. W3 is an experiment in Діагностика. The default is unchanged (no map window), and W2 stays
   the first choice after off.
2. **The ring at mid-height: 20 px (built, arcs 324) or 6 px (arcs 310).** 310 is closer to "the whole free space" but would show
   2–6 px slivers of BMW's own speed rule and drive-mode label at mid-height, **if** BMW draws them under Twin Dial (photo 14 says
   it draws its stock layout under a theme). One number in `TW_MAP.W3.arc`.
3. **The foot: 588 (built) or 600 with a notch** at the lower right round BMW's marker (one more uniform, about 40 bytes).
4. **The road bar:** keep it at [760, 556, 1160, 594] (built; its bottom 6 px are now over black) or raise it 6 px inside W3's foot.
5. **If the car shows BMW's box in W3's corners** for BMW's own map: either keep W2, or open W3 only while CarPlay leads
   (`guidingProvider`). That needs a rule that is not built.

## 7. Tests (each run alone, ports 8910–8915)

| Suite | Result | MW's checks in it |
|---|---|---|
| `bench/test-twin-layout.js` | 121 passed, 0 failed | 14: W3 against the layout. The arcs match the dials. The ring and its mid-height widths. The box against the tab bar, the bottom rule and the marker. The hole meets no BMW lamp, gap, strip, marker, panel or stock mid-height item (W2 neither). Tab bar and bottom row opaque. Nothing of ours where W3 opens beyond W2 (the engine's own layer plan). Road bar inside W2. |
| `bench/test-twin-budget.js` (BENCH_PORT 8910) | 101 passed, 0 failed | Static: arcs normalised; the fp16 model and its bite. Live: `u_mapA` for W3 [380, 1540, 400, 324], zeros for W2 and shut. |
| `tools/twin-dial/test_shader.py` (the real shader on the Mac's GPU) | 56 passed, 0 failed | 10: zeros change nothing for W2. The ring. Open past the arc. The new corners open. Mid-height 704–1216. The tab rule opaque. Shut from y 588. Never outside x 618–1302. Every open pixel inside the box and ≥ 323 px from a centre. |
| `bench/test-twin-centre.js` | 250 passed, 0 failed | W3 under the same presence rule. The variant switch while open (§4, fails without the fix). An unknown variant is no window. |
| `bench/test-twin-cc.js` (8912) | 266 passed, 0 failed | none: the safety machine is untouched |
| `bench/test-twin-pages.js` (8913) | 100 passed, 0 failed | none |
| `bench/test-diag.js` (8911) | 106 passed, 0 failed | The option list is now off / W2 / W3 / W1 (it was off / W2 / W1). W3 writes `config.mapWindow = "W3"`. |
| `bench/test-mb-exp.js` (8915) | 46 passed, 0 failed | none: it rewrites every shader's `v_uv` |
| `tools/signals/check_signals.py` … `apps/twin_dial/app.json` | all declarations pass | — |

Every new check fails without its code. Examples: the layout test's first W3 check without `TW_MAP.W3`; the marker check with
the foot at 600 (seen); the stock-item checks with arcs of 310 (by construction: 690 / 1230 at y 400); the §4 checks without
the fix (seen). The full runner was not run (the lead runs it at merge).

## 8. Mockups (`bake/work/car-issues/pub/MW/`, git-ignored; made by `tools/shots/twin/mapwin.py --port 8914`)

The bench renderer is the app's real layers from a bench snapshot, replayed on the Mac's GPU by the gallery's painter.

- **Under our layers, our own stand-ins only**: a drawn map (grey land, white roads, a 100 px grid) over the whole map slot
  ("slot"), or the same map inside BMW's box with photo 41's fall-off ("box"); `backdrop.py`'s boxes for BMW's lamps; BMW's
  gear letter.
- **The road name** is the test fixture's "вулиця Тестова".
- **No photo and no BMW artwork** is in them.

| File | What |
|---|---|
| `mw_<day|night>_<road|noroad>_slot.png` | W2 (top) over W3 (bottom), the whole panel, labelled |
| `mw_centre_<day|night>_<road|noroad>_slot.png` | The middle [420, 100, 1500, 640], W2 beside W3 |
| `mw_day_<road|noroad>_box.png`, `mw_centre_day_*_box.png` | The same over BMW's box (H2 / H3): W3 then looks like W2 with black corners |
| `mw_geometry.png` | Outlines: the dials, the tab bar, the bottom rule, W2, W3, photo 41's box, the stock mid-height items, the marker, BMW's panel, the belt group and the gear gap |
| `mw_states.json` | The bench states each picture came from |

Not related to MW and not investigated: in the night snapshots (W2 and W3 alike) the media tab's cell shows a warning
pictogram before the radio icon. The day snapshots do not show it. The night states ran after the day states on the same
bench without a restart.
