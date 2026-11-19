# Brief: the bottom strip and the speed-limit signs, measured to the pixel and rebuilt on a baked mask

Audience: twelve agents (packages R, ZL, ZC, ZR, T, O, H, P, M, F, X, D) working **in parallel**, each in its own git worktree. Read §0–§5 and your own package in full; skim the other packages so you know where your edge is. The lead (the main session) merges, bakes the final mask and shows the owner the bench.

This is meant to be an **aggressive** analysis. We have been approximating BMW's shapes with rectangles and round numbers and getting it wrong for a week. Do not round to convenient values, do not trust an earlier number because it is written down, and try to falsify every measurement of your own on a second photo before you publish it.

## 0. What the owner saw and said (2026-11-16)

On the bench, build `integrate/bottom-row` (`3c8ef46`), layout "6WB Comfort + G20":

- **"Just an open rectangle. The info field is not rectangular on the original photo."** In `bottomRow = allG20` our window over BMW's check-control panel is a plain box (`CC_BOX`), while BMW's panel has an arc on top and slanted ends.
- **"The round sign is no longer round, it got an asymmetry."** The black ring round BMW's speed-limit sign is visibly lopsided. Suspects: the ring is clipped by a layer rectangle, or today's commit `3bfce09` (window r 42 → 40, ring reach 8 → 10 px).
- **"In 'all ours' we do not draw the clock."** Today the clock is ours only once a time zone is known.
- **"There is still no full coverage, the geometry is not repeated, there is no proper flowing round the stock elements."**
- What they asked for: an almost pixel-by-pixel comparison of every zone where our layers and BMW's meet; everything taken apart into layers; the exact geometry of every element and font kept; colour deltas worked out against the fact that the display should be black there; textures baked for "all ours"; and in "all BMW's" a transparency that hugs the stock elements as tightly as possible. "If the shapes are rough, add a gradient of a couple of pixels that reads as a shadow, or come up with something yourself."

Earlier findings you start from (read `<session scratchpad>/bottom-strip-audit.md`): a 660 × 13 px uncovered band at y 652–666 under our message; `nav_bg` clipping 37 % of BMW's symbol; BMW's status marker blacked out in both modes; `CC_BOX` 17 px too wide; `_wbPatches` capped at four rects with `CC_TOP` last.

## 1. What the owner has decided

1. **Scope:** the bottom strip plus the speed-limit signs, in both see-through layouts (`wb_dials` = "6WB Comfort + G20", `wb_sport_dials` = "6WB Sport + G20"), in both modes of the one `bottomRow` setting.
2. **`allG20` ("all BMW's"):** we draw nothing in the strip. Our opaque background must flow tightly round each stock row item (range, clock, sign, belts, message panel with its symbol, status marker, road label, odometer, mode label, gear, temperature). BMW's rails, hooks, dashes, glows and dial remnants stay hidden.
3. **`all6WB` ("all ours"):** everything BMW draws *under* a theme is covered; we draw range, **clock**, temperature, sign, road label and message ourselves. Only safety telltales and elements BMW draws *over* a theme keep a hole.
4. **The clock is always ours in `all6WB`.** Time zone from the car's signals that are already read (`config.clock`), with the `timeZone` setting and `config.utcOffsetMinutes` as fallbacks.
5. **Fallback is all-or-nothing.** If any one element of the row cannot be drawn, the whole row goes back to BMW. No more per-element hand-back (it produced three texts stacked on each other).
6. **Standby** (P, engine off) **is always BMW's row**, in both modes. We do not draw an odometer.
7. **Our message is our own design**, not a copy of BMW's panel, and for now **without a window for BMW's symbol**. One short line. Two variants go to the bench for the owner to choose: **A** a severity marker plus the headline in the road label's slot, no frame; **B** the same with a hairline above and a coloured segment under the text instead of the marker.
8. **Cyrillic is baked into our fonts.** The cluster runs in Ukrainian; today `wb_cap` / `wb_capb` have 0 Cyrillic glyphs, so any car text latches the element back to BMW.
9. **Rectangles are replaced by a baked mask** (§4.4).
10. **A diagnostic build for the car is agreed in principle** (a solid-colour theme to photograph what BMW draws over us). It ships only on a separate yes.
11. **Allowed:** baking textures, drawing or obtaining pictograms, using the internet and any catalogue.

