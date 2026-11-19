# NV: the next turn — where the car carries it when CarPlay guides

Package NV of `docs/AGENT-BRIEF-car-issues.md`. Branch `car-issues/nv` from `main` at `8421a64`, worktree `.claude/worktrees/ci-nv`.
Two commits, Signal Probe only (plus its design table and its test). No version bump (still 0.2.0 in `app.json` and `probe.js`), no push, no car.
Evidence labels: **proven** = shown by a log line or the catalogue; **hypothesis** = not yet confirmed by the car. All times UTC (the car's clock is UTC+2).

## 0. In short

1. **What the owner saw is most likely BMW's own CarPlay manoeuvre box, a picture the head unit renders, not data the cluster draws.**
   On 2026-11-18, `cluster-render.log` shows, three times (13:02:27, 13:41:43, 18:30:31), the stock map content `0x1000300`
   (slot `0x100`) being unmapped and within 0.1 s the content `0x1000310` mapped in slot `0x103`; when the route ends
   (13:13:27, 14:15:19) the box goes and the map comes back. cluster-render's own configuration names `0x01000310` "the stock HMI's
   CarPlay maneuver panel" (class 0; slot `0x103` on this car; "VERIFIED on both cars"). At 18:30:31 the CarPlay route of the parked
   read had just started (S7 was switched on at 18:31:15 and read `guidingProvider = CAR_PLAY`). So "the turns in the place of the map"
   is the HMI's DCSM content `0x1000310` replacing the map content: **proven for 2026-11-18**, a **hypothesis for the 2026-11-19 sighting**
   until the 11-19 `cluster-render.log` is read (section 8).
2. **Data still exists somewhere on the head unit** (the HMI draws the box from something), but not in BMW's guidance fields: on 11-18
   the box was mapped from 18:30:31 and `Guidance.conclusiveDart`, `nextIntersection`, `currentLaneGuiding` and the cluster's own
   `LaneGuiding` (`maneuverShown = GUIDING_MANEUVER_NOT_DISPLAYED`) were all empty at 18:31:18 (**proven**, parked; the drive's lines
   are the lead's live reading only). The catalogue has three places the manoeuvre could travel as data that no probe has read yet
   (section 3): the projected-mode guiding service CarPlay writes into (`ProjectedModeGuiding`), the guidance engine's event lists
   (`GuidanceEventList.viewsChanged`) and its rendered contents (`GuidanceRenderedContent.viewContentChanged`, which carries a RAMSES
   scene id and the event it shows). Only the drive can tell whether any of them is filled while CarPlay guides.
3. **New set S16 "Наступний поворот"** (key `set16`, item P25): **22 new entries on 8 new interfaces** (11 fields, 11 events), plus all
   15 fields of S7 and the cluster's two view fields (`kombi-config`, `display-kombi`) through `also: ['S16']`: **39 fields on 15 proxies**
   when S16 is on alone. Read only. App total 250 entries, 94 interfaces, 50 events. `check_signals.py` passes.
4. **Twin Dial today cannot show the box in `авто`**: the map window's presence rule (S2) wants BMW's guidance `ACTIVE` or "map without
   a route", and with CarPlay BMW's guidance stays `INACTIVE`; with the slot rule (S1, `config.mapSlots`) it would even close the window,
   because the map content leaves slot `0x100` exactly when the box comes. In `карта` the window opens on the cluster's map view and the
   box would show through it **if** slot `0x103` lies under W2 (its place on the panel is not known: section 6). Decision for the lead.
5. Tests: `bench/test-signal-probe.js` **171 passed, 0 failed** (was 138/154: +S16 checks, vectors for the 22 entries, the bench build
   with the S16 checkbox). `bench/test-probes.js` 25/0, `tools/car/dry_run.js` 34/0, `tools/car/test_session.py` 12/0.
   `tools/car/test_car_tools.py` 46/1: the one failure (`test_diffs_change_only_what_they_name`, `config.tabKey`) **fails the same on
   `main`** without this branch (checked by stashing): not ours.

## 1. What was checked

- The catalogue `reference/car/2026-11-15/someip/someip-interfaces.json` (689 interfaces): every interface whose name or any field,
  event, method or nested member matches manoeuvre / turn / junction / guid / lane / roundabout / intersection / navi / projected /
  carplay / HUD / kombi / cluster / RAMSES / scene / stream; then the types of every candidate (scratch scripts, not committed).
- The probe log of 2026-11-18 (`reference/car/2026-11-18-collect1/apps/signal_probe/signal_probe.log`): S7's 16 s, parked.
- `cluster-render.log` / `.log.1` / `status.txt` of the same collect (2026-11-14 to 2026-11-18 18:35), and cluster-render's
  configuration with its comments (`reference/car/2026-11-15/cluster-render/cluster-render.conf`).
- The installed apps' manifests (`reference/car/2026-11-18-collect1/by-uid/*/package.json`, `session.json`).
- Twin Dial's tab 5 logic (`shared/twin_pages.js` `mapState`, `parseSlots`, `navPage`, `routeLive`, `dartOf`, `lanesOf`) and
  `apps/twin_dial/app.json` (`navView`, `diagMapWindow`, `config.pages`).
- `reference/hu/` holds one picture only (nothing about guidance). The 2026-11-19 logs (`reference/car/2026-11-19-collect2/`) had not
  arrived during this round.

## 2. The evidence for "a rendered picture"

### 2.1 The DCSM timeline of 2026-11-18 (`cluster-render.log`, numbers only)

| UTC | map content `0x1000300` (slot `0x100`) | box content `0x1000310` (slot `0x103`) | belt panel `0x1000314` (slot `0x103`) | cluster-render's own line |
|---|---|---|---|---|
| 13:02:27.407–.416 | map status 1 → 0 | 0 → 1 | 0 → 3 (parked) | "guide_auto: the box is mapped" |
| 13:13:27.125–.160 | 0 → 1 | 1 → 0 | stays parked | "no box mapped: not held" |
| 13:41:43.344–.426 | 1 → 0 | 0 → 1 | 0 → 3 | "the box is mapped" |
| 14:15:19.409–.460 | 0 → 1 | 1 → 0 | 3 → 0 | "no box mapped" |
| 18:30:31.879–.923 | 1 → 0 | 0 → 1 | 0 → 3 | "the box is mapped" (to the end of the log, 18:35:02) |

Each time the map leaves and the box comes within 9–82 ms, and the reverse at the end (1–44 ms): the HMI swaps its map for its box in
the centre of the cluster. cluster-render.conf (lines 148–200 of the 2026-11-15 copy): "the slot of the stock HMI's CarPlay maneuver
panel (content 0x01000310, class 0: in 0x105 on one car, in 0x103 on another ...)"; "0x01000314, the seat-belt-warning panel ... always
is [parked], behind the box, with a route; VERIFIED on the car". `status.txt` at 18:35: `slot 0x103 status 2: 0x1000314 class 1024 map 3,
0x1000310 class 0 map 1`, `target map ... idle`.

The 18:30:31 swap is 44 s before S7 was switched on (18:31:15.287) and read `guiding-provider = CAR_PLAY`, `carplay-guidance = true`,
`guidance-state = INACTIVE` (18:31:18): the box came with the CarPlay route.

### 2.2 Who else could have drawn it

- **Turn View** (`iteriodev_turnview` 1.5.1, uid 12219, with its native tool and a cluster-render drop-in) is installed. Its manifest
  says it draws the front camera with a turn card and turn signs "from BMW Maps, Apple CarPlay or Android Auto", and on iDrive 7 it is
  "off at every start of the car" until switched on in its app. On 11-18 its client made its five layers on `map` (13:01:47, 18:18:55)
  but the `map` target was **never shown** after 09:54 (no "target map: shown" line, `status.txt`: "target map ... idle"). So the 11-18
  boxes were not Turn View. If it had been on, cluster-render's guide hold would have hidden BMW's box (it holds slot `0x103` "while the
  map is shown"). **For 11-19 this must be checked**: if the owner switched Turn View on, the turns may have been its card over the front
  camera (section 8). Turn View's own claim also says the CarPlay manoeuvre is reachable as data by a native tool on this head unit
  (how, its manifest `manifest.json` would say: decision D4).
- Waze on cluster (`robertn_waze_cluster`) draws on `map` too; not shown on 11-18 after 09:54.

### 2.3 Verdict (as far as the catalogue and the logs go)

**The cluster shows a picture rendered by the head unit** (a RAMSES / DCSM content of BMW's HMI, `0x1000310`), not data the cluster
draws itself: proven for 11-18, the most likely reading of 11-19. Nothing in the catalogue lets a reader observe the picture's
transport: `RamsesService` (instances `IC_HMI`, `HU_HMI`, `Navi_HG`, `Navi_JV`) and `RICServiceGuidance` / `RICServiceGuidanceEnh`
(`ric_setManeuverData`, `ric_setManeuverListData`, `ric_setLaneGuidingData`, ...) carry it as **method calls** between the head unit
and the cluster: a client cannot subscribe to another client's calls, and calling them is forbidden. What a reader **can** try is the
data the HMI draws the box from, if it travels over SOME/IP: S16's candidates. If S16 finds them empty during a CarPlay turn, the box is
fed inside the HMI (from the CarPlay process directly), and the only way to show the turn in Twin Dial is the map window (section 6).

## 3. The candidates (catalogue of 2026-11-15)

Ranked by how likely they are to carry CarPlay's manoeuvre as data, then by what they decide. "Text" = a member that can hold a road
name / instruction / view name: it is printed as a length only. g / n / e = getter / notifier / event id.

| # | Interface [instance] | Member | Ids | Type | Text | S16 tag | Why / what it decides |
|---|---|---|---|---|---|---|---|
| 1 | `navigation.guiding.GuidanceEventList` [Main_Instance] | event `viewsChanged` | e 32770 | `ViewsUpdate`: map view name → {sequence, indexed position, `eventsInView[]` of `GuidanceEventUnion`}; car distances | view names; description texts | `event-list` | the guidance engine's event list per registered view (the HMI's cluster / HUD views): manoeuvres as **data** (intersection direction, roundabout exit, branch, destination, lane change, distance, guiding step) if CarPlay's events enter BMW's engine |
| 2 | `navigation.guiding.GuidanceRenderedContent` [Main_Instance] | event `viewContentChanged` | e 32770 | {viewId, `RenderedContent`: `ramsesSceneId`, occurrence, `contentType` (`HIGH_GUIDING`, `JUNCTION_VIEW`, ...), `associatedEvent` (`GuidanceEventUnion`), bar graph} | view id; description texts | `rendered` | the engine's **rendered** guidance content: a scene id (compare with the DCSM content ids) **and** the event it shows; if filled while the box shows, both questions are answered at once |
| 3 | `navigation.projectedmodeguiding.ProjectedModeGuiding` [Main_Instance] | fields `currentRoad`, `waypoints`, `currentLaneGuiding`, `activeRouteChange` | g 3/5/7/1, n 32770/32771/32772/32769 (setters 4/6/8/2: CarPlay's side writes them) | `Description`, `DestinationInfo`, `CurrentLaneGuiding`, enum | road text | `pm-road`, `pm-waypoints`, `pm-lanes`, `pm-route-change` | the service the projected mode **writes into** BMW's guidance. `waypoints` is the same type as `Guidance.nextDestinationInfo`, which was filled: if `pm-waypoints` mirrors it, this is the path CarPlay uses. Its manoeuvres go in by **methods** `addEventsOnRoute` (11), `updateEventsOnRoute` (13), `passEventOnRoute` (12), `setGuidanceState` (9): not observable; their effect would show in #1 / #2 |
| 4 | same | event `streamCommandChanged` | e 32773 | request / resize / stop a stream: render size, category rect, safe area (screen px) | no | `pm-stream` | the engine asking the projected mode for a **video stream** (CarPlay's map for a cluster area). A request here = the "picture" is a CarPlay video, not BMW's box |
| 5 | `navigation.guiding.Guidance` [Main_Instance] | events `streamStatusChanged`, `activeRouteChanged`; field `currentRoute` | e 32801, 32800; g 8 / n 32776 | enum `AVAILABLE/STREAMING/ERROR/UNAVAILABLE`; cause enum; session / handle | no | `stream-status`, `route-changed`, `current-route` | the other end of #4 (`STREAMING` = a CarPlay stream is on); whether BMW's engine holds a route object while CarPlay guides |
| 6 | `a4a.carplay.CarPlayManager` [CarPlayManagerInst1] | `projectedGuidanceMetadataAvailable`; event `onScreenChange` | g 10 / n 32775; e 32773 | bool; `NATIVE/PROJECTED` | no | `cp-metadata`, `cp-screen` | whether the phone sends route-guidance **metadata** at all (the iAP2 feed a native box needs); who owns the centre screen |
| 7 | `am.carplay.CarPlay` [CarPlayAMInst1] | event `changeModes` | e 32769 | app states (`AS_TURN_BY_TURN` + flag), resources | `reasonStr` (skipped) | `cp-modes` | the moment CarPlay's turn-by-turn mode starts / stops: the clock for the drive's notes |
| 8 | `remoteinstrumentcluster.RICServiceGuidance` [1] | event `ric_sendEvent` | e 32770 | handle, event id / type, `ManeuverListEvent` (ranges), `DisplaySwitchEvent` (`HMI_GUIDING_LABELS_MAP`, `HMI_ROADBOOK_*`) | no | `ric-guidance` | the cluster side of the remote-cluster guidance widget (the methods carry the data the other way): a display switch or list event while the box shows would place the box on this path |
| 9 | `remoteinstrumentcluster.DisplaySink` [1] | event `requestSinkSwitch` | e 32769 | aspect ratio, resolution, display selection | no | `display-sink` | a display-sink switch (a video sink in the cluster) at the moment the box or a stream appears |
| 10 | `navigation.NavigationCAN` [1] | `calculationDataNavigation` | g 1 / n 32769 | distance to destination + unit, arrival h / min | no | `nav-can` | the old CAN feed to the cluster: does it carry CarPlay's destination too (like `nextDestinationInfo`)? |
| 11 | `FeedbackLaneGuiding` [1] | event `correctionDataLaneGuiding` | e 32769 | best / highlighted lanes, distances to the next manoeuvre | no | `lane-feedback` | the lane-guidance correction (camera ↔ navigation); a distance-to-manoeuvre that moves during a CarPlay turn would be data |
| 12 | `sdi.aap.AndroidAutoProjectedManager` [AndroidAutoProjectedManagerInst1] | `projectedGuidanceMetadataAvailable`, `isVideoStreaming`, `featureConfiguration` | g 3, 1, 7 / n 32771, 32769, 32775 | bool, bool, {metadata, vehicle status} | no | `aa-metadata`, `aa-video`, `aa-features` | Android Auto's twins (not this car's phone; cheap, for completeness) |
| — | `Guidance` (S7, now also S16) | `conclusiveDart`, `nextIntersection`, `currentLaneGuiding`, `currentRoad`, `nextDestinationInfo`, `guidanceState`, `guidingProvider`, `isOnGuidableRoad`, map-without-route, 3D junction | as S7 | | road text | S7 tags | **empty with CarPlay on 11-18 (proven parked, live-read on the drive)**; the reference for a BMW route |
| — | `LaneGuiding` [1] (S7) | `laneGuiding`, `extendedLaneGuiding` | g 1, 2 / n 32769, 32770 | lane bits, distances, `statusDisplayGuidingManeuver`, `displayedManeuverID` | no | `kombi-lanes`, `kombi-lanes-ext` | the cluster's own manoeuvre display status: `NOT_DISPLAYED` while the CarPlay box was mapped (18:31:18): it describes BMW's own manoeuvre display, not the CarPlay box |
| — | `KombiIF` / `Vehicle2` (S6) | `statusConfigurationKombi5`, `statusDisplayKombi` | as S6 | cluster centre view (`KARTENANSICHT` on 11-18) | no | `kombi-config`, `display-kombi` | the view switch the box depends on |

Searched and **not read** (why):

| Interface | What it is | Why not |
|---|---|---|
| `remoteinstrumentcluster.RICServiceGuidance` / `RICServiceGuidanceEnh` methods (`ric_setManeuverData`, `ric_setManeuverListData`, `ric_setGuidingData`, `ric_setLaneGuidingData`, `ric_trans*`) | the head unit pushing guidance models to the cluster's widget | methods: only the caller and the provider see them; calling them is a side effect (forbidden) |
| `ramsesservice.RamsesService` (instances `IC_HMI`, `HU_HMI`, `Navi_HG`, `Navi_JV`, tests) | the RAMSES transport itself: `Navi_HG` / `Navi_JV` = the navigation's high-guiding and junction-view scenes | methods only; the scenes are pictures (scene graphs), cluster-render already sees the DCSM side |
| `distributedrendering.SceneProvider` events `newScenesAvailable` / `scenesBecameUnavailable` | scene ids + friendly names, instances 1–8 | which instance is the navigation's is unknown (8 proxies); names are text; cluster-render's `dcsm:` lines already give the content ids. Candidate for a later round if #2 is silent |
| `displaydataserver.DisplayDataServer`, `displaydata.DisplayDataServer2` | the HUD's feed (S8 / S14 read it) | **no navigation member in either struct** (only lane-keeping arrows): the HUD's turn arrows, if any, do not come this way |
| `KombiIF` (58 fields) | the cluster's settings | only the navigation display type (`kombi-config`, already read) |
| `FASuS.statusDisplayFASManeuver`, `displayForesight2.laneGuidingManeuverID` | a driver-assistance manoeuvre display; foresight | unrelated to navigation turns; `displayForesight2` is read whole in S14 |
| `HMIControlAD.controlNavigationVisualization` | automated driving's road clearance | unrelated |
| `navigation.mapcontrol.MapControl`, `mapsettings.MapSettings` | the centre display's maps | no cluster map type in `MapType`; positions |
| `carplayproperties.CarPlayProperties`, `projectedmodesproperties.ProjectedModesProperties` | text → text property maps | text only: keys could say whether a cluster map stream is configured, but the rule prints texts as lengths (decision D5) |
| `NavigationStatus.nextDestination` / `currentLocation`, `Positioning`, `MapControl.currentMapPosition` | addresses, coordinates | positions: never |
| `speech.voiceguiding.VoiceGuiding`, `sdi.projected.SpeechProjectedMode` | BMW's voice; Siri | no manoeuvre |

## 4. Certain from the catalogue and the logs, and what only the drive can tell

Certain:
- BMW's guidance fields that S7 reads have no manoeuvre while CarPlay guides (11-18, parked; the drive read live). `nextDestinationInfo`
  is CarPlay's destination as BMW's data.
- The cluster's "turns in the place of the map" on 11-18 are the HMI's DCSM content `0x1000310` in slot `0x103` replacing the map
  content `0x1000300` in slot `0x100` (section 2.1).
- The catalogue has a projected-mode path into BMW's guidance engine (`ProjectedModeGuiding`: fields CarPlay writes, events-on-route
  methods), an engine output per view (`GuidanceEventList`, `GuidanceRenderedContent`), and a stream path (`ProjectedModeGuiding.
  streamCommandChanged` ↔ `Guidance.requestStream` / `streamStatusChanged`). None of them has been read on the car.
- The picture's transport (RAMSES, RIC methods) is not readable by an app.

Only the drive can tell:
- whether `GuidanceEventList.viewsChanged` / `GuidanceRenderedContent.viewContentChanged` carry CarPlay's manoeuvres (→ data, our own arrow);
- whether `ProjectedModeGuiding` is filled (→ the path), and `pm-waypoints` matches `next-destination`;
- whether a stream is requested (`pm-stream`, `stream-status = STREAMING`) (→ the box is a CarPlay video, not BMW's drawing);
- whether `cp-metadata` is true (→ the phone sends metadata at all);
- the same chain with BMW's own route (S7 should fill; does `rendered` show `HIGH_GUIDING` with the same scene id as a DCSM content?).

If S16 is silent or empty for every candidate during a CarPlay turn, the honest answer is: **no interface an app may read carries
CarPlay's manoeuvre**; what is left is the destination block Twin Dial already shows (remaining distance, arrival, travel time, road)
and the map window showing BMW's box (section 6).

## 5. The set S16 in Signal Probe

- `apps/signal_probe/fields.js`: 22 entries (`set: 'S16'`, `p: 'P25'`) and `also: ['S16']` on all 15 S7 fields and on `kombi-config`,
  `display-kombi`; the set `{ id: 'S16', key: 'set16', name: 'next turn', uk: 'Наступний поворот' }`; defaults gap 2000 ms, cap 400.
- `apps/signal_probe/format.js`: `eventParts` (the manoeuvre part of `fmtDart`, unchanged output), `mapEntries` (a map as an object,
  `{key, value}` / `[key, value]` pairs or a `Map`: the binding's shape is unknown), `fmtViews`, `fmtRendered`, `fmtStream`;
  `fmtTree` may take a field's own mixed-case enum pattern (`enumNames`, only `ric-guidance`: `^(Event|Model)(_[A-Za-z0-9]+)*$`,
  checked against all 67 enum names its type can carry, and that it lets no free text through).
- `apps/signal_probe/probe.js`: the three formatters; an event list goes out in parts of at most 1000 characters like the tree values.
- `apps/signal_probe/app.json`: `signals` synced (`tools/car/probe_fields.js sync`), the checkbox `set16` "S16 Наступний поворот (P25)".
- `design/car/probe_fields.json` re-exported; `bench/signal-probe-vectors.json` + 22 vectors (canaries in every text, coordinates and
  a bearing in the values: none reaches a line).

Example lines (from the vectors; numbers invented):
```
probe: P25 event-list #1 t=..: views=1 shape=object [id-len=17 seq=4 valid=true offset=0 total=7 events=4 {intersection direction=LEFT geometry=TWO_WAY_FORK_TAKE_LEFT dist-len=3 unit=METER raw=350 step=GS2 ...} {roundabout exit=2 angle=270 side=RIGHT ...} {branch branch=HIGHWAY_OUT_RIGHT ...} +1 more] toFinal=5700 toNext=5700 travelled=1250 sinceDest=1250
probe: P25 rendered #1 t=..: view-len=17 scene=4097(0x1001) occurrence=12 seq=4 type=HIGH_GUIDING options=ALWAYS_ON+BAR_GRAPH bar=60 labels=LEFT tags=3 event={intersection direction=RIGHT ...}
probe: P25 pm-stream #1 t=..: command=requestStream size=1088x480 rect=0,0+1088x480 safe=rect 40,20+1008x440
probe: P25 stream-status #1 t=..: streamStatus=STREAMING
probe: P25 cp-modes #1 t=..: appStates=1 [appStateID=AS_TURN_BY_TURN stateFlag=true speechMode=SM_NONE] resources=1 [...]
```

Privacy: road, instruction and view names, the CarPlay reason text: lengths or skipped; waypoint position and `directionToDestination`
(a bearing), road-link coordinates, source / POI ids: never printed (tested). Read only: `r` only with a non-zero getter, `s`, events as
`_broadcasts`; no `w`, no method. `ProjectedModeGuiding`'s fields have setters (CarPlay's side writes them): declared `rs`, never `w`.

Volume (the kit's log turns over at 256 KB, one older file kept): moving distances are slowed down: event list every 5 s (cap 300),
rendered content 2 s, projected waypoints and lane feedback 5 s, CAN feed 10 s, rare events 250 ms. Estimate from the vectors' line
lengths: **about 0.5 KB/s with a CarPlay route** (BMW's manoeuvre fields static), **about 1 KB/s with BMW's route** (S7's manoeuvre
fields move too): one log file per 4–8 minutes. Collect after each leg, or the lead reads live (`tail -F`). The event list is the
longest line (about 670 characters per view; at most 4 views x 3 events, then parts).

Not verified: the firmware's own generator was not run (its copy from the car is not here); the app now declares 94 interfaces, about
116 interface configs with the platform's 22 (0.2.0: 86 / about 108; the largest install known to have worked is 0.1.0's 62 / 84,
which ran on 11-18). Whether a map arrives as an object, pairs or a `Map` is
unknown (all handled). Whether subscribing the remote-cluster events (`RICServiceGuidance`, `DisplaySink`) is answered at all.
`Guidance`'s two events are subscribed on the same proxy as its fields (one watch; precedent: Driver Profiles' `UserAccount` mixes fields
and an event and installs). With S7 on first, S16 adds the two events to the running proxy with `on()` (risk R5 of CP-REPORT: no car
precedent for a late `on`); **switch S16 on alone, before S7**.

## 6. Twin Dial: the map window and the arrow page

**The window's presence rule against the box** (`shared/twin_pages.js` `mapState`, Twin Dial 0.1.3, `config.mapWindow` from
Діагностика "Вікно карти", default off):
- `авто` (default): opens only when the cluster is in map view **and** (BMW's `guidingState = ACTIVE` or "map without a route"). With
  CarPlay BMW's guiding is `INACTIVE` (CF3) and "map without a route" read `false` (11-18): **the window stays shut**, tab 5 shows our
  CarPlay route page. The box is not visible.
- `карта`: opens whenever the cluster is in map view (`statusNavigationDisplayType = ...KARTENANSICHT...`, which the car reported on 11-18):
  **the window opens and BMW's box shows through it if slot `0x103` lies under W2** [700, 173, 1220, 600]. The box's place and size on
  the panel are not known (cluster-render only knows our 16x16 hold scene for that slot). The owner's sighting ("in the place of the
  map") suggests the centre area. Photo test in section 7.
- `config.mapSlots = true` (S1, the slot rows; off today): `parseSlots` reads slot `0x100` only, and the box comes exactly when the map
  content leaves slot `0x100` → "slot 0x100 has no map" → **the window would close for the box**. If the window is to show the box, S1
  must also count slot `0x103` (or `0x105`) with content `0x1000310` mapped (one regex; Twin Dial's file, not this package: D2).
- cluster-render's guide hold hides BMW's box only while its `map` target is shown (Waze on cluster, Turn View); Twin Dial draws on
  `panel` and does not trigger it.

**The arrow page with BMW's own route** (`config.pages.guidance`, off, "dark until tested"; tab 5 shows `guide` when
`NavigationStatus.guidingState = ACTIVE` and the setting is on). What it reads, and what S7 must show on the drive for it to be switched on:

| Twin Dial needs | From (S7 tag) | What the drive must show |
|---|---|---|
| `guidingState = ACTIVE` | `guiding-state` (`NavigationStatus`) | `ACTIVE` during the BMW route (it said nothing at all in 16 s on 11-18: never seen) |
| the manoeuvre kind | `dart` (`Guidance.conclusiveDart.maneuver`): `intersection.direction` (`TurnDirection`), `branch.branchType`, `roundabout.exitNumber` + `driveDirection`, `destinationManeuver.position`, `laneChange.changeSide` | a `kind=` other than `pointOfInterest` before each turn; the enum names as the catalogue's (Twin Dial's tables match the catalogue's numbers if numbers come) |
| the distance | `dart` `formattedDistance.rawDistance` | `raw=` counting down to the turn, in **metres** whatever `unit=` says (CF3 proved metres for `nextDestinationInfo` only) |
| the roundabout exit | `dart` `exit=` | 1–6 (icons exist for 1–6 per side) |
| the lanes | `lanes` (`currentLaneGuiding.rows[0].lanes[].directionToFollow` / `isHighlighted`) | `rows>=1` with `*` on the recommended lanes near junctions |
| the road | `road` (`currentRoad`) | `elements>=1` (drawn, never logged) |
| arrival / distance | `next-destination` | as with CarPlay |

If S16 shows CarPlay's manoeuvres in `event-list` or `rendered`, they are the same `GuidanceEventUnion` that `dartOf` already reads:
the CarPlay route page could gain an arrow by reading the first event of the cluster's view (D3).

## 7. The drive protocol for the owner

Before: Signal Probe 0.2.x with S16 installed (the lead's build); **Turn View off** and **Waze on cluster off or HUD only** (either one
hides BMW's box); the cluster's centre display set to **Map view** (iDrive: Instrument cluster > Central display area). A passenger
operates the phone and the probe menu; the driver never touches the menu while moving. Write down the **car clock time** (UTC+2) and
what the cluster shows at every step (no street names: "left turn 300 m", "roundabout exit 2").

Leg A — CarPlay route (5 minutes, 2–3 turns, a roundabout if one is near):
1. Parked, engine running. Signal Probe: only **S16** on (it includes S7). Press "Позначка" → M1. Note the time.
2. Start the route in CarPlay (Apple Maps or Google Maps: note which). Wait until the cluster shows the turn box. Mark M2, note the time,
   photograph the cluster (whole panel, with BMW's own cluster theme, not Twin Dial: the box's place on the panel is wanted).
3. Drive. At each turn: the passenger marks just before it (M3, M4, ...) and notes the manoeuvre and the distance the box shows; one
   photo of the box with a distance on it. At a roundabout: note the exit number shown.
4. After the last turn, stop the route in CarPlay; mark; note whether the map came back.
5. Parked: the lead collects `signal_probe.log` and `.log.1` and `cluster-render.log` (or reads them live during the leg).

Leg B — BMW navigation route (5 minutes, 2–3 turns):
1. Parked. S16 stays on. Mark. Start a route in BMW's own navigation (iDrive) to a nearby destination. Mark when the cluster shows the
   first guidance. Photograph the cluster.
2. Drive the same way: mark before each turn, note the manoeuvre and distance shown; one photo with an arrow.
3. End the route; mark; collect as above.

Optional Leg C — Twin Dial: Twin Dial on, tab 5, Діагностика "Вікно карти" = W2, Навігація = **карта**, a CarPlay route: photograph
whether the box shows through the window. Then Навігація = авто: the CarPlay route page (no box).

The lead reads, per leg: S16's `cp-metadata`, `cp-modes`, `pm-*`, `event-list`, `rendered`, `stream-status`, `pm-stream`, `ric-guidance`,
`display-sink`, `nav-can`, `lane-feedback`, and S7's `dart` / `next-intersection` / `lanes` / `guiding-state` lines against the marks;
in `cluster-render.log` the `dcsm:` lines of contents `0x1000300`, `0x1000310`, `0x1000314` and `guide_auto`; the +30 s / +120 s summaries
(which providers are up, which fields stayed silent; the +120 s getter asks for silent fields).

## 8. What the 2026-11-19 logs would settle (when `reference/car/2026-11-19-collect2/` arrives)

- `cluster-render.log` about 12:00–12:30 UTC: a `content 0x1000310 map status 0 -> 1` with `content 0x1000300 map status 1 -> 0` at the
  time the owner saw the turns = BMW's box (the hypothesis of §0.1 confirmed); a `target map: shown` with client `iteriodev-turnview` =
  Turn View's card instead.
- Turn View's app log or the journal: whether it was switched on.
- Any `signal_probe.log` with S7 lines while the box was mapped: the same emptiness, or not.

## 9. Decisions for the lead

- **D1** Merge and install: S16 adds 8 interfaces (94 declared, about 116 interface configs with the platform's 22): no car precedent
  for that many (0.1.0's 84 configs is the largest known to have installed; 0.2.0's 108 not yet seen on the car). Fallback if the install
  fails: drop `ric-guidance`, `display-sink`, `nav-can` and `lane-feedback` (four interfaces fewer) by deleting their table lines and
  running `node tools/car/probe_fields.js sync` (the tests' counts follow the table except the fixed totals 250 / 95 / 43).
- **D2** Twin Dial's map window and the box: (a) leave as is (the owner chooses `карта` to see the box through W2); (b) make S1's slot
  rule count the box in slot `0x103`/`0x105` (needs `config.mapSlots`, a Twin Dial change); (c) let `авто` open the window during a CarPlay
  route when the box is mapped. Anything that changes Twin Dial's default needs the owner's word. Wait for Leg C's photo first.
- **D3** If S16 shows CarPlay's events as data: a Twin Dial arrow on the CarPlay route page from `GuidanceEventList` (dark until tested).
- **D4** Read Turn View's SOME/IP manifest (`manifest.json` of uid 12219, interface names only, nothing else of its code): it reads
  CarPlay's manoeuvre somehow, and its declarations would name the path in one look.
- **D5** `CarPlayProperties` / `ProjectedModesProperties` (text → text maps): print their **keys** (identifier-like, e.g. a cluster-map
  flag) or keep the rule "texts as lengths only" (not read today).
- **D6** `SceneProvider` scene announcements (8 instances, friendly names as text) for a later round if `rendered` is silent.

## 10. Hand-off

- Branch `car-issues/nv`: `aa5a04b` (S16), `78e94a8` (S16 rates). Touches only `apps/signal_probe/*`, `bench/test-signal-probe.js`,
  `bench/signal-probe-vectors.json`, `design/car/probe_fields.json`. No shared file, no other app.
- `bake/work/car-issues/pub/NV/READY`.
