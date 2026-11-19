# Brief: the cruise set-speed simulator — an interactive tool with graphics, settings and the full scenario catalogue

Audience: two agents working in parallel, **SC** (the core: models, scenarios, batch runs) and **SU** (the page: animation, charts, settings), or one agent doing both in that order. The parent document is `docs/cruise-set-speed-brief.md` (read it first: the concept, the phases, the thresholds, what is known about the car). This brief is its Phase 5, made buildable now. **Not started: the owner asked for briefs only (2026-11-19).** The lead merges and pushes; you never push and never connect to the car.

## 0. Why build it before there is any car data

No drive has been recorded yet, so the cruise's real behaviour (does it brake, how late, how hard) is unknown. The simulator therefore has two jobs, in this order:

1. **Now, with assumed plants:** show *which* plant, sensor and command-path numbers the concept needs. The main result is a **feasibility map** ("it works if the cruise brakes at least X m/s² with a dead time under Y s and the gap arrives at Z Hz"), and from it a ranked list of what the first drive must measure.
2. **Later, with identified plants and recorded traces:** the verdict of the parent brief (go / conditional / no-go). Loading a measured plant and a logged drive must need no code change: a JSON file each.

Everything the tool shows while the plant is assumed says so on screen ("assumed plant: <preset>"), in every export and in every report. A result from an assumed plant is never worded as a verdict.

The whole thing is offline. Nothing here reads from or writes to a car.

## 1. Shape of the tool

- **Language: JavaScript, no build step, no dependency.** The core runs under Node (the Mac has Node 22) for batch runs and in the browser for the page, from the same files. Reason: the estimator and the set-speed logic are the code that would one day run on the head unit (Node 8.16), so they are written once. **`estimator.js` and `controller.js` are Node 8.16 syntax, pure functions of their inputs and state, no clock, no I/O, no randomness of their own.** (The parent brief said Python "unless decided otherwise"; this is the decision, for that reason. Plant identification from logs can still be Python later: it hands over a JSON.)
- **Place:** `tools/cruise-sim/` (tracked: code, scenario definitions, small fixtures; no pictures, no result dumps — the Store refuses a repository archive over 64 MiB). Results and reports go to `bake/work/cruise-sim/` (git-ignored).
- **Deterministic:** every random draw comes from one seeded generator in the run's config; the same config gives the same bytes on Node and in the browser. A run is fully described by one JSON object, which the page can export, import and put in the URL.
- **Time:** vehicle dynamics at a fixed 10 ms step; the controller at its own rate (5–10 Hz, a setting); sensors and the command path on their own clocks with their own jitter.

### 1.1 The contract between the core and the page

```
run(config) -> {
  meta:    { config, seed, plant: { name, assumed: true|false }, version },
  t:       Float64Array,                      // seconds, the 10 ms grid (or a decimated one, config.out.dt)
  series:  { name -> Float64Array },          // see 1.2
  events:  [ { t, kind, ... } ],              // commands sent / applied / dropped, target changes, warnings, driver actions, failures
  metrics: { name -> number | boolean },      // section 5
  verdict: { pass: boolean, failed: [metric names] }
}
step API: createSim(config) -> { step(), state(), done }   // for the animated page: the same model, advanced in real time
```

SC owns this contract and publishes it in the first hour (`tools/cruise-sim/CONTRACT.md`) with one working scenario, so SU builds against real output from the start. Changes to it are announced in `bake/work/cruise-sim/pub/NOTES.md`.

### 1.2 Series every run carries

Truth: ego position, speed, acceleration; each other vehicle's position, speed, acceleration, lane; the true gap and true relative speed to the true lead; road grade. Measured: gap, relative speed, target id, validity, as the sensor model delivers them (with their sample times). Estimated: gap, relative speed, lead speed, lead acceleration. Control: the desired gap, `v_ref`, the set speed commanded, the set speed the cruise holds, the driver's maximum, the cruise state. Safety: time gap, TTC, the warning flag and its reason, the gap that would be needed. Driver: pedals, overrides.

## 2. Models (SC)

Each model is a small module with a parameter object, a documented default, and its ranges for the settings page. Every parameter has a unit in its name or its description.

### 2.1 The stock cruise (the plant) — several, selectable

