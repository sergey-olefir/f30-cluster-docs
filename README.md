# f30-cluster docs

The documentation of [**f30-cluster**](https://github.com/sergey-olefir/f30-cluster): custom instrument-cluster
apps for a BMW G20 3 Series with an iDrive 7 head unit (MGU). Two cluster apps redraw the car's 1920 × 720 panel:
**f30_cluster**, the look of the older F30's 6WB dials in three themes, and **Twin Dial**, a twin-round-dial design
drawn from scratch. Next to them: **Signal Probe** (a read-only probe of what the car sends), a cruise-control
simulator, and the bench tools that let all of it be built without the car. The apps only listen: nothing is written
to the car.

Everything here was written while building the project, mostly by a team of AI agents working to the owner's
briefs, and checked on the bench and then in the car. The code, the tools and the tests are in the code repository:
**https://github.com/sergey-olefir/f30-cluster**. This repository holds the texts and pictures only; it was split
off because the head unit's app store refuses a repository archive over 64 MiB.

## Start here: the article

How the whole thing was done, from the first photo of the panel to two working cluster apps.

- **[Українською](article/article.uk.md)**: «Два щитки для однієї «трійки»: приладова панель BMW G20, перемальована
  командою ШІ-агентів»
- **[In English](article/article.en.md)**: "Two clusters for one 3 Series: a BMW G20's instrument cluster, redrawn by
  a team of AI agents"

Its pictures are in [`article/images/`](article/images/) (the ones with text in both languages, `*_uk.png` and
`*_en.png`); [`article/sources.md`](article/sources.md) says where every fact and number comes from;
[`article/README.md`](article/README.md) is the publishing checklist and how to re-make the images;
[`article/CHANGES-r4.md`](article/CHANGES-r4.md) is the last revision's change log.

## How to read the paths

These documents were written inside the code repository, under `docs/`.

- A path written as `docs/<path>` means `<path>` in **this** repository (for example `docs/twin-dial/SPEC.md` is
  [`twin-dial/SPEC.md`](twin-dial/SPEC.md)). The agent briefs (`docs/AGENT-BRIEF-*.md`) are in
  [`internal/briefs/`](internal/briefs/).
- Any other path (`apps/`, `shared/`, `bench/`, `tools/`, `bake/src/`, `design/`, `kit/`) is in the
  [code repository](https://github.com/sergey-olefir/f30-cluster).
- `bake/work/…` and `reference/…` are the project's local working folders (renders, car logs, reference pictures).
  They were never published, so those paths are mentions, not links.
- Versions, branches and commits named in a report are the code repository's at the time of writing. A report
  describes its own round; a later one may supersede it (the specifications say so where they do).

## Index

### Specifications

| Document | What it is |
|---|---|
| [G20-DESIGN-SPEC.md](G20-DESIGN-SPEC.md) | The stock G20 cluster, measured: a specification for drawing it ourselves |
| [F30-FONT-SPEC.md](F30-FONT-SPEC.md) | The F30 6WB's typeface, measured |
| [ROAD-SHIELDS.md](ROAD-SHIELDS.md) | Road-number shields for the bottom row |
| [BOTTOM-ROW-PACKAGE.md](BOTTOM-ROW-PACKAGE.md) | The bottom row: who draws the sign, the message and the road label |
| [twin-dial/SPEC-R3.md](twin-dial/SPEC-R3.md) | Twin Dial specification, round 3 (current; extends [rev 2](twin-dial/SPEC.md)) |
| [twin-dial/ENGINE-CONTRACT.md](twin-dial/ENGINE-CONTRACT.md) | Twin Dial engine contract |
| [twin-dial/MOCKUP-SPEC-R3B.md](twin-dial/MOCKUP-SPEC-R3B.md) | Twin Dial round 3b mockup specification |
| [twin-dial/FV-SPEC.md](twin-dial/FV-SPEC.md) | The vehicle ahead on Twin Dial |
| [cruise-set-speed-brief.md](cruise-set-speed-brief.md) | Cruise-control set-speed controller: signals, simulation, feasibility |
| [cruise-sim/CONTRACT.md](cruise-sim/CONTRACT.md) | The cruise simulator's contract |
| [DRIVER-PROFILES-APP.md](DRIVER-PROFILES-APP.md) | Driver Profiles, a profile manager app (bench only) |

### Protocols for the car

| Document | What it is |
|---|---|
| [car/SESSION.uk.md](car/SESSION.uk.md) | The car session, step by step (Ukrainian) |
| [car/RUNBOOK.md](car/RUNBOOK.md) | The lead's runbook for that session |
| [twin-dial/PROBE-PROTOCOL.md](twin-dial/PROBE-PROTOCOL.md) | Twin Dial probe: what to do at the car |
| [research3/MK-PROTOCOL.md](research3/MK-PROTOCOL.md) | Trip computer and odometer: the five-minute check |
| [cruise/DRIVE-PROTOCOL.md](cruise/DRIVE-PROTOCOL.md) | The first calibration drive with the cruise recording |
| [OVERLAY-PHOTO-PROTOCOL.md](OVERLAY-PHOTO-PROTOCOL.md) | The photo protocol for the stock G20 elements |
| [CAR-SNAPSHOT-2026-11-15.md](CAR-SNAPSHOT-2026-11-15.md) | What the car's head unit is and runs (a read-only snapshot) |

### f30_cluster: the 6WB bake and the stock G20

- The bake: [BAKE-REPORT.md](BAKE-REPORT.md), [BAKE-P0-inventory.md](BAKE-P0-inventory.md),
  [BAKE-P0-report.md](BAKE-P0-report.md), [BAKE-C9-overlays.md](BAKE-C9-overlays.md),
  [BAKE-P10-levels.md](BAKE-P10-levels.md)
- The G20 simulation on the bench: [G20SIM-REPORT.md](G20SIM-REPORT.md), [B](G20SIM-B-structure.md),
  [C](G20SIM-C-icons-text.md), [D](G20SIM-D-live.md)
- The overlay (our dials over BMW's scene): [OVERLAY-REPORT.md](OVERLAY-REPORT.md),
  [OVERLAY-PLAN-DIALS.md](OVERLAY-PLAN-DIALS.md), [OVERLAY-PARTS-REPORT.md](OVERLAY-PARTS-REPORT.md)
- The bottom strip: [strip/](strip/) (D, F, H, M, O, P, R, T, X, ZC, ZL, ZR)
- Polish rounds: [A](POLISH-A-REPORT.md), [B](POLISH-B-REPORT.md), [C](POLISH-C-REPORT.md),
  [D proposal](POLISH-D-PROPOSAL.md) and [report](POLISH-D-REPORT.md), [E](POLISH-E-REPORT.md),
  [F](POLISH-F-REPORT.md), [ONE-SWITCH-DONE.md](ONE-SWITCH-DONE.md)
- On the car: [CAR3-G](CAR3-G-REPORT.md), [CAR3-H](CAR3-H-REPORT.md), [CAR3-I](CAR3-I-REPORT.md),
  [CAR3-J](CAR3-J-REPORT.md), [CAR3-K](CAR3-K-REPORT.md), [CAR4-L](CAR4-L-REPORT.md) (pictures in
  [car3/](car3/), [car4/](car4/))
- Audits: [FEATURE-AUDIT.md](FEATURE-AUDIT.md), [PERF-AUDIT.md](PERF-AUDIT.md),
  [PERF-NEEDLE-REPORT.md](PERF-NEEDLE-REPORT.md), [UI-NATIVE-REPORT.md](UI-NATIVE-REPORT.md)

### Twin Dial

- Round 1 (design): [VR](twin-dial/VR-REPORT.md), [VD](twin-dial/VD-REPORT.md), [VC](twin-dial/VC-REPORT.md),
  [VT](twin-dial/VT-REPORT.md), [VF](twin-dial/VF-REPORT.md), [VS](twin-dial/VS-REPORT.md),
  [VG](twin-dial/VG-REPORT.md)
- Round 2 (the build): [A1](twin-dial/R2-A1-REPORT.md), [A2](twin-dial/R2-A2-REPORT.md),
  [B](twin-dial/R2-B-REPORT.md), [C](twin-dial/R2-C-REPORT.md), [D](twin-dial/R2-D-REPORT.md),
  [E](twin-dial/R2-E-REPORT.md) and its [gallery](twin-dial/R2-GALLERY.md), [F](twin-dial/R2-F-REPORT.md)
- Round 3 (mockups and the 0.1.1 build): [K1](twin-dial/r3/K1-REPORT.md), [K2](twin-dial/r3/K2-REPORT.md);
  [KA](twin-dial/r3b/KA-REPORT.md), [KB](twin-dial/r3b/KB-REPORT.md), [KC](twin-dial/r3b/KC-REPORT.md);
  B1–B7 in [twin-dial/r3b/](twin-dial/r3b/); [R3B-INTEGRATION.md](twin-dial/R3B-INTEGRATION.md),
  [FIN-REPORT.md](twin-dial/FIN-REPORT.md)

### The car's signals, fixes and issues

- Research round 3: [MB](research3/MB-REPORT.md) (and its [upstream draft](research3/MB-UPSTREAM-DRAFT.md)),
  [MC](research3/MC-REPORT.md), [MK](research3/MK-REPORT.md), [MP](research3/MP-REPORT.md),
  [MT](research3/MT-REPORT.md)
- The signal session: [CA](car/CA-REPORT.md), [CL](car/CL-REPORT.md), [CP](car/CP-REPORT.md),
  [CQ](car/CQ-REPORT.md); a parser dry run in [car/dry-run/](car/dry-run/) (synthetic data)
- Fixes from the car: [CF1](car-fix/CF1-REPORT.md), [CF2](car-fix/CF2-REPORT.md), [CF3](car-fix/CF3-REPORT.md)
- Car issues: [CT](car-issues/CT-REPORT.md), [GL](car-issues/GL-REPORT.md), [MW](car-issues/MW-REPORT.md),
  [NV](car-issues/NV-REPORT.md), [TX](car-issues/TX-REPORT.md)

### Cruise control

- The recorder and the signal map: [cruise/RC-REPORT.md](cruise/RC-REPORT.md)
- The simulator: [SC1](cruise-sim/SC1-REPORT.md), [SC2](cruise-sim/SC2-REPORT.md),
  [SC3](cruise-sim/SC3-REPORT.md), [SU1](cruise-sim/SU1-REPORT.md), [SU2](cruise-sim/SU2-REPORT.md)

### Internal

[`internal/briefs/`](internal/briefs/): the briefs the AI agents worked from, one per round. They are working notes
(rules, packages, file paths of the time), kept because the article describes how the work was organised.

## Privacy

Before publication every file was scanned and cleaned: no VIN, keys, tokens, account or profile data, coordinates,
street or place names from the drives, media titles, contacts, phone numbers or the car's mileage; image metadata was
stripped, and street names were blurred or painted out in the pictures. Values that look personal but are not
(the bench's fixtures such as an odometer of 48 215 km, an invented street, an invented radio station) are test
data. Two reports about the car's driver profiles and its Digital Key are not published.

## Licence

See [LICENSE.md](LICENSE.md).

- **Texts, diagrams and renders** (everything not listed below): [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
- **Photographs of the real car's panel** (in the article's `photos-early_*.png` and `rectify_*.png`) and the bench's
  stock-G20 backdrop made from them (it shows through in the G20 simulation frames): the owner's photos, all rights
  reserved; they may be shown with this documentation.
- **Code snippets and the scripts in `article/src/`**: MIT, as the code repository.
- BMW, iDrive, CarPlay and other names are trademarks of their owners, used only to say what the software runs on.
  This is an independent hobby project, not affiliated with or endorsed by BMW.
