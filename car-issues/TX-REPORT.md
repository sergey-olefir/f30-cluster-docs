# car-issues TX — "compressed textures": our theme not drawn, lags, artefacts, wrong until a reboot

Branch `car-issues/tx` (worktree `.claude/worktrees/ci-tx`), from `main` 8421a64. Not pushed, no version bump, no car connection.
Status words: **proven (bench)**, **proven (code)**, **proven (data)** (the car's saved logs / our files), **hypothesis** (the car has not confirmed it).

## 0. The answer first

**Two separate faults, both ours to avoid, neither in BMW's content.**

1. **The picture in photo B is a row-pitch fault on the rendering side, not in the app.**
   The app sends the right bytes. Somewhere between cluster-render's write and the cluster's GPU, an **R8 texture whose rows are not a multiple of 4 bytes** is re-read at the padded pitch: 672 bytes a row instead of 670.
   - The shape is **proven (bench, our own bytes)**: reading `speedo@r8` / `tacho@r8` that way reproduces photo B's diagonal tick streaks, the tachometer's hub ring as a blue ellipse and the red-zone smear at the lower right (`bake/work/car-issues/pub/TX/shear-sim.png`).
   - The discriminator is **proven (data)**: in photo B the 144-wide R8 oil mini, sent at the same time by the same path, is intact. Every R8 texture the car has drawn right has 4-aligned rows.
   - **Which component holds the state is a hypothesis:** cluster-render's in-place write, or the KOMBI's renderer. **The app does not hold it**: its bytes are right, and the same bytes arrive after a restart.
   - Twin Dial's gear is the same fault, on the 1066x70 `td_big` atlas (GL found the same independently; see §6).
2. **"It does not draw, it lags, only a reboot helps" after a switch is a leak in the app (and mirrored in the cluster-render service for the life of the connection).** This is **proven (bench), fixed on this branch.**
   - A live change of **"Lighter scene textures"** dropped the old scene textures while the old layers still bound them. The service refuses such a drop (`asset in use`) and the client keeps the asset. The engine forgot it, so nothing ever dropped it again.
   - After one switch (off → r8), connection 1 sat at its **16-asset cap** and 5 new textures were refused (`too many assets`). Three dial layers kept an **unbound sampler**, which the car does not draw (memory note on RAMSES). Pixel memory was 14.6 MB instead of 5.97 MB.
   - Every leaked asset rides along on each whole-scene re-send (+4.39 MB per KOMBI map after the first switch): the lag.
   - It lasts as long as the app process, which in practice means until a reboot.

**Can the default path reach it?**
- **f30 with the setting Off, no switch:** no. Every R8 texture it makes is 4-aligned (bench, 7 layouts: 448, 480, 832 and 1908 wide; the 2026-11-18 car log adds 1072). The leak needs the setting's own drop, and with the setting Off the orphan sweep never fired (bench, 10 layout and theme changes).
- **Twin Dial:** yes, for the shear, on every start. Its `td_big` atlas is 1066 wide (Twin Dial 0.1.3; 0.1.0's gear atlas was an aligned 912). That is GL's fix. Twin Dial has no leak path.

## 1. What was checked

| what | how | result |
|---|---|---|
| the switch, live | scratch driver `repro_switch.js` against the bench (`BENCH_PORT` 8931), f30 Comfort, `diagSceneTex` off → r8 → etc2 → off → r8 → etc2, the scene read after each step | the leak, §2 |
| every drop path | reading `shared/hud.js`, `shared/diag_settings.js`, `shared/mb_exp.js`, the client | only `Diag.changed()` drops assets before the layers that bind them |
| who makes assets | `grep '.asset('` | only hud.js `_atlas`, `_scene`, `_exactAtlas`, `_blank`; all four register in `hud.atlases` |
| the protocol | upstream `PROTOCOL.md` (copy in `bake/work/research3/MT/upstream/`) | `drop`: "an asset no layer samples (else refused: `asset in use`)"; 16 assets per client, 32 in all; ETC2 sent once, charged once |
| the bench model | `bench/server.js` | models `asset in use`, both asset caps and 16 MiB; the leak reproduces there exactly |
| photo B's shape | `shear-sim.png`: our `@r8` value × colour, read at 670 vs 672 a row; `td_big` at 1066 vs 1068 | photo B and photo A's gear strokes |
| R8 widths, all | every `hud/scene/*.json` of both apps, the bench's atlases over f30's layouts, the 2026-11-18 cluster-render log | §3 table |
| a healthy session | `reference/car/2026-11-18-collect1/cluster-render/` and app logs | what the next logs must be read against, §5 |

## 2. The leak, proven on the bench (f30 0.7.21, `main`)

| step (live) | pixel memory | assets (conn 1 + 2) | left behind | unbound effect layers | service refusals |
|---|---|---|---|---|---|
| Comfort, off (cold) | 11 963 960 | 9 + 2 | 0 | 0 | — |
| → r8 | 14 600 472 (cold r8: 5 965 040) | **16** + 4 | 10 | 3 (tacho, fuel, oil) | 10 × `asset in use`, 5 × `too many assets`, 5 binds `no such asset` |
| → off | 14 725 784 | 16 + 6 | 19 | 5 | 8 × `too many assets`, 8 binds refused |
| → r8 | 14 976 408 | 16 + 10 | 23 | 5 | 12 × `too many assets` |
| → etc2 | 15 101 720 | 16 + 12 | 25 | 5 | 8 × `too many assets` |

- Cause, **proven (code)**: `Diag.changed()` (shared/diag_settings.js) dropped every `:scene:` asset and deleted it from `hud.atlases`, then called `_rebuild()`, which dropped the layers.
  - The client's `drop()` keeps an asset a layer still binds and sends the drop anyway. The service refuses it (`asset in use`).
  - Nothing names the asset any more, so it stays in the client's map and in the service until the connection ends.
  - The client even replays it after a reconnect.
- Why `bench/test-diag.js` passed: it looks for `asset refused`, but the engine's line is `refused drop on 2: asset in use`.
- Why only a reboot cures it: the state lives in the app's client and in the service's per-connection asset table. Both end only when the app process ends, and nothing short of a reboot ends it on the car.
- The lag (from MT's per-texture bytes): after the first switch the 10 leaked RGBA8 assets (4 393 776 bytes) are written into every new scene. That is about 7.4 MB per KOMBI map instead of 3.0 MB: about 3.7 s instead of 1.5 s at the 2000 KB/s floor. The healthy 2026-11-18 log has 51 such re-sends.

**Fix (commit 50b5643):**
- `Diag.changed()` only forgets the scene textures.
- `HudApp._dropOrphans()` (hud.js, marked `car-issues TX`) runs in `_rebuildInner` right after the old layers are dropped and before anything new is made. It drops every asset a connection holds that `hud.atlases` does not name and no layer binds.
- The drop-before-make order (memory note on peak pixel memory) is kept.
- With the fix, every step of off → r8 → etc2 → off → r8 → Sport → Comfort → G20 on / off → etc2 → off gives exactly the cold start's pixel memory: 11 963 960 / 5 965 040 / 3 949 536 / 1 975 064 / 6 538 120. Nothing is refused, nothing is left behind, every sampler is bound.
- On the default path (10 layout, theme, G20 and off changes, setting Off) the sweep never fires.

## 3. The shear: R8 rows that are not a multiple of 4 bytes

| texture | app / mode | size | row bytes mod 4 | on the car |
|---|---|---|---|---|
| strip_mask | f30, default | 1908x353 R8 | 0 | right (many sessions) |
| collar_r8 | f30, default | 480x180 R8 | 0 | right |
| glyph atlases | f30, default | 448x111, 832x92, 1072x90 R8 | 0 | right |
| face_speedo / face_tacho | Twin Dial | 616x560 R8 | 0 | right ("the dials stay right") |
| panel_mask | Twin Dial | 536x112 R8 | 0 | right |
| td_gear atlas (gear, 0.1.0) | Twin Dial 0.1.0 | 912x50 R8 | 0 | no broken gear reported on 11-18 |
| **td_big atlas** (speed conn 1, **gear conn 2**) | Twin Dial 0.1.3 | **1066x70 R8** | **2** | **gear broken (photo A)**; speed right |
| fuel@r8 / oil@r8 | f30, `r8` | 144x144 R8 | 0 | **intact in photo B** |
| **speedo@r8, tacho@r8** (+ `_d`) | f30, `r8` | **670x670 R8** | **2** | **sheared in photo B** |
| **range_seg@r8 / ed_seg@r8** | f30, `r8` and `etc2` | **346x68 / 346x70 R8** | **2** | not seen yet |
| window@r8 | f30, `r8` | 320x286 R8 | 0 | — |
| dials, minis, window `@etc2` | f30, `etc2` | 672x672, 144x144, 320x288 etc2rgb | (blocks) | **never drawn on a car** |

Sources: files in `apps/*/hud/scene/`; the bench's atlas lines; `reference/car/2026-11-18-collect1` (cluster-render `asset … (r8 WxH)` lines and the app's `cluster-render status` lines).

Photo B, read against the simulation:
- The needles, the hubs, our speed-limit sign and the figures (image sheets) are right. The figures stand on a circle.
- Only the baked dial faces are sheared. A circle read 2 px further a row turns into parallel diagonal streaks, wrapping twice across the 670 px.
- The tachometer's red hub ring and its blue inner ring become the ellipse beside the needle.
- The R8 oil mini (aligned) is intact.
- So photo B is **mode `r8`**: `off` sends 2680-byte RGBA rows, `etc2` sends 672-wide blocks, and neither can shear like this. This is a prediction the 11-19 log can check (§5).

### Ranked hypotheses for *where* the pitch goes wrong, and why only sometimes

| # | hypothesis | fits | does not fit / open | status |
|---|---|---|---|---|
| H1 | **The first whole write of an asset with a new scene is right; a write into a scene already up ("in place") is re-read at the 4-byte pitch** (GL's best fit, which I share) | photo A: same atlas size on two connections; the speed's atlas is written once, the gear's gets in-place letter writes (7–19 writes per session on 11-18), and only the gear is wrong. f30: R8 dials made by a rebuild while the theme is up (drive mode, night, a live switch) → wrong; the next new scene (a KOMBI re-map, our theme off and on) → right; "sometimes" | "only a reboot helps" must then come from the leak (§2), or from no re-map happening before the owner rebooted | hypothesis |
| H2 | **A renderer-wide unpack alignment that flips at some moment and stays until a reboot** | "only a reboot", "one time clean, another time drifts" | photo A needs a coincidence: the flip came after the speed atlas's last write but before the gear's | hypothesis |
| — | asset ids reused after a drop | — | the client never reuses a ref (`nextRef++`); a reused id would show another whole texture, not a 2 px a row shear | **killed (code + shape)** |
| — | a texture made under an old key with another size or format (R8 read as ETC2, RGBA as R8) | — | every bind names the new ref. A `v8` shader on an RGBA asset would give wrong colours, not a shear. Wrong-format reads would be block noise (ETC2) or a ×4 stretch (RGBA as R8), not a 2 px a row slant | **killed (code + shape)** |
| — | drop and make on different connections | — | each texture lives on one connection, one socket, in order | **killed (code)** |
| — | an upload refused at the pixel-memory peak, the layer left bound to the old texture | — | a refused asset leaves the new layer unbound: **not drawn** (proven, §2), never drawn from old bytes | **killed for the shear; proven as "does not draw"** |
| — | the shader variant and the texture set changing in different frames | — | both are made in the same rebuild on the same connection; a layer waits until its asset has pixels | **killed (code)** |
| E | `etc2` only: the KOMBI refuses `etc2rgb` | would give "does not draw" in `etc2` | no car session has run it | hypothesis; the log names it (`asset N: the renderer refused its WxH etc2rgb texture`) |

Why a reboot clears it:
- H1: a new process builds into a new scene, and every asset is then a first whole write.
- H2: the renderer's GL state is reset.
- The leak: a new process and a new connection.

## 4. Safe behaviour, delivered and proposed

| | status |
|---|---|
| The old set is dropped only after the layers that bind it, and before the new set is made (leak fix + regression test) | **done, commit 50b5643** |
| `test-diag`'s "nothing refused" check matches the engine's `refused <op> on` line | **done** |
| Every R8 asset's rows a multiple of 4 | GL builds it for the atlases (`_exactAtlas`, `_atlas`). For f30's `@r8` files it is decision D1 (they are only used by a non-default option, and padding at load time is not exact: §7) |
| The format chosen only at start (the setting says "after restart") | decision D3 |
| A refused upload → our theme off with one log line | decision D4 (changes the default path's behaviour) |
| `etc2` / `r8` locked until the car has passed §5 | decision D2 |

**What the owner should do today:**
- Keep Діагностика → **"Lighter scene textures" Off** (the shipped default) and do not switch it while driving.
- If any of our pictures looks wrong, reboot. That works, and with the setting Off f30 makes no texture of the kind that breaks.
- In Twin Dial, until GL's fix ships, the gear letter in the hub may show garbage after a gear change. BMW's own gear letter in its gap is never covered and stays right. Read the gear there.

## 5. The written car test (for the owner; the lead reads the logs)

Build: the lead's merge of this round (TX's leak fix + GL's switch). Parked. Engine idling or off with the ignition on. Before anything else, a **belt-lamp check**: with BMW's own cluster, unbuckle and confirm the belt lamp and the parking-brake lamp show. For every photo, note the **car clock time** and include the whole cluster.

1. **Reboot** the head unit. Photo 1: BMW's own cluster (our themes off).
2. **f30, cold start in r8.** In f30 set Діагностика → Lighter scene textures = **r8**, then reboot again, so the first build is r8 and no live switch happens. Choose 6WB Comfort. Photo 2.
   - Prediction: H1 and H2 both say right.
3. **A rebuild while our theme is up.** Drive-mode switch Comfort → Sport → Comfort (or Comfort + G20 on, then off). Photo 3, in Comfort.
   - H1: dials sheared, the oil and fuel minis intact.
   - H2: right, unless the flip has come.
4. **A new scene.** If photo 3 is sheared: set f30's theme to **Off** (BMW's own), wait **20 s**, set Comfort again. Photo 4.
   - **H1 says right again; H2 says still sheared.** This is the step that separates them.
5. **Live switches** (checks the leak fix): r8 → etc2 → off → r8, **15 s** each, one photo per step (photos 5a–5d).
   - With the fix every dial is drawn, whatever its shape.
   - In etc2: either right, or the dial layers missing with a renderer refusal in the log (E).
6. Set Lighter scene textures back to **Off**. Photo 6.
7. **Twin Dial** (map window never opened in this run). Brake held, standing: P → R → N → D, one photo of the hub per letter. Then do the same with GL's aligned-atlas option (GL's report has the option's name). Then open the map window once and repeat D.
8. If anything is still wrong at the end: first **theme off 20 s and on** (photo), then **reboot** (photo). This shows the smallest cure.

**What the lead reads afterwards** (app logs in `oap_*/data`, `cluster-render.log`, the journal; times against the photo times, car clock = UTC+2). These are the app's lines:
- `scene texture "speedo": 670x670 R8 asset (speedo@r8)`: when each R8 texture was made.
- `diag: diagSceneTex …`
- `refused …` (with the fix: none).
- `assets: dropped N that nothing uses any more` (new, from the fix): it must appear only at live switches.
- `layer N … error` / `waiting`.
- `cluster-render status: … asset N r8 670x670 of f30_cluster on panel: K writes, F failed`.
- Twin Dial's `gpu text atlas "td_big" …` and GL's new in-place-write lines.

These are cluster-render's lines:
- `client N (f30_cluster): asset N on panel (r8 670x670)`: when the asset arrived.
- `target panel: its scene goes whole to the renderer again: N KB`: each new scene.
- `INCIDENT target panel lost: slot locked` / `back after` (re-maps), `start: cluster-render` (service restarts), any `E` line or `renderer refused its … texture`.
- **H1 is proven if:** every sheared photo comes after an R8 asset was made or written while the scene was up, and the next `goes whole` before a right photo.
- **H2 is proven if:** a right → wrong change with no asset write between them, staying wrong through a `goes whole`.
- The journal: KOMBI or Dcsm2 resets in the same minutes.

**What the 2026-11-19 logs will settle when they arrive** (photo A ≈ 12:18 UTC, photo B ≈ 12:24 UTC):
1. Whether photo B was mode r8 (predicted: `(speedo@r8)` lines in f30's log before 12:24).
2. Whether a live switch came first. Before the fix it would show `refused drop … asset in use` and `too many assets`.
3. The order of f30's R8 asset writes and the last `goes whole` before 12:24 (H1 vs H2).
4. For Twin Dial: the gear atlas's writes, and whether a `goes whole` fell between the last letter write and 12:18.
5. Any cluster-render restart or KOMBI reset in between.

The 2026-11-18 collection shows what healthy looks like:
- no `refused` line in either app log;
- cluster-render logs no error for assets;
- `asset … r8 … : 1 writes, 0 failed` for every texture, and 7–19 writes on the gear atlas;
- 51 `goes whole` and 19 slot-lock incidents.

## 6. GL and TX

- **We agree, found independently:** one cause for both pictures (R8 rows not a multiple of 4 bytes, re-read at the padded pitch on the cluster's side). The default path reaches it in Twin Dial and not in f30.
- **GL adds the angle:** photo A's strokes lean 24–26°, which is 2 px a row on the **whole 1066 row** (26.6°), not a 41-px rect re-read alone (18.4°). GL also adds the in-place-write model, which I rank first (H1).
- **GL agrees the leak is a second, separate fault.**
- **Division of work:**
  - GL: `_exactAtlas` / `_atlas` aligned, `_exactPut` writing whole 4-px columns, Twin Dial's diagnostic switch, log lines for R8 alignment and in-place writes.
  - TX: `Diag.changed()` and `_dropOrphans`.
  - Neither of us touched the other's functions.
- **We differ only in emphasis:** GL says "only a reboot" fits the leak better than the shear. I agree under H1; under H2 the shear alone would need a reboot. Step 4 of §5 decides.

## 7. Decisions for the owner (nothing here is built in)

- **D1. f30's `r8` option.** Its dial and segment files are 670 / 346 wide, the shape that broke.
  - If `r8` stays, re-bake `speedo@r8`, `tacho@r8`, `speedo_d@r8` and `tacho_d@r8` 672 wide, with their colour maps computed on the 672 grid (`src: [670, 670]`), and `range_seg@r8` / `ed_seg@r8` 348 wide. Also add the padding factor to `u_decoR`, as `u_baseR` already has it.
  - Padding at load time without a re-bake is **not** exact: the tachometer's colour map shifts by up to 74/255 on 617 colour values at the red zone (measured).
  - Otherwise remove `r8`.
- **D2. Lock `r8` and `etc2`** (hidden, or a second confirm) until §5 has passed.
  - `etc2rgb` has never been drawn on this car. The car runs cluster-render 0.6.9 (`versions-ls`), which takes ETC2 by protocol, but no session has proved the KOMBI draws it.
  - `etc2` also sends the 346-wide R8 segments.
- **D3. The texture format applied only at start** (the setting labelled "after restart", as `ccDisplayed` is). With the leak fixed a live switch is clean on the bench. Under H1, though, every R8 texture made while our theme is up is at risk, and a start always builds into a new scene.
- **D4. A refused upload → our theme off, with one log line,** instead of a half-built theme. This changes f30's default behaviour when the service is short of assets or memory (for example, two apps on at once). The sketch: in the `refused` handler, on op `asset` or `bind` during a build, set `this.problem` and build `off`.
- **D5. A guard for the future:** the engine refuses (or pads) any R8 asset whose rows are not a multiple of 4, plus a test over every app's textures and atlases. The lead decides whether this lands with GL's atlas change.

## 8. Tests (each run alone, worktree `ci-tx`)

| suite | port | result |
|---|---|---|
| `bench/test-scene-tex.js` | 8933 | **72 passed, 0 failed** (54 before + 18 new in §4 "the setting switched live"); **without the fix: 58 passed, 14 failed** (first: `refused drop on 2: asset in use`) |
| `bench/test-diag.js` | 8934 | 104 passed, 0 failed |
| `bench/test-twin-budget.js` | 8935 | 95 passed, 0 failed |
| `bench/test-twin-cc.js` | 8935 | 52 + 266 passed, 0 failed |
| `bench/test-twin-tabs.js` | 8935 | 252 passed, 0 failed |

Not run: the full runner (the lead's, at merge).

Files: the fix is in `shared/diag_settings.js` and `shared/hud.js` (`_dropOrphans`, one call in `_rebuildInner`). The tests are in `bench/test-scene-tex.js` and `bench/test-diag.js`. The scratch tools `repro_switch.js` and `shear_sim.js` stayed in the session's scratchpad; the picture is `bake/work/car-issues/pub/TX/shear-sim.png`.