- **P1** first-order lag with dead time and acceleration limits (the parent brief's starting model).
- **P2** the same with separate parameters for speeding up and slowing down.
- **P3** second-order (for an overshoot the first-order cannot show).
- **Braking authority as a mode of each:** *lift-off only* (deceleration = drag + engine braking, speed- and gear-dependent, about −0.3 to −1 m/s²), *mild braking* (down to about −1.5 … −2), *firm braking* (down to about −2.5 … −3). These three are the **presets** the first results are reported for.
- Around all of them: aerodynamic and rolling drag, road grade (the cruise fights it with finite authority: on a downhill a lift-off-only cruise overspeeds), a step-size nonlinearity option (large setpoint steps answered harder), setpoint resolution (1 km/h, optional 5), a maximum accepted change rate, the minimum and maximum active speed, **dropping out below the minimum speed** (the cruise switches off and the car coasts: what then?), suspend on the driver's brake, resume, override by the accelerator, and the cruise's own behaviour when the set speed is far below the current speed.
- A plant is loadable from JSON (the future identified model) and marked `assumed: false` only then.

### 2.2 Other vehicles

- Scripted profiles: piecewise speed or acceleration, sine, steps, ramps, stop, stop-and-go.
- A human-like driver model for leads and for the car behind us (the Intelligent Driver Model with noise and a reaction delay), so a platoon of several cars ahead produces realistic waves and a follower behind shows what our braking does to them.
- Lane changes as events (cut-in, cut-out) with a lateral duration, so the sensor model can acquire the new target gradually.
- Replay: a recorded lead-speed trace from a JSON or CSV file (later: the logged drives; now: a few synthetic "recorded-like" traces as fixtures).

### 2.3 The sensor (a camera, not a radar)

Range limit and a noise that **grows with distance** (camera range error is roughly proportional to distance squared; make the law a setting), a separate noise on relative speed or none at all (the speed may have to be estimated), quantisation, a native sample rate (2–25 Hz, a setting), latency with jitter, random dropouts and bursts, outliers, a frozen value (the last sample repeated), target-id behaviour on cut-in / cut-out (acquisition delay, a transition where the gap is a blend or jumps), loss on curves by curvature and range, a wrong target (the car in the next lane on a curve) for a while, a ghost target, degraded modes (rain, low sun: shorter range, more dropouts), and the "no target" value.

### 2.4 The estimator, the controller, the warning

Exactly the parent brief's Phases 3 and 4: an alpha-beta and a Kalman variant with re-initialisation on target change; the set-speed law with its optional feed-forward term, the rate limits, quantisation with hysteresis, the driver's maximum and the cruise minimum; the warning by needed distance and by TTC. Every number of the parent's tuning table is a setting. Add what the failure-mode table of the parent brief demands: staleness checks on every input, the hold-then-release behaviour after target loss, no acceleration into a curve on a lost target.

Two **baselines** run on the same scenario for comparison: (a) an ideal adaptive cruise that commands acceleration directly (what a real ACC could do), (b) the human-like driver model. They answer "how much is lost by acting through the set speed only".

### 2.5 The command path

Delay, jitter, a maximum accepted rate, a drop probability, a silent rejection, duplicated or out-of-order delivery, "accepted but applied late", and the head unit's pauses (a stall of 0.2–2 s at random). All settings; all unknown on the car, so all swept.

### 2.6 The driver

Sets and changes the maximum speed; overrides with the accelerator; brakes (the cruise suspends) and resumes; and **reacts to a warning**: a reaction-time distribution (for example 0.8–2.5 s) and a braking level (4–8 m/s²). The reaction model is what turns "a warning was given" into "was it given early enough": every scenario that ends in a warning is scored with the driver reacting, with the driver reacting late, and with the driver not reacting.

### 2.7 Failures, injectable at any time or at the worst time

The app stops (commands stop, the warning stops); the head unit restarts (30–60 s of nothing); the gap freezes; the sensor goes blind; a command is lost exactly when a decrease was needed; the set speed sticks; the cruise drops out by itself. "At the worst time" means a search over the injection time for the worst outcome, not one hand-picked moment.

## 3. The scenario catalogue (SC) — as complete as it can be made

Scenarios are **data**: a file each in `tools/cruise-sim/scenarios/`, with a name, a description, the cars and events, the parameter grid it is swept over, and which metrics decide it. The list below is the minimum; add what you find missing and say so. Each family is run at ego speeds 40, 60, 90, 120 and 150 km/h where it makes sense, on a flat road and on grades, in clear and degraded sensing.

| Family | Scenarios |
| --- | --- |
| **A. Free driving** | No target at steady speed · set-speed steps ±1, ±5, ±10, ±20, ±40 km/h · uphill, downhill 3 / 6 / 10 % · crest and dip · the driver changes the maximum while free |
| **B. Approach** | A slower lead from far, speed difference 10 … 80 km/h · a lead at the cruise's minimum speed · a lead slower than the minimum · **a standing obstacle / the tail of a motorway queue at 90, 120, 150 km/h** (out of the concept's reach: scored on the warning only) · approach on a downhill · approach in rain mode |
| **C. Following** | Steady, at each time gap · the lead's speed oscillates (amplitude 2 … 15 km/h, period 5 … 60 s: the string-stability sweep) · a human-like lead (noise) · a platoon of 3–8 human-like cars ahead with a braking wave running back through it · the lead accelerates away · the lead drifts slowly · long following (30 min) for chatter and drift · a car behind us following closely |
| **D. Lead braking** | To a lower speed at −0.5, −1, −1.5, −2, −3, −4, −6 m/s² · to a standstill · a brake tap and release · braking that starts while we are still closing · braking on a downhill · two brake events in a row |
| **E. Cut-in** | Gap 5 … 60 m, the cutting car slower, equal, faster · cut-in then brake · two cut-ins in a row · a cut-in during our acceleration back to the maximum · a motorcycle (narrow, acquired late) · a truck (acquired early) |
| **F. Cut-out** | Revealing a slower car (20 … 80 km/h slower) · **revealing a standing car** · revealing a faster car (do we accelerate too eagerly?) · the lead leaves on an exit ramp while slowing |
| **G. Sensing faults** | Target lost on a curve for 1 … 10 s, then back · lost and replaced by the next-lane car · a ghost target for 0.5 … 3 s · frozen gap · dropout bursts · outliers · latency spikes · low native rate (2 Hz) · range cut to 40 m · the lead beyond range, then entering it at a large speed difference |
| **H. Road and envelope** | 30 km/h zones (near the cruise minimum) · 150 … 180 km/h · a speed-limit change (the driver lowers the maximum behind a faster lead) · merging traffic at an on-ramp · a roundabout or junction approach behind a lead (the lead stops: warning only) · stop-and-go traffic (out of scope: how does the system leave its envelope, and does it say so?) |
| **I. Driver** | Accelerator override while following · brake and resume · resume with a lead much closer than at suspension · the maximum raised by 30 km/h behind a slow lead · cancel · every warning scenario with the three reaction cases of §2.6 |
| **J. System failures** | Every failure of §2.7 in families B–F, at the worst time · the command path at its worst settings · combined: a failure during degraded sensing |
| **K. Replay** | Recorded lead traces (fixtures now, the car's logs later) · the same traces with a cut-in spliced in |
| **L. Mixed driving** | A generator that strings scenarios of the families above into hours of driving with realistic frequencies (settings: share of motorway / road / town, traffic density), for the "warnings per hour" and "driver interventions per hour" numbers |

## 4. The page (SU): graphics and settings

One static page, `tools/cruise-sim/ui/index.html`, opened from the disk or a tiny local server; no library from the network (it must work offline); the charts and the scene are drawn on canvas by our own small code. It works at laptop width; a phone is not a target.

### 4.1 The scene (animated)

A road seen from the side or from above, scrolling with our car: our car, the lead, every other vehicle, lanes and lane changes, the grade as a slope. Drawn on it: the measured gap and the true gap (so the sensor's error is visible), the desired gap as a zone, the needed distance of the warning, the target the sensor currently holds (and a mark when it is the wrong one or none), the warning state, the driver's actions, a failure in progress. A readout beside it: speed, set speed (commanded and held), maximum, time gap, TTC, the cruise state. Play, pause, step, speed ×0.25 … ×16, a scrubber over the whole run.

Optional, if cheap: a small preview of the cluster picture of `docs/AGENT-BRIEF-front-vehicle.md` (near / middle / far level and "closing") fed from the simulated sensor, and an export of the simulated gap events in the shape the Twin Dial bench can replay. Do not build the cluster graphic here; only the hook.

### 4.2 Time charts (linked to the scrubber, zoomable)

Speeds (ours, the lead's, commanded and held set speed, the maximum, the estimate of the lead) · gap (true, measured, estimated, desired, needed) · time gap and TTC with their thresholds · accelerations (ours, the lead's, the limits of the plant) · the event lane (commands sent / applied / dropped, target changes, warnings, driver, failures). Any two runs can be overlaid (A / B): the same scenario with two parameter sets, or our controller against a baseline.

### 4.3 Settings

Grouped as the models are: scenario (family, scenario, its variations, the seed) · plant (the preset or a loaded file, then every parameter) · sensor · estimator · controller and warning · command path · driver · failures. Every control shows its unit, its default and its range; a changed value is marked; "reset group"; presets can be saved, named, exported and imported as JSON; the whole state lives in the URL so a run can be sent as a link. Changing a setting re-runs at once when the run is short, otherwise on a button.

### 4.4 Batch views

- **The scenario matrix:** every scenario × its variations as a grid of pass / fail cells for the current parameter set, with the deciding metric in each cell; a click opens that run in the scene.
- **The feasibility map:** a heat map over any two parameters (for example the plant's maximum deceleration × its dead time; the command delay × the sensor rate), each cell the share of the catalogue that passes or a chosen metric; the three plant presets marked on it. This is the tool's main picture while the plant is assumed.
- **Sensitivity:** for the current set, which parameter moved by ±30 % changes the verdict (a tornado chart).
- **Tuning sweep:** the parent brief's tuning table (standstill gap, time gap, gap-closing time, rate limits, warning TTC) swept, best sets listed with what they cost in comfort.
- Long batches run in the core under Node (`node tools/cruise-sim/sweep.js <plan.json>`) and the page loads the result file; short ones may run in a worker in the page.

### 4.5 The report

A button, and a Node command, that write one self-contained HTML file into `bake/work/cruise-sim/reports/`: the configuration, the plant and whether it is assumed, the scenario matrix, the feasibility maps, the worst cases with their charts, the metrics against the thresholds, and the list "measure this first on the car, because the verdict turns on it".

## 5. Metrics (SC)

Per run: minimum gap, minimum time gap, minimum TTC; collision (yes / no, and the impact speed) for each driver reaction case; warnings (count, reason, **lead time**: how long before the last moment a braking driver could still avoid contact), missed warnings (a contact or a near miss without one) and nuisance warnings (a warning the cruise would have handled); the driver's interventions needed; gap error RMS in steady following; the oscillation gain in family C (output amplitude over input amplitude, per period: above 1 means it amplifies waves); our peak deceleration and jerk, acceleration RMS (comfort); what we do to the car behind (its minimum time gap and peak deceleration); setpoint changes per minute and whether the command rate limit was hit; time lost against the ideal adaptive cruise baseline; time spent outside the envelope.

Per catalogue: the parent brief's pass thresholds, each a setting (they are proposals), the pass share per family, the per-hour numbers from family L, and robustness (all of it again with ±30 % on the plant and at the command path's worst settings).

## 6. Tests

- Analytic checks of every model: a first-order step response against its formula; a stopping distance; TTC of two constant speeds; the dead time visible to the sample; the rate limiter; quantisation with hysteresis.
- Determinism: the same config gives the same result on Node and in the browser build of the core (a hash of the series).
- Every scenario file loads, runs and finishes; a snapshot of the metrics of a fixed reference set guards against silent changes.
- `estimator.js` and `controller.js` parse under Node 8 syntax rules and touch nothing global.
- The failure injections do what they say (the command really stops; the gap really freezes).
- Runner: `node tools/cruise-sim/test.js`, wired into `tools/twin-dial/run_all_tests.sh` only if it stays under a minute.

## 7. Rules

- Worktrees `<repo>/.claude/worktrees/cs-core` (branch `cruise-sim/core`) and `.../cs-ui` (branch `cruise-sim/ui`) from `main`; symlink `reference`, `bake/work` and `docs` from the main checkout. Never `git add` under `docs/`, `reference/`, `bake/work/`. No big tracked files.
- Offline only: no car, no network fetch at run time, no dependency download. If you want a public driving dataset for family K, propose it in the report with its licence; do not download it.
- No setter, no method, no car code is touched: this round changes nothing under `apps/` or `shared/`.
- Local server ports 8980–8989 only if a server is needed at all (8080 / 8150 are the owner's). You cannot `kill` processes; no browser automation: the page's logic is tested through the core, and you describe how to check the page by eye.
- Interface text in English; the report's summary also in Ukrainian for the owner.
- Hand-in: the code; `tools/cruise-sim/README.md` (how to open it, how to run a sweep, how to load a plant and a trace); the first report on the three assumed plant presets; `docs/cruise-sim/SC-REPORT.md` and `SU-REPORT.md` (through the symlink, not committed) with what was built, what was left out and why, and the owner's decisions; `READY` in `bake/work/cruise-sim/pub/<ID>/`. Final message: branch, last commit, test totals, the headline of the first feasibility map (worded as "with an assumed plant"), and the ranked list of what the first drive must measure.
