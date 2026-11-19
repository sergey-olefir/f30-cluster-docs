# GL: the garbage in the hub, the drifting digits

Car issues of 2026-11-19, package GL. Branch `car-issues/gl` (worktree `.claude/worktrees/ci-gl`), commit `943899d`. Not pushed, no version bump. No car session. The 2026-11-19 logs had not arrived when this report was written.

Status words: **proven (code)** means read in our source; **proven (bench)** means shown by our own engine on the Mac; **proven (data)** means read in the 2026-11-18 car logs; **hypothesis** means the car has not confirmed it.

## 0. In short

- **The cause (picture: proven on the bench; location: hypothesis).** The gear's glyph atlas is an R8 texture with rows of **1066 bytes**, and 1066 is not a multiple of 4. If that texture is read with its rows **1068** bytes apart (a 4-byte row alignment), you get exactly photo A's hub: slanted strokes at 2 px a row (26.6°; the photo measures about 24°), made of pieces of neighbouring cells. f30's `@r8` dials (670 bytes a row) read 672 apart give photo B (TX found this independently; we agree). **Every one of our textures that broke on the car has rows that are 2 bytes off a multiple of 4. Every one that never broke is aligned.**
- **Why the gear and not the speed (proven in code).** Both are "gpu" texts on `td_big`, each with its own 1066-wide atlas. The speed only ever shows figures, and those come from the atlas's first whole upload. The gear's letters (P R N D S M, and "D1", "M3" and so on) are not tabular figures. `_exactCodes` cuts them into 41-px cells, and `_exactPut` writes those cells **into the atlas in place**. That is the only place in the whole engine where an R8 texture is written after its first upload. Photo A shows a correct speed "0" next to a broken gear: one texture written once, one written in place.
- **What still needs the car (hypothesis).** Where the 4-byte pitch is applied (cluster-render's in-place write, or the cluster's upload) and why it happens only sometimes. The best fit to every observation is this: an asset's first pixels that arrive with a **new scene** are right, and pixels written **in place while the scene is up** are re-read with the padded pitch.
- **What I built (commit `943899d`).**
  - Every glyph atlas now has rows that are a multiple of 4 bytes. `td_big` goes from 26 cells to 28 cells (1066x70 → 1148x70).
  - Every rect written into an atlas now spans whole 4-texel columns (41 → 44 px).
  - The shader is unchanged and the picture is pixel-identical (bench).
  - f30_cluster is unchanged byte for byte. Its only GPU text atlas, `wb_big` (832 = 26 × 32), was already aligned.
  - A Діагностика switch in Twin Dial, **"Цифри на GPU (тест)"**, has three options: off = the fix; **"атлас 0.1.3"** = 0.1.3's layout back, for the car test; **"пікселями"** = the speed and the gear drawn as image text.
  - Log lines, numbers only.
- **The car test (§6)** takes one parked session of about ten minutes. Shift through P R N D a few times with each option of the switch. If 0.1.3's layout breaks the letters and the default does not, the hypothesis is confirmed.
- **Decisions (§8):** whether the aligned atlas ships as Twin Dial's default before the car test (I built it as the default), and whether the test switch stays in the release.

## 1. The path, end to end, and every way a right text can become wrong glyphs

The gear is `twin.json` widget `gear`: text `{{td:gear}}`, font `td_big`, box [298, 365, 462, 435], `"gpu": true`, `conn` 1 (the second connection), `maxLen` 2. The speed is `speed`: box [1410, 374, 1670, 444], `conn` 0, `maxLen` 3.

`_gpuTexts` turns each of them into one `atext` effect layer (shader `ATEXT`, `shared/shaders.js`) over an exact atlas (`_exactAtlas`):

- The pitch is the figure advance, 82 px.
- The cells are half a pitch, 41 px. Lengths 1 and 2 (gear) and 1 to 3 (speed) start on different half-pitch phases.
- The gear's layer has 4 cells of 41 = 164 px. The speed's has 6 cells.
- Each atlas holds 10 figures in 20 cells plus 6 spare cells: 26 × 41 = **1066 x 70**, R8, sampled nearest.
- Its key is `<conn>:exact:td_big:82/41x70:0`, so the two atlases are separate assets on separate connections.

Car log of 2026-11-18: `gpu text atlas "td_big": 10 figures in 20 cells and 6 spare cells of 41x70, 1066x70 R8 asset` (proven, data).

