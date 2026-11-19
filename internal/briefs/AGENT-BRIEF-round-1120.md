# Brief: the owner's round of 2026-11-19 evening — six packages

Audience: six agents in parallel, **F1, T1, T2, AC, AA, DR**. Base: **`main`** at `35dddc9` or later (Twin Dial 0.1.5, f30_cluster 0.7.22, Signal Probe 0.4.0, the cruise simulator in `tools/cruise-sim/`; every suite green). The lead merges, bumps versions and pushes; you never push, never bump a version, never connect to the car.

## 0. What the owner reported (2026-11-19, evening drive)

1. **f30_cluster, see-through G20 layout with our bottom row:** "the text overlaps again". Photo D (`bake/work/car-issues2/in/photo-D-f30-g20-row.jpg`, car clock 19:28, night): the range's pump pictogram and the "km" after the range figure are **doubled**, two copies a few px apart: ours over BMW's own.
2. **f30_cluster, the road label:** a national road number (letter + two figures) is drawn as plain text before the street name, **without its shield**. BMW draws it on a coloured plate. `docs/ROAD-SHIELDS.md` (and `design/shields/preview.html`) already specify the Ukrainian shields; nothing was wired.
3. **Twin Dial: "the figures drifted at a random moment"** — after 0.1.4's aligned atlas? Which version was on the car is being asked (the lead will tell you). Photo A (`bake/work/car-issues2/in/photo-A-twin-map-gear.jpg`) is the earlier case.
4. **Twin Dial, messages:** tab 2 often shows our fixed words **"Повідомлення / на панелі внизу"** (the pointer page) instead of the message itself. Only "door open" was ever shown as our card; **switching DSC off** showed BMW's own panel and our pointer. The owner: *"why point at the bottom if we can show it in the centre?"*
5. **Every metric an adaptive cruise needs must be captured** on the next drives.
6. **The docs and the article go to a separate repository** of the owner's GitHub account (`docs/` left the main repository on 2026-11-19 because the Store refuses an archive over 64 MiB).
7. *(added during the round)* **A separate app for the adaptive cruise:** for now it only **reads and draws**; first **a park of mockups**.

Photo D shows a street name: never copy it into anything, never write it; describe by numbers only. `bake/work/` and `reference/` are git-ignored and read-only for you.

## 1. Rules (as every round)

