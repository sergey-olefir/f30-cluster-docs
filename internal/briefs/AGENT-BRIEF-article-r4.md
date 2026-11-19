# Brief: the article, revision 4: impersonal register, much deeper technical detail

Audience of this brief: one agent, package **ART4**. You revise an existing article. Read this whole brief before you start.

Audience of the article: engineers and car enthusiasts who read DOU / Habr. They enjoy a shader, a homography or a
SOME/IP field name, but they know nothing of BMW internals or of this repository. The text must stay readable for them:
every deep section opens with one plain sentence of why, then the mechanics, then a number, then a picture.

## 0. Where things stand, and what the owner said (2026-11-19)

The article exists, revision 3: `bake/work/article/article.uk.md` (lead, ~5 400 words), `article.en.md` (~6 300 words),
`images/` (11 pictures per language), `sources.md`, `README.md`, `src/` (the scripts that made the pictures), `render/`.
Read all of it first, in full, including `sources.md` and the scripts in `src/`. The first brief is
`docs/AGENT-BRIEF-article.md` (its rules still hold unless this brief changes them).

The owner's feedback on revision 3, verbatim (Ukrainian) and in English:

> «"Власникові авто бракувало" в статті мені не подобається. ми можемо її взагалі обезособити. таке наче ми пишемо
> реферат, а не історію від третьої особи. … Зараз все виглядає трішки сухо, я хочу більше деталей: як ми рахували як
> розслоювати зображення на елементи? як рахували тіні, тощо?»
>
> "I don't like 'the car's owner was missing…'. We can make the article fully impersonal, as if we were writing a
> technical paper (реферат), not a story told in the third person. … It all looks a bit dry; I want more detail: how did
> we work out how to split an image into layers / elements? how did we compute the shadows, and so on?"

Feedback from earlier revisions that must SURVIVE this one (do not lose any of it):
- the real scale: about 700 signals listened to; sessions of up to an hour in the car over SSH; up to 15 agents in
  parallel at the peak and more than 60 agents over the project; gigabytes of raw data;
- the signal collector app (Signal Probe) is a highlight: what it is, how it works, what it found;
- technical detail on how textures were baked, how overlaps were found, how fonts were chosen;
- the early "creepy" photos of the real cluster before the scales were drawn (`photos-early_*.png`).

## 1. Task 1: the register (impersonal, like a technical paper)

- **No persons at all.** Remove "власник / власник авто / власникові / за підрахунком власника / owner", "я", and
  third-person storytelling ("he decided", "his reaction was"). Also avoid "ми" as a character in a story. Write as a
  technical paper / реферат does: impersonal and passive constructions.
  - Ukrainian: «шкали виміряно з фотографій», «для цього застосовано гомографію», «рішення ухвалено за макетами»,
    «на авто перевірено того ж вечора», «з'ясувалося, що…», «виявилося…», «підхід такий: …». Impersonal -но/-то forms,
    «було + participle», infinitive constructions («треба було…», «вдалося…»), noun phrases («вимірювання дало…»).
  - English: passive voice and nominal style ("the scales were measured from photographs", "the fix was verified on the
    car the same evening", "the decision was taken on the mockups"). An occasional "we" in the English is tolerable only
    where the passive gets clumsy; prefer none.
- **The human / AI-team split stays a topic, but impersonally**: "the work was done by a team of AI agents (one lead
  agent, parallel package agents) under human direction: a person set the goals and rules, took decisions, photographed
  and pressed buttons in the car; the agents measured, drew, wrote code, tests and reports." Then never personalise again.
- **Numbers that only the owner stated** (60+ agents, an hour in the car) were marked "за підрахунком власника". Now
  mark them impersonally as estimates not checked against files: «за оцінкою (у файлах не перевірено)», "by estimate,
  not checked against the files", or check them against git / the worktree list / logs and drop the qualifier if they
  hold. `sources.md` keeps saying where each comes from (there you may write "the owner's statement").
- Quotes of the owner's words ("макети дуже круті", "стрілка ніби злегка підвисає") become impersonal: «оцінка за
  макетами була однозначно позитивна», «на авто помітно, що стрілка ніби злегка підвисає». No quotation marks around a
  person's words.
- Keep it readable: impersonal is not bureaucratic. Short sentences, concrete nouns, no «здійснено реалізацію».

## 2. Task 2: much more technical depth (the core of this revision)

The owner wants the HOW. For each topic below: read the code and the reports, understand the real algorithm, and explain
it with its real parameters (thresholds, radii, sizes, counts, error in px, MiB, ms). Include formulas where the code
has them, and short snippets of OUR code (5–20 lines; GLSL from `shared/shaders.js`, Python from the bake tools).
Every number goes in `sources.md` (file + line or report + section, or commit). Where the reports say "hypothesis", the
text says so. Do not invent a mechanism the code does not have: if you cannot find how something was done, say in your
final message what is missing, do not guess.

### 2.1 Splitting the 6WB image into elements and layers (the owner's first question)