| # | Way it could go wrong | Verdict | How |
|---|---|---|---|
| 1 | **R8 rows not a multiple of 4 bytes, re-read at a 4-byte pitch** | **leading hypothesis**. The picture is proven on the bench. The location and the "sometimes" are not proven | §2, §3 |
| 2 | The atlas replaced, dropped or made again under a live layer (tab change, map window, `mbRebuild`, drive mode, night look, start-up hold) | **ruled out** (code) | None of these changes the exact atlas's key, so the same atlas object and asset ref survive every Twin Dial rebuild (`_sceneNeeds` keeps the key, `_dropsUnused` drops only unused keys). Client refs are never reused (`nextRef++`). A wrong atlas would also give whole wrong glyphs, not slanted strokes |
| 3 | A uniform names cells of an atlas that is not the bound one | **ruled out** (code) | `x.atlas` is the object that was bound (`_create`), and the codes come from it. Each atlas has exactly one layer |
| 4 | Two connections with no order between them | **ruled out** (code) | The gear's layer, its atlas, its bind, its uniforms and its pixel writes all go on connection 2: one socket, in order |
| 5 | fp16: atlas coordinates in `mediump` | **ruled out** (code) | ATEXT is `precision highp float`. The cell maths runs in cells (≤ 32), and uv is in 0..1. Texel 1066 needs 11 bits; highp has 23. It would also be wrong on every frame and for the speed too |
| 6 | The atlas's size against the car's limits | **ruled out** (data) | 1066x70 R8 was accepted (`asset 84 r8 1066x70 ... 0 failed`, status of 2026-11-18). 1148x70 is within 2048 |
| 7 | "D" against "P" in `td_big` | not a cause by itself | Both are letters (advance 78 and 74, against 82 for the figures), so both go through spare cells written in place |
| 8 | Order of a cell's pixels and its uniform: a uniform goes out at once, a frame waits while the socket is blocked | **hypothesis, transient only** | For a fraction of a second the new code can show the spare cell's old content (another value's slice). It heals at the next drain, so it cannot explain a picture that stays |

## 2. The evidence

### 2.1 Proven (data): which textures broke, and their rows

The textures and their sizes come from the cluster-render status and the app logs of 2026-11-18, and from `apps/*/hud/scene/*.json`.

| Texture | Format, size | Row mod 4 | On the car |
|---|---|---|---|
| Twin Dial `face_tacho`, `face_speedo` | R8 616x560 | 0 | never broken ("its dials stay right") |
| Twin Dial `panel_mask` | R8 536x112 | 0 | never broken |
| Twin Dial `td_big` exact atlas (speed and gear) | R8 1066x70 | **2** | **the gear broke** (photo A) |
| Twin Dial 0.1.0 `td_gear` exact atlas | R8 912x50 | 0 | no broken gear reported (owner to confirm), although its 57-px cells were written in place up to **19 times** a session (`asset 11 r8 912x50 ... 19 writes`) |
| f30 `speedo(_d)@r8`, `tacho(_d)@r8` | R8 670x670 | **2** | **broken** (photo B: TX) |
| f30 `range_seg@r8`, `ed_seg@r8` | R8 346x68 / 346x70 | **2** | in both lighter modes (TX) |
| f30 every RGBA8 texture, `strip_mask` 1908, `collar_r8` 480, `wb_big` 832, `fuel@r8` / `oil@r8` 144 | | 0 | never broken |

The `td_gear` row is the useful one. It shows that rects whose **own** width is not a multiple of 4 (57 px) were harmless inside an aligned atlas. This points at the asset's **whole rows** being what gets re-read, which fits §2.3.

### 2.2 Proven (code): the gear is the only "gpu" text written in place

`shared/hud.js` holds exactly one R8 rect write: `_exactPut`. Every other asset upload is whole. Every other rect write is RGBA (rows of 4·w bytes, always aligned).

I ran the engine's own `_gpuTexts` / `_exactAtlas` / `_exactCodes` against a recording client (`bench/test-gpu-rows.js`).

- **The gear.** 38 values (P R N D S M – D1..D8 M1..M8 S1..S8 and a gear-change sequence) cause **80 in-place writes** (82 in the 0.1.3 layout).
- **The speed.** 11 values (0 to 299) cause **none**. Only its "–" (unknown) writes 2 cells.

### 2.3 Proven (bench): the angle and the content of photo A's strokes

Measured on photo A (a crop of the hub and of the speed's "0", both scaled by the same factor):

- **Scale.** The "0" is 63 x 65 px of ink on the panel (`td_big` mask) and 187 x 180 px in the crop: 2.97 crop px per panel px across, 2.77 down.
- **Size.** The gear's garbage spans about 470 x 175 crop px, which is about **158 x 63 panel px**. That is all 4 cells of the layer (164 px), so the value had two characters: in D at a standstill the token is `D` + the gear, probably "D1".
- **Angle.** The long strokes rise at a slope of about 0.42 in the crop, or about 0.45 once the vertical scale is corrected: **about 24°**, rising to the right ("/").

The engine's real atlas bytes for "P", then "D1", were read under four models (scratch renders, not kept):

| Model | Shear | Angle | Look |
|---|---|---|---|
| whole atlas re-read at 1068 (4-byte pitch of 1066) | 2 px a row | **26.6°** | a "7" at the left, long parallel "/" strokes, pieces of the spare cells to the right and, low down, of row 0's figures: "pieces of other glyphs". **Closest to the photo** |
| the 41-px rect read at 44 (4-byte pitch of the rect alone) | 3 px a row | 18.4° | many thin stripes, all from the cell's own glyph. A worse fit |
| the rect read at 42 (2-byte pitch) | 1 px a row | 45° | too steep |
| tight (what the protocol says, and the bench does) | none | – | "D1" right |

The slant direction matches too. A stride larger than the data moves each row's content left as y grows, which gives "/" strokes, as in the photo.

The same 4-byte pitch applied to f30's `speedo_d@r8` / `tacho_d@r8` (670 → 672) gives photo B: parallel diagonal streaks of ticks and the hub ring turned into a tilted ellipse (my render, and TX's `bake/work/car-issues/pub/TX/shear-sim.png`).

### 2.4 Hypothesis: why only sometimes, and why only a reboot cures it

The upstream protocol says the service "keeps [an asset's] pixels, writes them whole into each new scene of its target and in place as they change". My best fit to every observation is that the **in-place** path re-reads an R8 texture's whole rows at a 4-byte pitch, while the **new-scene** path is right. This fits each observation:

- **Photo A's speed is right and its gear is wrong.** The speed's atlas was written once, at a build. The gear's was written again in place.
- **The faults start at a switch.** f30's lighter-mode textures are made at a live switch, while the panel scene is up, so they take the in-place path. Twin Dial's gear writes a new cell whenever a value with new cells appears.
- **The faults are intermittent.** Whether a write lands in a new scene or in place depends on timing. Examples: whether the panel scene was released between a build's drops and its new layers; whether the socket was blocked, so that the gear's first letter rode inside the atlas's pending whole upload (`cluster_render_client.js`: rects merge into `whole` until it is sent).
- **"Only a reboot cures it."** After a reboot the first build goes into a new scene. Nothing short of that re-sends a sheared texture by the right path (a Twin Dial rebuild keeps the atlas, `byKey` keeps the cells).

This is not proven. Two things would settle it: cluster-render's source, or its log of an in-place write. It could also be GL state on the cluster that is sometimes 4 and sometimes 1 (TX's alternative). **The change in §3 removes the condition in both cases**: no row and no rect of a glyph atlas is ever 2 bytes off.