**The safety rule, not negotiable:** a BMW warning is never hidden unless our own text for it is on the screen in the same frame. Belt and airbag telltales are never covered in any state.

## 2. Ground truth

### 2.1 The new stock set (owner's originals, added 2026-11-16)

`reference/g20/`, PNG, 5712 × 4284 unless noted, roughly 2–2.5 photo px per panel px (measure it). Shot hand-held.

| Id | File | What it shows | View |
|---|---|---|---|
| 60 | `60_grid-over-stock_P_2026-11-16.png` | **our calibration grid theme over the stock cluster**: major lines every 100 px with labels, calibration bars; stock strip visible beneath | frontal |
| 61 | `61_stock_D_limit50_road_2026-11-16.png` | stock, D, map, limit 50, road label, AUTO H | frontal |
| 62 | `62_stock_P_msg-2line_sym-i_limit50_belts_2026-11-16.png` | stock, P, **two-line message** with an "i" symbol in the panel, "i" marker on the right, 4 belts + airbag, limit 50 | frontal |
| 63 | `63_stock_P_limit50_marker-i_service_2026-11-16.png` | stock, P, no message, "i" marker, belts, limit 50, a faint odometer ghost at the bottom centre | frontal |
| 64 | `64_stock_P_limit30_belt1_road_2026-11-16.png` | stock, P, map, limit 30, **one** belt telltale, road label, no marker | frontal |
| 65 | `65_stock_standby_odometer_2026-11-16.png` | stock, **standby**: odometer in the centre, mode label in the bottom row, no sign, no belts | from upper right, glass reflections |
| 66 | `66_stock_P_msg-1line_limit30_marker-warn_2026-11-16.png` | stock, P, **one-line message** with a symbol, warning-triangle marker, belts, limit 30 | from upper right, reflections |
| 67 | `67_stock_P_limit30_marker-warn_road_service_2026-11-16.png` | stock, P, no message, warning-triangle marker, road label, belts, limit 30 | from upper right, reflections |
| 68 | `68_stock_D_nolimit_msa_map_2026-11-16.png` (4032 × 3024) | stock, D, empty sign "---", auto start/stop icon, road label | frontal |

Still missing: a night shot and a no-overtaking sign. Neither blocks you; say where it limits a conclusion.

### 2.2 Older material

- `reference/g20/08, 09, 14, 17` — stock, low resolution (0.85–0.9 photo px per panel px). `10` — the grid over stock, low resolution. `39`, `40` — "g20-look"; `design/g20-spec/measured.json` describes 39 as **stock**, an earlier audit called both "ours" from the file name. **Their status is disputed: package H settles it by content, nobody assumes.**
- `reference/ours/34, 41, 42, 43` — **our** theme with BMW's row showing. `docs/G20-DESIGN-SPEC.md` §5.1 rests on 42 and 43, which is why its claim "no fill under the message" is suspect.
- Our current build on the car, 2048 × 1536, in `<session folder>/images/`: `56.jpg` (row left to BMW), `58.jpg` (`all6WB`), `59.jpg`, `60.jpg` (close-ups by the sign). These are the evidence for **what BMW draws over our opaque paint**.
- The owner's bench screenshots of the complaints, same folder: `65.png`, `66.png` (our message over a green matte and over the stock backdrop), `67.png` (the rectangular window over BMW's panel), `68.png` (the lopsided sign), `69.png` (`all6WB` on the matte: the open box, no clock).
- `design/overlay/g20_comfort.json` — the catalogue: `elements` is a **list** of 39 objects with `id`, `bbox`, `poly`, `colour`, `states`, `relation` (`under` 20, `unknown` 12, `over` 4, `gone` 3). Measured on photos 08–14 only.
- `design/g20-spec/measured.json` + `tools/g20spec/*` — the previous strip measurement and its tools (`zoom.py` warps a panel box out of an original photo at n× using the models in `bake/work/overlay/rect/H.json`; `measure.py`, `fontfit.py`, `fingerprint.py`). **Reuse the tools, re-verify every number.**
- `tools/overlay/rectify.py` — the rectification model (homography + one radial term; grid crossings by NCC; SIFT for the rest). Read its docstring before inventing your own.

### 2.3 What the photos already tell us (low-resolution reading, to be verified)

