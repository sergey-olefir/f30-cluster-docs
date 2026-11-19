# Brief: a gallery of every state the theme can be in

Audience: one agent that builds a small tool and runs it. The parallelism is in the tool (several benches and a render pool at once), not in the number of agents: this is mechanical work.

## 0. What the owner asked for (2026-11-16)

"I want to look at all the signals we support: messages, navigation, speed limits and so on, in as many different combinations as possible. Just make the screenshots and put them in a folder, so that I do not have to play with the settings. It must be as parallel as possible."

So: a folder of pictures the owner can leaf through, one per combination, each labelled with what was fed and what the engine made of it. No fiddling, no explanations needed to read it.

## 1. How the pictures are made: the replay, not a browser

Do **not** use Chrome or any browser automation. Headless Chrome with software GL hung on this machine today and pinned the CPU, and agents cannot `kill`.

Use package O's replay instead (`tools/strip/O/`: `benchio.py`, `snap.py`, `states.py`, `render.py`). It snapshots the scene exactly as the bench streams it (every layer, its compiled shader, live uniforms, sampler bindings, the raw pixels from `GET /px`) and runs the app's **real fragment shaders** on the Mac's GPU through moderngl, stacking the layers as `bench/page.html` does. It was checked against the owner's own screenshot of the bench: median error 0.000. A replayed frame at scale 1 **is** the panel's pixels, 1920 × 720.

Each picture is our layers composited over a backdrop:

- **`car`** (the main picture): a real photo of the stock cluster, rectified to panel px by package R (`bake/work/overlay/rect/61.png … 68.png`), chosen to match the state: driving with a limit and a road label → 61; driving with the empty sign → 68; P → 63 (or 64, 67); a one-line BMW message → 66; a two-line one → 62; standby → 65. This is the closest thing to the car we have.
- **`matte`** (see-through layouts only): flat green, so every pixel we leave to BMW is obvious.

BMW's side of a `car` picture is whatever that photo shows; it does not follow the fed signals. Therefore in the "all BMW's" row mode only the states that have a matching photo are meaningful, and the manifest must say for every picture whether its backdrop matches the fed state (`backdrop_matches: true | false`).

## 2. The matrix

### 2.1 Contexts (layout × row mode)

| Id | Settings |
|---|---|
| `comfort` | Theme 6WB Comfort, `g20` off |
| `comfort-g20-bmw` | 6WB Comfort, `g20` on, `bottomRow = allG20` |
| `comfort-g20-ours` | 6WB Comfort, `g20` on, `bottomRow = all6WB` |
| `sport` | 6WB Sport |
| `eco` | 6WB Eco Pro |

Set `byMode` off while a context is held, or the drive mode will switch the theme under you. "6WB Sport + G20" cannot be loaded through any setting today; leave it out and say so in the report.

### 2.2 Axes

Signals (fed through `POST /fake` scenes from `bench/fake-signs.js`, manual values through `POST /signals`, the drive mode through `POST /drive`):

