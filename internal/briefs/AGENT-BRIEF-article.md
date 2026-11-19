# Brief: the article — how we built a custom instrument cluster for a BMW G20 (Habr / DOU style)

Audience of this brief: one agent, package **ART**. Audience of the article: smart engineers and car enthusiasts who read DOU / Habr — curious, technical enough to enjoy a shader or a SOME/IP field name, but not specialists in BMW internals. It must read easily: a story with a thread, technical details as seasoning, not a manual.

## 0. What the owner wants (2026-11-18)

"I need to write a proper article on how we did all of it: how we listened to signals, where we started, how we baked, how we computed the geometry and the area. A bit of technical detail, but on the whole it should read easily for a smart person. Add the best couple of screenshots of Comfort mode and of Twin. It needs the evolution of our app. Imagine we are writing for Habr or DOU."

Owner's decisions on the form:
- **Two versions: Ukrainian (lead, for DOU) and English** (a faithful adaptation, not a word-for-word translation).
- **Brands are named openly:** BMW G20, the F30's 6WB cluster, a cluster in the style of the Audi virtual cockpit. (This article lives OUTSIDE the public repository — see §1; inside the repository the rule "no maker's names in tracked files" still holds.)
- **By the owner, in the first person** ("я", with "ми" for the owner working with Claude as the AI pair — say plainly that the work was done with an AI agent team; that is part of the story). **No personal data of the car or the owner:** no VIN, no plate, no addresses, no places, no account names, no phone, no e-mail, no GitHub user name unless the owner adds it. The byline is left as a placeholder `[Автор]` / `[Author]`.
- **Pictures: renders only, no photos of the real cluster.** The owner: photos where the G20 cluster shows through look creepy because the seams are visible. Make **mockup-style images**: our own renders (bench replays, state galleries, mockups) presented cleanly — optionally placed in a drawn, neutral cluster surround (a dark rounded panel with a soft shadow, like a product shot), colours matched as we did for Twin Dial. No BMW artwork, no reference photos of other makers, no screenshot that shows the owner's profile picture or any personal text.

## 1. Rules

- Work in `bake/work/article/` (git-ignored): `article.uk.md`, `article.en.md`, `images/`, `sources.md` (where each fact and number came from: file, commit, report). **Nothing of the article is committed to the public repository.** No push. You may read the whole repository, all branches (`git log --all`), the docs, the reports, the bake reports, the galleries under `bake/work/`.
- **Facts must be true and checkable.** Every number (layers, MiB, bytes, px, ms, fps, field names, versions, dates) comes from a report, a test, a log summary or a commit; `sources.md` says which. If something is a hypothesis in the reports, the article says so in plain words ("ми думаємо", "схоже, що") — no invented certainty. Don't promise things not built.
- **Privacy:** the car's logs (`reference/car/`) may inform the story only as numbers and enum names already summarised in the reports; never a VIN, an id, a coordinate, a road, a station, a contact. The home-directory path, local IPs, SSH keys and fingerprints never appear.
- **Safety framing:** say clearly that this is a hobby project on a rooted head unit, that BMW's warnings are never hidden (the rule and how it is enforced), and that readers should not copy write-paths to their car. No instructions for rooting, for coding the car or for anything that writes to the car.
- Images: PNG, sized for the web (1600–2000 px wide for full-cluster pictures), each with a caption in both languages in the text. Use Python/Pillow via `UV_CACHE_DIR=<main checkout>/bake/work/.uvcache uv run -q --no-project --with numpy --with pillow python -I <script>`.

## 2. Material to read (start here)

