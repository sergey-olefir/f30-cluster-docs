# SC2 report — sensing and control of the cruise simulator

2026-11-19 · agent SC2 · branch `cruise-sim/sc2` (worktree `.claude/worktrees/cs-sc2`), last commit `da284e2`.
Every number below comes from an **assumed plant** (SC1's presets `liftoff` / `mild` / `firm`) and an **assumed camera** (my defaults). None of them is a verdict on the car.

## 1. What was built

All files are in `tools/cruise-sim/`. Each module is wrapped in an IIFE, registers as `self.CS.<name>` in the page and `module.exports` in Node, and follows the contract's module shape `{ NAME, SCHEMA, PRESETS?, create(params, rng) }`. Every SCHEMA entry has a unit, a default and a range (or options).

| File | What it is | Node 8 / pure |
| --- | --- | --- |
| `core/sensor.js` | The front camera (KAFAS). Range noise grows with distance: `sd = sd0 + sd100·(d/100)^exp`, default exp 2, AR(1)-correlated, plus a scale error. Relative speed: lagged, noisy and quantised, or absent (`vrelMode: 'none'`). Native rate 2–25 Hz, latency with half-normal jitter (delivery stays in order), single dropouts and exponential bursts, outliers. A frozen transport (`repeat` with the same seq and time, `repeatNewSeq`, or `silent`); blind (`invalid` with `qual: 'blind'`, or `silent`). Target ids on cut-in / cut-out: acquisition delay (motorcycle ×2, truck ×0.6), release delay, track memory, and a jump or a blend between targets, the id changing at the start or the end. Loss on curves and the next-lane car taken for the lead both come from one geometric model: the camera predicts our lane from the yaw rate and misses a fraction (`curvePredErr`) of the curve; this also covers curve entry. Field of view, range, ghosts (random or forced; standing ghosts close at our speed), weather (`rain` / `lowSun` / `fog` / `night`), and "no target" as null, a 4093-like sentinel, or a sentinel still flagged valid. Seven presets (`ideal clear rain lowSun fog night slow2Hz poor`); every random effect has its own forked stream. Also exports `geom.trueLead / gapTo / yRel` (used by the baselines). | Node 8 syntax (not required to be pure; it is the world model) |
| `core/estimator.js` | The Phase 3 estimator, four variants: `kalman` (state [d, v_rel], our acceleration as a known input), `kalmanCA` (adds a_lead as a Singer process), `alphaBeta` (growing-memory warm-up after a reset), `raw`. Re-initialisation on an id change or on two consistent distance jumps (one jump is an outlier and is skipped). Coasting through dropouts, then the track is dropped. Staleness from silence, a repeated sequence number or time, and optionally identical values. Prediction to "now" from the message's own time, or from the receive time minus an assumed latency. `est = { valid, id, gap, vrel, vlead, alead, stale, reset, resetReason, coasting, settled, sensorFault, reason, age, sinceCorr, n }`. | **yes** |
| `core/controller.js` | The Phase 4 set-speed law with every tuning-table entry (d0 5 [3–8] m, h 2.0 [1.5–3] s, τ 4 [2–8] s, down / up 3 [1–5] / 1 [0.5–2] km/h/s), plus feed-forward of a_lead·T_lag (optional), the driver's maximum (lowering acts at once), the cruise minimum, quantisation to the cruise step (1 / 5 / 10 km/h) with hysteresis, a maximum change rate, resend of a lost command (`send` flag), hold-then-release after loss or stale input, no acceleration into a curve on a lost target (`curvature` or `yawRate` input), no commands while the cruise is suspended, off or dropped out (re-synced from the held set speed on resume), and two additions of mine explained in §4: `kinLimit` (option, off) and `capAboveEgo_kmh` (on). | **yes** |
| `core/warning.js` | `d_needed = d0 + v_c²/(2|a_min|)` and `TTC < TTC_warn` (3 [2–4] s), an on / off delay, and a sensor warning when the input stays stale or blind for `sensorWarnAfter_s` within `sensorWarnWindow_s` of a target. Additions explained in §4: `limitByLoop` (on: uses `min(|a_min|, aCmdMax)`), `leadBraking: 'estimate'` (option, off: the lead keeps braking at the estimated a_lead to a stop; closed-form `maxClosing`), `tDead_s` (option, 0). | **yes** |
| `core/baselines.js` | `idealAcc`: a constant-time-gap ACC that commands acceleration (gains, −3.5 / +2 m/s², jerk limit 2.5 m/s³, actuator lag 0.3 s; by default it sees the truth, optionally the same camera through the estimator). `human`: the Intelligent Driver Model as our driver, with correlated perception noise and a 1 s reaction delay. Flat SCHEMA (`acc_*`, `hum_*`); the engine passes `cfg.ego.params`. | Node 8 syntax |

Tests (mine): `tests/sensor | estimator | controller | warning | baselines | node8-pure | closedloop-sc2 .test.js`, plus helpers in `tests/stubs/sc2-*.js`. They speak SC1's runner format (`# pass N` / `# fail M`).

## 2. Test totals

- **SC2 branch alone: 91 passed, 0 failed** (closed-loop file skipped: no `core/sim.js` on this branch).
- **SC2 + SC1 `22b57af` in a scratch copy** (nothing of SC1's edited): **176 passed, 0 failed, 14 files**. 101 of these are mine, including the 10 closed-loop tests through the real engine and SC1's browser-path test with my files loaded.
- **The Node 8 proof** (`tests/node8-pure.test.js`, 13 tests):
  - Node's own bundled acorn (`node --expose-internals`, no dependency) parses all five files at `ecmaVersion: 2017`.
  - Twelve negative controls are rejected: `?.`, `??`, object spread, object rest, class fields, private fields, optional catch binding, `||=`, numeric separators, BigInt, regex lookbehind, `for await`. A positive control of Node 8 syntax is accepted.
  - No post-Node-8 built-in API is used (`.flat`, `Object.fromEntries`, `globalThis`, and others).
  - The three pure files name no clock, randomness, I/O, timer or `require`.
  - At run time they work in a `vm` context with `Date`, `Math.random` (made to throw), `console`, `process`, `require` and the timers removed. The context's globals are the same before and after, and the outputs are byte-identical between two instances and against the Node-loaded module.
  - Loaded as classic scripts into one context (as the page does), they add only `CS`.

## 3. Contract changes

Requested in `pub/NOTES.md`; **all adopted by SC1 in `14fd6d2`**:

1. `controller.step` input gets `curvature` (1/m at ego's x, + = left) and `yawRate`.
2. The engine sends when `out.send === true` (a change or a **resend**), else on a change of `vSet`.
3. `warning.step` input gets `aCmdMax = controller.rateDown_kmhps / 3.6`.
4. The series `meas.wrong` (from `meas._truth`); the freeze copy keeps my `seq`.

Conventions agreed with SC1:

- `x` is the front bumper; gap = `o.x − o.length − ego.x`.
- Lane + and lat + are to the left; curvature + is a left turn.
- The `world.faults` keys my sensor reads: `sensorBlind gapFrozen ghost sensorRangeCut latencyExtra dropBurst sensorDegraded`.
- The sensor's `preset` is **not** a SCHEMA key: SC1's `schema.withPreset` applies `sensor.PRESETS`, so a preset survives the defaults merge.

Additive output fields:

- **meas:** `seq`, `qual`, `_truth`.
- **est:** `resetReason`, `coasting`, `settled`, `sensorFault`, `reason`, `age`, `sinceCorr`, `n`.
- **controller output:** `send`, `y`, `target`, `reason`.
- **warning:** `raw`, `aUsed`.

No further contract change is needed.

## 4. Findings (assumed plant, assumed camera)

1. **The down rate limit caps the loop's deceleration, and the tuning table's range cannot meet the parent's S3 threshold.**
   - 3 km/h/s is 0.83 m/s². The table's whole sweep, 1–5 km/h/s, is 0.28–1.39 m/s². The cruise cannot slow faster than its set speed falls, so the `mild` and `firm` plants behaved exactly alike: the controller, not the plant, was the limit.
   - Lead 90 → 40 km/h, `firm` plant, current defaults, the driver reacting to warnings (cells: minimum time gap / warnings):

     | rateDown | lead −0.5 | −1 | −1.5 | −2 | −3 m/s² |
     | --- | --- | --- | --- | --- | --- |
     | 3 km/h/s | 1.91 / 0 | 0.63 / 1 | 0.90 / 1 | 0.84 / 1 | 0.67 / 1 |
     | 5 | 2.03 / 0 | 1.51 / 0 | 0.41 / 1 | 0.57 / 1 | 0.48 / 1 |
     | 8 | 2.05 / 0 | 1.85 / 0 | 1.41 / 0 | 0.62 / 1 | 0.44 / 1 |
     | 12 | 2.05 / 0 | 1.87 / 0 | 1.61 / 0 | 1.33 / 0 | 0.76 / 1 |

   - "Lead braking ≥ 1.5 m/s² without a warning" needs about 8 km/h/s; −2 m/s² needs about 12. SC3 should sweep `rateDown_kmhps` to 12. The SCHEMA range stays the parent's 1–5 until the owner widens it (see §5).
2. **The warning must know that limit, or it trusts braking the loop never asks for (fixed: `limitByLoop`, adopted by the engine).**
   - On `demo.approach` (120 → 80 km/h from 150 m), the parent formula with the mild or firm plant's `a_min` (−1.8 / −2.8 m/s²) gave **no warning** and a minimum time gap of 0.29 s.
   - With `min(|a_min|, aCmdMax)` (the engine now passes `aCmdMax`) the warning comes at 4.2 s, the driver brakes, and the minimum time gap is 1.35 s with a minimum gap of 23 m.
3. **The parent's `d_needed` assumes the lead holds its speed, so under sustained lead braking it warns late.**
   - The option `leadBraking: 'estimate'` predicts the lead braking to a stop.
   - Lead 90 → 40 km/h, `firm` plant, 3 km/h/s: the first warning moves from 3.5–10 s to 1.0–2.6 s after the lead starts braking.
   - With `kalmanCA` as well, the minimum time gap is **1.8–2.0 s at every level from −1 to −4 m/s²**. The parent formula gives 0.0–0.36, with collisions at −3 and −4.
   - Left **off** by default to keep the parent formula; my recommendation is to turn it on (§5).
4. **A resume brought the old set speed back, and the down rate limit made the cruise accelerate into a slower lead (fixed: `capAboveEgo_kmh`).**
   - The sequence: the driver taps the brake, SC1's driver model auto-resumes, and the cruise holds the old 90 km/h at 66 km/h behind a 50 km/h lead.
   - The rate limit then lowered 90 at 3 km/h/s while the cruise accelerated at 1.5 m/s²: a collision.
   - A set speed above our own speed only accelerates us, so it is now lowered at once to `max(target, v + 2 km/h)`.
   - The same applies to family I ("resume with a lead much closer than at suspension").
5. **Setpoint chatter.**
   - Steady following on the default camera: **34–42 changes per minute** with the parent's one-step hysteresis. The threshold is ≤ 20.
   - `hystSteps 1.5` gives 19, `2` gives 11, `3` gives 2 per minute. The gap error RMS stays about 1.1 m (the threshold 0.3·h·v is 15 m).
   - A longer τ does not help (33 per minute). A calmer estimator (q 0.3) helps a little (27 per minute).
   - Default left at 1 step (the parent's rule); 2 recommended.
6. **Sensor warnings need persistence (fixed: `sensorWarnAfter_s` = 1 s; burst defaults lowered to 0.0005 per sample, 0.5 s).**
   - At the first burst defaults (about one 1 s dropout every 50 s), "stale = warn" made dozens of nuisance warnings per hour. The parent's limit is ≤ 1 per hour.
   - The controller still holds the set speed immediately on stale input; only the warning waits.
   - **Dropout bursts per hour and their lengths** therefore drive the "warnings per hour" metric, and they are unknown on the car.
7. **Estimator: no variant meets the Phase 3 target (v_rel noise < 0.3 m/s and added delay < 0.3 s) on the assumed camera.**
   - Steady following; lag measured as the v_rel estimate's delay behind the truth while the lead brakes at −2 m/s²:

     | Variant | v_rel sd at 30 / 60 / 90 m | Lag |
     | --- | --- | --- |
     | kalman q=1 (default), camera v_rel (lag 0.5 s) | 0.18 / 0.26 / 0.34 | 1.05 s |
     | same, camera v_rel without lag | 0.18 / 0.26 / 0.34 | 0.78 s |
     | kalman q=4, camera v_rel without lag | 0.28 / 0.43 / 0.58 | 0.32 s |
     | kalman q=1, gap only | 0.34 / 0.49 / 0.59 | 1.30 s |
     | kalmanCA aSd=1, camera v_rel (lag 0.5) | 0.28 / 0.43 / 0.60 | 0.55 s |
     | alphaBeta 0.5 / 0.05, camera v_rel | 0.26 / 0.69 / 1.37 | 0.79 s |

   - The 2-state Kalman (constant relative speed) lags a steadily braking lead by construction.
   - `kalmanCA` removes most of the filter's share, so what remains is mostly the camera's own speed lag (assumed 0.5 s).
   - The deciding numbers are the camera's **internal v_rel lag and noise**, and whether `speedX` is a relative speed at all.
8. **SC1's driver model (reported to SC1).**
   - It reacts once per `warn.on`: it brakes for 1 s, releases, and auto-resumes into a warning that is still on, with no second reaction.
   - This caused the one collision left in the lead-braking table (2-state Kalman, −1.5 m/s²). It is not a defect of the loop, but it affects every "driver reacts" score.

## 5. Open decisions for the owner

1. **Widen the tuning table's down rate limit** (now 1–5 km/h/s) to about 12 km/h/s? Inside 1–5, S3 (≥ 1.5 m/s²) cannot pass whatever the plant (finding 1). The comfort cost of 8–12 km/h/s is a cruise braking at 2.2–3.3 m/s² on command.
2. **Make `leadBraking: 'estimate'` and `kalmanCA` the defaults?** (Finding 3: from minimum time gaps of 0.1–0.4 s to about 2 s, with the same driver model.) Today the defaults are the parent's.
3. **Hysteresis 2 steps instead of 1?** (Finding 5: the setpoint-change threshold is met only then.)
4. **The identical-values freeze check** (`frozenSameN`) is off. It catches a frozen payload whose counter still advances (`repeatNewSeq`), but a smooth, quantised camera output can honestly repeat. Decide after the first logs.
5. **My additions that are on by default:**
   - `limitByLoop` (the warning trusts only the deceleration the loop can ask for);
   - `capAboveEgo_kmh` 2 (the resume fix);
   - `sensorWarnAfter_s` 1 s;
   - `resendAfter_s` 1 s.

   Each can be set back to the parent's behaviour.

## 6. Left out, and why

- **Lateral dynamics of our own car and lane keeping.** The sensor takes lane and lat from the world as given. The yaw rate is v·κ of the road, not a measured value with its own noise.
- **An occlusion model** (a truck ahead of the lead hiding it). Not asked for; the camera sees every car in range and field of view.
- **The camera's object list** (`keep-dist`, up to six objects) as a second source. The sensor gives the single front-vehicle message the car's `DataFrontVehicle` carries.
- **Identifying the estimator's parameters from logs.** That needs the drive; the estimator's own noise model (`rGap*`, `rVrel*`) is a setting that matches the sensor defaults.

## 7. How to check

- `node tools/cruise-sim/test.js` on the merged tree runs everything. The closed-loop file prints the steady-following chatter with the default hysteresis.
- `node tools/cruise-sim/tests/estimator.test.js` prints the estimator characterisation table for the default camera.
- On the page: the sensor and estimator settings appear from `CS.sensor.SCHEMA` / `CS.estimator.SCHEMA`. With `sensor.curvePredErr` at 0.9 on a curved road, the lead is lost in the curve and the set speed stays flat until the curve ends (`meas.wrong` marks a next-lane target).

## Підсумок українською (для власника)

Усе нижче — на **припущених** моделях круїзу й камери, це не вердикт.

**Що зроблено.** Готові модель камери, естиматор (Kalman, Kalman з прискоренням лідера, alpha-beta), закон уставки швидкості, попередження і два базові варіанти для порівняння (ідеальний ACC і водій-людина за IDM). Естиматор, контролер і попередження написані синтаксисом Node 8 і чисті (без годинника, випадковості та вводу-виводу); тест це доводить. Разом із рушієм SC1 проходять 176 тестів.

**Головні знахідки:**

1. **Обмеження швидкості зниження уставки визначає, наскільки сильно система може гальмувати.** 3 км/год/с — це лише 0,83 м/с², і вся таблиця налаштувань (1–5 км/год/с) не досягає порогу «лідер гальмує на 1,5 м/с² без попередження». Для цього треба близько 8 км/год/с.
2. **Попередження за формулою з брифу спрацьовує запізно.** Воно розраховує на гальмування круїзу, якого контур ніколи не запитує, і вважає, що лідер тримає швидкість. Виправлення (врахувати ліміт контуру й прогнозувати гальмування лідера) разом з Kalman-CA тримають мінімальний часовий інтервал близько 2 с при гальмуванні лідера від −1 до −4 м/с².
3. **Після відновлення круїзу стара уставка розганяла авто на повільнішого лідера.** Виправлено: уставка вище за нашу швидкість одразу знижується до неї.
4. **Гістерезис в 1 крок дає 34–42 зміни уставки за хвилину** (поріг — 20). З гістерезисом у 2 кроки — 11.

**Що виміряти на авто насамперед:** затримку й шум відносної швидкості камери, частоту та тривалість пропусків повідомлень і реальну межу гальмування круїзу.