- **Speed limit** (11): `limit-50`, `limit-80-rain`, `limit-130-warn1`, `limit-30-warn2-school`, `limit-55-mph`, `limit-end`, `limit-unknown`, `limit-none`, `limit-invalid`, `nopass-car`, `nopass-truck`.
- **Message** (15): none (`cc-clear`), `cc-red`, `cc-yellow`, `cc-green`, `cc-orange`, `cc-white`, `cc-long`, `cc-long-name`, `cc-three`, `cc-frame-2`, `cc-uk`, `cc-cyrillic`, `cc-greek` (falls back to BMW's row by design), and the two never-read states `cc-unread`, `cc-unread-root` (only right after a restart).
- **Road / navigation** (5): `road-none`, `road-street`, `road-shield`, `road-guiding`, `road-cyrillic`.
- **Clock** (2): `clock-plus2` (ours), `clock-none` (the row waits and is BMW's).
- **Motion** (8), manual values as in `tools/strip/O/states.py` (`DRIVE`, `PARK`, `P63`, `STANDBY`): P idling; D at 30; D at 50; D at 72 inside a 50 limit (over the limit); D at 130; high rpm near the redline; P with the engine off (`condition` 7); standby (`condition` 5, held 1.5 s).
- **Drive mode** (3) via `/drive`: Comfort 4, Sport 1, Eco Pro 7. **Gearbox program** (3): D, S, M (`program`).
- **Auto start/stop** (5): `msa` 0–4.
- **Values at their extremes**: ambient −12.5 and +38.0; coolant cold (the cold-engine rev guard) and hot; range 12, 368 and 9999; fuel nearly empty.

Settings (through `POST /setting`): `ccStyle` A / B (with a message showing), `limitArc` auto / on / off (while over the limit), `modeLabel`, `gearTray` gear / letter / empty, `wbFoot` hair / chrome / none / glass, `dialShade` off / weak / normal / strong, `stockMap`, night (`nightDim` always with `nightLook` plain / amber / ice / red, and off), `baked`, `badge` none and two samples, `badgeBMW`.

Read `apps/f30_cluster/app.json` and `bench/run-app.js` for the authoritative lists; if an axis has a value this brief missed, add it.

### 2.3 Which combinations

The full product is astronomically large and mostly noise. Build four sets:

- **A. One-factor sweeps.** For every context, start from one baseline (D at 50, `limit-50`, `road-street`, `clock-plus2`, no message, Comfort, day, default settings) and vary each axis alone through all its values. About 70 pictures per context.
- **B. Pairwise interactions**, for the two see-through contexts, over the signal axes (limit × message × road × motion × clock): a covering set in which every pair of values of two different axes occurs at least once. For `comfort-g20-bmw` keep only motion/message states that have a matching photo (§1).
- **C. Scenarios.** About twenty hand-picked, realistic frames per main context: town at 50 with a street name; motorway at 130 with a no-overtaking sign; a school zone with the second warning level; rain supplement; mph; a long Cyrillic road label together with a long message; three messages cycling; a message while over the limit; start-up; P with the engine off; standby; standby with a message; night amber with a message; cold engine near the redline; Sport mode in the S program; Eco Pro.
- **D. Diagnostics**: the calibration grid, the map calibration and the three mattes, one picture each.

**Deduplicate by image hash.** A picture identical to its context's baseline is not saved twice; it is listed in the index as "no visible effect" under that axis value, which is itself worth knowing (for instance that the bottom-row signals change nothing in an opaque theme).

Expect roughly 700 distinct pictures; report the real count.

## 3. Running it in parallel

- **Six benches at once**, ports **8181–8186**, each with its own `BENCH_DATA_DIR`, each started in the bounded form (a Python wrapper that ends it after an hour; see `docs/AGENT-BRIEF-strip.md` §3 "Bounded processes"). Start them one after another, not all at once, so the first build of the app is not raced. Do not touch 8080 or 8150 (the owner's).
- **Split the work by context**, so a bench rarely switches layout; the two see-through contexts are the largest and get two benches each.
- **Snapshot and render are separate stages.** A worker per bench drives the state and saves the snapshot; a pool of render processes (moderngl standalone contexts) turns snapshots into pictures. Neither waits for the other.
- **`_rowFail` is a sticky latch.** `POST /restart` at the start of each group, order each group so that the latching states (`cc-greek`, a throw) come last, and restart after each of them. The never-read states are only reachable right after a restart.
- **Never save a picture of the wrong state.** Before each snapshot, poll `GET /probe` until it reports what was asked (as `states.py` does with its predicates); on a timeout, record the combination as failed with the probe's last answer instead of saving it.
- Everything ends by itself; nothing is left running. Report any port still listening.

## 4. What lands in the folder

`<repo>/bake/work/shots/<YYYYMMDD-HHMM>/` (git-ignored; the backdrops are photo pixels, so nothing here is ever committed):

```
index.html              a contact sheet: thumbnails grouped by set and context, a text filter, click for full size,
                        a switch between the car and matte backdrop
manifest.json           one entry per picture: context, every setting and signal fed, the probe's own/fail/standby
                        answers, the backdrop and whether it matches, the image hash, duplicates folded into it
A-sweeps/<context>/<axis>/<value>.png  (+ .matte.png)
B-pairs/<context>/<limit>__<message>__<road>__<motion>__<clock>.png
C-scenarios/<context>/<nn>-<name>.png
D-diagnostics/<name>.png
failed.json             combinations that never reached their state, with the reason
```

Every thumbnail in `index.html` carries a one-line caption from the manifest: the row's owner ("row: ours" / "row: BMW's — waiting for a time zone" / "row: BMW's — missing glyph") and what was fed. The page must work opened straight from disk (`file://`), with no server and no external resources.

## 5. Rules

- Base: `git checkout -B shots integrate/strip` (tip `541832a` or later). Symlink `reference` and `bake/work` from `<repo>`; `git status --porcelain` must not list them.
- The tool lives in `tools/shots/` (tracked, committed on your branch). It must be re-runnable with one command, and take `--only <set|context|axis>` so a part can be redone.
- Python through the repo's convention: `UV_CACHE_DIR=<repo>/bake/work/.uvcache uv run -q --no-project --with numpy --with pillow --with moderngl python -I …`.
- No Chrome, no push, no version bump, no car. Change nothing in `shared/` or `apps/`; if a state cannot be reached without an app change, list it instead.
- Only the bench's own invented texts are fed. Nothing read off a photo goes into a file name, the manifest or the report.

## 6. Proof and report

- Planned, produced, deduplicated and failed counts, per set and context.
- No error or refusal line in any bench log during the run (layer cap, pixel memory, uncaught exception).
- One sanity check of the replay on this build: render the baseline of `comfort-g20-ours` and compare it with the bench page's own drawing rule the way package O did; say how.
- The wall-clock time of the run and the one command that repeats it.

Your final message is all the lead sees: the folder's path first, then the counts, anything that failed and why, the axis values that turned out to have no visible effect, and anything in the pictures that looks like a defect (a clipped text, an overlap, a wrong colour), with the file name of each.
