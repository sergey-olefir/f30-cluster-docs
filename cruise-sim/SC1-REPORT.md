# SC1 report: cruise simulator, engine and world

2026-11-19 · agent SC1 · branch `cruise-sim/sc1` (worktree `.claude/worktrees/cs-sc1`) · head `22b57af` · 11 commits on `main` (9517e89)

## Короткий підсумок (українською)

Рушій симулятора готовий і перевірений: `core/sim.js` (`run`, `createSim`), модель штатного круїзу (P1, P2, P3; пресети
lift-off / mild / firm; опір повітря, кочення, ухил; квантування уставки, вимкнення нижче мінімальної швидкості,
призупинення гальмом, відновлення, перевизначення педаллю газу), інші машини (сценарні профілі, IDM із шумом і затримкою
реакції, відтворення записаних треків, перестроювання), водій із моделлю реакції на попередження, канал команд (затримка,
джитер, ліміт частоти, втрати, тиха відмова, дублі, перестановка, зависання головного пристрою) і інжекція відмов, зокрема
`at: 'worst'` як пошук найгіршого моменту. 75 тестів SC1 проходять (2 с); разом із модулями SC2 — 169. Усі 154 файли
сценаріїв SC3 і всі 4358 їхніх варіантів проганяються на рушії без жодної помилки (26 с). Результат детермінований і
однаковий у Node та у браузерному завантаженні (перевірено хешем). Усі числа круїзу — **припущені**, доки не буде
ідентифікованої моделі з логів машини; результат так і позначений (`meta.plant.assumed`).

## What was built (all under `tools/cruise-sim/`)