- The history: `git log --all --reverse --format='%h %ad %s' --date=short` (versions 0.x → 0.7.20 of f30_cluster, Twin Dial 0.1.0 → 0.1.1, the research rounds), `README.md`, `CLAUDE.md`, `docs/*.md` (FEATURE-AUDIT, G20-DESIGN-SPEC, BAKE-*, OVERLAY-*, CAR3-*, CAR-SNAPSHOT-2026-11-15, PERF-AUDIT, PERF-NEEDLE-REPORT, BOTTOM-ROW-PACKAGE, ONE-SWITCH-DONE, UI-NATIVE-REPORT, …), `docs/strip/`, `docs/boxes/` (branches), `docs/research3/`, `docs/twin-dial/` (SPEC.md, SPEC-R3.md, V?-REPORT, r3/, r3b/), `docs/car/` (the signal session), `docs/car-fix/` (on branches `car-fix/*`), the memory notes are summarised in reports (fp16 mediump, unbound samplers, KOMBI shader size, peak memory, getter id 0, ccList, TRIP button bit, speed-limit source).
- The galleries and renders: `bake/work/shots/*/` (f30 state galleries; the owner opened `bake/work/shots/20261116-2238/index.html`), `bake/work/twin-dial/r2/E/gallery/`, `bake/work/twin-dial/mockup/` (r1 mockups, r3, r3b with contact sheets), `bake/work/trip-graph/mockups/`. A fresh gallery of both final apps will be rendered by the lead after the 0.1.1 / 0.7.21 build; ask the lead (in your final message) for it if you need a better frame, or render one yourself with `tools/shots/run.py` / `tools/shots/twin/run.py` on `main` (headless, no browser, ports 8980+, one at a time).

## 3. The story to tell (a suggested arc — improve it)

1. **The itch.** A G20 with BMW's flat digital cluster; the wish for the F30's analogue 6WB look. The MGU head unit, the community's App Kit, cluster-render; what it means to draw on someone else's cluster (we only add layers on top of BMW's scene; BMW's lamps and warnings must stay visible).
2. **First light.** The first themes on the bench and on the car; the platform's hard limits learned the hard way: 8 layers per connection, 16 MiB of pixel memory counted twice, the KOMBI resetting on a too-big shader, `mediump` being fp16 on the car, a sampler without a texture making a layer silently vanish, a getter id of 0 breaking the install. (Short war stories, one paragraph each.)
3. **Geometry and area.** How the cluster's visible area was measured (the hood line, BMW's telltales, the belt group, the gear letter), how layouts were fitted (dial units R, fits with clearances of 3–4 px), the "black box" mystery and what the research found.
4. **Baking.** From shaders to baked textures: measuring the 6WB from photos, baking faces, the bottom strip mask, fonts with Cyrillic, pictograms of our own; budgets in MiB and bytes per shader.
5. **Listening to the car.** The signal catalogue (hundreds of SOME/IP interfaces), why "it's in the catalogue" ≠ "the car sends it" (`newCCMessage` silent for 25 min), the probe app, the session at the car: the TRIP button found as bit 0x40, the "shown now" message signal (`statusDisplayedCCMessage` vs the dead 65535), the speed limit hiding in a camera event (class × 5), cruise in the HUD feed, the trip computer's factors checked against the car's own page.
6. **Twin Dial.** A second cluster from scratch in the style of the Audi virtual cockpit: references measured and rectified, the spec, mockups, the owner's choices, live tabs, messages in the centre with the safety machine (150 ms, the per-frame audit), the build by parallel agents.
7. **Performance.** Why one app was fast and the other heavy (textures re-sent on every drive-mode change; R8 / ETC2 / residency options), the start-up triple build.
8. **Working with an AI agent team.** Briefs, parallel packages, worktrees, merges, tests that must fail when the feature is broken, what went wrong (rate limits, wrong assumptions in reports, the "feet" that weren't in the references) — honest and short.
9. **The evolution** as a compact timeline (version → what changed → a thumbnail), and **what's next**.

Length: about 3 000–4 500 words in Ukrainian; sections with short headers; 6–10 images; 2–4 short code / config snippets (a shader fragment, an app.json signal declaration, a decision table) — only our own code.

## 4. Images to make

- **Hero:** a clean product-style shot of f30_cluster Comfort and of Twin Dial (renders in a neutral drawn surround, colours as on the bench / as matched for Twin Dial).
- **Before / after** pairs for the evolution (an early theme vs today; Twin Dial mockup vs the built app).
- **Diagrams** (our own drawing): the layer stack over BMW's scene (BMW's lamps showing through), the protected zones (belt group, gear gap, the box), the message safety sequence (window open → our card → window closed), the signal path (car ECU → SOME/IP → our app → cluster-render → KOMBI).
- Optional: a contact-sheet strip of states (night, message, tabs).
Every image is ours; captions say what is shown.

## 5. Deliver

`bake/work/article/article.uk.md`, `article.en.md`, `images/*.png`, `sources.md`, and a short `README.md` (how to publish: DOU accepts Markdown-ish text and uploaded images; Habr likewise). Finish with a short final message: where the files are, word counts, the list of images, any fact you could not verify, and any better frame you want the lead to render.
