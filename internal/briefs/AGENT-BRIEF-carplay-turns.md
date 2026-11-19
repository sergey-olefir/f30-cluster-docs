# Brief: CarPlay's next turn as data — GuidanceEventList.queryContent in Signal Probe and as a test mode in Twin Dial

Audience: one agent, **CT**. Base: **`main`** at `9517e89` or later (Twin Dial 0.1.4, f30_cluster 0.7.22, Signal Probe 0.3.0, every suite green). The lead merges, bumps versions and pushes; you never push, never bump a version, never connect to the car.

## 0. What is known

- With a CarPlay (or Android Auto) route, BMW's `Guidance` (0xFB1A, inst 128) stays `INACTIVE`; `conclusiveDart`, `nextIntersection` and the lanes are empty or a placeholder (car, 2026-11-18; `docs/car-issues/NV-REPORT.md`). The cluster still shows BMW's own CarPlay turn box, as a picture.
- **The phone's manoeuvres exist as data** in `de.bmw.infotainment.navigation.guiding.GuidanceEventList` (service 64283 = 0xFB1B, major 5, instance 128 — confirm the instance in the catalogue). IterioDev's Turn View reads them by **polling the method `queryContent` (methodId 4)**. It is public, MIT/other: `github.com/IterioDev/car-apps`, `tools/iteriodev-turnview/nav.js` (`queryHex`, the `events` branch of `nav.payload`, `maneuverOf`, `textMeters`) and `turnview.js` (the poll: only while the provider is not BMW, one call in flight, every `EVENTS_MS`). Copies are in the lead's scratchpad: `<session scratchpad>/tv/`. Read them; **do not copy their code** (licence and style): write ours from the catalogue.
- Their request (stateless, the HMI's own views untouched): one `GuidanceEventListConfig` with `viewport.startPoint.anchorPoint` = standardReferencePoint `CCP`, no offsets, `maxElementCount` 4, no end points, `includedActiveTypes` empty, `includedTypes` = `INCLUDE_ACTIVE_MANEUVERS`, `INCLUDE_DESTINATIONS`, `INCLUDE_TUNNELS`, `INCLUDE_ALONG_MAIN_ROAD`; `options` empty, `minimumUpdateIntervalMs` 0, `segmentItemRasterLength` 0.
- Their reading of the reply: `viewContent[0].eventsInView[]`, each a manoeuvre union `{type, value}` (intersection / branch / roundabout / destinationManeuver / tunnel / alongMainRoad, and other kinds that are not the turn). **CarPlay's `rawDistance` is not the distance** (Apple Maps sends 1); the distance is `formattedDistance.distance` (text, e.g. "250" or "1,2") with its `unit`. Apple Maps sends `uid` 0. The first way-ahead event with a distance is the next turn. The service sends no notifier for this without a view of one's own (so `viewsChanged` stays silent: do not rely on it).

## 1. The owner's word (2026-11-19)

The owner approved calling **`queryContent` only** — a read. **Never** `setViewConfig`, `setViewportConfig`, the `registeredViews` setter, or any other method; never a method on any other interface. This is the first method call our apps make: keep it in one small module, declared with `_methods: ["queryContent"]` on that interface only (`tools/signals/check_signals.py` checks it), called through the kit's `signals.proxy` / `signals.call` (`kit/runtime/signals.js`). Whether the car's ACL lets an OAP app call it is **unknown**: log the outcome of the first call (answered / error text / timeout), never crash, never retry faster than the poll.

Poll only while the route is the phone's (`Guidance.guidingProvider` is `CAR_PLAY` or `ANDROID_AUTO`), one call in flight, every 2 s, a timeout of 3 s, and stop when the provider goes back or the feature is off.

## 2. Package

### 2.1 Signal Probe

- A new entry kind for a polled method (the table, `tools/car/probe_fields.js sync`, the vectors and the tests as 0.2.0 / S16 did; `docs/car/CP-REPORT.md` §13 is the pattern). Put it in **S16 "Наступний поворот"**.
- What the line carries: the call's outcome; the number of events; for each of the first 4: its type, direction / branch type / exit number / drive direction, `formattedDistance` (the number and the unit — a number, so it may print), `rawDistance`, whether a uid is set, `distanceToDestination` if present. **Texts (road names, descriptions, signposts) print as lengths only.** No positions.
- Mind the volume (256 KB rotation): print only when the reply changed, plus a count of unchanged replies.

### 2.2 Twin Dial: a test mode on tab 5

- A Діагностика setting **"Повороти CarPlay (тест)"** (key `diagCarplayTurns`, off by default), config `config.carplayTurns`. Off: nothing changes, no call is made.
- On, with the phone guiding: tab 5 shows the **arrow page** (the one built for BMW's own guidance, `shared/twin_pages.js`, dark until tested) fed from the event list instead of `conclusiveDart`: the arrow for the type and direction, the distance from `formattedDistance`, the road name if the event carries one (on screen only, never in a log). The CarPlay route page (destination block) stays the fallback whenever the call fails, returns no turn, or is older than 6 s.
- Use the same decoder as the probe (a shared module, e.g. `shared/carplay_turns.js`).
- Respect `navView` ("Навігація" авто / карта / стрілки): in "карта" with the map window open, the map wins; say in the report what you chose for "авто".
- No new layer: the page is the existing `tdpage` layer. Glyph coverage for every character the page can draw (memory: `td_*` fonts).

## 3. Rules (as every round)

- Worktree `<repo>/.claude/worktrees/ci-ct`, branch `car-issues/ct` from `main`; symlink `reference`, `bake/work` and `docs` from the main checkout. Never `git add` under `docs/`, `reference/`, `bake/work/`.
- Read-only otherwise: fields `r` with a non-zero getter id, `s`, events `_broadcasts`. Never a setter. Node 8.16 syntax. Privacy: no texts in logs (lengths only), no positions, no names.
- Bench ports 8950–8959 only (8080 / 8150 are the owner's); run your suites one at a time; do not run the full runner (the lead does).
- A test that fails without each feature. Bench fixtures with the real shapes from the catalogue (invented neutral road names).
- Report `docs/car-issues/CT-REPORT.md` (through the symlink, not committed): what was built, what the car must show, a short car protocol (S16 on alone, then a CarPlay route with three turns and a roundabout if near; then Twin Dial tab 5 with the test mode on), and open decisions. `READY` in `bake/work/car-issues/pub/CT/`. Final message: branch, last commit, test totals, decisions.