- Standby is a different arrangement of the strip, not a variant of the normal one.
- The marker right of the panel (none / warning triangle / "i") is independent of the message panel: 63 and 67 have a marker and no panel.
- The message takes the road label's place; the panel is a shaped rail, the symbol sits inside it on the left.
- The belt group varies: one telltale in 64, four plus the airbag elsewhere.
- The sign always sits in the same seat, hard against the small speed readout on its left and the belts on its lower right.

## 3. Rules for every package

- **Base.** Your worktree starts on `main`. First: `git merge --ff-only integrate/bottom-row` (tip `3c8ef46`); if that refuses, `git checkout -B strip/<pkg> integrate/bottom-row`. Then, with `F30_MAIN=<repo>`: `[ -e reference ] || ln -s $F30_MAIN/reference reference` and `[ -e bake/work ] || ln -s $F30_MAIN/bake/work bake/work`. `git status --porcelain` must not list either; if it does, stop and say so.
- **No push. No version bump. No car** (no ssh, nothing sent to it). Commit on your own branch only.
- **No Chrome, no browser automation of any kind.** An earlier agent's headless Chrome hung and pinned this machine at 674 % CPU. **This session's permission layer denies `kill` to agents**, so anything you start you cannot stop. Work numerically: read uniforms from `GET /events`, state from `GET /probe`, baked pixels from `GET /px`, and compute. If a picture is truly needed, ask the lead in your final message.
- **Bounded processes.** If your package has a bench port, start the bench so that it ends by itself, in the background:
  `python3 -c "import subprocess,os; p=subprocess.Popen(['node','bench/server.js','f30_cluster'],env=dict(os.environ,BENCH_PORT='<port>',BENCH_DATA_DIR='<your scratch dir>')); exec('try:\n p.wait(timeout=10800)\nexcept subprocess.TimeoutExpired:\n p.terminate()')"`
  Ports: **O 8161, M 8162, F 8163, X 8164, D 8165.** 8080 and 8150 belong to the owner; read 8150 if you like, never write to it. Report your port and pid at the end.