How a photograph-and-render of an analogue cluster became a set of separately drawn, separately animated elements:
what the elements are (face, numerals, ticks, rings, chrome, hub, needle, red zone, warning areas, glass, …), how each
was isolated and measured, which became baked static layers and which stayed live in a shader, and why the split is
where it is (the 8-layer cap per connection, 16 MiB of pixel memory counted twice, the KOMBI's shader size limit, what
must animate). Start with:
- `bake/README.md`, `docs/AGENT-BRIEF-bake-6wb.md`, `docs/BAKE-P0-inventory.md` (circle / ellipse fits with RANSAC,
  sub-pixel edges, perspective, the reflections pointing at the window), `docs/BAKE-P0-report.md`,
  `docs/BAKE-P10-levels.md` (levels and colour), `docs/BAKE-REPORT.md`, `docs/BAKE-C9-overlays.md`;
- `bake/src/` packages `p0`, `C1`…`C10`, `P9`, `P10`, `G20`, `common`, `qc.py`, `validate_manifest.py`, and `bake/manifest`,
  `bake/report`;
- `tools/bake_scene.py` (packing baked layers into RGBA scene textures, straight alpha), `apps/f30_cluster/hud/baked/`,
  `apps/f30_cluster/hud/scene/`, the layer stack in `shared/hud.js` and `shared/shaders.js`;
- `docs/G20-DESIGN-SPEC.md`, `docs/F30-FONT-SPEC.md`, `docs/PERF-AUDIT.md`, `docs/PERF-NEEDLE-REPORT.md`.
- Twin Dial's own decomposition: `tools/twin-dial/BAKE.md`, `bake_face.py` (drawn at 3x, reduced with LANCZOS; R8 faces
  plus a regions JSON), `face_shade.py` (the numpy reference of the shader decode: material from the byte value and the
  polar position, smoothstep bands, linear light), `test_shader.py` / `compare_face.py` (GPU shader checked against numpy),
  `docs/twin-dial/SPEC.md`, `SPEC-R3.md`, `ENGINE-CONTRACT.md`, `docs/twin-dial/r3b/B1-REPORT.md`, `R3B-INTEGRATION.md`.
  The "one byte per pixel encodes the material, the shader decides the colour" idea deserves a clear explanation and a
  picture.

### 2.2 Shadows, light, glow (the owner's second question)

Find every place where shade and light are computed or baked, and explain how: needle shadows, ring and bezel inner
shadows, the face gradient / vignette, the hub, reflections removed or kept, glow around pointers, the night look
(linear-light multiply), the red zone's antialiased edge. Search `shared/shaders.js`, `bake/src/**`, `tools/twin-dial/**`,
`tools/g20sim/**` for shadow, shade, blur, offset, glow, vignette, gradient, gamma, linear, sRGB. Known starting points:
- `tools/g20sim/calib_glow.py`: the stock pointer glow's gain fitted by least squares, photo − base = g · (glow₁ − base),
  over the pointer zone with other elements masked (`docs/G20SIM-REPORT.md`, `G20SIM-D-live.md`);
- `docs/BAKE-P0-inventory.md`: where the light comes from in the reference photos and why it was judged a reflection;
- `docs/POLISH-A-REPORT.md` and the other POLISH reports, `docs/OVERLAY-REPORT.md`, `docs/OVERLAY-PARTS-REPORT.md`,
  `docs/strip/M-REPORT.md`, `docs/strip/ZR-REPORT.md`, `docs/twin-dial/VC-REPORT.md`, `VT-REPORT.md`, `FIN-REPORT.md`,
  `MOCKUP-SPEC-R3B.md`, `R2-B-REPORT.md`, `r3/K1-REPORT.md`;
- the fp16 lesson (`mediump` is half precision on the car: a `length()` of pixel offsets over 256 px overflows; highp and
  scaled lengths) belongs here if a shadow or glow distance was where it bit.
If a shadow is a drawn gradient rather than a computed one, say that plainly; that is also an answer.

### 2.3 The rest of the "how", deeper than revision 3