- Worktree `<repo>/.claude/worktrees/r1120-<id>` on a new branch `r1120/<id>` from `main`; symlink `reference`, `bake/work` and `docs` from the main checkout. Never `git add` under `docs/`, `reference/`, `bake/work/`. No big tracked files (the Store limit is on the whole repository).
- Car: read-only signals only — `r` only with a NON-ZERO getter id, `s`, events `_broadcasts`; the one approved method is `GuidanceEventList.queryContent` (already in use). Never a setter or any other method; never driver-assistance settings. `tools/signals/check_signals.py` passes for every app you touch.
- **Safety rule (hard):** a BMW warning is never hidden unless our own text for that exact message is on screen in the same frame. BMW's lamps, the belt group, the gear gap [1380, 660, 1460, 720] are never covered.
- Platform limits: 8 layers per connection, 32 in all, 16 MiB pixel memory; the KOMBI resets on too-big effect shaders; `mediump` is fp16 (`highp` for pixel distances over 256); every sampler bound; **every R8 texture 4-aligned in width**; Node 8.16 syntax in apps.
- Privacy: no VIN, keys, tokens, Digital Key or BMW ID material, coordinates, street names, media titles, contact names, numbers, the real mileage; texts logged as lengths only; no home-directory or temp paths in anything tracked or published.
- No other maker's name, logo, font or bitmap in anything committed or published.
- Ports: F1 8900–8909, T1 8910–8919, T2 8920–8929, AC 8930–8939, AA 8940–8949 (8080 / 8150 are the owner's). Suites one at a time; do not run `tools/twin-dial/run_all_tests.sh` (the lead does). No `kill`, no browser automation. Shared files: marked hunks `// r1120 <ID>:`. T1 and T2 both touch Twin Dial: keep to your own functions.
- Hand-in: `docs/r1120/<ID>-REPORT.md` (through the symlink, not committed), `READY` in `bake/work/r1120/pub/<ID>/`, final message: branch, last commit, test totals, decisions for the owner. Findings the car has not confirmed are hypotheses and are worded so.

## 2. Packages

### F1 — f30_cluster: the doubled range, and the road shields

- **The doubling.** In the see-through G20 layouts with the bottom row "all6WB" we draw the range, the clock, the temperature, the road label and the sign over BMW's own row. Photo D shows BMW's range pictogram and "km" still visible under ours with an offset. Find why (the strip mask / black under our row not covering BMW's range group at night or at this range width; a changed BMW position; our text placed a few px off; the row falling back to BMW's while ours is still drawn — `_rowFail`, standby, the time-zone wait). Measure on the photo against `docs/G20-DESIGN-SPEC.md` and the strip mask (`design/strip/`, `apps/f30_cluster/hud/scene/strip_mask.json`). Fix what is proven, with a test that fails without it; check every other element of the row for the same fault (clock, temperature, road, sign).
- **The shields.** Wire `docs/ROAD-SHIELDS.md` into the road label: parse the road number at the start of the label text (М / Н / Р / Т / E and their Latin look-alikes), draw it on its plate in the specified colours, the street after it. Drawn by us (geometry, our font), no image from anywhere. Unknown prefixes stay plain text. Test with invented road names. Mind the row's width (a long name is clipped as today) and night colours.
- f30 changes its shipped behaviour here: say exactly what, for the owner.

### T1 — Twin Dial: show the message in the centre instead of pointing at the bottom

- Read `shared/twin_cc.js` (`layoutTab2`, the keep-open conditions of `docs/twin-dial/SPEC-R3.md` §5.4 / SPEC rev 2 §8.4, D7, `POINTER`) and find **every reason the pointer page is chosen** instead of the card. For the DSC-off case list which reason fires most likely (no ccList entry for the shown number? an entry with no name? a glyph the font lacks? D7 not yet confirmed? the "shown now" number not matching?). Use the probe logs in `reference/car/2026-11-18-collect1/` for what ccList and displayed carry on this car.
- **The change the owner asks for:** whenever we can read the message (its name in the list), **tab 2 shows our card in the centre**, even while BMW's panel stays open at the bottom because the rule does not yet allow covering it. The card in the centre does not cover BMW's panel, so the safety rule is untouched: the bottom panel's open / covered logic must stay exactly as it is (prove it with the per-frame audit of `bench/test-twin-cc.js`). The pointer page stays only when there is really nothing of ours to show (no readable text), and then it says why in one short line where that helps.
- Log one line per message period with the reason the panel was kept open (numbers and enums only).
- Fonts: every character of every message name must be in `td_msg` (Cyrillic); report any gap you find in BMW's catalogue texts if available.

### T2 — Twin Dial: the figures drift at random

- GL's fix (0.1.4: `_exactAtlas` 4-aligned, `diagGpuText`) is on `main`; read `docs/car-issues/GL-REPORT.md` and `TX-REPORT.md`. If the owner confirms the drift on 0.1.4 or later, the 4-byte hypothesis is not the whole story. List every R8 / GPU-text asset Twin Dial makes **after** the first scene (in-place writes, atlases made late, sheets re-made) and every one whose width is not a multiple of 4 or whose in-place write rectangle is not 4-aligned in x and width (`sheet` image layers are RGBA: check their dirty-rect writes too), on every path: tab changes, the map window, messages, night, CarPlay turns, the front-vehicle test mode.
- Add a **guard** in the engine: no R8 asset or R8 rectangle write that is not 4-aligned can be made (assert in the bench, log and pad on the car), with a test.
- Make the diagnosis possible on the car: the log line per in-place write already exists (GL) — add what the next drive needs to tell which text drifted and when (numbers only), and a short car protocol.

### AC — capture every adaptive-cruise metric on the next drives

- From `docs/cruise-set-speed-brief.md`, `docs/cruise/RC-REPORT.md` (the recorder, the S14 additions), `docs/cruise-sim/SC3-REPORT.md` (the ranked list of what to measure) and `docs/AGENT-BRIEF-front-vehicle.md`: make **one recording set "ACC"** in Signal Probe that holds everything the parent brief's Phase 1 table, the simulator's ranked list and the front-vehicle display need (front vehicle, KAFAS object lists as far as the budget allows, cruise state and set speed, own speed / acceleration / yaw / grade, pedals and brake, wheel speeds, the collision-warning request, drive mode, gear). Budget it against the recorder's quota; propose the quota (`maxDiskUsage`) the drive needs and say what happens on the car if it is raised.
- **The lead's collection:** extend `tools/car/collect.sh` (read-only commands only, the same refusal filter) to copy `data/rec/` of Signal Probe, with sizes; dry-run tested.
- **The analysis after the drive:** one command that turns the recordings into the Phase 1 signal-quality report (native rate, jitter, gaps, units against the anchors, latency estimates), the Phase 2 step-response fits from the driver's set-speed steps (dead time, time constant, max acceleration and deceleration — the simulator's plant JSON as output, `assumed: false`), and a short summary for the owner. Test it on synthetic recordings made with the simulator (`tools/cruise-sim`).
- The drive protocol (`docs/cruise/DRIVE-PROTOCOL.md`) updated with the ACC set: what to drive, in which order, safely, by hand.

