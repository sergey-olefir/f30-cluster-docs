# Brief: the car session — read every signal we need, and everything we deferred or skipped

Audience: four agents, packages **CA, CP, CL, CQ**, working in parallel, each in its own git worktree; then the lead and the owner at the car. Read §0–§4 and your own package in full; skim the others. **No agent ever connects to the car.** The agents prepare three things: a complete register of what the car must answer, one probe that reads it, and the tools and the protocol that turn one session at the car into decisions. The lead reads the logs (read-only) and the owner does the presses.

## 0. Why

Since round 1 we have drawn from signals that the car's catalogue lists but that nobody has seen arrive on this car. The car has already proven that a catalogue field may never be sent (`newCCMessage`: silent for 25 minutes with messages showing) and that a field may arrive with a meaning we guessed wrong (0.7.19's "a listed entry is a shown message"). Three rounds of work now wait on the same answer: Twin Dial's trip page, media, phone, guidance arrow, cruise, tyres, the tab button; f30_cluster's message window; and a list of experiments and photos that were postponed each time the car was offline. The owner's instruction (2026-11-18): **one brief for reading all the signals we need from the car — everything we deferred or skipped — as detailed as possible.**

The aim is that **one prepared session** (plus, where a drive is needed, one drive) answers all of it, and that nothing is forgotten because it was written in a report nobody reopened.

## 1. Rules