## 3. What is built (commit `943899d`, marked `car-issues GL`)

`shared/hud.js`:

- **`_exactAtlas`.** The columns are chosen so that each row, cols × cw, is a multiple of 4 (`alignedGrid`). A one-row atlas gets a few more cells, and the extra ones are spare. A multi-row atlas gets fewer columns. No shader change: ATEXT's `u_grid` stays (cols, rows) and W = cols × cw exactly.

  | | Atlas | Spare cells | Bytes | Pixel memory |
  |---|---|---|---|---|
  | `td_big` before | 1066x70 | 6 | 74 620 | 149 240 |
  | `td_big` after | 1148x70 | 8 | 80 360 | 160 720 |

  Twin Dial makes two of these atlases, so the total is +11 480 B uploaded and +22 960 B of pixel memory, about 0.02 MiB.
- **`_exactPut`.** The cell goes into a kept copy of the atlas (`a.buf`). It is then sent as a rect of whole 4-texel columns around the cell, with the neighbouring texels as they are: 44x70 (3 080 B) instead of 41x70 (2 870 B).
- **`_atlas`** (the plain `atext` atlases). The same guard on its columns. Sixteen columns are always aligned, so only a cell wider than 128 px changes. No shipped layout has one: the calibration layout's `atext` uses `unit`.
- **`config.gpuText`.**
  - `"legacy"` brings back 0.1.3's layout: 1066 wide, 41-px rects, and its own atlas key, so a switch makes the atlas again.
  - `"pixels"` draws every "gpu" text as an image text.
  - Anything else gives the aligned layout. app.json carries no `gpuText`.