### AA — a separate app for the adaptive cruise: read and draw; a park of mockups first

The owner wants a **separate app** (not a Twin Dial page) that, for now, **only reads and draws**: the vehicle ahead, the gap, closing speed, the cruise state and set speed, the collision-warning state as information, later the outer loop's suggested set speed. **Nothing is written to the car.**

- **Where it can draw** — decide with the owner by mockups, because only one app may own the cluster's `panel` target (f30_cluster or Twin Dial draw it): the cluster's **`map` target** (the 1088 × 480 central map window, the slot Turn View uses; it can show while either theme runs?), the **HUD** (800 × 480), and the **centre screen** (RHMI + canvas). Read the `mgu-app-dev` skill's references (`cluster-and-hud.md`: targets, slot arbitration, the layer cap) and say what each target allows and costs and what has been seen on the car.
- **A park of mockups**: at least three visual directions × the placements above × every state (no target, far, middle, near, closing, closing fast, cruise off / standby / active / limiter, data stale, BMW's own collision warning active, the loop's suggested set speed shown), day and night. Own style (it must sit next to both f30_cluster's 6WB look and Twin Dial's), our own generic car shape, no maker's cues. Use `reference/vc/13_*` and `14_*` for the idea only. PNG files in `bake/work/r1120/pub/AA/` with a contact sheet per direction and one per placement, and an `index.html` that shows them all with the state names.
- **A skeleton app** `apps/acc_view` (manifest, `main.js`, read-only signal declarations reusing `shared/front_vehicle.js` and `shared/limit_src.js` where they fit, the bench running it on the `map` target with injected values), **off by default, not in the Store manifest until the owner says**; no new shared behaviour for the other apps.
- The spec `docs/r1120/AA-SPEC.md`: the placements, the directions, the states, the data each picture needs and whether it is seen, declared or unknown on this car, and the decisions for the owner.

### DR — the docs and the article as their own repository

- Stage `docs/` (the lead's local copy, 73 MB) as a repository in `bake/work/r1120/docs-repo/` (a plain folder with its own `git init`, **no remote, no push** — the lead creates the GitHub repository and pushes after reviewing your scan). Proposed name: **`f30-cluster-docs`** (the lead's choice; say if another is better).
- **Privacy scan, then scrub, before anything is staged:** every file, text and pictures' metadata (EXIF: strip it from every image; location in photos). Find and remove or replace: home-directory and temp paths (`<home>`, `/private/tmp/...`, scratchpad paths, the account name in them), VIN-like strings, keys, tokens, Digital Key / BMW ID material, coordinates, street and place names from the owner's drives, media titles, contact names, phone numbers, the car's real mileage and odometer figures from the car (fixtures and invented numbers are fine; say which you kept), email addresses. **Leave out entirely** anything about the Digital Key or the owner's driver profiles (`DIGITAL-KEY-PROFILE-REPORT.md`, `OFFLINE-USER-REPORT.md`, and any other you find) unless the scan proves it holds none of the above — list what you left out.
- Agent briefs (`AGENT-BRIEF-*.md`) are internal working notes: put them under `internal/briefs/` and scrub them the same way, or leave them out — recommend which.
- Structure: a README (what the project is, a link to the code repository `github.com/sergey-olefir/f30-cluster`, an index of the article, specs and reports), the article (`article/`, Ukrainian and English) prominent, licences stated (the article's images are the owner's photos and our renders). Relative links inside docs that pointed into the code repository become absolute links to it.
- Size: report the total; keep any single file under GitHub's 100 MB limit (none should come close).
- Hand-in: the staged folder with one commit, `docs/r1120/DR-REPORT.md` with the scan results (counts by kind, every file changed or left out, nothing private quoted in the report itself), and the exact `git remote add` / push commands for the lead.
