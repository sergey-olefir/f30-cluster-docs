# CT: CarPlay's next turn as data (GuidanceEventList.queryContent)

Package CT of `docs/AGENT-BRIEF-carplay-turns.md`. Branch `car-issues/ct` from `main` at `9517e89`, worktree `.claude/worktrees/ci-ct`.
Three commits. No version bump (Signal Probe 0.3.0, Twin Dial 0.1.4), no push, no car.
Evidence labels: **proven** = a test, the catalogue or a log shows it; **hypothesis** = not yet confirmed by the car.

## 0. In short

1. **One small module makes the one call**: `shared/carplay_turns.js`. It builds the request from the catalogue's types, calls
   `queryContent` through the kit's `signals.proxy` / `signals.call`, decodes the reply, picks the next turn, and runs the poll
   (only while wanted, one call in flight, every 2 s, timeout 3 s, every 10 s after five failures in a row). It logs the first
   call's outcome (answered / the error text / timeout) and every change of outcome, and never throws. Both apps pack it
   (`app.json "shared"`); it is the only file that makes the call (tests: one call site; of `shared/*.js` the only one that names
   the interface; the probe's own files make no call).
2. **Signal Probe S16 "Наступний поворот"** has a new entry kind, a polled method (flags `m`, declared `_methods`): `next-turn`.
   It asks only while S16 is on and `Guidance.guidingProvider` is `CAR_PLAY` / `ANDROID_AUTO`. A line only when the reply changed,
   with the count of unchanged replies. S16 now has 23 entries; 40 fields are on with S16 alone. App total 251 entries.
3. **Twin Dial**: Діагностика **"Повороти CarPlay (тест)"** (`diagCarplayTurns` -> `config.carplayTurns`, off by default). On, with
   the phone guiding: tab 5 shows the existing arrow page (`guide`, the one `tdpage` layer) fed from the event list: the arrow, the
   phone's own distance and unit, the road the turn names, the destination footer. The CarPlay route page stays the fallback when
   the call fails, finds no turn, or its turn is older than 6 s. Off: no proxy, no call, nothing changes (tested).
4. **Declarations added** (both apps, the same): `"de.bmw.infotainment.navigation.guiding.GuidanceEventList": { "_instances":
   "Main_Instance", "_methods": ["queryContent"] }` (Signal Probe keeps its S16 field `registeredViews` and event `viewsChanged`
   there too). `check_signals.py`: **all declarations pass** for both apps (`method queryContent: methodId 4`, instance
   `Main_Instance` = 128).
5. **Whether the car's ACL lets an OAP app call it is unknown.** Precedents (not ours): f30_cluster 0.7.18 declared
   `_methods ["requestDefaultSymbols"]` and installed, so a `_methods` declaration installs on this car (**proven** for the install);
   the public MGU app "Redline" (cauliflowermossgrape/idrive7-apps) called `queryContent` through the same OAP binding on a G21 with
   CarPlay and logged `NO_ERROR` replies with the phone's turns (2026-11-09, **proven on a G21, a hypothesis for this car**). Its
   reply log gave the shapes our bench fixtures copy (not its code).

## 1. What was built

### 1.1 `shared/carplay_turns.js`

- `request()`: `{ viewConfigs: [ { viewport: { startPoint: { anchorPoint: { standardReferencePoint: 'CCP' }, offsets: [] },
  maxElementCount: 4, endPoints: [], includedTypes: [INCLUDE_ACTIVE_MANEUVERS, INCLUDE_DESTINATIONS, INCLUDE_TUNNELS,
  INCLUDE_ALONG_MAIN_ROAD] }, options: [], minimumUpdateIntervalMs: 0, segmentItemRasterLength: 0 } ] }`. Checked against the
  catalogue by a test: every struct member is given, the anchor union has one member, every enum value is an item. Note: this car's
  catalogue has no `includedActiveTypes` (Turn View's iDrive 8 schema does); the request follows our catalogue.
- `decode(reply)`: `GuidanceEventListError`, `viewContent[0].eventsInView` (the first 4), `currentDistances`. A union arrives keyed
  by its member (`{ intersection: {...} }`, the binding's shape) or as `{ type, value }`; enums by name or number; a UInt64 `uid` as a
  digit string, a number, an `{ low, high }` pair or a buffer (printed only as set / zero).
- The distance: **the text** `formattedDistance.distance` with its unit (`"250"` METER, `"1,2"` KM -> 1.2, `"1,234"` -> 1234);
  `rawDistance` is never the distance (Apple Maps sends 1). No number or unit NONE = no distance.
- `nextTurn`: the first event of a turn kind (intersection, branch, roundabout, destinationManeuver, alongMainRoad, laneChange)
  **with a distance**. A tunnel, a POI or the event after the next one (text `" "`, unit NONE) never stands in for the turn.
- `Poller`: see section 0. A call that never settles holds the next one back (one in flight); our own guard notes "timeout" 3.5 s
  after the call, and one line says so after 15 s. A failed call is tried again at the next poll, after five failures in a row only
  every 10 s.

### 1.2 Signal Probe (S16, entry `next-turn`)

- `apps/signal_probe/fields.js`: the entry `{ tag: 'next-turn', iface: GuidanceEventList, field: 'queryContent', flags: 'm',
  instance: 'Main_Instance', gapMs: 2000, fmt: 'turns' }`; `manifestSignals()` writes `_methods`; `tools/car/probe_fields.js`
  sync / check / export know `m` (the export carries `methodId 4`).
- `apps/signal_probe/probe.js`: the provider is taken from S7's `guiding-provider` (also on with S16); `tick()` drives the
  `Poller`, which uses the **watch's own proxy** of GuidanceEventList (S16 already watches its `registeredViews` and
  `viewsChanged`), or one proxy of its own when the watch gives none (the bench). Failures go through the same line machinery.
- `apps/signal_probe/format.js` `fmtTurns`: per event (first 4) kind, direction / branch / exit + angle + side / position + flag /
  main-road direction / lane side / tunnel length, the phone's distance (its number when the text is a number, else the text's
  length) and unit, `raw`, `uid=set|zero`, `toDest` when present, step, the description as elements / characters; then `next=` (the
  event the cluster would draw) and the car's two distances. No text, no road link, no source id.
- Example lines (from the bench run):

```
probe: poll P25 next-turn: on (every 2 s, timeout 3 s)
probe: poll P25 next-turn: first call: answered, 2 events (calls 1, answered 1, errors 0, timeouts 0)
probe: P25 next-turn #1 t=5: answered events=2 total=2 [intersection direction=LEFT geometry=UNDEFINED dist=400 unit=METER raw=1 uid=set toDest=2400 step=GS2 desc=1/14] [destinationManeuver position=RIGHT flag=true stopover=0 dist-len=1 unit=NONE raw=500 uid=set toDest=2400 step=GS1 desc=1/10] next=0 toFinal=0 toNext=-1
probe: P25 next-turn #12 t=31: answered events=2 ... next=0 toFinal=0 toNext=-1 (+4 unchanged)
probe: P25 next-turn #1 t=1: error: Access denied ...        (a refused call: the binding's text, ASCII, at most 120 characters)
probe: poll P25 next-turn: off (off) (calls 14, answered 14, errors 0, timeouts 0)
```

- Volume: one line per changed reply (about 350 characters), at most one per 2 s, S16's cap 400 lines: about 0.2 KB/s while moving,
  14 minutes of changes. Unchanged replies (parked) make no line.

### 1.3 Twin Dial (tab 5)

- `shared/twin_pages.js`: `cpSync` (from the page's own 150 ms evaluation) owns the `Poller`; `cpLive` = test mode on, the phone
  guiding, BMW's guidance neither ACTIVE nor CALCULATING_ROUTE, a turn read within 6 s. `navPage`: map (if open) -> BMW's arrow page
  (if BMW guides and `pages.guidance`) -> **CarPlay's arrow page** -> the CarPlay route page -> calc / road / none.
- The arrow page's tokens read CarPlay's turn when the page chose it: the pictogram as for BMW's manoeuvre (`dartOf` on the event;
  along the main road = a slight bend; an unknown direction = the dot, never a guessed arrow), the roundabout's exit and traffic
  side, the phone's number with a "." and its unit (м, км, mi, ft, yd), the road the turn names (`description` text; empty if
  none), the footer from `nextDestinationInfo` (arrival, distance). Lanes stay BMW's `currentLaneGuiding` (empty so far with CarPlay).