- **Branch and worktree:** `git worktree add <main checkout>/.claude/worktrees/car-<pkg> -b car/<pkg> main`; symlink `reference` and `bake/work` (`git status --porcelain` must not list them). Commit on your branch. No push, no version bump, no merge; nothing changes in `apps/f30_cluster` or `apps/twin_dial` on your branch (the probe is its own app, §CP).
- **The car is read-only, and only the lead touches it.** Agents: no ssh, no network request to the car's address, no assumption that it is online. On the car, ever: never `kill` an OAP app process (only `systemctl restart oapcontroller`, and only the lead, with the owner's word); never a setter; never a method call with a side effect; never `KombiIF.standardCCMessage` / `extendedCCMessage`; never `A19B`; never a write to a driver-assistance setting; never `requestDefaultSymbols` (BMW's bitmaps are never requested, baked, logged or committed).
- **Read-only signals only:** flags `r` (with a NON-ZERO getter id) and `s`; `tools/signals/check_signals.py` must pass on every declaration; a notifier-only field is `s`. Events (`_broadcasts`) are not declared unless the lead says so for a named one.
- **Privacy in every log line and every file:** numbers, enum names and lengths only. Never a media title, a station name, a contact name, a phone number, a call list, a device name, a road name, a coordinate, a bearing, a VIN, an identity number of a modem, a key or a token. A text member is logged as its length; a picture as its format and byte count; a frequency as its band. Tests must plant canary strings in every text member and prove none reaches the log (as `bench/test-twin-probe.js` does).
- **No maker's name** in tracked files; no reference image in a tracked file; no home-directory path in a tracked file (find the repository from `__file__`).
- **No browser automation; no process you cannot stop yourself;** ports 8080 and 8150 are the owner's (use 8500 + your number); Python through `UV_CACHE_DIR=<main checkout>/bake/work/.uvcache uv run -q --no-project --with … python -I <script>`.
- **Evidence labels in everything you write:** *arrives* (a log line on this car shows it — cite the file and the line's shape), *arrives-other-app* (another app's log on this car), *not sent* (proven silent on this car, with the minutes observed and what was done meanwhile), *unseen* (never subscribed on this car). The catalogue (`reference/car/2026-11-15/someip/someip-interfaces.json`, git-ignored) has no scaling: every unit is a hypothesis until a reading is compared with something the car itself displays.
- **Outputs:** `docs/car/<PKG>-REPORT.md`, data in `design/car/*.json`, tools in `tools/car/`, scratch in `bake/work/car/<PKG>/`, a `READY` note in `bake/work/car/pub/<PKG>/`. Finish with a short final message: branch, last commit, what is ready, what the lead must decide.

## 2. Where things stand (2026-11-18)

- **On the car:** f30_cluster **0.7.20**, Twin Dial **0.1.0**, Driver Profiles 0.2.2, cluster-render (the client saved on 2026-11-15 is 0.5.12), other apps (car_vitals, Waze on cluster). Logs are plain files: `/var/opt/public/oap/oap_<uid>/data/<app>.log` with `settings.json` beside them (f30_cluster's uid is 12211; Twin Dial's is assigned at install). cluster-render keeps a journal and a `status.txt`.
- **On `main` (pushed, versions not bumped, so not on the car):** research round 3 — f30's message `release` fix (off), the `sceneTex` texture variants (off), the black-box experiment keys `mbGap` / `mbCover` / `mbRebuild` (off), Twin Dial's trip factors and silent-field log (on), `test-*` suites all green.
- **On branches, not merged:** Twin Dial round 3 — `twin-r3/{sd,sf,sb,sm,sp,sn,sv,sc,k1,k2}`. Five of them added probe fields to `shared/twin_probe.js` and `apps/twin_dial/probe/` independently: **SM** (+10, media), **SP** (+31, phone), **SN** (+13, guidance and the map-view setting), **SV** (+25, sign, cruise, technical, others), **SB** (buttons, in a separate `probe-p13.json`). Their item numbers collide (three packages used "P13"), and their edits to the same files conflict.
- **Saved car material on the Mac** (`reference/car/2026-11-15/`, git-ignored): the catalogue, snapshot logs of ours and of other apps (car_vitals among them), cluster-render's journal 2026-11-13…14 and `status.txt`. **No car log later than 0.7.12's is saved** — in particular nothing of 0.7.18–0.7.20 and nothing of Twin Dial. Whatever the lead read on 2026-11-17 exists only as notes in commit messages and reports.
- **The specifications that wait on this:** `docs/twin-dial/SPEC-R3.md` (§3.4 the tab key, §4.1 trip and tyres, §4.3 media, §4.4 phone, §4.5 map and guidance, §5.4 "shown now", §8 the sign, §9 cruise), `docs/research3/{MC,MK,MB,MT,MP}-REPORT.md`, `docs/twin-dial/PROBE-PROTOCOL.md` (P1–P12), `docs/twin-dial/r3/{SB,SM,SP,SN,SV,SC}-REPORT.md`.

## 3. The register so far (CA verifies, corrects and completes it)

Four kinds of item. **L** = answered by reading a log that the installed apps already write (no new build). **P** = answered by the probe (a signal read). **X** = an experiment: a build or a setting, then a photo or a log. **F** = a photo or an observation only.

### 3.1 L — logs the car already has

| # | Question | App and line | Decides | Source |
|---|---|---|---|---|
| L1 | Why f30's window over BMW's panel stays open | f30_cluster.log: `active cc: number …`, `check-control list: …`, `no ccList entry for number N`, `check-control frames: …`; `settings.json` (`hudLayout`, `g20`, `bottomRow`) | which of cases A1–A16 it is → which `release` setting | research3/MC §6 (13-row table). Trap: `no ccList entry` alone does not prove a hang |
| L2 | Does `statusActiveCCMessage` notify at all, what it rests on (0, 65535, the last number), does its number equal the ccList id | same lines | D7 for Twin Dial; `noneValues` | MC, SPEC rev 2 §8.4 |
| L3 | The clock's fallback | f30_cluster.log: `clock (local) raw: … -> N`, `clock: ours shows HH:MM, UTC+… from the car's shown clock` | that `statusShowTimeDate` arrives and is local time | 0.7.19 note in app.json |
| L4 | Twin Dial's trip fields | twin_dial.log: `trip average|travel|departure raw (P4): …` | arrive or not; raw values against iDrive's trip page | research3/MK |
| L5 | The odometer member | twin_dial.log: `odometer raw (P3): …`; and the owner's memory: did the bottom-left number show on 0.1.0 | arrives; factor 1 km | MK |
| L6 | Phone state | twin_dial.log: `phone state raw (P9): …` | `aggregatedCallState` arrives | r3/SP |
| L7 | Media | twin_dial.log: any `playerMetadata` / `MediaSource` line (lengths) | `HmiMediaState` on `HMI_CID` arrives | r3/SM |
| L8 | The second witness | twin_dial.log: `displayed cc: number N` | `Vehicle2.statusDisplayedCCMessage` arrives, returns to 0 | r2/D, MC |
| L9 | Layout builds at start | both logs: the layout-build lines in the first 10 s (the bench predicts 3) | whether the start-up fix is worth it | research3/MP |
| L10 | Upload accounting | cluster-render's journal: its upload cap and per-minute KB lines; "goes whole … KB" lines | the real cost of a theme switch | MP, MT |
| L11 | The map slot under Twin Dial | cluster-render's journal / status: slot 0x100 rows while Twin Dial's panel (0x10004f8 in 0x203) is up | was BMW's map there when no black box showed (theory H-M) | r3/SN, research3/MB |
| L12 | cluster-render's version | its status text | ≥ 0.5.12 for ETC2 | MT |
| L13 | The 0.7.18 probes' results never recorded | f30_cluster.log (if still present): `ccListRiskClass`, `CheckControlMaster.basicList` lines | which marks a stored message | r1/VS §2 item 6 |
| L14 | The speed-limit sign in f30 | f30_cluster.log: the sign's lines | that `TrafficSignDisplayControl` arrives (the round-3 brief claimed it; no line is in the repository) | r3/SV |
| L15 | Day/night | both logs: `day/night source: …` | the car's value vs the sun; the meaning of 0 | app.json note |

### 3.2 P — signals for the probe

Final numbering (CP renumbers the branches to this):

| Set | Item | Fields (interface . field) | Count | The owner does | Decides | From |
|---|---|---|---|---|---|---|
| **S1 messages** | P1 | `KombiIF.statusActiveCCMessage`, `Vehicle2.statusDisplayedCCMessage`, `KombiIF.statusMenuCCMessage`, `KombiInterface_2.ccList` (ids, colours, symbol ids, name lengths) | 4 | open a door, wait until BMW hides it, open two doors | "shown now" = the list id; rest value; the frame counter | VS |
| | P2 | `CheckControlMaster.basicList`, `KombiInterface_2.ccListRiskClass` | 2 | same | the stored-message mark; risk class as a sort key | VS, SC |
| **S2 computer** | P3 | odometer: `statusRangeTankStatus.mileageKilometre`, `MileageSupreme`, `VehicleInformation.Range`, `SecureMileage` | 4 | read the cluster's odometer | factor, invalid value | VS, MK |
| | P4 | trip: `statusBordcomputerAverage / TravelData / TravelStart / Status2`, `ResetLogic3.bcDataAll`, `ResetLogic`, `ResetLogic2`, `BcPersonal`, `LSCDataSink` | 9 | read iDrive's trip page; reset the trip in iDrive | factors, which computer is "since start", the departure time | MK |
| | P5 | units: `UnitBN2020`, `statusKombiExternalDisplay`, `Vehicle.statusNewCurrentUnitsStatusExt` | 3 | — | km / miles, °C, bar | VS, SV |
| | P6 | `KombiIF.dataInstrumentPack` (`engineTemperatureEvaluated`), `CDS.statusEngineTemperature` | 2 | cold start to warm | the car's own coolant verdict | VS |
| **S3 body** | P7 | `DoorsWindowsRoofMirrors.CentralLockingSystemAndPanelStatus2`, `A4A.statusA4ALids`, `Vehicle2.statusZVStatus` | 3 | each door, boot, bonnet | notifies at all; sends once at subscribe | VS, SV |
| | P10 | voltage: `BasicDataSet.basicDataSet` (the voltage member only), `LevelMonitor.systemVoltage`, `CDS.statusBattery2` | 3 | engine off / on | a plausible 12–14.8 V | VS |
| **S4 media** | P8 | `HmiMediaState` on `HMI_CID`: `playerMetadata`, `trackProgress`, `mainMenuMetadata`, `menu`; `MediaSource.activeSourceId`, `sources`; `TunerFmAmDab.currentStationFmDabAm`; `MediaPlayer` ×5 on `CarPlayMedia`; `listOfMainConnections` | 12 | FM, DAB, USB, Bluetooth, **CarPlay**; play, pause, skip, change source, mute | which line is the title; progress; the source's icon kind; radio detail | SM |
| **S5 phone** | P9 | `VoiceCall` (`Call1`), `VoiceCallManager`, `ConnectivityAggregator`, `DevicePresentation.deviceList`, `Telephone` (both instances), projection telephony | 32 | connect; ring the car from a second phone; answer; hang up; miss a call; CarPlay on; **photo of a ringing call with Twin Dial on and off** | the call's states; battery / signal; whether `Telephone` is the car's own modem (silent during a phone call); whether Twin Dial hides BMW's call pop-up | SP §5.1 |
| **S6 view and key** | P11 | `HMIContext` / `HMIContext2` `currentICWidget`, `currentICGraphicalContext`; `KombiIF.statusKombiSelection`, `statusKombiSelectionAvailability`, `statusConfigurationKombi5` (`statusNavigationDisplayType`), `Vehicle2.statusDisplayKombi` | 7 | change the cluster's view in iDrive and back | the cluster's map-view setting; an indirect key | VS, SB, SN |
| | P13 | buttons: `SteeringWheelPushButton` ×11; `body.Light.IndicatorSteeringColumn`, `…Complete`; `FrontCCU.statusRotaryEncoderState`, `statusPushButtonsState`; `Lifecycle.entertainmentModeTrigger` | 16 | SB §3.2: the stalk's on-board-computer button short ×3 (a long press may reset a page: skip or choose a page whose reset is acceptable), the stalk's other switches, thumbwheel up / down / press with and without a BMW list open, every other wheel button once, the controller, the same with a message showing | **the tab key** (SB §3.3 decision table) | SB |
| **S7 navigation** | P12 | `NavigationStatus.guidingState`, `Guidance.guidanceState`, `nextIntersection`, `conclusiveDart`, `currentLaneGuiding`, `nextDestinationInfo`, `currentRoad` (length) | 7 | a guided route: a turn, a roundabout, a fork, the destination | the manoeuvre kinds, the distance's unit against the car's own text | VS, SN |
| | P14 | `Guidance.visualizationInKombiNonNaviEnabled` and SN's other map-presence fields | 6 | cluster view = map, with and without a route | when BMW's map exists | SN |
| **S8 assistance** | P15 | `TrafficSignDisplayControl.displaySpeedLimitAndNoPassingRestriction`; `displayDriverAssistanceLongitudinal`; the HUD feed's set speed / distance / limiter; the third set-speed source | 8 | drive past signs; cruise on, set, +/−, cancel, resume; limiter on / off | the sign arrives; **the set speed and the active function** (SPEC-R3 §9) | SV |
| **S9 technical** | P16 | `FES.statusSwitchDriveDynamicSport2` (boost), `FES.statusVehicleDynamicEngine`, `…MaxValue`, `CDS2.statusInjectionVolume` (oil), `Vehicle2.statusOilDipstick2`, TPMS `statusTyrePressure` / `Temperature` / `TargetTyrePressure` / `TyresStatus` (getters, polled) | 9 | a drive under load; read iDrive's tyre page | boost's scale; tyres' scale and whether the car notifies | SV |
| | P17 | others: positioning heading, altitude, `cbsDataServiceList` (service due) | 8 | — | only whether they arrive | SV |

Roughly 135 fields on about 45 interfaces. Nobody has ever watched that many at once on this car (research3/MK watched 22 and called the load untested): hence **sets** switched one or two at a time (§CP).

### 3.3 X — experiments (a setting or a build, then a photo or a log)

| # | Experiment | Key / build | Photograph or read | Proves | Source |
|---|---|---|---|---|---|
| X1 | f30 with "Change with the drive mode" off | a setting, no build | the box after mode switches | whether the rebuild brings the black box | research3/MB §4 |
| X2 | Twin Dial with a clear layer over the gear letter | `mbCover` (clear, black, grey) | the hub of the speedometer | rectangle versus pixels | MB |
| X3 | f30 with the gap cut out of its layer rectangles | `mbGap` | the box; the notch in Comfort's strip | the fix | MB |
| X4 | forced rebuilds | `mbRebuild` | the box | theory (b) | MB |
| X5 | f30 textures | `config.sceneTex` = `r8`, then `etc2` | the dials close up; "goes whole … KB" over three Sport → Comfort switches | picture quality; that `etc2rgb` draws at all on this cluster | research3/MT §8 |
| X6 | the map window | `mapWindow` W1 (and W2), `mapWindowForce`, `mapSlots` | the centre in P and D, with and without a route, day and night; the centre with each cluster-view setting | BMW's map shows through on the panel's own id; its extent; what else BMW draws there; that the app can read the slot rows | r3/SN |
| X7 | f30's message release | `config.ccActive.release` (with SC's narrowing) | a door message, a second message, a message that returns | the window closes, and never over a listed message | MC, r3/SC |
| X8 | message timing | Twin Dial 0.1.1's card | slow-motion video of a message appearing and of leaving tab 2 | that 150 ms covers the upload and the panel's opening; that one turn's writes on several connections land together | r3/SC |
| X9 | the band's level and the night palette | `bandGain`; day and night | one fixed-exposure photo against the bench render (mid-tones only) | the level; the night multipliers | SPEC rev 2 §7, §17.2 |
| X10 | the KOMBI accepts the round-3 shaders | 0.1.1 installed | the cluster after start, a mode switch, day/night | no reset; no missing layer | SPEC-R3 §12 |
| X11 | switch-time film | — | 60 fps film through a mode switch on each app | MP's predictions (1–2.8 s incomplete back to Comfort; nothing on Twin Dial) | MP |

### 3.4 F — photos and observations only

| # | What | Why | Source |
|---|---|---|---|
| F1 | Is BMW's own speed-limit sign visible anywhere with Twin Dial up | SV: we cover it at (538, 621) | r3/SV |
| F2 | The side lamps and the belt lamp with Twin Dial up (belt unbuckled, a door open) | the safety check before driving | SPEC rev 2 §17.2 |
| F3 | A close photo of a dial's lower ends on the car | the owner's inner-ring finding against SD's measurement | r3/SD |
| F4 | "Photo 16" of `docs/OVERLAY-PHOTO-PROTOCOL.md`: which side lamps BMW draws over us | six of seven lamps have relation "unknown" | r1/VG §1.4 |
| F5 | Does the HUD show the cruise set speed | how urgent SPEC-R3 §9 is | lead |
| F6 | The cluster's own cruise and limiter display on the stock cluster (Twin Dial off) | the model for our symbols and states | SPEC-R3 §9 |
| F7 | The stock cluster's media line, phone list and call pop-up (Twin Dial off) | what the RIC services draw that we cannot read | SM, SP |
| F8 | Ukrainian message names: the longest the car shows | the card's three lines | r3/SC |

## 4. Packages

### CA — the audit: nothing deferred is lost
Sweep **every** place a car question may hide and reconcile it with §3: all of `docs/` (the reports of rounds 1–3 and of research 3, `SPEC.md`, `SPEC-R3.md`, `PROBE-PROTOCOL.md`, `FEATURE-AUDIT.md`, the car reports `CAR3-*`, `CAR4-*`, `OVERLAY-*`, `BAKE-*`, `PERF-*`, `docs/boxes/*` and `docs/strip/*` on their branches), every `_about` / `_comment` in both apps' `app.json` that says UNCONFIRMED or hypothesis, every `hypothesis` / `unseen` / `after probe` / `needs the car` in `design/**/*.json`, the unmerged branches `twin-r3/*`, `boxes/*`, `fix/*`, and commit messages since 0.7.12 (`git log --all`). For each claim found: the exact question, the file and line that asks it, what would answer it (L / P / X / F), whether §3 already has it, whether an answer already exists somewhere (then cite it and close the item), and what depends on it (which spec section or setting is blocked).
Deliver `design/car/register.json` (machine-readable: id, kind, question, fields, steps, decides, blocks, source, status) and `docs/car/CA-REPORT.md`: the reconciled register, the items §3 missed, the items §3 has that are already answered, and contradictions between reports (two reports stating different things about the same signal). Publish the register early (`pub/CA/`): CP, CL and CQ build from it.

### CP — one probe for everything: the app `signal_probe`
A **separate app** `apps/signal_probe` (id `signal_probe`, title "Signal Probe", its own `loggingId`), that **draws nothing on the cluster** (no cluster-render connection, no layer) and only subscribes and logs. Reasons: the probe then cannot harm either cluster app; it can be installed beside both and run with either or with the stock cluster (needed for F6, F7, P13 with BMW's lists); a failed declaration fails only the probe; and an app may declare one instance of an interface, so a second app doubles what can be read.
1. **Unify** the five branches' probe code and field lists (`twin-r3/{sm,sp,sn,sv,sb}` on top of `main`'s `shared/twin_probe.js`, `apps/twin_dial/probe/`, `tools/twin-dial/probe_build.py`) into the app, renumbered to §3.2's P1–P17, with the formatters and the privacy rules of each package kept (per-field minimum gaps, rate limit, line cap, summaries naming silent fields at 30 s / 2 min / 10 min, one read-only getter ask for a silent field).
2. **Sets** (§3.2 S1–S9), each switchable in the app's own centre-screen menu, default all off; the log says which sets are on at every start and change. Because declarations are static in `app.json`, state plainly what the kit and the platform do with 45 declared interfaces whose handlers are idle (cost at start, the generated ACL, the load) — measure on the bench what can be measured, and offer the alternative of two or three probe apps (`signal_probe_a/b/c`) if one manifest is too heavy; recommend one.
3. **Interfaces with several instances** (`HMI_CID`, `CarPlayMedia`, `Call1`, `Main_Instance`, `Inst1`, `Front`): a table of which instance each field is read on and why; where a second instance would be worth a second app.
4. **Log format:** one line per change, `probe: P<n> <field-tag> #<count> t=<s>: key=value …`, numbers, enum names (capitals, digits, `_`), lengths; a marker line the owner can trigger (the app's menu: "Позначка N") so the protocol's steps can be found in the log; the session's start line with the app's version and the sets.
5. **Tests:** `bench/test-signal-probe.js` — every field injected and its exact line compared; canaries in every text member; a table-versus-catalogue check that no text member is read as a number; Node 8.16 syntax (the car's Node: no `?.`, `??`, object spread, optional catch binding); `check_signals.py` on the manifest; the f30 and twin suites unchanged; the app starts on the bench with nothing injected and exits cleanly (an OAP app that exits twice is uninstalled by the platform: no uncaught exception anywhere).
6. Twin Dial keeps its own small log lines (trip, odometer, phone state, displayed cc); remove nothing from it.
Deliver the app, `docs/car/CP-REPORT.md` (the field table with ids, flags, instance, set, formatter; the load discussion; how to install it from the Store), `design/car/probe_fields.json`.

### CL — the tools: from logs to decisions
1. **A read-only collector for the lead** (`tools/car/collect.sh` + a README): given the car's address, copies **only** the app logs, their `settings.json`, cluster-render's journal and status text into a new dated folder under `reference/car/<date>/` (git-ignored); every command is a read (`cat`, `ls`, `journalctl` read options) over ssh; it prints what it will run before running it; it never restarts, writes or kills. It records the installed apps' versions and uids (so Twin Dial's and the probe's data folders are found).
2. **A parser** (`tools/car/parse_logs.py`): reads those logs and fills `design/car/answers.json` — per register item: answered / not yet, the evidence lines' shapes (numbers only), the value decided. It implements every decision table of the reports as code: MC §6 (the window's case), MK (trip and odometer factors; the distance ÷ time cross-check), SB §3.3 (the tab key: which bit of the stalk byte moved on which step, using the marker lines), SM / SP / SN / SV's tables, the L-items of §3.1.
3. **Config patches** (`tools/car/make_patches.py`): from `answers.json`, the exact `app.json` changes each answer implies (`config.tabKey`, `config.trip.factors`, `config.odometer`, `config.ccActive.release` / `noneValues`, `config.cruise`, media and phone field paths, tyre scales, `mapWindow` presence rule) as reviewable diffs with the evidence beside each; never applied automatically.
4. **A privacy check** on everything collected before anything derived is committed: a scan for text that should not be there (letters in fields that must be numbers, long strings, coordinates), and a redaction rule; derived files carry numbers only.
5. **Run it now** on the material already on the Mac (`reference/car/2026-11-15/`: car_vitals' log for tyres and boost, cluster-render's journal for the slots and uploads) so the tools are proven before the session, and so L10–L12 and the car_vitals facts are answered today where the old material allows; mark each such answer with its date.
Tests with synthetic logs for every table (and with deliberately misleading ones: the MC trap of §3.1 L1). Deliver the tools, `docs/car/CL-REPORT.md`.

### CQ — the session protocol for the owner (and the lead's runbook)
Two documents, from CA's register and CP's sets:
1. **`docs/car/SESSION.uk.md` — for the owner, in Ukrainian**, written for someone standing at the car with a phone: what to bring (a second phone to ring the car, the CarPlay cable, a USB stick with music, a tripod or a holder for photos), how long each block takes, and the blocks **in the order that needs the fewest installs and the least driving**: parked with the engine off → parked with the engine on (P) → one short guided drive → one drive with cruise and a limit sign. Each step: what to press, for how long, what to wait for, which photo or video to take (framing, exposure note: mid-tones matter, whites clip), and when to press "Позначка N" in the probe's menu. Warnings in place: the stalk button's long press may reset a trip page; never operate the menu while moving (the drive steps name a passenger or a stop); belt and door checks first.
2. **`docs/car/RUNBOOK.md` — for the lead:** before the session (which builds must be on the car: the probe app, Twin Dial 0.1.1 with its experiment keys, f30 with its keys; which need the owner's approval; the order of installs), during it (when to collect logs, which sets to switch on for which block, how to verify a block's log before the owner moves on), after it (parser, patches, which spec sections each answer unblocks, what to save to memory). A one-page checklist of all register items with a tick box.
3. **A dry run on the bench:** the protocol's parked blocks replayed against the bench with injected signals and the probe app, to prove the log contains what the parser needs for every step and that the marker lines line up.
Deliver both documents, the dry run's log and parser output, `docs/car/CQ-REPORT.md` (the time budget per block, what cannot be done in one session and why).

## 5. Order and hand-offs

All four start at once. CA publishes the register first (it starts from §3, so the others are not blocked); CP publishes the set list and the log format early (CL's parser and CQ's protocol depend on them); CQ's dry run needs CP's app and CL's parser and comes last.

## 6. What the lead does with it

Merges the four branches; asks the owner to approve the installs (the probe app; the builds carrying the experiment keys); runs the session with the owner by `SESSION.uk.md`; collects and parses; brings the answers and the patches to the owner; updates `SPEC-R3.md` (every **after probe** becomes verified, refuted or re-planned); writes the result into memory.