- **Geometry and overlap search** with the stock G20 scene: `tools/overlay/` (`rectify.py`: homography plus the phone
  lens' cubic radial residual of ±3.5 px; `catalogue_g20.py`, `relations_g20.py`, `inventory_parts.py`, `plan_dials.py`),
  `tools/ref/` (`grid_homography.py`, `sift_align.py`: CLAHE, SIFT 6 000 features, Lowe ratio 0.78, RANSAC),
  `docs/OVERLAY-PHOTO-PROTOCOL.md`, `OVERLAY-PLAN-DIALS.md`; the protected zones; the bottom strip mask with its lanes
  (`tools/strip/bake_mask.py`, `docs/strip/`); the "black box" (`docs/boxes/` on branches, `docs/research3/`,
  `docs/research3/MB-UPSTREAM-DRAFT.md`, the f30 diagnostic options `diagMbGap` / `diagMbCover` in
  `apps/f30_cluster/app.json`). **News of 2026-11-19: on the car, the diagnostic experiments did hide the black
  rectangle.** Which option did it (the gap cut round the gear letter, the extra cover layer, or the drive-mode switch)
  is not known to you yet: write the black-box section so that it ends with "a diagnostic experiment on the car hid it"
  and leave one clearly marked placeholder `[МЕХАНІЗМ: уточнити]` / `[MECHANISM: to confirm]` for the lead to fill; do
  not guess which option worked.
- **Fonts**: `docs/F30-FONT-SPEC.md`, `tools/f30font/`, `tools/bake_fonts.py` (atlases of coverage bytes in JSON; the car
  never rasterises), how candidates were compared with the references, Cyrillic coverage.
- **Shader engineering on a weak target**: the KOMBI reset at ~26.7 k characters of shader, one look per layer source
  with `#if STYLE` resolved in JS, unbound samplers dropping a layer, the getter id 0 install failure.
- **The signal work** stays as is, plus one short paragraph in "what next": Signal Probe 0.2.0 (commit `819854b`) adds
  two sets for a drive, S14 (distance control: the vehicle ahead, the distance control's object list with distances,
  speeds and accelerations, the collision-warning request, the car's own accelerations) and S15 (the front camera's and
  the fused object lists). Nothing of it has been seen on a drive yet; say so. Source: `docs/car/CP-REPORT.md` §13.

Length: the Ukrainian text may grow to about 8 000–10 000 words; keep the English a faithful adaptation. Structure the
depth so a reader can skip it: short main sections, and inside them clearly headed technical subsections
(«Як це працює», "How it works"), or a closing «Технічні подробиці» part with one subsection per topic. Your call; the
text must not become a manual.

## 3. Pictures

Add at least 5 new pictures (both languages for any with text), PNG, 1600–2000 px wide for full-cluster pictures.
Suggested, adapt to what you find:
1. **Exploded layer view** of f30_cluster Comfort: the baked layers of `apps/f30_cluster/hud/baked/` and the live shader
   layers drawn as a tilted stack with labels, BMW's own scene at the bottom.
2. **Element decomposition** of one dial: our render with each element outlined and named (face, ticks, numerals,
   rings, hub, needle, red zone), and which are baked vs live.
3. **Shadows / light**: the same dial with the shade terms off and on (render it with the real code if a switch exists;
   otherwise a diagram of the terms with their formulas), plus the glow fit if it is clear.
4. **Twin Dial's R8 face**: the raw byte image beside the decoded colour face, with the material bands named.
5. **Rectification**: a G20 photo from `reference/g20/` (the owner's own photos; crop to the cluster, blur the map and
   any street name as `photos-early_*` does) warped onto the 1920 × 720 panel grid, with the residual.
6. Optional: a font atlas tile, the strip mask lanes, a timeline of the layer budget.

Image rules: renders, our own diagrams and the owner's own G20 photos only. **Never** put into the article an image from
`reference/6wb/`, `reference/vc/` or any other reference picture of another maker or from the internet (those are
third-party photos and BMW / Audi artwork): describe what was measured on them and show our own result instead. No
screenshot with the owner's profile picture (`<scratchpad>/shots/` and similar), no personal text, no map with a street
name. Re-use and extend the scripts in `bake/work/article/src/`; keep every new script there.

## 4. Rules (all of them hold)

- Work only in `bake/work/article/` (git-ignored). **Never edit a tracked file, never commit, never push.** You may read
  the whole repository and all branches (`git log --all`, `git show <branch>:<path>`).
- **Never connect to the car**, no SSH, no `<car SSH key folder>`. Never print or copy anything from `<car SSH key folder>/`.
- No browser automation. If you need a fresh render: the bench and the gallery tools on `main`
  (`tools/shots/run.py --out <folder>`, `tools/shots/twin/run.py --real --out <folder>`), headless, ports 8980–8999, one
  bench at a time, output under `bake/work/article/render/`. Python with
  `UV_CACHE_DIR=<repo>/bake/work/.uvcache uv run -q --no-project --with numpy --with pillow [--with opencv-python-headless] python -I <script>`.
- Privacy: no VIN, plate, address, place, road, station, contact, phone, e-mail, account or GitHub name, no IP, key or
  fingerprint, no home-directory path, in the texts, the pictures or `sources.md`. The car's logs inform the text only
  as numbers and enum names already in the reports.
- Safety framing stays: a hobby project on a rooted head unit; BMW's warnings are never hidden (the rule and how it is
  enforced); no instructions for rooting, coding or writing to the car.
- Brands: named openly in the article (BMW G20, F30's 6WB, "in the style of the Audi virtual cockpit"); the article
  lives outside the repository.
- Facts true and checkable; hypotheses flagged; nothing promised that is not built.

## 5. Deliver

- `article.uk.md`, `article.en.md` (revision 4), new pictures in `images/`, updated `sources.md`, `README.md` (image
  order and captions updated), the new scripts in `src/`.
- `CHANGES-r4.md`: what changed and why, a list of every removed personal phrase with its replacement (a table is fine),
  and the new technical sections with the code / report each is based on.
- Keep revision 3 as `article.uk.r3.md` and `article.en.r3.md` before you overwrite (copy first).
- Final message (short): where the files are, word counts, new pictures, any fact you could not verify, any mechanism
  you could not find in the code, and anything you want the lead to render or check.
