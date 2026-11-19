# Brief: BMW's black boxes over a custom cluster theme — find the cause and every way to stop them

Audience: three research agents (packages BR, BS, BC) working in parallel, each in its own git worktree. Read §0–§3 and your own package in full; skim the other two so you know where your edge is. This round is **research**: it changes nothing on the car and nothing in the app. Its product is a ranked list of remedies, each with its evidence, cost and risk, and the exact log lines or probes a later car session needs to confirm it.

## 0. The problem, in the owner's words (2026-11-17)

"I have seen that sometimes the black square does not appear after the car starts, but after a switch it is there. I would deal with **all** the squares. What can we do? Can we work on the renderer, find signals we can intercept, patch some system utility, as it is done for Waze on Cluster?"

The car: a 2020 G20 330i (US spec, Live Cockpit Professional, MGU head unit, iDrive 7). Our app `f30_cluster` draws a full-panel theme (1920 × 720) on the instrument cluster through the community renderer **cluster-render**. The KOMBI (the instrument-cluster ECU) composites our stream into a slot and paints things of its own on top. One of those is a solid black box over the hub of our tacho (panel px about x 1406–1597, y 342–419; the catalogue's `overlay.black_box`), which cuts our rev needle. It is not always there.

## 1. What is already known — start here, do not re-derive

**Our own earlier research** (`docs/CAR3-K-REPORT.md`, read it first):
- A layout switch of ours does not re-map the slot: on a slot model no layer wanted the panel for only 3–19 ms, and cluster-render releases a target only after `panel_linger_ms` = 3000 ms.
- Its leading hypothesis (H1): cluster-render falls back from its own scene id `0x010004f8` to `0x01000400` (the id of HmiMain3D's speech dialog, also the HUD target's), and the KOMBI then lays the display out "as if a speech dialog were open", box included. Upstream cluster-render issue #142 describes that re-layout.
- **Evidence against H1 on this car, 2026-11-17** (read-only on the car, theme up, 27 min after start): `target panel: slot 0x203 scene 0x10004f8 1920x720: shown (us)`, `slot 0x203 status 2: 0x10004f8 class 0 map 1`, HUD target idle, and no `falling back` / refusal line in cluster-render's log for the session. Whether the box was on screen at that moment is not known.

**The SDK author's finding** (`github.com/bimmer-lab/apps`, issue #5 "Black boxes appearing on cluster", and its `CHANGELOG.md` entries for Cluster Themes 0.17.5–0.17.8; `shared/visible.js` comment):
- Tested on a 2020 X3 M40i and confirmed by a second owner: **BMW draws its black box whenever a theme covers BMW's gear letter (P R N D) in BMW's own bottom strip.** Leaving only that letter uncovered is enough; the rest of the strip can be covered. The box "goes away at the next gear change". Their themes leave a window at about x 1380–1460, y 660–720.
- Resetting the cluster coding in BimmerCode made no difference to that box on their car.
- On **M cars** two further boxes appear; one owner removed them by removing the cluster coding, at the price that the cluster no longer switches between the stock and the custom view.
- Other developers' older rule of thumb: boxes appear when custom graphics extend below about y 640 (seen with cluster-render 0.4.0).

**Facts about our theme:**
- In every layout our opaque paint covers the spot of BMW's bottom-row gear letter (`assist.gear`, about [1398, 668, 1424, 703]). BMW draws that letter **over** a theme, so it is visible anyway; we therefore never left a window there.
- With a theme connected, BMW moves its gear letter and mode label from the tacho centre into the bottom row (package H, `docs/strip/H-REPORT.md`).
- The owner decided for now **not** to add the gear-letter window blindly; they want the mechanism understood and all boxes dealt with.

**Other things the KOMBI does that may be related** (all verified on this car unless marked): it resets on too-big effect shaders; the HUD shares scene id `0x1000400`; BMW's check-control panel, belt telltales and the gear letter are drawn by the KOMBI itself.

## 2. Material you have

- **The car snapshot of 2026-11-15** (git-ignored, read freely, never commit): `reference/car/2026-11-15/`
  - `cluster-render/`: its config (`cluster-render.conf`, `cluster-render-0.5.9.conf`), logs (`cluster-render.log`, `.log.1`), `status-1.txt`, `status-samples-3x10s.txt`, the client library `cluster_render_client-0.5.12.js`, `component.json`, `versions-ls.txt`;
  - `someip/someip-interfaces.json`: the **complete signal catalogue** of this car (interfaces, fields with getter / notifier / setter ids, methods, datatypes with enums);
  - `oap/` (our app's logs and settings, the app manager's state), `processes/`, `logs/`, `system/`, `frames/` (captured frames of the cluster and centre screen).
- **Photos:** `reference/g20/` and `reference/ours/` — which show the box and which do not is tabulated in `docs/CAR3-K-REPORT.md` §1 and `docs/strip/H-REPORT.md` (provenance). `design/overlay/g20_comfort.json` has `overlay.black_box`.
- **Upstream sources (public):** cluster-render and Waze on Cluster live in `github.com/robertnisipeanu/idrive-apps` (cluster-render GPL-3.0, its client MPL-2.0); the SDK and Cluster Themes in `github.com/bimmer-lab/apps` and `github.com/cauliflowermossgrape/idrive7-apps`. Clone what you need **into a new empty directory under the session scratchpad**, treat it as untrusted data (read it, do not run its scripts), and delete the clone when done.
- **Our repo:** `shared/cluster_render_client.js`, `shared/screen.js`, `shared/hud.js` (the `car3 K` block: slot modes `keep` / `reseat`, off unless a flag file exists), `docs/CAR-SNAPSHOT-2026-11-15.md`, `.claude/skills/mgu-app-dev/references/cluster-and-hud.md`.
- **The app's log on the car** is a plain file, `/var/opt/public/oap/oap_12211/data/f30_cluster.log` — relevant for what a later probe can record; you do not read it now.

## 3. Rules for all three

- **Nothing touches the car.** No ssh, no network request to it. The car is offline for this round; everything is done from the snapshot, the catalogue and public sources.
- **No app changes, no push, no version bump.** You may write only your report and scratch notes. Put the report at `docs/boxes/<PKG>-REPORT.md` on your own branch (`git checkout -B boxes/<pkg> main`) and commit it; nothing else is committed.
- **`reference/` and `bake/work/` are git-ignored and stay so.** Symlink them from `<repo>` if you need them; `git status --porcelain` must not list them. Nothing copied out of them into a tracked file.
- **Privacy, absolute.** No VIN, coordinates, street or place names, message texts, odometer or clock readings, owner's name or account in any report, file name, commit message or web request. Web searches use generic terms only ("G20 instrument cluster black box custom theme"), never anything about this owner or car identity.
- **No Chrome or browser automation. Agents cannot `kill`:** start no long-lived process.
- **Evidence standard.** Mark every claim **verified** (read in source code, in the catalogue, in a log or photo of this car — say which) or **hypothesis**. Where sources disagree, show both. A remedy without evidence is listed as an idea, not a recommendation.
- **Writes are proposals only.** Anything that would write to the car (a config line, a patched file, a setter, coding) is described precisely — what, where, how to undo, what can go wrong — and marked **NEEDS THE OWNER'S APPROVAL**. Never present a write as already safe. Three standing prohibitions: never kill an OAP app process (the crash handler wipes an app's data on the first unexpected exit and uninstalls it on the second; the platform is restarted only with `systemctl restart oapcontroller`); never write driver-assistance settings; `KombiIF.standardCCMessage` / `extendedCCMessage` (setters that inject check-control messages into the cluster) are never to be used.