- **Log lines (numbers only).**
  - At atlas creation: `..., rows 1066 bytes: NOT a multiple of 4 (config.gpuText "legacy")`. Nothing is added when the rows are aligned.
  - At each in-place write: `gpu text atlas "td_big" on connection 2: cell 22 written in place, rect 900,0 44x70, write 3`. The first 20 writes are logged, then every 50th.

`shared/diag_settings.js`: a change of `gpuText` asks for a rebuild. The rebuild drops the layers before the atlas, so the asset is never dropped while a layer still binds it. This is not TX's §2 path (`sceneTex`), which drops assets first.

`apps/twin_dial/app.json`: the setting `diagGpuText`, "Цифри на GPU (тест)", a choice of вимк. / атлас 0.1.3 / пікселями, default off. It is on the DIAG screen, skipped from SETTINGS, and listed in `config.diag.settings`.

f30_cluster is unchanged: the same `wb_big` atlas byte for byte and the same 7 uploads rect for rect (tested). No f30 setting was added.

## 4. f30_cluster's GPU texts (0.7.14 on)

**Not affected** (proven in code and on the bench). The only one is 6WB Sport's digital speed (`wb_sport.json`, `wb_sport_dials.json`, font `wb_big`, on connection 2): pitch 64, cells of 32 px, 26 cells, so **832x92**, rows a multiple of 4. Its in-place writes (the "--" cells) are 32 px wide at multiples of 32, also aligned. f30's photo B is the scene textures (`@r8`, 670 and 346 wide), not a GPU text: TX's package.

## 5. With TX: one cause or two

TX's notes (`bake/work/car-issues/pub/TX/NOTES.md`, read at 15:04 and again before this report, unchanged) and mine agree.

- **Agree: one cause for both pictures.** R8 rows 2 bytes off a multiple of 4, re-read at a 4-byte pitch. TX found it from the f30 side and I found it from the gear; the tables match.
- **Agree: where it sits is a hypothesis.** The default f30 path (setting Off) cannot reach the shear, because every R8 texture it makes is aligned. Twin Dial's default reached it on every start (`td_big` 1066), and it showed whenever the gear wrote in place.
- **Differ: the shader.** TX suggested padding the atlas width and changing the shader's divisor. That is not needed: choosing the columns makes the rows aligned with the shader untouched (one less KOMBI shader risk).
- **Differ (small), on the model.** TX leaves open "a GL state that flips" against "a second upload path". The photo's angle (2 px a row, not 3) and the 0.1.0 `td_gear` atlas (aligned rows, unaligned 57-px rects, up to 19 in-place writes, no reported fault) favour **whole rows re-read on an in-place change** over **the rect's own rows**. Both are covered by the change.
- **TX's second finding (the live `diagSceneTex` switch drops assets that are still bound, so they leak):** a separate fault. It does not draw shear. It fits "does not draw, lags, only a reboot". It is not in my code: my switch rebuilds and does not drop first, and the bench trace confirms that no asset is dropped while bound.

## 6. The car test (for the owner; parked, engine running, foot on the brake, belt lamp check first)

**Twin Dial, the new build, tab 1, map window off.**

1. **Default (Діагностика "Цифри на GPU (тест)" = вимк.).**
   - Shift P → R → N → D → N → R → P, holding each for 2 s. Do it twice.
   - Then change tabs, open the map window (W2) and close it, and do one P → D → P after each.
   - Photograph the hub only if a letter is wrong. **Expected: never wrong.**
