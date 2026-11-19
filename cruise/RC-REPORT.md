# RC report: the recorder (Phase 0) and the signal map (Phase 1, desk part)

2026-11-19, agent RC, branch `cruise/rc` (worktree `.claude/worktrees/rc`). Brief: `docs/cruise-set-speed-brief.md`. Nothing here was run on the car; every "on the car" statement below is a hypothesis until the first recorded drive.

## 0. In short

- **Signal map.** The catalogue (`reference/car/2026-11-15/someip/someip-interfaces.json`) has the **brake pedal** (driver braking from the DSC, twice, and the brake-light switch), the **accelerator pedal** (angle, and a flag "driver demand above / below the assistance system's"), the **four wheel speeds**, five ego-speed sources, three longitudinal-acceleration sources (plus two more), three grade sources. It has **no cruise buttons**: the steering-wheel cruise keys go to the DSC, the head unit only sees their effect (the set speed and the cruise state). 13 fields were added to S14 (read only: `s`, `rs` with a non-zero getter, events as `_broadcasts`), S10's gear is also read with S14; `check_signals.py`: all declarations pass (99 interfaces, 5 of them new).
- **Recorder.** A recording mode inside Signal Probe (`apps/signal_probe/recorder.js`): menu rows **"Почати запис S14"** (start / stop) and **"Набір для запису"** (S14, S14+S15, S13), the existing **"Позначка N"** row writes a marker into the recording. Every value of every field of the chosen set, unthrottled, one CSV line per sample, gzip, files of 4 MB of text in `data/rec/`, a 15 MB quota with the oldest files removed, a full disk stops the recording and never the app.
- **Budget (S14, 58 fields, plausible native rates, 1 142 samples/s): 35 KB/s of text, 8.8 KB/s on disk, about 30 minutes in the 15 MB quota.** Twice those rates: 17 KB/s, about 15 minutes. CPU on the Mac 4.2 ms per second (about 2–3 % of one core of the head unit's Atom A3960, estimated). The binding limit is the app's own disk limit (`maxDiskUsage` 20 MiB), not the head unit's free space (9.7 GB on `/var/opt`).
- **Converter.** `tools/car/rec_convert.py`: one table per recording (sparse, or on a common clock with `--resample MS`), per-field CSVs, markers, and a report of each field's native rate, jitter, gaps, the recorder's drops, the car's missed messages (from the payload counters) and the payload clock against the receive clock.
- **Protocol** for the first drive: `docs/cruise/DRIVE-PROTOCOL.md` (Ukrainian, then English).
- Tests: `bench/test-signal-probe.js` 175 / 0, `bench/test-signal-recorder.js` 60 / 0 (a 20-minute simulated drive, failures, the real app on the bench with a burst of injected values), `tools/car/test_rec_convert.py` 9 / 0.

## 1. The signal map

"Units" below are the catalogue's: it gives **no units and no scaling** for any member (only names, integer widths, floats, enums). A unit is written only where the kit already uses a scaling verified on the car (`kit/runtime/car.js`), or where the type is a float whose name says it; everything else is "raw" until the drive calibrates it.

### 1.1 Brief signal → catalogue member

| Brief signal | Probe tag (set) | Interface . member | Field / event | Getter / notifier / event id | Type | Unit (source) |
|---|---|---|---|---|---|---|
| **Brake pedal: driver braking** | `brake-driver` (S14, new) | `de.bmw.chassis.VehicleStabilizationManagement` . `StabilizationManagement` . `statusBrakingDriver` (+ `statusBrakingMSA`, `qualifierFunctionFDR/ABS/ASC`, `statusUnitRunDSC`) | field | – / 32769 | UInt8 | raw (a flag or a level; to be seen) |
| Brake pedal, infotainment copy | `brake-driver-hu` (S14, new) | `de.bmw.infotainment.vehicle.A4A` . `statusA4AFDRControl` . `statusBrakingDriver`, `qualifierFunctionFDR` | field | 1 / 32769 | UInt8, UInt16 | raw |
| Brake-light switch (pedal touched) | `brake-switch` (S14, new) | `de.bmw.powertrain.Powertrain` . `InformationVehicleDrive` . `dummy2` (type `StatusBrakeLightSwitch`: `KEINE_BLS_MINUS_BETAETIGUNG` 0 / `BLS_MINUS_BETAETIGUNG` 3 / faults), `dummy3` clutch, `dummy4` MSA button | field | – / 32773 | enum | enum |
| Brake light lit | `brake-light` (S14, new) | `de.bmw.body.Light` . `LampExterior` . `controlFunctionBrakeLight` (`OFF_BL` / `ON_BL`), `controlFunctionBrakeForceDisplay` | field | – / 32770 | enum | enum |
| **Accelerator pedal** | `accel-pedal` (S14, new) | `de.bmw.powertrain.AcceleratorPedal` . `AngleAcceleratorPedal` . `actualValueAngleAcceleratorPedal`, `...Virtual`, `gradientActualValueAngleAcceleratorPedal`, `qualifierActualValueAngleAcceleratorPedal`, `dummy2` (type `StatusInterfaceDriverAssistenceSystem`: driver demand larger / smaller than / no assistance-system demand) | field | – / 32769 | UInt16, enum | raw angle (BMW: usually 0.025 %, unverified) |
| **Driver override of the cruise** | `drive-torque` (S14, new) | `de.bmw.powertrain.Powertrain` . `WheelMomentDrive4` . `statusInterfaceDriverAssistenceSystemTargetLimit` (`FAHRPEDAL_GETRETEN__FAHRERANFORDERUNG_GROESSER_FAS...` = pedal pressed, driver demand above the cruise's), driving direction wish, drivetrain engaged, rear-axle speed | field | – / 32771 | enum, UInt16 | enum |
| Override, display side | `longitudinal` (S8, also S14) | `TrafficSignDisplayControl` . `displayDriverAssistanceLongitudinal` . `dispXCC.acceleratorPedalPressed`, `dispXCC.brakeOnlyModeActive`, `valueRequestedSpeed` | field | – / 32772 | Bool, UInt16 | – (the interface was down on 2026-11-18) |
| **Cruise buttons** | — | **not in the catalogue.** The MFL cruise keys go to the DSC; `SteeringWheelPushButton` (S6) has only the infotainment keys. A press shows as a set-speed change (below) | — | — | — | — |
| **Cruise set speed** | `hud-data`, `hud-data2`, `ldm-guidance`, `assist-avail`, `display-speed`, `ad-velocity` | `DisplayDataServer.functionalLibraryData.cruiseCtrlSpeedHUD`; `DisplayDataServer2.functionalLibraryData.ldmSetSpeed.ldmSetSpeedValue` (seen: km/h, 65535 none); `FASuS.displayDriverAssistenceSystemLongitudinalGuidance.valueRequestSpeed`; `MultimodalControlDriverAssistance.functionAvailabilityDriverAssistance.valueSetSpeed` | fields | 1/32769; 1/32769; –/32770; 2/32769 | UInt16 | km/h (DisplayDataServer2, seen) |
| **Cruise state** | same | `ldmIconId` (DCC / SLD / ...), `ldmIconState` (`ACTIVE` / `STANDBY` / `OVERRIDE` / `BRAKE_ONLY` / ...), `ldmSetSpeedColour`, `activeFunctionDriverAssistenceSystemLDM` (`XCC_AKTIV` / `SLD_AKTIV`), `proposalAvailabilityDCC` | fields | as above | enum | enum. `OVERRIDE` is the display's own override state: compare it with `drive-torque` |
| **Wheel speeds** | `wheel-rpm` (S14, new) | `de.bmw.powertrain.Powertrain` . `ActualValueRPMWheel` . `actualValueRPMWheelFLH`, `actualValueRPMwheelFRH` (sic), `...RLH`, `...RRH`, four qualifiers, `dummy` = `StatusRPMSource` | field | – / 32770 | UInt16 | raw rpm (scale to calibrate against the speed) |
| **Ego speed** (5 sources) | `speed` (S14) | `de.bmw.chassis.SpeedAcceleration` . `vehicleSpeed` . `speedVehicleCentreOfGravity` + standstill, qualifier | field | 2 / 32783 | UInt16 | raw |
|  | `dynamics-long` (S14) | `SpeedAcceleration` . `VehicleDynamicDataLongitudinalAndMassVehicle` . `vehicleSpeedLongitudinal` (+ `longitudinalAccelerationDrivingDirection2`, `emptyMassVehicle`) | field | – / 32781 | UInt16 | raw |
|  | `velocity` (S14, new) | `de.bmw.powertrain.vehicledynamics.VelocityAndVehicleConditionProvision` . `velocityVehicle` . `velocityVehicleCog.velocityVehicleCog` | event | 32792 | Float | m/s by the name of a float (to confirm) |
|  | `velocity-long` (S14, new) | same . `velocityVehicleLongitudinal` . `velocityVehicleLongitudinal.velocityVehicleLongitudinal` | event | 32794 | Float | m/s (to confirm) |
|  | `car-speed2` (S14, new) | `de.bmw.infotainment.vehicle.Vehicle2` . `statusCarSpeed2` . `speedVehicleCentreOfGravity` | field | 8 / 32776 | UInt16 | **km/h × 1/64** (kit `car.js`, read on the car by f30_cluster and Twin Dial) |
|  | `display-speed` (S14) | `StatusDisplayXADAndSpeed.statusDisplaySpeed.displayedValueSpeed` | field | – / 32769 | UInt16 | the displayed speed |
| **Longitudinal acceleration** | `accel-long` (S14) | `SpeedAcceleration` . `LongitudinalAccelerationCentreOfGravity` . `longitudinalAccelerationCentreOfGravity` (+ error amplitude, qualifier) | field | – / 32771 | UInt16 | raw |
|  | `accel-long-cog` (S14) | `de.bmw.chassis.AccelerationLongitudinalLateralProvision` . `accelerationLongitudinalCog` | event | 32774 | Float | m/s² (to confirm) |
|  | `accel-long-est` (S14) | same . `vehicleAccelerationLongitudinalEstimated` | event | 32795 | Float | m/s² (to confirm) |
|  | `cds-accel-long` (S14) | `de.bmw.infotainment.vehicle.CDS2` . `statusLongitudinalAccelerationMassCenter` | field | 43 / 32811 | UInt16 | **m/s² = raw × 0.002 − 65** (kit `car.js`, used on the car) |
|  | `dynamics-long` (S14) | `longitudinalAccelerationDrivingDirection2` (above) | field | – / 32781 | UInt16 | raw |
| **Road grade** | `gradient` (S14) | `de.bmw.chassis.SurfaceInformationProvision` . `longitudinalGradientSurface` (+ `...Fast`) | event | 32794 | Float | % or rad (to confirm) |
|  | `grade-xview` (S14, new) | `de.bmw.infotainment.drivingexcitement.XView` . `statusGradientRoadway` . `actualValueLongitudinalGradientRoadway` | field | 2 / 32770 | UInt16 | raw (the infotainment path: the likeliest to be up) |
|  | `lane-slope` (S14, new) | `de.bmw.chassis.VehicleModel` . `LaneSlope` . longitudinal, longitudinal fast, transverse | field | – / 32777 | UInt16 | raw |
| Engine speed | `engine-rpm` (S14, new) | `de.bmw.infotainment.vehicle.CDS` . `statusEngineSpeed2` . `actualValueRPMEngineCrankshaft` | field | 25 / 32793 | UInt16 | **rpm × 0.25** (kit `car.js`) |
| Gear | `gearbox-status` (S10, now also S14) | `de.bmw.infrastructure.VehicleInformation` . `statusGearboxDrive` | field | – / 32776 | enum | P/R/N/D, G1..G10 |

Not added, and why: `CDS.statusHandBrake`, `VehicleStabilizationManagement.StandStillFunction` (parking brake, auto hold: below the cruise's range); `MassProvision.massVeh` (S14 already reads `emptyMassVehicle`); `DrivingVector`, `CentralOdometryProvision.odometryValues` (the latter carries the car's own x / y odometry position: left out for privacy); `CDS.datenFahrdynamiksensorErweitert` (roll / pitch rate: S14 has both). The setters (`MultimodalControlDriverAssistance.multimodalControlDriverAssistance`, `FASuS.iDriveFAS`, ...) are never declared.

### 1.2 Counters and time stamps inside the S14 payloads

Of the 58 fields S14 reads (its own 50, the seven of S8 it shares, S10's gear), these carry a counter or a time stamp. The recorder puts the counter in its `ctr` column and the time stamp in `pts` (`<seconds>.<nanoseconds>`, exactly as sent); every other member is a value column. The full table of all 58 (with the member counts) is `bake/work/cruise/pub/RC/s14-counters.md`.

| Tag | Interface . member | Kind | Ids | Counter (`ctr`) | Time stamp (`pts`) |
|---|---|---|---|---|---|
| `longitudinal` | TrafficSignDisplayControl.displayDriverAssistanceLongitudinal | field | g - / n 32772 | `counter` | — |
| `ldm-guidance` | FASuS.displayDriverAssistenceSystemLongitudinalGuidance | field | g - / n 32770 | `aliveDisplayDriverAssistenceSystemLongitudinalGuidance` | — |
| `gearbox-status` | VehicleInformation.statusGearboxDrive | field | g - / n 32776 | `alivStatusGearboxDrive` | — |
| `keep-dist-1` | KeepDistanceObjectsLegacy.keepDistanceObjects1 | event | event 32769 | `keepDistObjects1.keepDistanceObject1SyncCounter` | — |
| `keep-dist-2` | KeepDistanceObjectsLegacy.keepDistanceObjects2 | event | event 32770 | `keepDistObjects2.keepDistanceObject2SyncCounter` | — |
| `keep-dist-params` | KeepDistanceObjectsLegacy.keepDistanceParameters | event | event 32771 | `keepDistParameters.keepDistanceParameterSyncCounter` | `keepDistParameters.KeepDistObjNanoseconds`, `keepDistParameters.KeepDistObjSeconds`, `keepDistParameters.keepDistTimeSync` |
| `view-input` | AssistedDrivingViewFunctionInput.assistedDrivingViewFunctionInputIbVip | event | event 32769 | — | `assistedDrivingViewStdTimestampNanoseconds`, `assistedDrivingViewStdTimestampSeconds`, `assistedDrivingViewStdTimestampSyncState` |
| `display-speed` | StatusDisplayXADAndSpeed.statusDisplaySpeed | field | g - / n 32769 | `counter` | — |
| `display-xad` | StatusDisplayXADAndSpeed.statusDisplayXAD | field | g - / n 32770 | `counter` | — |
| `ad-velocity` | HMIControlAD.desiredDisplayedVelocity | field | g - / n 32770 | `counter` | — |
| `ad-state` | HMIControlAD.stateAD | field | g - / n 32771 | `counter` | — |
| `warn-brake` | FASuS.requestWarnBrakeCoordinatorDisplay | field | g 65 / n 32833 | `aliveRequestWarnBrakeCoordinatorDisplay` | — |
| `speed` | SpeedAcceleration.vehicleSpeed | field | g 2 / n 32783 | `aliveSpeedVehicle` | — |
| `accel-long` | SpeedAcceleration.LongitudinalAccelerationCentreOfGravity | field | g - / n 32771 | `aliveLongitudinalAccelerationCentreOfGravity` | — |
| `accel-lat` | SpeedAcceleration.LateralAccelerationCenterOfGravity | field | g - / n 32770 | `aliveLongitudinalAccelerationCentreOfGravity` | — |
| `yaw-speed` | SpeedAcceleration.YawSpeedVehicle | field | g - / n 32777 | `aliveYawVelocityVehicle` | — |
| `dynamics-long` | SpeedAcceleration.VehicleDynamicDataLongitudinalAndMassVehicle | field | g - / n 32781 | `aliveVehicleDynamicsDataLongitudinal1` | — |
| `accel-long-cog` | AccelerationLongitudinalLateralProvision.accelerationLongitudinalCog | event | event 32774 | `counter` | — |
| `accel-lat-cog` | AccelerationLongitudinalLateralProvision.accelerationLateralCog | event | event 32773 | `counter` | — |
| `accel-long-est` | AccelerationLongitudinalLateralProvision.vehicleAccelerationLongitudinalEstimated | event | event 32795 | `counter` | — |
| `yaw-rate` | YawRateProvision.yawRate | event | event 32791 | `counter` | — |
| `gradient` | SurfaceInformationProvision.longitudinalGradientSurface | event | event 32794 | `counter` | — |
| `accel-pedal` | AcceleratorPedal.AngleAcceleratorPedal | field | g - / n 32769 | `aliveAngleAcceleratorPedal` | — |
| `brake-driver` | VehicleStabilizationManagement.StabilizationManagement | field | g - / n 32769 | `aliveStatusStabilisationDSC` | — |
| `brake-switch` | Powertrain.InformationVehicleDrive | field | g - / n 32773 | `aliveInformationVehicleDrive` | — |
| `brake-light` | Light.LampExterior | field | g - / n 32770 | `aliveControlLampExterior` | — |
| `drive-torque` | Powertrain.WheelMomentDrive4 | field | g - / n 32771 | `aliveWheelMomentDrive4` | — |
| `wheel-rpm` | Powertrain.ActualValueRPMWheel | field | g - / n 32770 | `aliveActualValueRPMWheel` | — |
| `velocity` | VelocityAndVehicleConditionProvision.velocityVehicle | event | event 32792 | `counter` | — |
| `velocity-long` | VelocityAndVehicleConditionProvision.velocityVehicleLongitudinal | event | event 32794 | `counter` | — |
| `lane-slope` | VehicleModel.LaneSlope | field | g - / n 32777 | `aliveTiltRoadway` | — |

What the counters are: `alive*` are BMW's 4-bit alive counters of the CAN / FlexRay signal (0..14, 15 = invalid), `counter` the 16-bit end-to-end counter of the SOME/IP event, `*SyncCounter` the keep-distance list's own 8-bit counter. A jump over +1 is a message the car sent and the head unit never handed over: the converter counts them (`counter missed`). Only two S14 payloads carry a **measurement time**: `keep-dist-params` (`KeepDistObjSeconds` / `KeepDistObjNanoseconds`, both UInt16, so the "nanoseconds" is a scaled unit, not ns: the converter's payload-clock line will show which) and `view-input` (a standard time stamp, seconds and nanoseconds, with its sync state as a value column). Every chassis signal has a counter but no time: their latency can only be estimated by cross-correlation (the brief's method), the counters give the drop rate.

## 2. The recorder (Phase 0)

### 2.1 Where: inside Signal Probe

Inside, not a separate app: the probe already has the field table, the lazy subscription, the menu and the marks, and the car has installed it; a second app would need its own install (another 99-interface manifest), loggingId, uid, data folder and crash budget. The cost inside is one check per value in the probe's handler (`apps/signal_probe/probe.js`, hunks marked `// cruise RC:`). A separate app would only win if Signal Probe's crash record were already used up.

### 2.2 How it works

- **Menu** (`menu.js`): "Почати запис S14" starts (it switches the set and its checkbox on if needed, then routes the set's fields, including what other sets share with it through `also`, to the recorder); the same row stops; while recording it says `Запис S14 іде, m:ss. Натисніть: стоп` / `12 345 зразків, 1,2 МБ, вільно 13,8 МБ` (and `втрачено N` if the queue ever dropped). "Набір для запису" cycles S14 → S14+S15 → S13 (`config.recorder.choices`), fixed while recording. "Позначка N" writes `#m,<t>,<n>` into the recording as well as the log line. The recording does **not** resume after an app restart: one press per drive.
- **Files** (`data/rec/rec-<session>-<nnn>.csv.gz`): the format is in the header of `recorder.js`. In short: `#rec` header per file (app version, session, file number, the head unit's wall clock at the file's start, the monotonic time), `#f` per field (its columns, in the payload's own member order, again when the shape changes), `#n` per enum name (a code; names stay names in the converter), `#m` marks, `#end` the last line (missing = the app or the head unit stopped). A sample: `t_ms,field,seq,ctr,pts,values...`, `=` for "the same values as this field's line before" (86 % of the integer fields' lines in the simulation).
- **Time**: `t_ms` is `process.hrtime` (monotonic) at the moment the handler runs, ms since the start; `seq` per field from 1 (a gap = the recorder dropped samples: the queue was full); `ctr` / `pts` from the payload.
- **Privacy**: numbers and enum names only. A string that is not an enum name is written empty and counted (`texts=` in `#end`); buffers, the end-to-end header (crc, dataID, cHL, the top-level length), anything named latitude / longitude / WGS and whatever a field's tree spec skips (S15's own dead-reckoned position and heading) are never written. Only S13, S14, S15 can be chosen (no text member in their types; the test checks every field of them with canary text and an injected coordinate).
- **Writing**: lines are queued in memory and written once a second (`flushMs`), compressed in the libuv thread pool (`zlib.gzip`), appended with `fs.write`, one write in flight. Each flush is its own gzip member, so a file cut by a crash reads up to its last flush. Files are cut at 4 MB of text (about 1 MB on disk), never more than half the quota.
- **Quota**: after every write the recorder's own files are summed; the oldest closed files are removed while the folder is over 15 MB (`quotaMB`); a file not of the recorder is never touched. If the open file alone were over it, the recording stops.
- **Never the app**: every fs / zlib call is in a try or a callback that catches; a write error (ENOSPC, EDQUOT, EACCES, EIO, a broken gzip) stops the recording with `ЗУПИНЕНО: <code>` in the row and one log line; the queue is capped (2 MB of text: samples beyond are dropped and counted, `seq` shows where); a hostile value (a cycle, a throwing getter, a 100 000-entry list) is one counted error. At the app's exit the queue is written synchronously.
- **The log** while recording: the recorded fields' throttled lines are left out (`quietLog`), so the 256 KB log keeps the marks, the summaries and the `rec:` lines, and no CPU goes into formatting lines nobody reads.

### 2.3 The budget

Assumed native rates (the first drive replaces them; `rateOf()` in `bench/test-signal-recorder.js`): chassis and powertrain signals and the chassis events 50 Hz; gradient, lane slope 10 Hz; the camera's front vehicle 25 Hz, its gap 10 Hz; the three keep-distance lists 15 Hz; the infotainment (CDS, CDS2, Vehicle2, A4A, XView) and view input 10 Hz; display data and FASuS 5 Hz; the rest 1 Hz. S14 = 58 fields, **1 142 samples a second**.

| Case | Samples/s | Text B/s | On disk B/s | Minutes in 15 MB | CPU (Mac) ms/s |
|---|---|---|---|---|---|
| Plausible rates, a 20-minute simulated drive (values moving: noise on every float, slow integers, alive counters) | 1 142 | 35 037 | **8 761** | **30** | 4.2 |
| Twice the rates, 2 minutes | 2 284 | – | 17 257 | 15 | 12.0 |
| Upper bound: every column changes every sample (from the column counts) | 1 142 | ~69 000 | ~18 000–23 000 (x3–4) | 11–14 | – |

Per field at the plausible rates the 50 Hz float events dominate (`accel-long-cog`, `yaw-rate`, `accel-lat-cog`, `velocity`: about 2.1–2.4 KB/s of text each); the three keep-distance lists (65 columns each) would be 5 KB/s each if every object member moves. The `#f` / `#n` header lines cost about 230 B/s.

- **CPU**: 4.2 ms per second on the Mac (about 3.7 µs a sample including the gzip). The head unit's Intel Atom A3960 (4 cores, 1.9 GHz) is taken as 5–6 times slower single-threaded: **20–25 ms per second, 2–3 % of one core**, the gzip part off the main thread. The probe's quiet path saves the line formatting it would otherwise do on every value. The app's limit is `cpuLimit` 60.
- **Memory**: the queue is capped at 2 MB of text; the state per field is one line of text.
- **Disk**: `/var/opt` (the app's data folder, ecryptfs over ext4 with user quotas) had 9.7 GB free on 2026-11-15 (`reference/car/2026-11-15/system/df.txt`), so free space is not the limit. **The limit is the app's own `maxDiskUsage` = 20 MiB** (`package.json` from `kit/tools/build.py`, the same for f30_cluster: `reference/car/2026-11-15/oap/f30_cluster-dirs.txt`), which holds the log (up to 2 x 256 KB) and the settings too. How MainApp enforces it (an ext4 quota, a check, a kill) is unknown, so the recorder stays at 15 MB. The "вільно" in the menu is the room left in that quota. Node 8 has no `statvfs`; the recorder does not spawn `df`.
- **Exit criterion** ("a 20-minute recording survives"): met at the plausible rates (30 minutes fit); at twice the rates or with every value moving, 11–15 minutes fit, after which the quota removes the oldest file of the same recording. Two remedies, the lead's decision (section 5): raise `maxDiskUsage`, or record in blocks of 10–15 minutes and copy them off between drives.
- **Install**: S14 now needs 99 interfaces in the manifest (5 new: `AcceleratorPedal`, `VehicleStabilizationManagement`, `VelocityAndVehicleConditionProvision`, `XView`, `VehicleModel`). The largest manifest known to install was 88 configs (0.1.1, CP-REPORT §12); 0.2.0's 86 + platform 22 was built but the firmware's generator was not run. Risk R1 of CP-REPORT §7 grows a little.

## 3. The converter

`python3 tools/car/rec_convert.py <data/rec folder or files> -o <out> [--resample 100] [--max-age 300] [--parquet]` (plain Python 3; pandas only for parquet). Per recording (a "drive"): `fields/<tag>.csv` (every sample), `drive.csv` (sparse by time, or on a common clock with zero-order hold no older than `--max-age`), `markers.csv`, `report.md` / `report.json`: per field the samples, the native rate (from the median interval) and the mean rate, jitter (standard deviation of the intervals, p5..p95), gaps (an interval over three medians) and the longest, the recorder's drops (seq), the car's missed messages and repeats (counter, the wrap guessed: 15, 16, 256, 65536), the payload clock's interval and the spread of receive time around a fitted line against it (the reading path's jitter), the share of repeated values; per recording whether it ended with `#end`, the cut files, unreadable lines. Enum names stay names (the catalogue's numbers are one lookup away if wanted).

## 4. Tests

| Suite | Result | What |
|---|---|---|
| `BENCH_PORT=8901 node bench/test-signal-probe.js` | **175 passed, 0 failed** | the existing suite with the counts moved to 263 fields, 45 trees, 100 proxies, 13 new vectors (lines marked `// cruise RC:`) |
| `BENCH_PORT=8902 node bench/test-signal-recorder.js` | **60 passed, 0 failed** (42 s) | source (Node 8 syntax, no signal call, no child process); 89 fields of S13/S14/S15 from catalogue types with canaries and coordinates; counters / time stamps; a 20-minute simulated S14 drive (1.37 M samples, none dropped, every seq in order, 11 files, under the quota, the converter finds every rate and no drop); twice the rates; the quota removing an old recording; ENOSPC, EACCES, a missing folder, a broken gzip, a write that never returns, hostile values; inside the probe (routing, quiet log, marks); the real app on the bench (menu rows, start, 1 800 injected values in 0.2 s all in the file in order, mark, stop, restart = off, never exited) |
| `python3 tools/car/test_rec_convert.py` | **9 passed** | rates, jitter, gaps, drops vs counter misses, ditto / enums / a schema change across files, a cut gzip tail, sessions, resampling, the payload clock, bad lines, the CLI |
| `python3 tools/signals/check_signals.py ... apps/signal_probe/app.json` | **all declarations pass** (526 checks) | |
| `tools/car/test_session.py` 12 OK, `tools/car/dry_run.js` 34 / 0 | unchanged | |
| `tools/car/test_car_tools.py` | 46 / 1 | the one failure (`Patches.test_diffs_change_only_what_they_name`, `config.tabKey`) **fails on `main` too**, not this branch |

Node 8.16 itself was not run (no binary on the Mac); the check is the source scan as for the rest of the app. Side effect to know: `tools/car/dry_run.js` writes by default into the shared `bake/work/car/CQ/dry-run/`; running it refreshed that folder (regenerable, git-ignored).

## 5. Open decisions (the lead's / the owner's)

1. **`maxDiskUsage`**: keep 20 MiB (15 MB of recordings, 15–30 minutes of S14) or raise it in `app.json` (`"maxDiskUsage"`, read by `build.py`) to e.g. 100 MiB and `config.recorder.quotaMB` to 90 (2.5–5 h). Unknown whether MainApp accepts a larger value or how it polices it; untested on the car.
2. **Version**: the branch leaves Signal Probe at 0.3.0; the lead bumps.
3. **Install size**: 99 interfaces (R1). If the install fails, drop the five new interfaces' fields first (the pedals and the brake would then rest on `brake-driver-hu`, `brake-switch` via Powertrain, `drive-torque`).
4. **Resume after a restart**: off by design (a forgotten recording would run every drive); an option if a head-unit restart mid-drive turns out to be common.
5. **S15 in a recording** (S14+S15): its object lists are large (16 entries x up to 132 members); budget it from the first S14 drive before using it for longer than a few minutes.
6. **Merge with CT** (S16 on another branch): both change the counts in `bench/test-signal-probe.js` (fields, trees, proxies, the start line) and append to `bench/signal-probe-vectors.json`; every hunk of this branch there is marked `// cruise RC:`. After the merge: `node tools/car/probe_fields.js sync && node tools/car/probe_fields.js export`, then the counts are the sum.
