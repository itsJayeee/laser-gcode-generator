# Changelog

This project follows Semantic Versioning. A full snapshot of each release is archived under `versions/`.

## v1.19.2 (2026-08-06)
### Documentation
- Rewrote both manuals (zh-CN and en) in standard user-manual register: neutral instructional tone, formalized notes/tips/examples, removed conversational phrasing — while keeping the beginner-friendly plain-language vocabulary and the glossary. No behavior changes.

## v1.19.1 (2026-08-05)
### UI tweak
- Download button is no longer sticky: it sits in normal flow at the very bottom of the left panel (visible only when scrolled to the bottom), per user preference.

## v1.19.0 (2026-08-05)
### Canvas placement & canvas framing (physical alignment helper)
- New global "Place on actual canvas": enter real canvas width/height (inch or mm toggle) and position the artwork inside it (centered, or custom offsets from the canvas's left/bottom edges).
- Coordinate zero becomes the canvas bottom-left corner; the origin selector is hidden while canvas mode is on.
- "Frame before job" now traces the **canvas border** (not the artwork bbox) at zero power and ends with `M0` so the user can verify the traced frame matches the physical canvas before resuming.
- Preview draws the canvas frame (brown) with a zero-corner marker and fits the view to the whole canvas; a red warning appears when the artwork exceeds the canvas.
- Canvas placement participates in geoKey (geometry cache) and persists via workspace/globals like other settings.
- e2e: +10 tests (T11) covering inch→mm conversion, centered/custom placement, M0 pause, overflow warning, and fallback framing; 154 total passing.

## v1.18.0 (2026-08-05)
### New path mode: Circles (laser rings + motor powder dots)
- New per-layer path mode **Circles** (laser and motor layers). The source line itself is not drawn; circles are placed along it.
- **Laser**: a ring at each sample point. Circle size fixed or random (min–max diameter), spacing equal or random (min–max mm), optional solid fill via concentric rings with a **fill density** (ring step, mm). Rings enter the normal travel optimizer.
- **Motor (powder dots)**: instead of tracing a circle, the arm parks at each sample point and dwells with the vibrator on (`G0` → `S<amp>` → `G4 P<sec>` → `S0`), piling powder into a dot. Spacing equal/random; dwell fixed or random (min–max s); single amplitude%. Dwell params are emit-time only (not in GEOF), so editing dwell never recomputes geometry.
- All randomness is seeded from path coordinates (mulberry32) — same file always generates identical G-code.
- Preview: laser rings render as normal contours; powder dots render as filled dots.
- e2e: +14 tests (T10) covering mode buttons, ring generation, fill density, determinism of random size/spacing/dwell, dwell→G4 flow, dots-only output; 144 total passing.

## v1.17.1 (2026-08-05)
### Docs & UI copy: plain-language pass (no behavior changes)
- Rewrote both manuals (zh-CN and en) in plain, non-technical language for users without a coding background. Added a small glossary table at the top of each manual explaining the everyday working terms (G-code, .nc, S, $30, F, amplitude, M3/M4, M0, purge); removed programmer vocabulary (CSS classes/inline styles/inheritance, normalization, normal direction, deterministic seed, etc.) in favor of describing what the user sees and does.
- Renamed UI labels/hints (zh + en) to match: "对比度 γ" → "粗细反差" (Width contrast), "前瞻补偿" → "提前量 mm" (Lead distance), "线性映射(振幅范围)" → "不用校准数据" (No calibration data), "垂直笔画(法线)" → "垂直于笔画" (Across the stroke), "起始处置零 (G92 X0 Y0)" → "以当前停的位置为起点 (G92)" (Start from current position), "S 上限 ($30)" → "S 上限 (=机器里的$30)" (S max (= machine $30)). Motor centerline hint and import/default hints rewritten in plain language.
- CHANGELOG remains technical English by policy.

## v1.17.0 (2026-08-03)
### Test page: easier line selection & grouping
- **Click a card to select** — the second checkbox column is gone. Clicking a line card toggles its selection (highlighted border/background); Shift-click selects the whole range; clicking checkboxes or number inputs never affects selection. "Select all" remains.
- **Marquee selection on the preview**: hold **Shift and drag** on the preview canvas to draw a selection rectangle — every test line it touches is selected (added to the current selection). Plain drag still pans. Selected lines are drawn thicker in the preview, and selection stays in sync between canvas and list. Grouping is unchanged: select, then "Group selected lines" — a horizontal sweep grabs a row, a vertical sweep grabs a column.

## v1.16.2 (2026-08-03)
### Changed
- Motor layers default to **outline** path mode (per workflow decision): powder traces the contour by default; centerline (skeleton + width rhythm) and fill remain selectable per layer.

## v1.16.1 (2026-08-03)
### Fixed (bug-hunt review pass)
- Motor job mode: newly imported layers now default to **centerline** path mode (the historical powder behavior). v1.16.0 defaulted them to "outline", which silently changed closed shapes from skeleton-tracing to contour-tracing.
- Drag reorder: moving another card no longer shifts which layer is expanded — selection follows the layer object, not its index.
- Shift range-select on layer checkboxes could crash after importing a file with fewer layers (stale range index); guarded and reset on import.
- Group batch-edit inputs are now included in focus restoration after list re-renders.

## v1.16.0 (2026-08-03)
### Layer ordering = list order, with drag-to-reorder
- The per-layer "order" number field is gone. Processing order is simply the list, top to bottom (badge shows #1, #2, …). Click **Edit order** to enter reorder mode — cards collapse and become draggable — then **Done** to lock. Powder still always runs before laser in dual mode. Saved projects/memory restore their order automatically.

### Select-all + Shift range selection
- "Select all" checkboxes for layer enable, and on the Test page for both the export column and the select column (with indeterminate state). Shift-click any checkbox to apply the click to the whole range since your last click — in all three places.

### Test page: per-variable step refill
- "Refill from steps" is now split per variable: **Fill S / Fill F / Fill amplitude**. Refilling one variable never touches hand-edited values of the others.

### Motor layers: centerline / outline / fill path modes
- Powder layers now offer three path modes: **centerline** (previous behavior, width-rhythm amplitude modulation), **outline** (trace contours at constant amplitude), and **fill** (parallel powder lines inside closed shapes at the layer's fill spacing/angle, incl. perpendicular-to-stroke direction, constant amplitude). "Thicken" is meaningless for powder and maps to centerline.

### Focus no longer lost while editing
- The layer list and test-line list re-render on every change; the input focus (and cursor position) is now restored afterward, so Tab-through editing works.

### Always-visible download + progress
- The "Generate & optimize" button is removed (generation has been automatic since v1.11); the download button sits in a sticky footer that is always visible on both pages, with a live status line showing "Computing Ns…" during long jobs and "ready" after.

### Cleanups
- Switching from the Test page back to Create regenerates the SVG output (or clears the panel if nothing is imported), so a stale testgrid can no longer be downloaded by accident.
- Group batch-edit inputs now echo the group's value when all members agree.
- Calibration wizard dialog restyled (padded single-column layout, consistent buttons); Esc closes it.

## v1.15.4 (2026-07-29)
### Performance — faster "Computing…" on moderately complex files
- **Path ordering**: the grid-based nearest-neighbor now kicks in from 400 paths (was 1200). The 400–1200 band previously ran the O(n²×candidates) exact pass and was the main cause of long "Computing…" stalls. Benchmarks (800 closed contours): 1058ms → 81ms; 1200 contours: 1970ms → 82ms — with identical resulting travel distance.
- **Entry-candidate memoization**: candidate arrays for closed contours were rebuilt on every nearest-entry comparison during ordering; they are now cached per contour for the duration of a computation (n=300 exact pass: 588ms → 260ms, same output).
- **Import sampling**: per-element sampling cap halved (6000 → 3000 points); `getPointAtLength` is the import hot path in some engines, and the later 0.03mm simplification erases any fidelity difference.

### UI
- Removed the header subtitle.

## v1.15.3 (2026-07-29)
### Clarified — S value vs. power percent
- The relationship (power% = S ÷ $30 × 100; with $30=1000, S450 = 45% and S45 = 4.5%) was easy to misread. Laser power % and motor amplitude % inputs now show their live S equivalent next to the label (e.g. "Power% =S300"), updating with the $30 setting; the Test page shows a conversion legend. Manuals gained an explicit explanation.

## v1.15.2 (2026-07-29)
### Fixed — stats bar empty on the Test page
- Test-grid generation now fills the top stats bar: travel distance (grid runs in generation order, no optimization, so before/after are equal), cut length, estimated time (per-line feed rates + rapids), and G-code line count.

## v1.15.1 (2026-07-29)
### Performance & robustness (code review pass)
- **hatchFill scanline sweep**: fill hatching now uses an active-edge table (edges sorted by lowest point; each scanline only tests edges actually spanning it) instead of testing every edge on every scanline. ~2× faster on dense fills (3000-edge polygon at 0.05mm: 612ms → 296ms); output verified byte-identical across 450 randomized cases.
- **normColor context reuse**: color normalization previously created a new `<canvas>` per call — importing a large SVG created thousands of throwaway canvases. One shared context is now reused.
- **Dialog queue**: two in-page dialogs requested back-to-back previously clobbered each other — the first callback never fired and its flow silently died. Dialogs now queue and resolve in order.

## v1.15.0 (2026-07-28)
### Dedicated Test page (replaces the test grid dialog)
- The left column now has two tabs: **Create** (the full SVG workflow, unchanged) and **Test**. The Test tab swaps the whole left panel for the test workspace — no more modal — and generates G-code the moment you enter it; every subsequent change updates live.
- **Line list**: every generated line gets its own deterministic color (shown in the preview and as a swatch in the list), a per-line **export checkbox** (unchecked lines are excluded from the G-code), and inline S/F (or amplitude %) editing.
- **Manual grouping**: tick the select checkbox on any lines and press "Group selected lines". Groups can be renamed, toggle export for all members at once, batch-edit S/F/amplitude (one value applies to every member), and share one color in the preview; "Ungroup" restores individual lines. Group membership is annotated in the G-code comments.
- Device (laser/motor) is now chosen directly inside the Test page; the active tab, layout, cells, export flags and groups all persist across sessions. Downloads via a dedicated button as `testgrid.nc`.

## v1.14.1 (2026-07-28)
### Test grid: generates on open
- Opening the test grid dialog now generates the G-code immediately — the Generate button is never required (it remains as a manual refresh). Combined with v1.14.0's live editing, the whole flow is: open → tweak cells/settings → everything updates in real time.

## v1.14.0 (2026-07-28)
### Test grid: dual-direction steps + live editing after generation
- Both S and F (and motor amplitude) now have **row and column step increments**: cell = start + row × row-step + col × col-step. Sweep S down columns, F across rows, both at once, or zero one direction — any layout of the classic material-test card is expressible. Defaults keep the previous behavior (S per row, F per column).
- After the first Generate, the grid stays live: editing any cell, changing a setting, or refilling **auto-regenerates the G-code and preview immediately** — no repeated button presses. A normal SVG generation or a calibration pattern takes the output back over and stops the grid's auto-regeneration.

## v1.13.0 (2026-07-28)
### Test grid: per-cell editable S/F
- The grid dialog now renders a real editable table (the full Google-Docs-table idea): every cell has its own S and F inputs (amplitude % for motor), prefilled from the row/column step rules. Edit any cell freely; "Refill all cells from steps" re-applies the rules. Resizing rows/columns keeps existing cell values and fills new cells from the rules. Cell values persist with the other grid settings, and G-code annotations / the parameter map reflect the actual per-cell values.

## v1.12.0 (2026-07-28)
### Test grid generator
- New "Test grid" section opens a generator that needs no SVG: pick rows × columns with a Google-Docs-style sweep picker, set line length / row gap / column gap, and get stepped test lines in one click. Laser: rows step S, columns step F (S/F start + increment; M3/M4 selectable). Motor: rows step amplitude % with a fixed F (M3 forced). Each cell is annotated in the G-code (`; r2c3: S500 F1200`), a copyable parameter map is shown (highest-S row on top, matching the canvas), the footprint is displayed live, output previews on the canvas and downloads as `testgrid.nc`, and settings are remembered.

### In-page dialogs (fixes "Save project does nothing")
- All browser-native popups (`prompt`/`confirm`/`alert`, 15 call sites) are replaced with an in-page dialog. Browsers can silently suppress native dialogs ("prevent this page from creating additional dialogs"), which made Save project — and draft-discard confirmations — appear to do nothing. In-page dialogs are immune.

### Motor layers: single amplitude for uniform strokes
- With stroke style "Uniform" the layer now shows a single **Amplitude %** field (min/max/γ/look-ahead hidden — they have no effect on a constant-width line) and the G-code uses that amplitude directly, bypassing profile curves. Variable-width styles keep the full min/max/γ/look-ahead controls.

### Default feeds per device
- New layers default to F1000 (laser) / F3000 (motor); switching a layer's device or the job mode resets the layer feed to that device's default. Per-color parameter memory still overrides defaults for previously used colors.

### Preview
- Travel (rapid) dashed lines are now green (#4f7a3f) so cutting and travel paths are distinguishable at a glance; legend updated.

## v1.11.0 (2026-07-27)
### Fixed — project load failing intermittently
- "Load project" (and SVG / preset-JSON pickers) reset the file input **synchronously** while the file was still being read asynchronously; on some engines (notably WebKit) clearing the input invalidates the `File` mid-read, so loading silently did nothing. The reset now happens only after the read settles, and a read failure shows an explicit "invalid project file" alert instead of failing silently. The failure looked mode-dependent (reported as "can't load in motor mode") but was a timing race.
- Side fix from the same audit: selecting the same SVG file twice in a row did nothing (input value was never cleared); all three file pickers now clear after the read completes, so re-selecting the same file always works.

### Named project files
- "Save project" now prompts for a project name (defaults to the SVG's file name); the file downloads as `<name>.json` and the name is stored inside the project, so the next save suggests it again.

### G-code file name follows the SVG
- The downloaded `.nc` (and the name badge above the G-code preview) now uses the imported SVG's base name — `dragon.svg` → `dragon.nc`. The SVG name is stored in project files so it survives save/load. Calibration patterns still download as `calib.nc`.

### Material presets: powder field
- Presets gain a free-text **Powder** field (vendor · color · grain · batch) alongside material / thickness / effect. Ignition parameters depend on the powder × canvas combination, so one preset records one tested combination. Existing presets migrate automatically (empty powder). As always, preset values are estimates until measured.

### Material presets: draft mode with explicit Save
- The preset manager no longer persists every keystroke. All edits (fields, new, delete, import, load built-in) act on a draft; a new **Save presets** button writes them to storage. Closing with unsaved changes asks for confirmation — accidental edits or deletions are undone by simply closing the window.

### Calibration wizard: unified save semantics
- Profile deletion and quick drift-calibration previously took effect immediately; they now also act on a draft and only persist via **Save profile**, matching the preset manager. Closing with unsaved changes asks for confirmation.

## v1.10.0 (2026-07-23)
### Parameter persistence — no more re-entering settings
- All global settings and per-color layer parameters auto-save to browser storage and restore on reload; re-importing an SVG with the same stroke colors re-applies each color's last-used parameters (power, feed, mode, stroke style, powder settings, layer name)
- Project files: "Save project" exports the SVG artwork + all parameters + powder profiles as one JSON; "Load project" restores the entire workspace in one click — archivable and portable

## v1.9.1 (2026-07-23)
### Fixed — "Computing…" hang
- v1.9.0's raster-removal accidentally deleted the `setStat`/`fmtTime` helpers; any generation then threw inside the output builder and the Generate button stayed on "Computing…" forever. Helpers restored.
- Worker callbacks now wrap output building in try/catch, so any future exception surfaces as an error message instead of a stuck button.
- All `localStorage` access is wrapped in try/catch; restricted browser storage no longer kills the whole script on load.
### Cleaned
- Removed duplicate `l_strokeW` i18n keys; verified zero leftover raster references and no orphan functions.

## v1.9.0 (2026-07-22)
### UI restructure: job modes
- New top-level **Job mode** selector — Laser / Vibration motor (powder) / Dual process — remembered across sessions
- Laser mode hides all powder settings (motor offset, purge) and per-layer device selectors
- Motor mode hides material presets (a laser-material concept) and the motor offset (zero via G92 for a single tool); all layers dispense powder
- Dual mode keeps per-layer device selection with the full settings surface

### Removed
- Raster bitmap engraving (PNG/JPG import, grayscale/dither/threshold modes) removed to keep the tool focused on the vector powder+laser workflow; the last version with raster support is archived as `versions/gcode_generator_v1.8.0.html`

## v1.8.0 (2026-07-20)
### Dual-tool robot-arm painting pipeline (powder + laser)
- Per-layer device selection: Laser or Vibration motor (powder dispenser); both share one PWM channel, so G-code is emitted in two phases — PHASE 1 runs all powder layers, then `M0` pauses for the physical tool swap, then PHASE 2 runs all laser layers
- Motor layers trace the stroke centerline once (open paths as-is, closed shapes via skeleton extraction) with per-segment `S` amplitude modulation; powder phase uses `M3` constant mode so amplitude does not dip during accel/decel
- Width rhythm from stroke styles (uniform / taper / nib / brush) drives amplitude: design widths are normalized and stretched onto the usable amplitude range, with a contrast γ control
- Powder component profiles: per-component measured amplitude→line-width curves stored locally; amplitude is inverse-interpolated from the curve so different 3D-printed dispensers and powders reproduce the same design rhythm within their own dynamic range
- Calibration wizard: one-click stepped-amplitude test pattern (`calib.nc`), fill-in measurement table, plus a quick single-point drift calibration that rescales the whole curve before daily sessions
- Look-ahead compensation shifts amplitude changes earlier along the path to counter powder-flow lag
- Purge routine before the powder phase (position, duration, amplitude configurable) to establish powder flow
- Motor tool offset (dX/dY) applied to powder-phase coordinates so both tools hit the same lines; optional `G92 X0 Y0` zeroing at the current arm position for variable start points
- Preview renders motor layers with variable line width matching the powder rhythm

## v1.7.0 (2026-07-20)
### Performance — no more UI freezes on complex graphics
- All heavy geometry (skeleton thinning, rib generation, path ordering) now runs in a Web Worker built from the app's own pure functions; the UI stays responsive and a newer parameter change terminates the stale computation (latest-wins)
- Grid-accelerated nearest-neighbor ordering for large jobs: 20,000 segments ordered in ~1 s instead of the previous O(n²) freeze; 2-opt is applied adaptively only for small jobs
- Geometry cache keyed by geometry-affecting parameters: changing power / feed / passes / mode rebuilds G-code strings only, with zero geometry recomputation
- Preview rendering batches each layer into a single path and skips travel dashes above 8,000 contours; G-code textarea preview is truncated at 800 KB (downloaded file is always complete)
- Robustness: perpendicular fill falls back to fixed-angle hatching automatically when skeleton extraction yields nothing (fat, non-stroke shapes)

### Raster image support (PNG / JPG / JPEG / WebP / BMP)
- Drop or select a bitmap to enter raster engrave mode with its own parameter panel
- Three modes: Grayscale (per-pixel laser power modulation via M4, 32 quantized levels with run-length merging), Dither (Floyd–Steinberg error diffusion, ideal for cardstock), and B/W threshold
- Serpentine scanlines along X with skip-white travel; adjustable line spacing, max/min power, feed, invert, and skip-white threshold
- Images are downscaled to max 1600 px for processing; transparency is treated as white; preview shows the placed image with its bounding box

## v1.6.0 (2026-07-20)
- New stroke style "Gunpowder burst": noise-modulated stroke width + burn-spill ticks along stroke edges + scattered sparks around strokes
- Decorations use a deterministic seed (mulberry32), so the same file always produces identical G-code
- Regular outline pass is automatically disabled in gunpowder style
- App version embedded in the header UI and in the G-code file comment

## v1.5.0 (2026-07-20)
- New "Stroke style" option for Bold mode: Uniform / Tapered ends / Calligraphy nib (adjustable angle) / Brush (taper + nib)
- strokeToOutline and normal-direction ribs now support per-point variable width, with two smoothing passes to avoid outline jaggies
- Closed paths degrade gracefully (taper→uniform, brush→nib) to avoid a pinched seam
- New densify() polyline refinement to give width variation enough resolution

## v1.4.0 (2026-07-20)
- Corner overhang ribs: hatch lines extend past each vertex by halfW·tan(θ/2)+spacing and are clipped by the angle bisector, seamlessly tiling the corner fan
- Fixes wedge-shaped gaps on the outside of sharp turns (measured 0.00% true gap for 60°–150° turns)
- Fill mode measures local stroke width at each vertex to compute the overhang
- rayClip now supports nearest-inside-interval lookup for overhang samples slightly outside the polygon

## v1.3.0 (2026-07-20)
- Angle-bisector partitioning: rib families on both sides of a corner meet exactly at the bisector — zero overlap
- Occupancy-grid dedup (trimOverlapRibs): no double exposure at stroke crossings / skeleton junctions (measured 0.76% → 0.00%)
- Fill mode extends skeleton endpoints along the tangent to the contour boundary, covering stroke tips
- Sampling phase is continuous across segments, keeping spacing uniform through corners

## v1.2.0 (2026-07-20)
- New "Hatch direction" option: Fixed angle / Perpendicular to stroke
- In perpendicular mode every hatch line has length ≈ stroke width, so accel/decel and heat accumulation are uniform — fixes uneven cut-through on thick material caused by mixed long/short lines
- Bold mode generates perpendicular ribs along the centerline; Fill mode extracts the skeleton via raster thinning, then casts ribs along local normals clipped to the contour

## v1.1.0 (2026-07-20)
- Fixed SVG layer detection: color is now read via getComputedStyle, supporting CSS classes, inline styles, and inherited <g> attributes
- Fallback walks up the ancestor chain for stroke/fill attributes
- Fixes SVGs exported from Illustrator/Figma/Inkscape collapsing into a single black layer

## v1.0.0
- Initial release: SVG import with per-stroke-color layers, four path modes (centerline / outline / fill / bold), path optimization (nearest-neighbor + 2-opt), GRBL G-code generation, material preset manager