| File | What |
| --- | --- |
| `core/rng.js` | sfc32 seeded from a hash of a key; `fork(label)` = an independent stream keyed by root seed + label path (a new draw in one module never shifts another's); `next normal int pick uniform chance lognormal exp` |
| `core/schema.js` | `defaults`, `merge` (unknown keys throw), `validate` (range, type, enum, null, unknown), `check`, `withPreset`, `KMH` |
| `core/sim.js` | `run(config, deps)`, `createSim(config, deps)` (step / advance / state / result), recording on `out.dt`, events, exact 10 ms core metrics (`meta.core`), collisions (ahead / behind / side), `defaultConfig()`, `schemas()`, `hashResult()`, `LOAD_ORDER`, built-in scenarios `demo.approach` / `demo.free`, stand-ins (`STUBS`) for every SC2 / SC3 module until it exists |
| `core/plant.js` | the stock cruise: P1 (exact first-order + dead time), P2 (separate up / down τ and Td), P3 (second order, overshoots); demand clipped to [aMin, aMax] and an optional jerk limit, then realised through the powertrain (engine braking speed/gear-dependent, brake authority `brakeMax_ms2`, power / traction, drag, rolling, grade); presets liftoff / mild / firm; setpoint resolution, active range, accepted-change interval; states off / active / suspended / override / dropout; far-below behaviour; identified plant from a JSON file or object → `assumed: false` |
| `core/vehicles.js` | profiles const / piecewise / accel / steps / sine / stopgo / idm / replay / static; IDM with OU acceleration noise and a perception delay; lane changes (cut-in, cut-out, any lane, our own) with a cosine lateral move, `gap_m` / `v_kmh` / `dv_kmh` placement, `whenGap_m` trigger; trace parser (JSON / CSV); `gap`, `trueLead`, `follower`, `leaderOf` |
| `core/road.js` | piecewise-linear grade and curvature, exact elevation, lanes, speed limits |
| `core/driver.js` | scripted maximum / accelerator / brake / cancel / resume; reaction to the warning (nominal lognormal, late, none), braking level and build-up, release at the lead's speed, auto-resume (by hand below the cruise minimum first), optional speed-limit following and unwarned braking by TTC |
| `core/command.js` | delay, jitter, max accepted rate, drop, silent rejection, duplicate, reorder, applied-late, random head-unit stalls; preset `worst`; fixed draws per message so one probability never shifts the others |
| `core/failures.js` | appStop, huRestart (30–60 s, fresh app), gapFreeze, sensorBlind, commandLost (next decrease / all), setStuck, cruiseDropout, stall, and the sensor faults SC2's sensor reads (ghost, rangeCut, latencySpike, dropBurst, degraded); `searchWorst`: coarse 1 s pass + 0.2 s refinement, by collision / impact speed / smallest gap or any named metric |
| `test.js`, `tests/harness.js`, `tests/stubs/sc2.js` | runner (each file in its own process, `# pass` / `# fail` counts), a tiny harness, pinned stand-ins so engine tests keep their numbers when real modules land |
| `tests/*.test.js` | basics (rng, schema, road) 9 · plant 15 · vehicles 10 · command 7 · driver 8 · failures 10 · sim 16 = **75** |
| `README.md`, `CONTRACT.md` | how to run, load a plant and a trace, what is where; the contract kept current |

### The analytic and behavioural checks (brief §6, SC1's part)

- First-order step response equals `v_set + (v0 − v_set) e^(−(t−Td)/τ)` at every 10 ms sample (error < 1e-9); P2 with its
  own down parameters likewise; P3's overshoot matches `exp(−ζπ/√(1−ζ²))` within 0.03.
- Dead time visible to the sample: the plant alone (first non-zero acceleration at exactly 1.37 s for Td 0.37), and the
  whole path (command delay 0.2 + plant Td 0.5 → exactly 2.70 s for a command at 2.00 s).
- Stopping distance v²/2a within one step; TTC of two constant speeds = gap / closing speed at every sample; IDM settles at
  the analytic equilibrium gap `(s0 + vT)/√(1 − (v/v0)^δ)`; road elevation is the exact integral of the grade.
- Lift-off deceleration equals engine braking + drag + rolling; a lift-off cruise overspeeds on −8 %, a mild one holds it.
- Each failure does what it says (no command sent while the app is dead; the measured gap constant while the true one
  moves; exactly one decrease lost; the held set speed constant under setStuck; …).
- Determinism: same config → same bytes (hash of t, series, events, metrics); `meta.config` reproduces a run; another
  seed differs; `createSim` stepped in odd chunks equals `run()`.
- **The browser path**: every `core/*.js` loaded as a classic script into one shared `vm` context with no `require`
  (as the page does), in `LOAD_ORDER`; the result hash equals Node's, with the stand-ins and with the real SC2 modules.

### Integration checks (in a scratch copy, nobody's file edited)

- `cruise-sim/sc2` @ 9807bd5 + `cruise-sim/sc3` @ b5852fa on top of sc1: all 14 test files pass, **169 tests**; no stubs
  left in `meta.modules`.
- **All 154 scenario files and all 4358 variants** of SC3's catalogue run on the engine: 0 errors, 0 NaN, no collision in
  the first 0.5 s (a set-up error would show there); 26 s for all. 1836 of 4358 variants end in a collision (J 1034, E 202,
  L 153, D 121, G 81, F 79, B 65, H 59, K 22, I 12, C 8) — counted, not judged by SC1; most are runs where the driver does
  not react or reacts late, by design of the catalogue.
- Speed: a 90 s scenario ≈ 11 ms with stand-ins, ≈ 15 ms with SC2's modules; one hour of driving ≈ 0.3–0.5 s.

## Contract changes (all additive except the tick order; announced in `bake/work/cruise-sim/pub/NOTES.md`, written into `tools/cruise-sim/CONTRACT.md`)

1. **Tick order:** the others move after the plant (with ego), so sensor, driver and plant see one instant at t. Only the
   order of `vehicles.step` changed.
2. **Every core file is wrapped in an IIFE**, and the page loads them in `sim.LOAD_ORDER`. Without it the browser rejects
   the second file that declares a top-level name (found by SC1's and SU1's tests independently; SC2 and SC3 adopted it).
3. `create(params, rng, ctx)`; `run / createSim(config, deps)`; stand-ins and `meta.modules`.
4. Plant API `request(t, vSet, faults)` + `step(dt, { t, grade, driver, faults }, ego)`; `world.cruise.setEff`.
5. Geometry: x = front bumper, `gap(behind, ahead)`, `y`, lane + = left; `xFront_m` to place a car alongside.
6. Metrics: `metrics.compute(result, config)` then `metrics.verdict(metrics, thresholds, decide)` (SC3's API); SC1's exact
   numbers in `meta.core`.
7. Added series (`set.eff meas.id meas.wrong est.id warn.reason warn.ttc app.alive curvature elevation follower.v a.demand
   ego.lane ego.y veh.<id>.*`), meta (`vehicles ego road params enums core worst …`), events (`vehicle.lane driver.cancel`,
   `collision.from`), failure kinds, scenario events (`driverResume driverCancel`, `laneChange` of `'ego'`).
8. For SC2: controller input `curvature`, `yawRate`; output `send` (resend / suppress); warning input `aCmdMax`.
9. Controller clock: due at 0, 1/rate, …, on the first tick at or after (exact mean rate); the state at t = duration is
   recorded.

## Left out, and why

- **Wiring `test.js` into `tools/twin-dial/run_all_tests.sh`**: it would qualify (≈ 2 s alone, ≈ 4 s with SC2's tests), but
  that script is not SC1's file. One line for the lead to add after the merge.
- **Determinism outside V8**: `normal()` and the plant use `Math.log` / `Math.exp`; Node and Chrome agree bit for bit
  (tested). Firefox / Safari may differ in the last bit, so a hash taken there may not match; the physics would not.
- **A lateral model of our car in curves, wind, tyre slip, gear shifts**: out of scope; engine braking is a smooth
  speed-dependent value standing in for gears.
- Two intermediate commits (f8b4439, aab88d3) carried a failing new plant test (fixed in 503c828); the head is green.

## Open decisions for the owner / lead

1. **The assumed plant numbers.** Dead time 0.5 s, τ 2.0 s, aMax 1.5 m/s² for all presets; lift-off = no brake, demand
   limit −0.7 (physics gives about −0.5 … −0.8 on the flat, almost nothing on a 6 % downhill); mild = brake 2 / −1.8; firm
   = brake 3.5 / −2.8. They decide every assumed result; the feasibility map sweeps them anyway.
2. **What a brake press does to DCC**: modelled as `suspended` with the set speed remembered (resume restores it), and a
   set-speed write while suspended is rejected (`acceptWhenSuspended: false`). Both unknown on the car.
3. **The cruise's own cut-off**: drops out below 25 km/h, set range 30–210 km/h. To be read on the drive.
4. **The driver model's defaults**: brakes only if actually closing when he reacts; 6 m/s² held until at the lead's speed;
   resumes 3 s later; relies on the warning alone (`selfBrakeTTC_s: 0`), which makes "missed warning" visible but is
   pessimistic for the per-hour numbers of family L.
5. **A collision ends the run** (`stopOnCollision: true`), so metrics after a contact do not exist.
6. **`at: 'worst'`**: one worst failure per run; 1 s then 0.2 s; objective collision → impact speed → smallest true gap.
   Searching two failure times jointly was left out (cost grows with the square).
7. **SC3's E.cutin.faster** (x0 formula crossing 0) and E.cutin.slower at 5 m / 30 km/h end in side-swipes; reported to
   SC3 in NOTES; their call.