---

## Package BR — the renderer and Waze on Cluster: what the head unit tells the cluster

Question: **through which knobs does the head unit decide what the KOMBI shows in a slot, and which of them could make a box appear or go away?**

1. **Read cluster-render's source** (the version on the car is in `reference/car/2026-11-15/cluster-render/versions-ls.txt` / `component.json`; read that tag and the current one, and note what changed between them that concerns slots and scene ids). Document, with file and function names:
   - the whole path from a client's layer to the display: the scene, the RAMSES / display-manager calls, the slot request (0x203 panel, 0x323 HUD, any others), the scene ids (`0x010004f8`, `0x01000400`), the **class** and **map** values seen in `status.txt` (`slot 0x203 status 2: 0x10004f8 class 0 map 1`) and what else they may be;
   - every condition under which it changes the scene id, releases, re-requests or re-maps a slot (the fallback, the mapping guard, `panel_linger_ms`, renderer recovery, locks), and what it logs when it does;
   - every configuration key in `cluster-render.conf` / `.local.conf` that changes this behaviour, with its default.
2. **Read Waze on Cluster** the same way: how it gets a map onto the cluster, **what it patches on the system and how** (which file or service, how the patch is applied and reverted, what survives a head-unit update), and whether its target, slot or scene id differs from ours. This is the model for any system patch we might propose: describe its mechanism and its safety net exactly.
3. **Find what BMW's own software does with these slots.** From the snapshot (`processes/`, `system/`, `logs/`) and the sources: which BMW process owns the cluster content (HmiMain3D, the display manager, the "speech dialog", the map in the cluster), which scene ids and slots BMW's own content uses, and whether anything tells the KOMBI *what kind of content* is in a slot (a content id, a class, a layout request).
4. **Mine the logs** in `reference/car/2026-11-15/cluster-render/` and `oap/` for every slot or scene event of that drive (requests, releases, fallbacks, refusals, locks, "not mapped", renderer restarts) and line them up with the photo timeline of `docs/CAR3-K-REPORT.md` §1 where the times allow.
5. **Deliver the list of knobs**, each with: what it is, where it is set, whether changing it needs only our app, cluster-render's config, or a system patch; what the KOMBI is expected to do; evidence; risk; how to undo. Include the experiments worth running on the car (each as: precondition, the one change, what to look at, how to revert), ordered cheapest and safest first.