- **Waiting.** Never a foreground `sleep`. Wait for another package's file with a background `until [ -f <file> ]; do sleep 15; done` and do your independent work meanwhile. If the file is still missing when you run out of independent work, go on with provisional inputs (the low-resolution rects in `bake/work/overlay/rect/`, or your own quick registration), mark the results provisional, and upgrade them when the file lands.
- **Python.** System Python has no numpy. Use the repo's convention: `UV_CACHE_DIR=$F30_MAIN/bake/work/.uvcache uv run -q --no-project --with opencv-python-headless --with numpy --with pillow --with scipy python -I <script>`.
- **`_rowFail` is a sticky latch.** Once an element is handed back to BMW it stays BMW's until `POST /restart`. A Cyrillic string latches at once on the current fonts. Restart between bench cases or you will measure the latched state without knowing.
- **The generator drift trap.** `python3 design/gen_6wb.py apps/f30_cluster/hud/layouts` no longer reproduces the committed layouts: it moves every `wb_cap` / `wb_capb` text by 1–2 px, because the layouts predate the current font bake. **Only package F regenerates all layouts.** M, X and D: regenerate, keep only the hunks that are your change, revert the rest. Layout JSON is never merged by hand; the lead regenerates after merging.
- **Privacy, absolute.** The photos show street and place names, an odometer reading, service dates and check-control texts. None of it goes into any tracked file, report, script, file name, commit message or web request. Describe a box and its size, never the words. No VIN, no coordinates. `reference/` and `bake/work/` are git-ignored and stay that way: **no photo pixel, and nothing derived pixel-wise from a photo, in a tracked file.** Numbers measured on a photo are fine.
- **Not ours to ship.** Never download, commit or ship a BMW font file. Never put a bitmap taken from BMW's software, from a photo of it, or from the car snapshot in `reference/car/` into a tracked file: this repo is public. Pictograms are our own drawings (ISO 2575 defines the standard tell-tale shapes) or come from a permissively licensed set with the licence recorded next to the file.
- **Platform limits** (`.claude/skills/mgu-app-dev/references/cluster-and-hud.md`, and prove on the bench with `BENCH_LAYER_CAP=8`, `BENCH_PIXEL_MIB`): 8 layers per connection, 32 in all, 16 MiB of image pixel memory with each image counted twice, and the **peak during a layout switch** is what actually breaks (drop unused scene textures before making new ones). The KOMBI resets on a too-big effect shader: one look per layer source, measure bytes before and after. The car's `mediump` is fp16: a coordinate across 1920 px loses whole pixels, so use `highp` and scaled lengths. Every `sampler2D` must have an asset bound or RAMSES does not draw the layer.
- **Tests.** Build packages keep `node bench/test-bottom-row.js` (91 passed on the base) and `node bench/test-native-ui.js` (44) green and add tests that would have caught what they fix.
- **Evidence.** Every number carries the photo id(s), the method and an uncertainty. Mark every claim **verified** (measured on a photo, or read from the engine's own output) or **hypothesis**. Where two sources disagree, show both; do not pick one silently.
- **Shared files between build packages.** Keep edits in separate, commented blocks; do not reformat or reorder code you do not own. Ownership is listed per package.

## 4. Shared conventions

### 4.1 Coordinates and the region of interest

Panel 1920 × 720, x right, y down. **Continuous panel coordinates:** pixel (i, j) covers [i, i+1) × [j, j+1), its centre is (i + 0.5, j + 0.5). Boxes are `[x0, y0, x1, y1]`, half-open. `tools/overlay/rectify.py` uses OpenCV's convention (index = centre); convert at the boundary and say which convention each legacy number was in.

**STRIP** = x 0–1920, y 540–720. It contains the sign (centre about (537.3, 621.3), outer radius about 42) and the foot of both dials. Raster products of the strip come at **1×** (1920 × 180) and **4×** (7680 × 720); use 16-bit linear light wherever colour is the point.

Zones, with 10 px overlaps; an element belongs to the zone holding its centre: **L** x 0–700, **C** x 690–1250, **R** x 1240–1920.

### 4.2 States

Stock: `ST-DRIVE` (61, 68), `ST-P` (63, 64, 67), `ST-MSG1` (66, 17), `ST-MSG2` (62, 08), `ST-STANDBY` (65), `GRID` (60, 10). Sub-flags: sign {limit, empty, none}, belts {0, 1, 4 + airbag}, marker {none, warn, info}, road label {yes, no}.

Ours (package O renders all of them, for both layouts and both modes): drive with sign and road label; P; our message drawn; message while the row is latched back to BMW; standby (stage P, rpm 0); every sign state the bench has (limit, empty, end, none, no-overtaking); clock ours (a zone fed) and clock BMW's.

### 4.3 Element ids

Use the catalogue's ids and add new ones in the same style, recording each new id in your published index:

- **L:** `bottom.range` (pump icon, digits, unit), `bottom.clock`, `assist.speed_limit` (BMW's sign: ring, face, digits, empty state), `bottom.belt_rear`, `bottom.belt_driver`, `bottom.airbag`, new `assist.speed_foot` (the small speed readout at the speedo's foot), new `rail.left` (the speedo's bottom line and hooks), the feet of `structure.speedo` and `structure.fuel_mini` as neighbours.
- **C:** `assist.cc_panel` (outline, rail, any fill), `bottom.cc_info` (the symbol slot), new `cc.text` (one-line and two-line boxes), `bottom.street`, new `bottom.odometer` (standby, and the ghost in 63), new `rail.centre`, the foot of `map.region` as a neighbour.
- **R:** `assist.info_tacho` (the marker: none / warn / info), `assist.mode_label` (bottom-row position in standby), `assist.gear`, `bottom.temperature`, `assist.rpm_label`, `assist.off`, `assist.cold`, new `assist.msa`, new `rail.right`, the feet of `structure.tacho` and `structure.oil_mini` as neighbours.

### 4.4 The mask this work feeds

One baked texture over the STRIP replaces the four patch rects, `CC_BOX`, `CC_TOP`, the `nav_bg` hole and, if package M can unify it, the round sign window. Proposed channels (M may change the packing, not the meaning):

| Channel | Used when | Opaque everywhere except |
|---|---|---|
| `ours` | `all6WB`, row healthy, not standby | safety telltales, elements BMW draws over a theme, and whatever §X(e) decides for the marker |
| `bmw` | `allG20`, or `all6WB` fallen back, no message | each stock row item, hugged |
| `bmw_msg` | the same, BMW's message panel showing | the same plus the panel, its symbol and marker |
| `standby` | P with the engine off, both modes | the standby row items |

The belt hole always spans the full group. Every edge carries its own feather: hard where BMW's background behind it is black, a few pixels where it is not.

### 4.5 Publishing to each other

Tracked outputs go in your branch: scripts in `tools/strip/<pkg>/`, numbers in `design/strip/<pkg>/`, the report in `docs/strip/<PKG>-REPORT.md`. Because branches do not see each other's files, **also copy whatever another package needs** to the shared, git-ignored `bake/work/strip/pub/<PKG>/` and touch `bake/work/strip/pub/<PKG>/READY` when it is usable. Photo-derived pixels live only under `bake/work/strip/`.

## 5. Dependencies

R feeds ZL, ZC, ZR and T (rectified photos). O feeds ZL, ZC, ZR (our maps, for the diff). T feeds F (sizes). Everything else is independent. R and O: **publish early and incrementally**, three other packages are waiting on each of you.

---

## Package R — registration and photometry (foundation)

Owns: `tools/strip/R/`, `bake/work/strip/rect/`, the bench's photo list.

1. **Register every photo of §2.1 to panel coordinates**, then re-register 08, 09, 14, 17, 39, 40 with the same method. Start with 60: locate every visible major grid crossing (NCC, as `rectify.py` does for photo 10) and fit the homography plus lens term; this also gives the lens model of this phone. The stock photos have no grid: tie them through content that never moves (dial rails and hooks, tick marks, numerals, the rev band), masking everything that changes (needles, map, texts, telltales). Then **refine inside the STRIP** with a local model on strip-fixed landmarks and report the residual there separately from the whole-panel one.
2. **Publish per photo, frontal ones first** (60, 61, 62, 63, 64, 68, then 66, 67, 65, then the old ones): `bake/work/strip/rect/<id>_panel_2x.png` (3840 × 1440), `<id>_strip_4x.png` (7680 × 720, 16-bit), the model in `bake/work/strip/rect/H.json` **in the schema `tools/g20spec/zoom.py` reads**, and `<id>.READY`. Also write plain 1920 × 720 rects to `bake/work/overlay/rect/60.png … 68.png` and add them to the bench's photo dropdown (find where its list lives), so the owner can put them under the theme.
3. **Error budget.** For each photo: rms and max residual over the panel and over the STRIP, photo px per panel px, and a map of where the model is weakest. A photo whose STRIP residual exceeds 1 px is flagged; say what would fix it. Cross-check: the same fixed landmark measured in every photo must land within your stated error, and if it does not, your error estimate is wrong.
4. **Photometry.** Per photo: linearise; estimate the **display's black level** from panel areas that hold no content (not the bezel, not the hood); model the glare and reflection field (65–67 have strong reflections on the cover glass) and publish a `valid` mask of where colour can be trusted; normalise exposure between photos on a common reference (stock white text, the sign's red ring). Publish `photometry.json` with the numbers and their spread.
5. **Sharpness.** Estimate each photo's blur (edge-spread width in panel px) so the zone packages know how far an edge position can be trusted, and whether a 50 % threshold is biased by blooming on bright text.

Final message: the per-photo table (scale, residuals, black level, blur, usable for geometry / for colour), and anything a photo cannot be used for.

## Packages ZL, ZC, ZR — stock atlas, pixel diff and mask proposal, one zone each

Owns: `tools/strip/Z<x>/`, `design/strip/Z<x>/`, `bake/work/strip/zone-<x>/`. Read-only for everything else.

Each of you does the same four things for the elements of your zone (§4.3).

**1. The stock atlas.** For every element, in every state it appears in:
- its outline as a sub-pixel polygon and as a 4× alpha mask, from at least two photos where two exist; state how the edge was placed (threshold after local black subtraction, an edge-spread fit) and its uncertainty;
- its colour in linear light and sRGB, and any glow or halo around it as a radial profile;
- for text: baseline, cap height, x-height, figure height, stem, digit advance, tracking, and above all the **anchor**: which edge stays put when the string changes. You have several strings of the same field across photos (range, clock, temperature); use them. Then give the **widest string the field can hold** (from the glyph metrics, not from one photo) and the box that follows from it;
- neighbours and the clear gap to each, in px: these gaps are the corridors a mask edge can run in;
- whether BMW draws it under or over a theme, with the evidence (the session photos 56 and 58 and `reference/ours/*` show our opaque paint with BMW on top of it or not). The catalogue has 12 `unknown`; settle what you can and say what only package D's car photo can.
- Reconcile with `design/g20-spec/measured.json` and the catalogue: for every number of theirs in your zone, confirmed, or corrected with the delta.

**2. Unattributed light.** In every stock photo, mask every element you have catalogued (dilated by its blur) and list what is still lit in your zone above the black level: rails, hooks, dashes, glows, ghosts, anything. Each blob is either given an id or listed as unexplained with its box. Nothing in the zone may be left unaccounted for.

**3. The pixel diff**, once `bake/work/strip/pub/O/READY` exists. For both layouts, both modes and every state, classify each pixel of your zone: stock ink that must show and does; **CLIP** (must show, we cover it: give the summed alpha and the count); **LEAK** (must be hidden, shows); **DOUBLE** (ours drawn within a few px of visible stock ink doing the same job); **SAFETY** (any alpha of ours over a belt or airbag telltale or an over-element, footprint dilated by 3 px: always a defect); and **SEAM**. For seams, walk every edge of our coverage and measure the luminance step across it relative to the display's black, in units where stock white text is 1.0, plus ΔE2000 where the background is tinted. Rank the seams by how visible they are. What must show and what must be hidden in each mode is §1.2 and §1.3.

**4. The mask proposal** for your zone, per channel of §4.4: a list of shapes `{id, channel, kind: hole | cover, polygon, feather_in_px, feather_out_px, reason, evidence, clearance}` plus a 4× raster preview. Hug the element; set the feather per edge segment from the background luminance you measured behind it; never let a cover reach a safety telltale. Where an element's footprint varies (the belt group, a text of varying width), the hole follows the widest case. Say what the proposal costs you: every place where hugging one element forces a gap or a clip at a neighbour.

Zone specifics:
- **ZL.** The sign is yours and the owner named it. Measure BMW's ring (outer and inner radius, roundness, centre) to a tenth of a px on 61–64, 66–68, its face, its digits, the empty state. Explain exactly how close the speed readout and the belts come. The range field's pump icon changes between BMW's and ours; record both.
- **ZC.** The panel is yours and the owner named it. Deliver its true outline (arc, slanted ends, where the rail stops), for one and two lines; settle with stock photos (62, 66, 17) whether there is any fill under the message; the symbol slot's box and what sits in it; the road label's anchor and widest string; the standby odometer; and what the ghost in 63 is.
- **ZR.** The marker's three states, its shape and colour; the mode label's standby position; the gear letter (catalogued as drawn over a theme: confirm on 58, which shows two gear letters); the temperature's anchor and widest string including a negative value; the start/stop icon in 68.

Final message: the atlas table, the defect table with pixel counts, the ranked seams, the mask proposal in words, and the conflicts you could not resolve.

## Package T — typography

Owns: `tools/strip/T/`, `design/strip/T/`.

1. For every stock text in the STRIP and the sign's digits: the face's fingerprint (x/cap, stem/cap, figure/cap, advance/figure, "1" against "0"), sizes in px, tracking, whether figures are tabular. Build on `tools/g20spec/fontfit.py` and `fingerprint.py`; the new photos have twice the resolution the old measurement had.
2. Which of our baked fonts draws each of our row items (`wb_row`: range, clock, temperature; `wb_val`: units and the limit; `wb_cap`: road label, message; `wb_capb`) and how far each is from BMW's in height, advance, stem and baseline, in px. The owner likes these items **in BMW's places**: give the bake sizes and offsets that put ours there.
3. Publish `typography.json` to `bake/work/strip/pub/T/` for package F, early, even if provisional.
4. You may use the web to identify candidate free faces. No BMW font file is downloaded.

## Package O — our side, to the pixel

Owns: `tools/strip/O/`, `bake/work/strip/ours/`. Bench port 8161. Read-only for the app.

1. **Alpha and colour of every pixel of the STRIP**, for both layouts, both modes and every state of §4.2, at 1× and 4×, computed from the engine's own output: pull each layer's shader and live uniforms from `/events`, the baked collar and cover from `/px`, and re-implement the alpha path. The earlier audit did exactly this and matched a photo to about 4 px; its scripts are in the session scratchpad (`cover.py`, `cov2.py`, `compose.py`, `elem.py` and others beside `bottom-strip-audit.md`). Publish the base states first and touch `bake/work/strip/pub/O/READY`.
2. **A per-layer breakdown** for each state: which layer contributes what (background, collar, cover, each patch, `u_cc`, the dials' cut ring, `nav_bg`, each text widget with its real ink box and its widest string).
3. **The sign.** Sample our coverage round the sign's centre every degree, r 30–60 in 0.25 px steps, on the current tip and on the parent of `3bfce09`. Find the asymmetry the owner sees in `68.png`, name its cause to the pixel (a layer rectangle clipping the ring? the collar's edge? the changed radii?) and say whether reverting `3bfce09` removes it.
4. **The open box and the missing clock** in `69.png`: explain each from the uniforms.
5. Anything semi-transparent (alpha between 0.02 and 0.98) that is not an intended soft edge is a defect; list them all.
6. Budgets per state: layers per connection, pixel MiB, shader bytes of each effect source.

## Package H — history and claims

Owns: `tools/strip/H/`, `design/strip/H/`. Read-only for everything else.

1. **The genealogy of every geometry constant** that shapes the strip or the sign: `CC_BOX`, `CC_TOP`, `CC_TOP_SOFT`, `NAV_BAR`, `NAV_HOLE`, `SIGN_WIN`, each patch rect and each row text box in `design/gen_6wb.py`; the cut modes in `shared/hud.js`; the ring reach in `shared/shaders.js`; `shared/visible.js`; the catalogue's boxes; `measured.json`. For each: the commit that set it, the photo it was measured on, **whether that photo was stock**, and what it was assumed to hold. `CC_BOX` is already caught: measured on a photo where the mode label stood somewhere else.
2. **Photo provenance.** Every photo in `reference/g20/` and `reference/ours/`: stock, ours, or grid; state; resolution; which documents cite it. Settle 39 and 40 by looking at them.
3. **Claims against the new photos.** Go through `docs/G20-DESIGN-SPEC.md` (all of §5 and §8), `docs/BOTTOM-ROW-PACKAGE.md`, the OVERLAY, CAR3 and POLISH reports: every statement about the strip, checked against 61–68 (on R's rects once they exist), with a verdict.
4. Deliver the constants ranked by how likely they are wrong, and a list of document sections to correct.

## Package P — pictograms and catalogues (web)

Owns: `design/strip/pictograms/`, `tools/strip/P/`.

1. **What BMW's marker means** (no marker / warning triangle / "i") and when each shows; how long a message stays, hides and returns; what the strip does in standby. The G20 owner's manual and BMW's own documentation first. Compare with `config.ccm` and its `_timings` note in `app.json`.
2. **A catalogue from `cCSymbolID` and check-control ids to meanings**, from public sources, as far as it can be built. Which ids are safety-critical.
3. **Our own drawings.** The severity marker for variant A (warning triangle, "i", a neutral one) and the common message symbols (door, brake, tyre pressure, engine, washer fluid, belt), drawn by us after ISO 2575 as SVG, plus baked alpha at the sizes T and ZR measure. A provenance and licence note beside every file.
4. **Our sign art against the rules:** the speed-limit and no-overtaking signs per the Vienna Convention's proportions and against the stock ring ZL measures.
5. Find how icons are baked and referenced today (`shared/icons.js`, the layouts' `icon` widgets) so your files drop in.
6. No VIN and nothing personal in any search. Follow §3 "Not ours to ship" to the letter.

## Package M — the mask mechanism (build)

Owns: `shared/shaders.js`; a new, separate mask block in `shared/hud.js`; `tools/strip/bake_mask.py`; a new `strip_mask()` in `design/gen_6wb.py`. Bench port 8162.

1. **Establish what the renderer can do**, from the skill's references, `shared/cluster_render_client.js` and the bench: how an effect shader samples an image asset (the background already samples the collar and cover), filtering (nearest or bilinear), formats, the per-sheet pixel limit (the app logs "sheet limit 345600 px": a full-resolution STRIP is exactly 1920 × 180 = 345 600), the upload budget.
2. **Decide plain alpha at full resolution against a distance field at half resolution**, by measurement: edge position error, how the feather is controlled, memory, shader bytes. The owner wants a soft edge of a couple of pixels that reads as a shadow; a distance field would make its width one uniform.
3. **Prototype on the bench with a synthetic mask** built from today's rectangles, and prove that with it every state reproduces today's output (compare against O's maps, or your own readback, to 1/255). Then show a shaped hole and a feather working.
4. **fp16:** a texture coordinate across 1920 px in `mediump` is off by pixels on the car. Solve it and say how.
5. **The sign window.** It lives in the dials' layer today (`u_cut`). Say whether it can move into the mask, and at what cost; O's finding on the asymmetry is your input.
6. **The bake tool:** `tools/strip/bake_mask.py` takes the zone packages' shape lists (§ZL–ZR.4) and writes the texture and the layout entries.
7. **Budget table** before and after: layers, pixel MiB including the layout-switch peak, shader bytes against the KOMBI rule. Removing the patch, `u_cc` and hole code should pay for the mask; show it.
8. Keep the old rectangle path until the lead has switched over; remove it in a separate, last commit.

## Package F — fonts: Cyrillic (build)

Owns: `tools/bake_fonts.py`, `tools/fonts-src/`, `apps/f30_cluster/hud/fonts/`, the text-metric constants in `design/gen_6wb.py`, and the regeneration of all five 6WB layouts. Bench port 8163.

1. **Bake Ukrainian into every font that draws text the car supplies** (`wb_cap`, `wb_capb`, and check for others): the full Ukrainian alphabet in both cases, the apostrophe forms, guillemets, the dash, the number sign. Check that `tools/fonts-src/Roboto-wdth-wght.subset.ttf` holds the glyphs; if the subset dropped them, rebuild it from upstream Roboto (OFL), not from anything of BMW's.
2. **Measure the cost** per layout: atlas size, pixel MiB, the switch peak. Propose the wider Cyrillic set as an option with its price.
3. **Fix the generator drift for good.** Regenerate the five layouts, and for every text widget give the ink box before and after, computed, not eyeballed. Each move is either intended or undone. After you, `gen_6wb.py` reproduces the committed layouts exactly, and a test says so.
4. A Cyrillic message and road label must draw with no latch; add the bench scenes and the tests. Decide and document what happens to a character still missing (a replacement glyph, or the row falls back).
5. Take T's sizes from `bake/work/strip/pub/T/typography.json` if it is there before you finish; otherwise leave the sizes and say so.

## Package X — row logic and our message (build)

Owns: the row logic in `shared/hud.js` (`_own`, `_rowFail`, `_rowCheck`, `_wbCcSend`, `_ccOpen`, the clock's zone), `apps/f30_cluster/app.json` settings and config, `cc_row()` / `nav_row()` and the row parts of `design/gen_6wb.py`, `bench/fake-signs.js`, `bench/test-bottom-row.js`. Bench port 8164. Use today's rectangles for coverage; M's mask replaces them later.

1. **All-or-nothing fallback** (§1.5), logged once with the reason.
2. **The clock is ours in `all6WB`** (§1.4). `config.clock` has an unverified 15-minute factor: accept an offset only if it is a whole number of quarter hours within ±14 h, else treat it as unknown and use the next source. Log one line the owner can check against the car at a glance.
3. **Standby is BMW's row** (§1.6).
4. **Our message, variants A and B** (§1.7), behind a setting so the owner can flip them on the bench. One line, the headline only, cut to fit with an ellipsis, the road label hidden while it shows, BMW's panel fully covered while ours is drawn.
5. **The safety rule as code and as tests** (§1): the car says a message is showing and our text is not on the screen in that frame → BMW's panel is visible. One test per way it can happen: an unreadable struct, a missing glyph, a throw while drawing, the gap between the message arriving and our first draw.
6. **The marker:** if a signal tells us stored messages exist, propose drawing our own; if not, BMW's marker must stay visible in both modes. Evidence first.

## Package D — the diagnostic theme for the car (build, small)

Owns: a new `design/gen_matte.py` and its layout, one theme-list entry in `app.json`, an addition to `docs/OVERLAY-PHOTO-PROTOCOL.md`. Bench port 8165.

A calibration theme like `grid` and `mapcal`: the whole panel opaque in one flat colour (offer magenta, green and black) with a pixel ruler along the STRIP, using existing shaders only and the smallest possible layer and memory count. One photo of it on the car shows exactly which stock elements BMW draws **over** a theme, which settles the catalogue's 12 `unknown`. Deliver the theme, its bench proof, and a shot list for the owner (states, distance, angle, exposure). **It ships only when the owner says so.**

---

## 6. What comes back, and what happens next

Your final message is all the lead sees. Lead with the verdicts, then the tables; mark verified against hypothesis; name your branch and commits, what you published under `bake/work/strip/pub/`, any process you left running (port, pid), and what the owner must decide.

Then the lead merges F, X, M and D, regenerates the layouts, bakes the final mask from the three zone proposals with M's tool, runs the suites and the budgets, and puts both message variants and both modes on the bench over the new photos for the owner.
