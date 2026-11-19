# SC3 report — cruise simulator: scenarios, metrics, batch runs, report

2026-11-19 · agent SC3 · branch `cruise-sim/sc3` (worktree `.claude/worktrees/cs-sc3`) · last commit: 5b60940

**Everything below was computed with ASSUMED plants** (SC1's presets liftoff / mild / firm and the grids around them). No drive has been recorded. Nothing here is a go / no-go verdict; it says which car numbers the verdict will turn on.

## What was built

All under `tools/cruise-sim/` (my files only; nothing under `apps/` or `shared/`):

| File | What |
| --- | --- |
| `core/scenario.js` | Loader, validator, sweep expander (CONTRACT + additive keys: `params` with `"=expr"` arithmetic, `"{expr}"` in strings, `$repeat` in arrays, `extends` with `vehicles+` / `events+`, `variants`, `where`, `worstOver`, `reactions`, scenario-level `config`). `load(id)` returns a **runnable** concrete scenario (a file id → its default variant; a concrete id such as `D.brake#v=90,dv=20,a=-1/none` → that variant), `template(id)` the raw file, `expand`, `validate`, `list`, `toConfig(concrete, base)`, `bundleJs()` (`node core/scenario.js --bundle` writes the browser bundle `self.CS.scenarioBundle`). One table (`PATCHES`) maps scenario conditions to SC1/SC2 config keys. |
| `core/metrics.js` | 45 run metrics + 5 catalogue metrics, `SCHEMA` for the thresholds (the parent's proposals as defaults) and for 9 metric definitions, `compute(result, config)` (SC1's `sim.js` calls it), `verdict`, `worst`, `timeLost`. |
| `core/mixed.js` | Family L: event rates per road type (motorway / rural / town) × density, Poisson draws over the catalogue's variants, filler following / free time, `aggregate` → per-hour numbers. |
| `sweep.js` | Batch runner over a plan JSON, parallel child processes (all cores − 1), ideal-ACC baseline per variant, tornado points, writes `runs.jsonl` + `batch.json` (SU2's `cs-batch` shape) + `summary` (per point and family pass / safety shares, maps, ranking, tornado, per-hour, worst cases). |
| `report.js` | One self-contained HTML file (inline CSS + SVG, no script, nothing from the network): summary in English and Ukrainian, measure-first list, maps, tornado, family table, scenario matrix, metrics vs thresholds, the three reaction cases, per-hour table, tuning table, worst cases re-run and charted, the whole catalogue, configuration. |
| `scenarios/<A..L>/*.json` | 154 scenario files. |
| `scenarios/plans/*.json` | smoke, presets, map-authority-deadtime, map-sensor-command, tornado-mild, tuning. |
| `fixtures/traces/` | 4 synthetic recorded-like lead traces (3 JSON, 1 CSV; 15–17 KB each) and their deterministic generator `make.js`. |
| `tests/scenario,metrics,mixed,sweep.test.js` | 52 tests (SC1's harness). |

How to run: `node tools/cruise-sim/sweep.js tools/cruise-sim/scenarios/plans/presets.json` then `node tools/cruise-sim/report.js` (takes the latest run of every plan). Tests: `node tools/cruise-sim/test.js`.

## Definitions worth knowing

- **Critical moment**: true gap < 2 m, true TTC < 1 s, true time gap < 0.3 s, or contact. **Last moment**: the latest time a driver braking at 6 m/s² (no reaction time) still keeps 2 m to that lead, the lead doing what it really did. **Warning lead time** = last moment − warning onset (threshold ≥ 1.5 s). **Missed warning**: a critical moment with no warning in the 20 s before its last moment. **Nuisance**: on a run where the driver does not react, a collision warning not followed by a critical moment within 10 s (sensor-fault warnings never count).
- **Three driver-reaction cases**: every warn-only and out-of-envelope variant runs with SC1's `driver.reactionCase` `nominal` / `late` / `none`. The warning's timing and need are judged on `none` (identical to the others up to the reaction); contact is judged on `react`; `late` is reported, not decided.
- **Groups**: a variant group = one variant with all its reaction cases and all its worst-time injections (`worstOver`); a group passes only if every member passes. **Safety share**: groups whose failures are only comfort / chatter / nuisance / efficiency count as safe (the list is `sweep.SAFETY`).
- **Oscillation gain**: our speed's amplitude over the lead's, at the lead's sine period, after 2 periods. **Time lost**: distance behind the ideal-ACC baseline (same scenario, `ego.mode: idealAcc`) over our end speed.

## Adopted from the other agents (and what differs from CONTRACT's first draft)

- SC1: `driver.reactionCase`, fault kinds (`gapFreeze`, `ghost`, `commandLost`, `dropBurst`, `latencySpike { extra_s }`, …) with `duration`; `driverBrake { level_ms2 }`; sine `{ t0, phase_deg }`; IDM `*_ms2` keys, leaders by lane; `const` = vehicle `v0_kmh`; traces `{ t, v }` / CSV `t,v`; x0 = bumper gap; our own lane change (`laneChange { vehicle: 'ego' }`, added by SC1 on request).
- SC2: sensor presets `rain` / `lowSun` / `night`, `rateHz`, `range_m`; where no fault kind exists, the scenario uses the sensor's own settings through its `config` block: outliers (`outlierProb`), the next-lane car taken for the lead (a curve + `curvePredErr`), id flicker (`forget_s` 0 + `dropProb`). The ghost's distance is the sensor's choice (my first draft swept it).
- SU2: `batch.json` is their `cs-batch` shape; above 20 000 runs it keeps the runs at the preset points (all of them are in `runs.jsonl`).
- SU1: browser bundle via `node tools/cruise-sim/core/scenario.js --bundle > <path>`; I did not write under `ui/`.

## Real batches

I merged `cruise-sim/sc1` (14fd6d2) and `cruise-sim/sc2` (9807bd5) into my branch **locally** to run them (no stubs: `meta.modules` shows every real module). Together all tests pass.

| Plan | Runs | Points | Wall | Folder (bake/work/cruise-sim/runs/) |
| --- | --- | --- | --- | --- |
| presets (whole catalogue, A–L) | 39 222 (+4 358 ideal-ACC baselines) | 9 | 48 s | presets-20261119-195152 |
| map-authority-deadtime | 240 240 | 80 | 281 s | map-authority-deadtime-20261119-165649 |
| map-sensor-command | 109 600 | 40 | 40 min | map-sensor-command-20261119-* |
| tornado-mild | 71 240 | 26 | 35 min | tornado-mild-20261119-181216 |
| tuning | 101 376 | 144 | 63 min | tuning-20261119-191540 |

0 errors in any run. Report: `bake/work/cruise-sim/reports/report-20261119-175239.html` (also `report-latest.html`). Times vary with the machine's load (other agents ran their tests in parallel).

### 1. The first feasibility map (deceleration authority × dead time), with an assumed plant

Safety share of the catalogue (variant groups whose failures are at most comfort / chatter / nuisance), 12 variant groups per scenario file, family L excluded, mild preset otherwise:

With SC2's controller at the parent brief's start values (downward rate limit **3 km/h/s**):

| dead time \ cruise decel | −3.5 | −2.8 | −2.3 | −1.8 | −1.4 | −1.0 | −0.7 | −0.5 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 0.2 s | 74 | 74 | 74 | 74 | 74 | 74 | 70 | 70 |
| 0.5 s | 73 | 73 | 73 | 73 | 73 | 73 | 71 | 70 |
| 1.0 s | 72 | 72 | 72 | 72 | 72 | 72 | 71 | 70 |
| 1.5 s | 71 | 71 | 71 | 71 | 71 | 71 | 70 | 69 |
| 2.5 s | 69 | 69 | 69 | 69 | 69 | 69 | 68 | 69 |

With the downward rate limit opened to **12 km/h/s** (the cruise, not our limit, bounds the braking):

| dead time \ cruise decel | −3.5 | −2.8 | −2.3 | −1.8 | −1.4 | −1.0 | −0.7 | −0.5 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 0.2 s | 80 | 80 | 81 | 80 | 80 | 78 | 73 | 72 |
| 0.5 s | 80 | 80 | 80 | 80 | 79 | 77 | 72 | 71 |
| 1.0 s | 79 | 79 | 79 | 79 | 78 | 77 | 71 | 71 |
| 1.5 s | 78 | 78 | 79 | 78 | 77 | 76 | 72 | 71 |
| 2.5 s | 75 | 75 | 76 | 76 | 76 | 74 | 71 | 70 |

Approaches alone (family B), safety share, 12 km/h/s: 79–80 % at ≥ 1.8 m/s² and dead time ≤ 0.5 s; 67 % at −1.0; 36–43 % at lift-off (−0.5 / −0.7); at 2.5 s dead time 62–66 %.

**Headline (with an assumed plant):** the share that stays safe rises from ~71 % (lift-off only) to ~80 % once the cruise decelerates at **about 1.4 m/s² or more with a dead time of 0.5 s or less** (1.8 m/s² at 1 s, 2.3 m/s² at 1.5 s; at 2.5 s no deceleration reaches that band); beyond ~1.8 m/s² more authority buys little, and **with the parent brief's rate limit (3 km/h/s, range 1–5) the cruise's authority never matters** — the command ramp caps the deceleration at 0.83 m/s² (1.39 at 5 km/h/s), below the parent's own 1.5 m/s² requirement. The remaining ~20 % are scenarios no cruise can do (standing obstacles, hard braking, cut-ins at 5–10 m, failures at the worst time): they need the warning, and are scored on it.

### 2. The second map (sensor rate × command delay, mild, 12 km/h/s), with an assumed plant

Safety share: 78–80 % at command delay ≤ 0.2 s whatever the rate; a 1 s delay costs ~2 points, 2 s ~6 points; 2 Hz against 25 Hz costs ~2 points (cut-ins: 64–69 % against 71–74 %); sensor latency 0.5 s against 0.15 s costs another 2–3 points everywhere.

### 3. Tornado (±30 % around mild, 12 km/h/s)

Base: pass 62.6 %, safe 78.6 %. Change of the safety share, −30 % / +30 %: camera range −3.7 / +1.6; cruise deceleration −1.8 / +0.7; gap noise at 100 m −1.0 / +0.1; native rate −0.6 / +1.0; acquisition delay +0.9 / −0.4; dead time +0.7 / −0.3; max acceleration 0 / −0.6; latency +0.6 / −0.1; time constant +0.1 / −0.3; command delay and jitter ≤ 0.3; brake authority 0 (−1.8 m/s² binds first). ±30 % around mild sits above the map's knee, which is why the map (not the tornado) shows the deceleration as the first number.

### 4. Tuning sweep (mild preset; h × τ × rate down × TTC_warn; 704 variants × 144 sets)

Safety share by value (averaged over the others): rate down 1 / 3 / 5 / 12 km/h/s → 67 / 72 / 76 / 80 %; h 1.5 / 2 / 2.5 / 3 s → 70 / 74 / 75 / 76 %; τ 2 / 4 / 8 s → 72 / 74 / 75 %; TTC_warn 2 / 3 / 4 s → no change of either share (it moves only the warning's timing). Best sets: h 2–3 s, τ 8 s, 12 km/h/s → 82 % safe, 61–63 % pass, acceleration RMS 0.26–0.31 m/s², still 32–36 setpoint changes per minute. The parent's start set (h 2, τ 4, 3 km/h/s, TTC 3) ranks 86th of 144 (72 % safe).

### 5. Catalogue at the presets (whole catalogue, 1 140 variant groups + family L)

Family cells: pass % / safe %.

| Point | pass | safe | A | B | C | D | E | F | G | H | I | J | K | lead braking handled, no warning |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| liftoff | 55 % | 73 % | 84/100 | 40/40 | 8/88 | 81/81 | 66/67 | 55/66 | 53/74 | 57/64 | 81/87 | 43/43 | 31/38 | 0.5 m/s² |
| liftoff, command=worst | 53 % | 71 % | 84/100 | 34/34 | 7/86 | 77/77 | 64/66 | 54/65 | 53/74 | 55/62 | 66/77 | 44/44 | 44/50 | 0.5 m/s² |
| liftoff, rate down=12 | 56 % | 74 % | 84/100 | 44/44 | 8/87 | 85/86 | 63/67 | 55/66 | 53/74 | 55/62 | 74/81 | 48/48 | 38/44 | 0.5 m/s² |
| mild | 57 % | 74 % | 94/100 | 53/53 | 10/92 | 80/81 | 66/67 | 57/68 | 54/75 | 49/55 | 83/91 | 43/43 | 31/38 | 0.5 m/s² |
| mild, command=worst | 55 % | 72 % | 94/100 | 43/43 | 7/86 | 78/78 | 64/66 | 55/66 | 53/74 | 49/55 | 72/85 | 44/44 | 38/44 | 0.5 m/s² |
| mild, rate down=12 | 61 % | 80 % | 94/100 | 78/78 | 12/97 | 84/87 | 65/68 | 69/80 | 58/83 | 40/47 | 79/85 | 46/46 | 31/50 | 2 m/s² |
| firm | 57 % | 74 % | 94/100 | 53/53 | 10/92 | 80/81 | 66/67 | 57/68 | 54/75 | 49/55 | 83/91 | 43/43 | 31/38 | 0.5 m/s² |
| firm, command=worst | 55 % | 72 % | 94/100 | 43/43 | 7/86 | 78/78 | 64/66 | 55/66 | 53/74 | 49/55 | 72/85 | 44/44 | 38/44 | 0.5 m/s² |
| firm, rate down=12 | 61 % | 80 % | 94/100 | 80/80 | 12/97 | 86/88 | 65/68 | 71/82 | 58/83 | 40/47 | 79/87 | 46/46 | 31/50 | 3 m/s² |


Family C (following) passes 8–12 % but is 86–97 % safe: what fails is the setpoint chatter (40–70 changes per minute against ≤ 20) — SC2's controller, a tuning question, not a car number. Family H gets worse with the open rate limit (49 → 40 %): the out-of-envelope scenarios (junction, stop-and-go, zone 30) — a faster descent reaches the cruise minimum sooner and drops out; worth a look by SC2. The command path at its worst preset costs 2–3 points everywhere and 6–10 in B (approach) and I (driver).

Per hour of mixed driving (family L; event rates are my assumptions, core/mixed.js):

| Point | Mix | warnings / h | interventions / h | nuisance / h | missed / h | contacts / h if nobody brakes |
| --- | --- | --- | --- | --- | --- | --- |
| liftoff | L.commute | 24.2 | 22.3 | 3.1 | 2.40 | 12.30 |
| liftoff, command=worst | L.commute | 26.5 | 24.6 | 0.3 | 2.80 | 14.50 |
| liftoff, rate down=12 | L.commute | 22.0 | 20.1 | 3.4 | 2.40 | 10.20 |
| liftoff, rate down=12 | L.motorway | 24.0 | 22.3 | 4.2 | 3.50 | 14.20 |
| liftoff, rate down=12 | L.motorway.dense | 50.1 | 45.3 | 7.3 | 9.40 | 27.60 |
| liftoff, rate down=12 | L.rural | 15.7 | 15.7 | 3.5 | 0.20 | 7.80 |
| liftoff, rate down=12 | L.town | 37.3 | 33.2 | 0.0 | 0.40 | 1.60 |
| mild | L.commute | 23.9 | 22.0 | 4.1 | 2.40 | 9.90 |
| mild | L.motorway | 25.1 | 23.4 | 4.8 | 3.50 | 11.20 |
| mild | L.motorway.dense | 50.2 | 45.4 | 8.3 | 9.40 | 22.60 |
| mild | L.rural | 15.6 | 15.6 | 2.6 | 0.20 | 8.30 |
| mild | L.town | 44.0 | 39.9 | 6.4 | 0.40 | 14.10 |
| mild, command=worst | L.commute | 27.8 | 25.9 | 2.4 | 3.10 | 13.00 |
| mild, rate down=12 | L.commute | 11.2 | 9.3 | 0.4 | 3.30 | 5.30 |
| mild, rate down=12 | L.motorway | 8.4 | 6.7 | 0.3 | 4.60 | 5.70 |
| mild, rate down=12 | L.motorway.dense | 20.2 | 15.4 | 0.7 | 10.60 | 12.00 |
| mild, rate down=12 | L.rural | 6.9 | 6.9 | 0.3 | 0.20 | 3.10 |
| mild, rate down=12 | L.town | 33.8 | 30.9 | 0.0 | 0.40 | 11.10 |
| firm | L.commute | 23.9 | 22.0 | 4.1 | 2.40 | 9.90 |
| firm, command=worst | L.commute | 27.8 | 25.9 | 2.4 | 3.10 | 13.00 |
| firm, rate down=12 | L.commute | 10.9 | 9.0 | 0.4 | 3.30 | 5.30 |

Contacts per hour count the in-scope episodes run with the driver not reacting (the system alone), so they say how often the cruise alone would not cope, not how often a driver would crash. The parent's threshold is ≤ 1 warning per hour in real traffic: far off at every assumed point.

### 6. Findings beyond the map (hypotheses from simulation, not car facts)

1. **The parent brief's downward rate limit (3 km/h/s, range 1–5) caps our commanded deceleration below its own 1.5 m/s² requirement.** On the mild and firm presets alike, D.brake at −1.5 m/s² from 120 km/h ends in contact; with 12 km/h/s it passes at 1.8 m/s². Posted to SC2 in NOTES.
2. **Setpoint chatter**: steady following gives 40–70 changes per minute (C.human, C.long, C.osc) against the parent's ≤ 20; even the best tuning sets stay above 30. If the car's command path accepts only a few changes per minute, this is the next feasibility question after the deceleration.
3. **String stability** depends on the period: at 90 km/h the oscillation gain is 0.88 at a 20 s period and 1.10 at 60 s (C.osc): long, slow waves are amplified.
4. **Stub-era finding kept as a scenario**: B.approach.cruisemin (a lead at exactly the cruise minimum) caught a creeping collision with SC1's stub controller (set 31 km/h behind 30); with SC2's controller it passes — the scenario stays as a guard.
5. Lift-off only (−0.7 m/s²) handles lead braking without a warning only up to 0.5 m/s², whatever the controller.
6. **The parent's warning rule (needed distance / TTC) never warns about a close cut-in at a small speed difference**: E.cutin.slower at 15–20 m and 120 km/h leaves a 0.24–0.29 s time gap with no warning (TTC 4–8 s). These cut-ins are most of family L's missed warnings per hour. A time-gap term in the warning (SC2's file) would catch them; whether the driver should be warned there is the owner's call.

## Measure this first on the car (ranked by how far it moves the safety share, with an assumed plant)

1. **How hard the stock DCC slows down on a set-speed drop, and whether it uses the brakes at all** — the map's x axis: ~8 points between lift-off (−0.5 / −0.7 m/s²) and ≥ 1.4 m/s². Phase 2: large downward steps (−10, −20 km/h) on the flat and downhill.
2. **The command path's delay** (and whether the path exists at all, and how many changes it accepts) — ~6 points between 0 and 2 s. Paper only in this phase: it cannot be measured without writing to the car (a separate decision).
3. **Dead time from a set-speed change to the acceleration response** — ~4 points between 0.2 and 2.5 s; at 2.5 s no deceleration reaches the top band. Phase 2: ±5 km/h steps by the stalk.
4. **The camera's working range for the lead** — the tornado's first bar (−3.7 points at 84 m). Phase 1: approaches from far; the gap at the first valid `DataFrontVehicle` sample.
5. **Native rate and latency of `GapVehicleAhead.DataFrontVehicle`** on the head unit — ~2 points each (2 Hz vs 25 Hz; 0.5 s vs 0.15 s). Phase 0 recorder.
6. **Gap noise at long range** (camera: ∝ distance²) — Phase 1: steady following at several gaps against a calibrated distance.
7. **Target acquisition delay on a cut-in** — Phase 1 scenario 5 on video.

## The catalogue (154 files, 4 358 runs)

Scope: `in` = the cruise must handle it alone (run with the driver not reacting); `warnOnly` = beyond the concept, scored on the warning with the three reaction cases; `outOfEnvelope` = below the cruise minimum or stop-and-go, scored on saying so. "Groups" = variants without reaction cases and worst times; "runs" = with them.

| Family | Files | Groups | Runs | Added beyond the brief's table |
| --- | --- | --- | --- | --- |
| A. Free driving | 10 | 113 | 113 | 2 |
| B. Approach | 13 | 93 | 131 | 6 |
| C. Following | 13 | 169 | 169 | 5 |
| D. Lead braking | 12 | 159 | 382 | 5 |
| E. Cut-in | 12 | 207 | 565 | 4 |
| F. Cut-out | 8 | 65 | 137 | 3 |
| G. Sensing faults | 15 | 144 | 260 | 4 |
| H. Road and envelope | 9 | 47 | 111 | 3 |
| I. Driver | 8 | 47 | 59 | 2 |
| J. System failures | 41 | 80 | 1611 | 0 |
| K. Replay | 8 | 16 | 34 | 0 |
| L. Mixed driving | 5 | 786 | 786 | 0 |
| **Total** | 154 | 1926 | 4358 | 34 |

Added members (34): A.curves.free, A.free.adjacent; B.approach.curve, B.approach.degraded (low sun, night), B.approach.moto, B.approach.truck, B.approach.uphill, B.queue.crawl; C.osc.follower, C.square, C.steady.curve, C.steady.grade, C.truck.hill; D.brake.rain, D.brake.shortgap, D.brake.truck, D.brake.uphill, D.slowdown; E.cutin.aborted, E.cutin.drift, E.cutin.rain, E.cutin.withlead; F.reveal.none, F.reveal.slower.rain, F.reveal.truck; G.blind.long, G.curve.physical, G.id.flicker, G.noise.far; H.cruise.dropout, H.ego.lanechange, H.highspeed.brake; I.override.long, I.vmax.lower. Also beyond the table: the A step series split into up / down with ±1 … ±40; C.steady from 0.6 s; D to a crawl (10 / 20 km/h) as well as to a standstill; J's command path at its worst on five hosts and three combined failures (frozen gap in rain, blind sensor in rain, 2 Hz sensor with the worst command path); K's four traces each with a spliced cut-in; L as five mixes (motorway, dense motorway, rural, town, commute).

J ("every failure of §2.7 in B–F at the worst time"): 7 fault kinds (appStop, huRestart, gapFreeze, sensorBlind, commandLost (the first decrease), setStuck, cruiseDropout) × their meaningful hosts (B.approach Δv 40, D.brake −1.5, D.brake.hard −3, E.cutin.slower 30 m, F.reveal.slower Δv 40, F.reveal.none = "just after raising the set speed"), each at 7–8 injection times around the event × 90 / 120 km/h × 3 reactions; the group's worst counts.

| Id | Name | Scope | Decided by | Swept | Groups | Runs |
| --- | --- | --- | --- | --- | --- | --- |
| `A.crest` | Crest (up, then down) | in | overspeed_kmh, warnCount | v, g | 6 | 6 |
| `A.curves.free` | Winding road, no target (added) | in | warnCount, setpointChangesPerMin, overspeed_kmh | v, R | 6 | 6 |
| `A.dip` | Dip (down, then up) | in | overspeed_kmh, warnCount | v, g | 6 | 6 |
| `A.free.adjacent` | Overtaking a slower car in the next lane (added) | in | warnCount, peakDecel_mps2, setpointChangesPerMin | v, dv | 6 | 6 |
| `A.free.steady` | No target at a steady speed | in | warnCount, overspeed_kmh, setpointChangesPerMin, collision | v, sensing | 10 | 10 |
| `A.grade.down` | Downhill 3 / 6 / 10 % | in | overspeed_kmh, warnCount, setpointChangesPerMin | v, g | 15 | 15 |
| `A.grade.up` | Uphill 3 / 6 / 10 % | in | overspeed_kmh, warnCount, setpointChangesPerMin | v, g | 15 | 15 |
| `A.setstep.down` | The driver lowers the maximum (−1 … −40 km/h) | in | overspeed_kmh, setpointChangesPerMin, warnCount | v, step | 22 | 22 |
| `A.setstep.up` | The driver raises the maximum (+1 … +40 km/h) | in | overspeed_kmh, setpointChangesPerMin, warnCount | v, step | 24 | 24 |
| `A.vmax.change` | The driver changes the maximum several times while free | in | overspeed_kmh, setpointChangesPerMin, warnCount | v | 3 | 3 |
| `B.approach` | A slower lead from far (Δv 10 … 50 km/h) | in | collision, minTimeGap_s, minTTC_s, nuisanceWarnings, warnCount | v, dv | 18 | 18 |
| `B.approach.belowmin` | A lead slower than the cruise minimum (tractor, bicycle) | outOfEnvelope | collision, envelopeExitAnnounced, missedWarning | v, vl | 6 | 12 |
| `B.approach.cruisemin` | A lead at the cruise's minimum speed | in | collision, minTimeGap_s, minTTC_s, nuisanceWarnings | v | 3 | 3 |
| `B.approach.curve` | Approach in a long curve (added) | in | collision, minTimeGap_s, minTTC_s, nuisanceWarnings | v, dv, R | 8 | 8 |
| `B.approach.degraded` | Approach in low sun and at night (added) | in | collision, minTimeGap_s, minTTC_s, nuisanceWarnings | v, dv, sensing | 8 | 8 |
| `B.approach.downhill` | Approach on a downhill | in | collision, minTimeGap_s, minTTC_s, nuisanceWarnings | v, dv, g | 12 | 12 |
| `B.approach.large` | A much slower lead from far (Δv 60 … 80 km/h) | warnOnly | collision, missedWarning, warnLeadTime_s | v, dv | 7 | 21 |
| `B.approach.moto` | Approach a slow motorcycle (acquired late) (added) | in | collision, minTimeGap_s, minTTC_s, nuisanceWarnings | v, dv | 4 | 4 |
| `B.approach.rain` | Approach in rain mode | in | collision, minTimeGap_s, minTTC_s, nuisanceWarnings | v, dv, sensing | 6 | 6 |
| `B.approach.truck` | Approach a slow truck (acquired early) (added) | in | collision, minTimeGap_s, minTTC_s, nuisanceWarnings | v, dv | 4 | 4 |
| `B.approach.uphill` | Approach on an uphill (added) | in | collision, minTimeGap_s, minTTC_s, nuisanceWarnings | v, dv, g | 8 | 8 |
| `B.queue` | A standing obstacle / the tail of a motorway queue | warnOnly | collision, missedWarning, warnLeadTime_s | v, sensing | 6 | 18 |
| `B.queue.crawl` | The tail of a crawling queue (10 km/h) (added) | warnOnly | collision, missedWarning, warnLeadTime_s | v | 3 | 9 |
| `C.accel.away` | The lead accelerates away | in | overspeed_kmh, setpointChangesPerMin, peakAccel_mps2, timeLostPct | v, a | 6 | 6 |
| `C.drift` | The lead drifts slowly | in | gapErrRmsRel, setpointChangesPerMin, nuisanceWarnings, collision | v, s | 6 | 6 |
| `C.follower` | A car behind us following closely | in | collision, followerCollision, followerMinTimeGap_s, followerPeakDecel_mps2 | v, T | 9 | 9 |
| `C.human` | A human-like lead (IDM with noise) | in | collision, minTimeGap_s, minTTC_s, gapErrRmsRel, setpointChangesPerMin, nuisanceWarnings | v, sensing | 10 | 10 |
| `C.long` | Long following (30 min) for chatter and drift | in | collision, setpointChangesPerMin, gapErrRmsRel, nuisanceWarnings, minTimeGap_s | v | 2 | 2 |
| `C.osc` | The lead's speed oscillates (string stability) | in | collision, oscGain, minTimeGap_s, minTTC_s, setpointChangesPerMin | v, amp, P | 60 | 60 |
| `C.osc.follower` | Oscillating lead with a car behind us (added) | in | collision, oscGain, followerOscGain, followerMinTimeGap_s | v, amp, P | 6 | 6 |
| `C.platoon` | A platoon of 3–8 human-like cars with a braking wave | in | collision, minTimeGap_s, minTTC_s, nuisanceWarnings | v, n | 9 | 9 |
| `C.square` | The lead steps its speed up and down (added) | in | collision, minTimeGap_s, minTTC_s, setpointChangesPerMin, peakDecel_mps2 | v, dv, P | 12 | 12 |
| `C.steady` | Steady following, starting at each time gap | in | collision, gapErrRmsRel, setpointChangesPerMin, nuisanceWarnings, peakDecel_mps2 | v, h0 | 30 | 30 |
| `C.steady.curve` | Steady following on a winding road (added) | in | collision, minTimeGap_s, nuisanceWarnings, setpointChangesPerMin, curveLostSpeedGain_kmh | v, R | 6 | 6 |
| `C.steady.grade` | Steady following on a grade (added) | in | collision, minTimeGap_s, gapErrRmsRel, setpointChangesPerMin | v, g | 12 | 12 |
| `C.truck.hill` | Following a truck over a hill (added) | in | collision, minTimeGap_s, minTTC_s, setpointChangesPerMin | v | 1 | 1 |
| `D.brake` | The lead brakes to a lower speed (−0.5 … −1.5 m/s²) | in | collision, minTimeGap_s, minTTC_s, warnCount | v, dv, a | 21 | 21 |
| `D.brake.downhill` | The lead brakes on a downhill | warnOnly | collision, missedWarning, warnLeadTime_s | v, a, g | 12 | 36 |
| `D.brake.hard` | The lead brakes hard (−2 … −6 m/s²) | warnOnly | collision, missedWarning, warnLeadTime_s | v, dv, a | 28 | 84 |
| `D.brake.rain` | The lead brakes in rain (added) | warnOnly | collision, missedWarning, warnLeadTime_s | v, a, sensing | 4 | 12 |
| `D.brake.shortgap` | The lead brakes while we follow too close (added) | warnOnly | collision, missedWarning, warnLeadTime_s | v, h0, a | 12 | 36 |
| `D.brake.stop` | The lead brakes to a standstill or a crawl | outOfEnvelope | collision, envelopeExitAnnounced, missedWarning | v, vt, a | 27 | 54 |
| `D.brake.truck` | A truck ahead brakes (added) | warnOnly | collision, missedWarning, warnLeadTime_s | v, a | 4 | 12 |
| `D.brake.uphill` | The lead brakes on an uphill (added) | warnOnly | collision, missedWarning, warnLeadTime_s | v, a | 4 | 12 |
| `D.closing` | The lead brakes while we are still closing | warnOnly | collision, missedWarning, warnLeadTime_s | v, dv, g, a | 16 | 48 |
| `D.double` | Two brake events in a row | warnOnly | collision, missedWarning, warnLeadTime_s | v, h, a | 18 | 54 |
| `D.slowdown` | The lead slows down and speeds back up (added) | in | collision, minTimeGap_s, minTTC_s, warnCount, setpointChangesPerMin | v, a | 4 | 4 |
| `D.tap` | A brake tap and release | in | collision, minTimeGap_s, nuisanceWarnings, warnCount, peakDecel_mps2 | v, d | 9 | 9 |
| `E.cutin.aborted` | An aborted cut-in (added) | in | collision, nuisanceWarnings, peakDecel_mps2 | v, g | 4 | 4 |
| `E.cutin.brake` | Cut-in, then brake | warnOnly | collision, missedWarning, warnLeadTime_s | v, g, a | 8 | 24 |
| `E.cutin.double` | Two cut-ins in a row | warnOnly | collision, missedWarning, warnLeadTime_s | v, g, t2 | 8 | 24 |
| `E.cutin.drift` | A slow drift-in lane change (added) | warnOnly | collision, missedWarning, warnLeadTime_s | v, g | 6 | 18 |
| `E.cutin.duringaccel` | A cut-in while we accelerate back to the maximum | warnOnly | collision, missedWarning, warnLeadTime_s | v, g, tc | 8 | 24 |
| `E.cutin.equal` | A car at our speed cuts in | warnOnly | collision, missedWarning, warnLeadTime_s | v, g | 21 | 63 |
| `E.cutin.faster` | A faster car cuts in | in | collision, nuisanceWarnings, peakDecel_mps2, setpointChangesPerMin | v, g, dv | 24 | 24 |
| `E.cutin.moto` | A motorcycle cuts in (narrow, acquired late) | warnOnly | collision, missedWarning, warnLeadTime_s | v, g, dv | 12 | 36 |
| `E.cutin.rain` | A cut-in in rain (added) | warnOnly | collision, missedWarning, warnLeadTime_s | v, g, dv, sensing | 8 | 24 |
| `E.cutin.slower` | A slower car cuts in (gap 5 … 60 m) | warnOnly | collision, missedWarning, warnLeadTime_s | v, g, dv | 84 | 252 |
| `E.cutin.truck` | A truck cuts in (acquired early) | warnOnly | collision, missedWarning, warnLeadTime_s | v, g, dv | 12 | 36 |
| `E.cutin.withlead` | A car cuts into our gap while we follow (added) | warnOnly | collision, missedWarning, warnLeadTime_s | v, f, dv | 12 | 36 |
| `F.exit` | The lead leaves on an exit ramp while slowing | in | collision, minTimeGap_s, nuisanceWarnings, peakDecel_mps2 | v, a | 4 | 4 |
| `F.reveal.faster` | A cut-out reveals a faster car | in | collision, minTimeGap_s, peakAccel_mps2, setpointChangesPerMin | v, dv | 4 | 4 |
| `F.reveal.none` | A cut-out reveals an open road (added) | in | overspeed_kmh, peakAccel_mps2, setpointChangesPerMin, timeLostPct | v | 3 | 3 |
| `F.reveal.slower` | A cut-out reveals a slower car (20 … 40 km/h) | in | collision, minTimeGap_s, minTTC_s, warnCount | v, dv, gh | 18 | 18 |
| `F.reveal.slower.large` | A cut-out reveals a much slower car (50 … 80 km/h) | warnOnly | collision, missedWarning, warnLeadTime_s | v, dv, gh | 16 | 48 |
| `F.reveal.slower.rain` | A cut-out reveals a slower car in rain (added) | warnOnly | collision, missedWarning, warnLeadTime_s | v, dv, sensing | 4 | 12 |
| `F.reveal.standing` | A cut-out reveals a standing car | warnOnly | collision, missedWarning, warnLeadTime_s | v, gh | 12 | 36 |
| `F.reveal.truck` | A truck ahead reveals a slower car (added) | warnOnly | collision, missedWarning, warnLeadTime_s | v, dv | 4 | 12 |
| `G.blind.long` | The camera is blind for 30 / 60 s (low sun, dirt) (added) | warnOnly | collision, missedWarning, faultDetectDelay_s, lostTargetSpeedGain_kmh | v, d | 4 | 12 |
| `G.curve.loss` | Target lost on a curve for 1 … 10 s, then back | in | collision, minTimeGap_s, lostTargetSpeedGain_kmh, curveLostSpeedGain_kmh | v, L, lb | 30 | 30 |
| `G.curve.physical` | Target lost by curvature (sensor model), next-lane car present (added) | in | collision, minTimeGap_s, nuisanceWarnings, lostTargetSpeedGain_kmh, curveLostSpeedGain_kmh | v, R | 6 | 6 |
| `G.dropout.burst` | Dropout bursts | in | collision, minTimeGap_s, minTTC_s, nuisanceWarnings, setpointChangesPerMin | v, b | 6 | 6 |
| `G.frozen` | The gap freezes at its last value | warnOnly | collision, missedWarning, warnLeadTime_s, faultDetectDelay_s | v, d, lb | 24 | 72 |
| `G.ghost` | A ghost target for 0.5 … 3 s | in | collision, nuisanceWarnings, peakDecel_mps2, setpointChangesPerMin | v, d, wl | 16 | 16 |
| `G.id.flicker` | The target id flickers (same car) (added) | in | collision, nuisanceWarnings, peakDecel_mps2, setpointChangesPerMin | v, q | 4 | 4 |
| `G.late.entry` | The lead beyond range, entering it at a large speed difference | warnOnly | collision, missedWarning, warnLeadTime_s | v, dv, sensor.range_m | 6 | 18 |
| `G.latency.spike` | Latency spikes | warnOnly | collision, missedWarning, warnLeadTime_s | v, s | 6 | 18 |
| `G.lowrate.brake` | Low native rate (2 / 5 Hz), lead braking | in | collision, minTimeGap_s, minTTC_s, warnCount | v, sensor.rateHz | 4 | 4 |
| `G.lowrate.follow` | Low native rate (2 / 5 Hz), human-like lead | in | collision, minTimeGap_s, gapErrRmsRel, setpointChangesPerMin, nuisanceWarnings | v, sensor.rateHz | 4 | 4 |
| `G.noise.far` | Range noise at long gaps (added) | in | collision, setpointChangesPerMin, nuisanceWarnings, accelRms_mps2, estVrelErrRms_mps | v, sensing | 6 | 6 |
| `G.outliers` | Outliers in the gap | in | collision, nuisanceWarnings, peakDecel_mps2, setpointChangesPerMin | v, r | 4 | 4 |
| `G.range40` | Range cut to 40 / 60 / 80 m | warnOnly | collision, missedWarning, warnLeadTime_s | v, dv, sensor.range_m | 18 | 54 |
| `G.wrongTarget` | Lost and replaced by the next-lane car | in | collision, nuisanceWarnings, peakDecel_mps2, minTimeGap_s | v, sensor.curvePredErr | 6 | 6 |
| `H.cruise.dropout` | The lead slows below the minimum; the cruise drops out (added) | outOfEnvelope | collision, envelopeExitAnnounced, cmdWhileInactive | v | 2 | 4 |
| `H.ego.lanechange` | We change lane in behind a slower car (added) | warnOnly | collision, missedWarning, warnLeadTime_s | v, g, dv | 8 | 24 |
| `H.highspeed` | Following at 150 … 180 km/h | in | collision, minTimeGap_s, minTTC_s, warnCount, setpointChangesPerMin | v | 3 | 3 |
| `H.highspeed.brake` | The lead brakes hard at 150 … 180 km/h (added) | warnOnly | collision, missedWarning, warnLeadTime_s | v, a | 6 | 18 |
| `H.junction` | A roundabout or junction behind a lead (the lead stops) | outOfEnvelope | collision, envelopeExitAnnounced, missedWarning | v, a | 6 | 12 |
| `H.limit.change` | The driver lowers the maximum behind a faster lead | in | peakDecel_mps2, setpointChangesPerMin, warnCount, collision | v, d | 6 | 6 |
| `H.onramp` | Merging traffic at an on-ramp | warnOnly | collision, missedWarning, warnLeadTime_s | v, m, g | 12 | 36 |
| `H.stopgo` | Stop-and-go traffic (out of scope) | outOfEnvelope | collision, envelopeExitAnnounced, missedWarning | P | 2 | 4 |
| `H.zone30` | 30 km/h zone (near the cruise minimum) | outOfEnvelope | collision, envelopeExitAnnounced, missedWarning | vl | 2 | 4 |
| `I.brake.resume` | Brake and resume | in | collision, cmdWhileInactive, peakAccel_mps2, nuisanceWarnings | v, r | 6 | 6 |
| `I.cancel` | The driver cancels the cruise | in | collision, cmdWhileInactive | v | 3 | 3 |
| `I.override` | Accelerator override while following | in | collision, nuisanceWarnings, peakDecel_mps2 | v, p, d | 12 | 12 |
| `I.override.long` | Long override while the lead brakes (added) | warnOnly | collision, missedWarning, warnLeadTime_s | v | 2 | 6 |
| `I.reaction.sweep` | Hard braking ahead, the driver's reaction time and braking swept | warnOnly | collision | driver.reactionMedian_s, driver.brakeLevel_ms2 | 15 | 15 |
| `I.resume.close` | Resume with the lead much closer than at suspension | warnOnly | collision, missedWarning, warnLeadTime_s | v, dv | 4 | 12 |
| `I.vmax.lower` | The maximum lowered below the lead's speed (added) | in | peakDecel_mps2, setpointChangesPerMin, warnCount | v | 2 | 2 |
| `I.vmax.raise` | The maximum raised by 30 km/h behind a slow lead | in | collision, minTimeGap_s, peakAccel_mps2, setpointChangesPerMin | v | 3 | 3 |
| `J.appStop.B` | appStop during B.approach, at the worst time | warnOnly | collision, missedWarning, warnLeadTime_s | tf, v | 2 | 42 |
| `J.appStop.D1` | appStop during D.brake, at the worst time | warnOnly | collision, missedWarning, warnLeadTime_s | tf, v | 2 | 48 |
| `J.appStop.D3` | appStop during D.brake.hard, at the worst time | warnOnly | collision, missedWarning, warnLeadTime_s | tf, v | 2 | 48 |
| `J.appStop.E` | appStop during E.cutin.slower, at the worst time | warnOnly | collision, missedWarning, warnLeadTime_s | tf, v | 2 | 48 |
| `J.appStop.F` | appStop during F.reveal.slower, at the worst time | warnOnly | collision, missedWarning, warnLeadTime_s | tf, v | 2 | 48 |
| `J.appStop.R` | appStop during F.reveal.none, at the worst time | warnOnly | collision, missedWarning, warnLeadTime_s | tf, v | 2 | 42 |
| `J.cmdworst.B` | The command path at its worst settings during B.approach | warnOnly | collision, missedWarning, warnLeadTime_s, minTimeGap_s | command, v | 2 | 6 |
| `J.cmdworst.D1` | The command path at its worst settings during D.brake | warnOnly | collision, missedWarning, warnLeadTime_s, minTimeGap_s | command, v | 2 | 6 |
| `J.cmdworst.D3` | The command path at its worst settings during D.brake.hard | warnOnly | collision, missedWarning, warnLeadTime_s, minTimeGap_s | command, v | 2 | 6 |
| `J.cmdworst.E` | The command path at its worst settings during E.cutin.slower | warnOnly | collision, missedWarning, warnLeadTime_s, minTimeGap_s | command, v | 2 | 6 |
| `J.cmdworst.F` | The command path at its worst settings during F.reveal.slower | warnOnly | collision, missedWarning, warnLeadTime_s, minTimeGap_s | command, v | 2 | 6 |
| `J.combined.blind.rain` | The sensor goes blind during a rainy approach, at the worst time | warnOnly | collision, missedWarning, warnLeadTime_s, faultDetectDelay_s | tf, sensing | 1 | 21 |
| `J.combined.frozen.rain` | The gap freezes during rain, the lead braking, at the worst time | warnOnly | collision, missedWarning, warnLeadTime_s, faultDetectDelay_s | tf, sensing | 1 | 24 |
| `J.combined.lowrate.cmdworst` | A 2 Hz sensor and the worst command path, lead braking | warnOnly | collision, missedWarning, warnLeadTime_s, minTimeGap_s | command, sensor.rateHz, v | 2 | 6 |
| `J.commandLost.B` | commandLost during B.approach, at the worst time | warnOnly | collision, missedWarning, warnLeadTime_s | tf, v | 2 | 42 |
| `J.commandLost.D1` | commandLost during D.brake, at the worst time | warnOnly | collision, missedWarning, warnLeadTime_s | tf, v | 2 | 48 |
| `J.commandLost.D3` | commandLost during D.brake.hard, at the worst time | warnOnly | collision, missedWarning, warnLeadTime_s | tf, v | 2 | 48 |
| `J.commandLost.E` | commandLost during E.cutin.slower, at the worst time | warnOnly | collision, missedWarning, warnLeadTime_s | tf, v | 2 | 48 |
| `J.commandLost.F` | commandLost during F.reveal.slower, at the worst time | warnOnly | collision, missedWarning, warnLeadTime_s | tf, v | 2 | 48 |
| `J.cruiseDropout.B` | cruiseDropout during B.approach, at the worst time | warnOnly | collision, missedWarning, warnLeadTime_s | tf, v | 2 | 42 |
| `J.cruiseDropout.D1` | cruiseDropout during D.brake, at the worst time | warnOnly | collision, missedWarning, warnLeadTime_s | tf, v | 2 | 48 |
| `J.cruiseDropout.D3` | cruiseDropout during D.brake.hard, at the worst time | warnOnly | collision, missedWarning, warnLeadTime_s | tf, v | 2 | 48 |
| `J.gapFreeze.B` | gapFreeze during B.approach, at the worst time | warnOnly | collision, missedWarning, warnLeadTime_s, faultDetectDelay_s | tf, v | 2 | 42 |
| `J.gapFreeze.D1` | gapFreeze during D.brake, at the worst time | warnOnly | collision, missedWarning, warnLeadTime_s, faultDetectDelay_s | tf, v | 2 | 48 |
| `J.gapFreeze.D3` | gapFreeze during D.brake.hard, at the worst time | warnOnly | collision, missedWarning, warnLeadTime_s, faultDetectDelay_s | tf, v | 2 | 48 |
| `J.gapFreeze.E` | gapFreeze during E.cutin.slower, at the worst time | warnOnly | collision, missedWarning, warnLeadTime_s, faultDetectDelay_s | tf, v | 2 | 48 |
| `J.gapFreeze.F` | gapFreeze during F.reveal.slower, at the worst time | warnOnly | collision, missedWarning, warnLeadTime_s, faultDetectDelay_s | tf, v | 2 | 48 |
| `J.huRestart.B` | huRestart during B.approach, at the worst time | warnOnly | collision, missedWarning, warnLeadTime_s, faultDetectDelay_s | tf, v | 2 | 42 |
| `J.huRestart.D1` | huRestart during D.brake, at the worst time | warnOnly | collision, missedWarning, warnLeadTime_s, faultDetectDelay_s | tf, v | 2 | 48 |
| `J.huRestart.D3` | huRestart during D.brake.hard, at the worst time | warnOnly | collision, missedWarning, warnLeadTime_s, faultDetectDelay_s | tf, v | 2 | 48 |
| `J.huRestart.E` | huRestart during E.cutin.slower, at the worst time | warnOnly | collision, missedWarning, warnLeadTime_s, faultDetectDelay_s | tf, v | 2 | 48 |
| `J.huRestart.F` | huRestart during F.reveal.slower, at the worst time | warnOnly | collision, missedWarning, warnLeadTime_s, faultDetectDelay_s | tf, v | 2 | 48 |
| `J.huRestart.R` | huRestart during F.reveal.none, at the worst time | warnOnly | collision, missedWarning, warnLeadTime_s, faultDetectDelay_s | tf, v | 2 | 42 |
| `J.sensorBlind.B` | sensorBlind during B.approach, at the worst time | warnOnly | collision, missedWarning, warnLeadTime_s, faultDetectDelay_s | tf, v | 2 | 42 |
| `J.sensorBlind.D1` | sensorBlind during D.brake, at the worst time | warnOnly | collision, missedWarning, warnLeadTime_s, faultDetectDelay_s | tf, v | 2 | 48 |
| `J.sensorBlind.D3` | sensorBlind during D.brake.hard, at the worst time | warnOnly | collision, missedWarning, warnLeadTime_s, faultDetectDelay_s | tf, v | 2 | 48 |
| `J.sensorBlind.F` | sensorBlind during F.reveal.slower, at the worst time | warnOnly | collision, missedWarning, warnLeadTime_s, faultDetectDelay_s | tf, v | 2 | 48 |
| `J.setStuck.B` | setStuck during B.approach, at the worst time | warnOnly | collision, missedWarning, warnLeadTime_s | tf, v | 2 | 42 |
| `J.setStuck.D1` | setStuck during D.brake, at the worst time | warnOnly | collision, missedWarning, warnLeadTime_s | tf, v | 2 | 48 |
| `J.setStuck.D3` | setStuck during D.brake.hard, at the worst time | warnOnly | collision, missedWarning, warnLeadTime_s | tf, v | 2 | 48 |
| `J.setStuck.F` | setStuck during F.reveal.slower, at the worst time | warnOnly | collision, missedWarning, warnLeadTime_s | tf, v | 2 | 48 |
| `K.replay.motorway-dense` | Replay: motorway-dense (synthetic, recorded-like) | in | collision, minTimeGap_s, minTTC_s, setpointChangesPerMin, gapErrRmsRel, nuisanceWarnings, warnCount | sensing | 2 | 2 |
| `K.replay.motorway-flow` | Replay: motorway-flow (synthetic, recorded-like) | in | collision, minTimeGap_s, minTTC_s, setpointChangesPerMin, gapErrRmsRel, nuisanceWarnings, warnCount | sensing | 2 | 2 |
| `K.replay.rural` | Replay: rural (synthetic, recorded-like) | in | collision, minTimeGap_s, minTTC_s, setpointChangesPerMin, gapErrRmsRel, nuisanceWarnings, warnCount | sensing | 2 | 2 |
| `K.replay.town` | Replay: town (synthetic, recorded-like) | outOfEnvelope | collision, envelopeExitAnnounced, missedWarning | sensing | 2 | 4 |
| `K.splice.motorway-dense` | Replay: motorway-dense with a cut-in spliced in | warnOnly | collision, missedWarning, warnLeadTime_s | tc | 2 | 6 |
| `K.splice.motorway-flow` | Replay: motorway-flow with a cut-in spliced in | warnOnly | collision, missedWarning, warnLeadTime_s | tc | 2 | 6 |
| `K.splice.rural` | Replay: rural with a cut-in spliced in | warnOnly | collision, missedWarning, warnLeadTime_s | tc | 2 | 6 |
| `K.splice.town` | Replay: town with a cut-in spliced in | warnOnly | collision, missedWarning, warnLeadTime_s | tc | 2 | 6 |
| `L.commute` | Mixed driving: a commute (50 % motorway, 35 % rural, 15 % town) | in | per hour (catalogue level) | – | 238 | 238 |
| `L.motorway` | Mixed driving: motorway | in | per hour (catalogue level) | – | 179 | 179 |
| `L.motorway.dense` | Mixed driving: dense motorway | in | per hour (catalogue level) | – | 209 | 209 |
| `L.rural` | Mixed driving: rural roads | in | per hour (catalogue level) | – | 119 | 119 |
| `L.town` | Mixed driving: town | in | per hour (catalogue level) | – | 41 | 41 |

## Left out, and why

- J's injection time is a grid around the event (7–8 points), not SC1's `at: 'worst'` search (SC1 offers it in `failures.searchWorst`); a grid keeps every run deterministic and visible in the matrix. Switching a J file to `at: 'worst'` is a one-line change if wanted.
- The ghost target's distance is chosen by SC2's sensor (no fault option for it).
- Family L's event rates are my assumptions (orders of magnitude for a European commuter car); its episodes are independent runs (no stacking except where D.double, E.cutin.double and J stack on purpose).
- No public driving dataset was downloaded (offline rule). Candidates for family K, to propose to the owner: highD (German motorway, drone, free for non-commercial research use on request) and the Waymo Open Motion Dataset (non-commercial licence). Neither is fetched.
- The time-lost metric is only in plans with `baseline: true` (presets, smoke); the tuning table shows it as null.

## Owner's decisions needed

1. Whether the parent's downward rate limit (3 km/h/s, range 1–5) should be widened: as written it cannot meet the parent's own 1.5 m/s² requirement.
2. Whether ≤ 20 setpoint changes per minute stays the threshold (SC2's controller is at 30–70 in steady following), or the command path's real accepted rate replaces it once known.
3. The proposed thresholds I added beyond the parent's table (warning lead time ≥ 1.5 s, peak deceleration ≤ 3.5, jerk ≤ 2.5 m/s³, follower time gap ≥ 0.5 s, fault detection ≤ 2 s, speed gain on a lost target ≤ 2 km/h, critical time gap 0.3 s): all settings in `core/metrics.js` SCHEMA.

## Підсумок українською

Усе нижче пораховано **з припущеною моделлю круїзу** (пресети liftoff / mild / firm від SC1 і сітки навколо них). Поїздки ще не було, тому це не вердикт «так / ні», а перелік чисел авто, від яких вердикт залежатиме.

- **Каталог:** 154 файли сценаріїв у 12 сімействах (A–L), 4 358 конкретних прогонів; 34 сценарії додано понад таблицю брифу. Кожен сценарій має свою область (in / warnOnly / outOfEnvelope) і метрики, що його вирішують. Сценарії з попередженням проганяються тричі: водій реагує, реагує пізно (2,5 с), не реагує.
- **Перша карта здійсненності** (сповільнення круїзу × мертвий час): частка каталогу, що лишається безпечною, росте з ~71 % (лише скидання газу) до ~80 %, коли круїз сповільнює **приблизно від 1,4 м/с² з мертвим часом до 0,5 с** (1,8 м/с² при 1 с, 2,3 м/с² при 1,5 с; при 2,5 с цієї смуги не досягти). Понад ~1,8 м/с² більше гальмівної сили майже нічого не додає. Решта ~20 % — ситуації, яких жоден круїз не витягне (стояча перешкода, різке гальмування, врізання на 5–10 м, відмови в найгірший момент): там важить лише попередження.
- **Головна знахідка:** з лімітом зниження уставки з батьківського брифу (3 км/год/с, діапазон 1–5) сила гальмування круїзу взагалі не має значення: команда обмежує сповільнення до 0,83 м/с² (1,39 при 5 км/год/с), що нижче власної вимоги брифу (1,5 м/с² без попередження). Пресети mild і firm тоді поводяться однаково; гальмування лідера −1,5 м/с² на 120 км/год закінчується контактом. З лімітом 12 км/год/с те саме проходить.
- **Друга карта** (частота камери × затримка командного шляху): затримка до 1 с коштує ~2 пункти, 2 с — ~6; 2 Гц проти 25 Гц — ~2 пункти (здебільшого врізання); латентність 0,5 с — ще 2–3.
- **Налаштування:** найбільше важить ліміт зниження уставки (67 → 80 % безпечних від 1 до 12 км/год/с), далі часовий інтервал h (70 → 76 % від 1,5 до 3 с); поріг TTC попередження частки не змінює. Стартовий набір брифу — 86-й зі 144.
- **Інші знахідки:** у сталому слідуванні контролер змінює уставку 40–70 разів на хвилину (поріг брифу ≤ 20); довгі хвилі (період 60 с) підсилюються (коефіцієнт 1,10); правило попередження брифу (потрібна відстань / TTC) не попереджає про врізання на 15–20 м з малою різницею швидкостей, хоча часовий інтервал тоді 0,24–0,29 с.
- **Що виміряти першим на авто:** 1) наскільки сильно DCC гальмує при зниженні уставки і чи гальмує колодками взагалі; 2) затримку командного шляху і чи він існує взагалі (лише на папері в цій фазі); 3) мертвий час від зміни уставки до прискорення; 4) робочу дальність камери до лідера; 5) власну частоту і затримку `DataFrontVehicle`; 6) шум дистанції на великій відстані; 7) затримку захоплення цілі при врізанні.
- **Змішана їзда (сімейство L, частоти подій — мої припущення):** поїздка на роботу (50 % автомагістраль, 35 % заміська, 15 % місто): 24 попереджень на годину з контролером брифу (mild) і 11 з лімітом 12 км/год/с; поріг брифу — ≤ 1 на годину. Пропущені попередження (3.3 на годину) — майже всі від врізань на малій відстані.