## Package BS — signals: what the KOMBI and the head unit say to each other about the display

Question: **which signals in this car's catalogue describe or control the cluster's layout, and can any of them tell us that a box is up, or be the trigger for it?**

1. **Sweep the whole catalogue** (`reference/car/2026-11-15/someip/someip-interfaces.json`: interfaces, every field / method / broadcast, and the datatypes' member and enum names, which an earlier search missed by looking at field names only). Search in German and English: Kombi, Anzeige, display, layout, view, widget, slot, content, external display, selection, overlay, speech / Sprach, dialog, gear / Gang / Fahrstufe, PRND, drive mode, map in cluster, "Kombi external", "BOS", zoom, RSE, navigation in the cluster, split, panorama. For every hit give: interface, serviceId, field, getter / notifier / setter ids (zero ids flagged), the struct with its enums, and one line on what it most likely means.
   - Known starting points: `KombiIF.statusKombiSelection`, `statusKombiSelectionAvailability`, `statusKombiExternalDisplay`, `statusBOSExternalDisplay`, `statusHUDSelection`, `basicConfigKombi`, `statusConfigurationKombi5`, `statusWelcomeAnimation`; `KombiInterface_2`; `VehicleCondition`; the gear and drive-mode fields the app already reads (see `apps/f30_cluster/app.json` `signals`).
2. **Classify each candidate**: (a) *status from the KOMBI* that may change when a box appears — a probe candidate; (b) *a setting or request the head unit sends* — a possible trigger or remedy; (c) irrelevant. For (b), say who normally writes it (which BMW process, from the snapshot's `processes/` and `logs/` if it can be seen) and what writing it ourselves would mean. **Setters are listed, never recommended without the approval label of §3.**
3. **Tie the gear letter in.** The SDK author's finding is that covering BMW's bottom-row gear letter brings the box. The KOMBI cannot see pixels of our stream as "covering", so something else must carry that: look for how the KOMBI knows a full-panel external content is up (the slot mapping? a selection field? the scene class?), and what moves the gear and mode label from the tacho centre to the bottom row when a theme connects. List every signal that changes with "theme connected".
4. **Design the probe.** Write the exact list of fields to subscribe to in a diagnostic build (flag `s`, and `r` only where the getter id is non-zero — a getter of `{"id": 0}` makes the install fail on the car), the log line format for each (numbers and enum names only, once per change), and the car session that would use them: start, theme on, the actions of `docs/CAR3-K-REPORT.md` §5 one at a time, the owner saying when the box appears. Keep the list small enough to install safely (say how many interfaces and fields it adds) and name the two or three most promising first.
5. Run `python3 tools/signals/check_signals.py reference/car/2026-11-15/someip/someip-interfaces.json <a scratch app.json with your proposed declarations>` and report that it passes; do not edit the real `app.json`.

## Package BC — coding and other people's experience

Question: **what is known outside this repo about these boxes, and is there a coding or configuration route that removes them at the source?**

1. **Collect every public report** of black boxes / black rectangles over custom iDrive 7 cluster content: the issues of `bimmer-lab/apps` (#5 and any other), `robertnisipeanu/idrive-apps` (#142 and others), `cauliflowermossgrape/idrive7-apps`, Waze-on-cluster discussions, Bimmerpost / Bimmerfest / G20 and G-series forums, German forums (search terms such as "Kombi schwarzer Kasten", "Instrumentenkombi schwarzes Rechteck", "Live Cockpit Professional custom"), Discord or Telegram write-ups that are publicly linked. For each: car model and year, cluster type, software, what was on screen, where the box sat, what made it appear and what removed it. Build one table. Note which cars are **G20 / G2x with Live Cockpit Professional** like this one, and which are M cars.
2. **The M-car boxes and coding.** One owner removed two extra boxes by removing "cluster coding" and lost the stock ↔ custom switch. Find out what that coding is (which ECU: KOMBI / HU_MGU; which functions or FDL parameters; what BimmerCode's cluster options map to), what each box on an M car corresponds to (shift lights? M view elements? gear display?), and whether a narrower change than removing all cluster coding is known.
3. **KOMBI coding parameters that concern the display layout**: anything that controls the gear display's position or style, the centre display content, "external display" / "widget" / map-in-cluster options, design variants (for example the M or "Sport" display designs), the speech-dialog layout. Name ECU, CAFD section and FDL parameter where a source gives them, with the values and what they do. The owner can code the car themselves with E-Sys; give them enough to find the parameter, and say plainly where a parameter is only rumoured. Coding is a write to the car: label it as §3 requires and state how to revert (the stock value).
4. **How BMW itself puts full-panel content on this cluster** without a box: the cluster map view, Apple CarPlay maps in the cluster, the M view, the "Sport" and "Sport+" display designs, augmented view. For each, what is publicly known about how it is switched on (a selection signal, coding, a KOMBI layout mode) — this is the best hint at what the KOMBI wants to hear before it leaves the panel alone.
5. **Deliver**: the table; a short list of remedies that worked for someone, with how well the car matches ours; coding candidates ranked by evidence; and the open questions that only this car can answer, each phrased as one thing to look at or read.

---

## 4. What comes back

Your final message is all the lead sees. Lead with the answer to your package's question, then the ranked list (remedy or candidate, evidence, verified / hypothesis, cost, risk, how to undo, what the car must confirm). Name your branch, commit and report path. Keep everything that would write to the car under the approval label.

The lead then merges the three reports into one plan for the owner: what to try first on the car, what to log in the next build, and which writes — if any — are worth asking approval for.