- The log: the poll on / off, the first call's outcome and every change, and the turn's KIND once per change with the road as a
  length (`carplay turns: turn intersection left, a road of 14 characters`). Never a road name, never a distance (tested).
- Glyphs (tested against the atlases): the distance `td_dist` (digits and "."), the unit `td_sub` (м км mi ft yd), the exit
  `td_u21` digits; a road name is drawn with the Noto atlas and cut with "…" to its width (characters without a glyph are dropped by
  the painter, as for every road text today).

### 1.4 What I chose for "авто" (navView)

- **карта**, with the map window open (the cluster in map view): **the map wins** (unchanged).
- **стрілки**: never the map, so CarPlay's arrow page whenever it is live.
- **авто**: the map only by its strict rule (BMW's guidance ACTIVE or "map without a route"); with a CarPlay route BMW's guidance is
  INACTIVE, so the window normally stays shut and **CarPlay's arrow page shows**; a forced window (`mapForce`) still wins.

### 1.5 Bench

- `bench/carplay-turns-fixture.js`: replies in the catalogue's shapes as the binding hands them, CarPlay's way (rawDistance 1, the
  next-but-one event `" "` / NONE / 500, `detailedGeometry` UNDEFINED, `configSequenceId` 4294967295), a route of three turns and a
  roundabout with invented neutral road names, and a stand-in provider (`callMethod` answers only `queryContent`).
- `bench/run-app.js`: an app whose manifest declares `queryContent` on GuidanceEventList gets the stand-in (`hu.fakes`); `GET /probe`
  reports `carplayTurns: { calls, methods }`.

## 2. Tests (each suite run alone, ports 8951-8954)

| Suite | Result | Note |
|---|---|---|
| `bench/test-signal-probe.js` | 202 / 0 | was 171 on `main`; + the method's declaration and catalogue checks, the vector, the poll (provider gating, every 2 s, one in flight, timeouts, errors, slow-down, unchanged count, stop), the real build calling the stand-in |
| `bench/test-twin-carplay.js` (new) | 69 / 0 | the decoder, the declarations, off / on, the route, the fallbacks, BMW first, navView, glyphs, painting, the real Twin Dial build calling the stand-in |
| `bench/test-twin-centre.js` | 250 / 0 | |
| `bench/test-twin-pages.js` | 100 / 0 | 100 on `main` too |
| `bench/test-twin-tabs.js` | 252 / 0 | |
| `bench/test-twin-layout.js` | 121 / 0 | |
| `bench/test-diag.js` | 107 / 0 | the new setting passes the generic DIAG checks |
| `bench/test-gpu-rows.js` | 48 / 0 | |
| `bench/test-twin-probe.js` | 115 / 0 | |
| `bench/test-twin-budget.js` | 101 / 0 | |
| `bench/test-native-ui.js` | 47 / 0 | |
| `bench/test-twin-sim.js` | 39 / 0 | |
| `bench/test-twin-fonts.js` | 327 / 0 | |
| `bench/test-probes.js` | 25 / 0 | f30_cluster through the changed run-app.js |
| `tools/car/dry_run.js` | 34 / 0 | `--out` in my scratchpad (its default writes into `bake/work/car/CQ/dry-run`) |
| `tools/car/test_session.py` | 12 / 0 | |

A test fails without each feature: e.g. with the CarPlay branch of `navPage` disabled, `test-twin-carplay.js` gives 61 / 8.
`tools/twin-dial/run_all_tests.sh` was not run (the lead's). Add `twin-carplay` to its suite list (new suite, no count on `main`).

## 3. What the car must show, and a short protocol

Before: Signal Probe and Twin Dial of this branch installed (the lead's builds). Turn View and Waze on cluster off. Car clock times
noted (UTC+2). A passenger operates the phone and the menus; the driver never touches a menu while moving.

**A. Signal Probe, S16 alone** (parked, engine running): every set off but **S16**. Mark M1. Start a CarPlay route (note Apple Maps
or Google Maps). Mark M2 when the cluster shows BMW's turn box. Expect within 2 s:
`poll P25 next-turn: on`, then `poll P25 next-turn: first call: <outcome>`. This line answers the ACL question:
- `answered, N events` -> the call is allowed; read the `next-turn #1` line (kinds, `dist=`, `unit=`, `next=0`);
- `error: <text>` -> refused (the text says by whom); `timeout` -> no answer; `proxy: <text>` -> no proxy. Collect the log.

**B. The drive** (if A answered): a short CarPlay route with three turns and a roundabout if one is near. The passenger marks just
before each turn and notes the manoeuvre and distance the phone shows. Expect a `next-turn` line every 2 s while the distance
changes, `dist=` counting down in the phone's units, the kind and direction matching the phone, `exit=` at the roundabout, the next
turn moving to `next=0` after each turn, and at the end `events=0 ... next=none`. Then stop the route: `poll ... off`.

**C. Twin Dial** (if A answered): Twin Dial on, tab 5, Діагностика **"Повороти CarPlay (тест)"** on, Навігація = авто. A CarPlay
route: tab 5 should show "Навігація", the arrow, the phone's distance (e.g. `400 м`, `1.2 км`), the road the turn names and the
arrival footer; photograph it against the phone. Then a moment with no turn (route ended) -> the CarPlay route page. Then Навігація
= карта with the map window W2 (if the cluster is in map view) -> the map. Twin Dial's log: `carplay turns: first call: ...`,
`carplay turns: turn <kind> <direction>, a road of N characters`. Switch the test mode off afterwards.

## 4. Open decisions for the lead / the owner

- **D1 Install risk.** Both apps now declare a method. The precedent that a `_methods` declaration installs on this car is f30_cluster
  0.7.18 (`requestDefaultSymbols`). If an install fails anyway: Signal Probe - delete the `next-turn` line and run
  `node tools/car/probe_fields.js sync`; Twin Dial - remove the GuidanceEventList block from `signals` (the test mode then logs
  `error`/`proxy` and keeps the route page).
- **D2 Twin Dial declares the method always**, even with the test mode off (a declaration is not a call; no call is made while off,
  tested). Alternative: a separate test build. I kept it in the normal build so the owner can switch it on from Діагностика.
- **D3 Poll while tab 5 is not selected?** I followed the brief: the poll runs whenever the test mode is on and the phone guides,
  whatever tab shows. Restricting it to tab 5 would mean fewer calls but a 2 s gap on entering tab 5.
- **D4 The phone's distance as the phone writes it** (its number and unit, `mi` / `ft` for imperial) rather than our metric rounding
  (`fmtDist`). The CarPlay route page still uses our rounding for the remaining distance.
- **D5 Along the main road** is drawn as a slight bend left / right (no pictogram of its own). A tunnel never replaces the turn.
- **D6 Backoff** after five failures in a row: one call every 10 s (the brief: never faster than the poll). Say if a refused call
  should stop the poll for the rest of the route instead.
- **D7** If the car answers, switching the Twin Dial test mode on by default is the owner's call after photo C.

## 5. Hand-off

- Branch `car-issues/ct`: `cd53770` (Signal Probe S16 + the shared module + bench), `fa36d00` (Twin Dial test mode), `4c7c9fb`
  (a probe test count after the shared poller's change: the first call follows a new proxy at once). Touches
  `shared/carplay_turns.js` (new), `shared/twin_pages.js`, `apps/signal_probe/*`, `apps/twin_dial/app.json`, `bench/run-app.js`,
  `bench/carplay-turns-fixture.js` (new), `bench/test-twin-carplay.js` (new), `bench/test-signal-probe.js`,
  `bench/signal-probe-vectors.json`, `design/car/probe_fields.json`, `tools/car/probe_fields.js`.
- `bake/work/car-issues/pub/CT/READY`.