2. **"атлас 0.1.3" (0.1.3's layout).**
   - The same shifting.
   - **Expected under the hypothesis: R, N, D (or D1) slanted, at least some of them, as in photo A.** Photograph the first wrong letter, with the car clock in the frame.
3. **"пікселями".** The same shifting. **Expected: always right.** This is also the fallback if step 1 ever shows a wrong letter.
4. **Back to вимк.** Then, if time allows, switch f30_cluster → Twin Dial once (the panel scene stays up) and shift again.

| Outcome | Reading |
|---|---|
| Step 2 wrong, steps 1 and 3 right | The hypothesis is confirmed on the car. The fix stays and the switch can go |
| Step 2 right as well | The in-place path did not trigger this time (it is intermittent). Not conclusive: repeat step 2 after step 4 |
| Step 1 wrong | The row alignment is not the whole cause. Use "пікселями" and send the logs |

**What the lead reads afterwards:**

- `twin_dial.log`: the lines `gpu text atlas "td_big" ...` (1148x70 by default, 1066x70 with "NOT a multiple of 4" in step 2), `cell N written in place, rect x,0 44x70` (41x70 in step 2), and `diag: diagGpuText <option>`, with their times against the photos.
- `cluster-render.log` and `status.txt`: `asset N r8 1148x70 of twin_dial_2 ...: K writes, 0 failed`, and the `goes whole` lines (a new scene) around the shifts. A wrong letter after a "goes whole" and before any write would argue against the in-place model.
- The journal: any renderer error near the times of the photos.

## 7. What the 2026-11-19 logs would settle (not arrived)

1. **Photo A at 14:18 car time (12:18 UTC).**
   - In `twin_dial.log`: the build (`gpu text atlas "td_big" ... 1066x70`) and how long before the photo it was.
   - In `cluster-render`'s status: how many writes the gear's atlas had (`asset N r8 1066x70 of twin_dial_2: K writes`). K > 1 means in-place writes happened.
   - In the service log: whether a `goes whole` fell between them.
2. **Photo B at 14:24.** Whether the `@r8` textures were made at a live switch (an `asset N on panel (r8 670x670)` line while the scene was shown, with no new scene) or at a start. This is TX's reading too.
3. **The "drifting digits".** Whether the speed's atlas (`twin_dial`, connection 1) ever had more than one write (the "–" when the speed is unknown). If it did, the speed can drift too.

## 8. Decisions for the owner

1. **Ship the aligned atlases as Twin Dial's default before the car test?** I built it as the default.
   - **For:** the picture is identical (bench: 38 gear values and 11 speeds pixel for pixel), the cost is about 0.02 MiB, and it removes the only condition both car models need.
   - **Against:** the cause is not yet confirmed on the car.
   - The alternative is to keep 0.1.3's layout by default: `"gpuText": "legacy"` in app.json config, one line.
2. **Keep the Діагностика switch in the release?** It is needed for the car test (§6). It can go once the test is done.
3. **f30_cluster: nothing from GL.** TX's padding of the `@r8` files and the leak are TX's decisions to bring.
4. **Report to cluster-render's author?** Optional: "an R8 asset whose rows are not a multiple of 4 bytes is drawn as if they were, after an in-place write". Only after the car test, and only with the owner's word.

## 9. Tests

All suites were run alone, one after another, on ports 8920 and 8921. The full runner was not run.

| Suite | Result |
|---|---|
| `test-gpu-rows` (new) | 48 passed, 0 failed (sections 1-6 run without the bench; section 7 is a live switch on the bench). With main's `shared/hud.js`: 12 fail |
| `test-diag` | 104 / 0 |
| `test-twin-budget` | 95 / 0 |
| `test-native-ui` | 47 / 0 |
| `test-twin-sim` | 39 / 0 |
| `test-twin-cc` | 266 / 0 |
| `test-scene-tex` | 54 / 0 |
| `test-twin-trip` | 89 / 0 |
| `test-twin-limit` | 224 / 0 |
| `test-twin-tabs` | 252 / 0 |
| `test-twin-centre` | 242 / 0 |
| `test-twin-layout` | 108 / 0 |
| `test-twin-fonts` | 327 / 0 |
| `test-twin-pages` | 100 / 0 |
| `test-twin-bake` | 37 / 0 |
| `test-fonts` | 34 / 0 |
| `test-cc-release` | 75 / 0 |
| `test-strip-mask` | 40 / 0 |

`check_signals.py` for `apps/twin_dial/app.json`: all declarations pass. No signal was added.

Sources:

- Photos A and B: `bake/work/car-issues/in/`, described by numbers only.
- Car logs: `reference/car/2026-11-18-collect1/` (`cluster-render/status.txt`, `cluster-render.log*`, `apps/twin_dial/twin_dial.log*`).
- Protocol text: `bake/work/research3/MT/upstream/PROTOCOL.md` ("Frames", "An asset is not drawn ...").
- Fonts: `apps/twin_dial/hud/fonts/td_big.json` (advances: figures 82, D 78, P 74, N 83, R 77, S 71, M 101).
