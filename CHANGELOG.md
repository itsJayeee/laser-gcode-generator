# Changelog

This project follows Semantic Versioning. A full snapshot of each release is archived under `versions/`.

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
