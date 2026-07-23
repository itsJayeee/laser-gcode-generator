# LaserGRBL G-code Generator · User Manual

Applies to: **v1.10.0** | [中文版](MANUAL.zh-CN.md) | This manual is updated with every release; see the [CHANGELOG](../CHANGELOG.md) for history.

This tool is a **single-file HTML app** built for a robot-arm painting workflow: dual-tool powder dispensing (vibration motor) + laser burning. It also works as a general laser engraving/cutting G-code generator. No installation — open `gcode_generator.html` in a browser (Chrome/Edge recommended).

---

## 1. Quick start

1. Open `gcode_generator.html`;
2. Pick the **Job mode** at the top (Laser / Motor / Dual);
3. Drag an SVG into the import area;
4. Configure each color layer under "Layer parameters";
5. Wait for the preview to refresh (heavy work runs in the background — the UI never freezes);
6. Click "Download" to get a `.nc` file and feed it to GRBL.

**Minimal example**: draw a circle with a black stroke in Illustrator/Inkscape, export SVG, drop it in → it becomes one black layer → path mode "Outline" → power 30%, feed 1800 → download. The G-code traces the circle once.

---

## 2. Importing files (SVG)

Supports path/line/polyline/polygon/rect/circle/ellipse. **Layers are split by stroke color** — every distinct color becomes an independently configurable layer. Color detection handles all common conventions (CSS classes, inline styles, inherited `<g>` attributes, direct attributes), so exports from Illustrator / Figma / Inkscape split correctly.

> Example: to control "powder strokes" and "laser-only details" separately, draw them in two colors (e.g. red and black) in your design app; they import as two layers.

---

## 2.5 Parameter memory & project files

**Auto-memory**: all global settings and every color layer's parameters (power/feed/mode/stroke style, etc.) are saved automatically in the browser. Refreshing the page or reopening tomorrow restores everything; re-importing an SVG that uses the same colors re-applies each color's last parameters — mappings like "red = construction lines S550 F600" are set once and persist.

**Project files**: "Save project" in the import section bundles the **SVG artwork + all parameters (including powder profiles)** into a downloadable `.json`; "Load project" restores the whole state in one click. Keep one file per artwork or parameter scheme — archivable, portable across machines, Git-friendly. Note that browser memory is lost when browser data is cleared; save a project file for anything important.

---

## 3. Global settings

| Setting | Description |
|---|---|
| Output width (mm) | Physical width of the result; height follows the aspect ratio |
| Origin | Bottom-left (all-positive coordinates) or centered |
| S max ($30) | Must match GRBL `$30`, otherwise power percentages are wrong. LaserGRBL default is 1000 |
| Rapid G0 | Travel speed, used for time estimates |
| Air assist (M8/M9) | Inserts M8/M9 at start/end |
| Frame before job | Traces the bounding box at zero power to verify placement |
| Zero at start (G92 X0 Y0) | Defines the arm's **current position** as this job's origin. Essential for large work areas with variable start points |
| Motor offset dX/dY | Physical offset of the powder nozzle relative to the laser spot, see §9.6 |
| Purge before powder | Pre-runs the powder flow at a waste location, see §9.5 |

---

## 4. Path modes (laser layers)

Per layer:

- **Outline**: traces the original contours once. For cutting and line art.
- **Centerline**: extracts the skeleton of closed thick-stroke shapes and traces it once. Converts outlined fonts to single-line engraving.
- **Fill**: fills closed shapes with parallel hatch lines. See §5.
- **Bold (thicken)**: expands single-line paths into stroked bands and fills them. For single-line fonts / handwriting paths. See §6.

---

## 5. Fill parameters

- **Spacing (mm)**: gap between hatch lines. 0.1–0.2 for surface engraving; 0.05 with multiple passes to cut through thick material.
- **Hatch direction**:
  - **Fixed angle**: traditional, all lines one direction.
  - **Perpendicular to stroke**: every line runs perpendicular to the local stroke direction, length ≈ stroke width. **This fixes uneven cut-through on thick material** — long lines cruise at full speed (less energy/mm) while short lines are accel-dominated (more energy/mm), so fixed-angle hatching burns different stroke orientations differently. Perpendicular hatching makes every line the same length, so energy is uniform everywhere. Corners use angle-bisector partitioning + overhang ribs; crossings are deduplicated — zero gaps, zero overlap.
- **Fallback**: on "fat" (non-stroke-like) shapes skeleton extraction can be empty; the tool automatically falls back to fixed-angle hatching.
- **Fill + outline**: traces the contour after filling for clean edges.

> Recipe (thick cardstock lettering that burned unevenly): spacing 0.05, direction "Perpendicular to stroke", S30 F1800, engrave mode (M4), and make sure GRBL `$32=1`.

---

## 6. Bold mode & stroke styles

**Stroke width** sets the nominal width (mm). **Stroke style** controls how width varies along the stroke:

| Style | Effect | Use |
|---|---|---|
| Uniform | Constant width | Standard bold lines |
| Tapered ends | Smoothly tapers to 20% at both ends, like pen entry/exit | Handwritten feel |
| Calligraphy nib | Width varies with stroke direction (thin horizontals, thick verticals); **nib angle** adjustable — 30° is the classic italic angle | Calligraphy |
| Brush | Taper + nib combined | Signatures |
| Gunpowder burst | Noise-modulated width + burn-spill ticks along edges + scattered sparks; randomness is deterministically seeded (same file → identical G-code every time) | Explosive / gunpowder aesthetics |

Bold fills also support the perpendicular hatch direction, and rib length follows local width — ribs shorten at tapers automatically, keeping energy uniform. Closed paths have no endpoints, so taper degrades to uniform (brush degrades to nib).

---

## 7. Job modes

Section 0 at the top selects the job mode; only settings relevant to that mode are shown:

| Mode | Behavior |
|---|---|
| **Laser engraving** | All layers are laser; motor offset, purge and other powder settings are hidden; material presets available |
| **Vibration motor (powder)** | All layers dispense powder; material presets hidden (a laser-material concept); motor offset hidden (zero via G92 for a single tool); purge available |
| **Dual process (powder then laser)** | Per-layer device selection; all powder/dual-tool settings visible; output has two phases with an M0 tool swap between |

The selection is remembered across sessions.

## 8. Devices & the dual-tool pipeline (robot arm: powder + laser)

Each layer has a **Device** selector: **Laser** or **Vibration motor (powder)**. Both share one PWM channel, so G-code is emitted in two phases around a physical tool swap:

```
G92 X0 Y0            ← optional: zero at current position
(purge)              ← optional
; PHASE 1 / POWDER   ← all motor layers (M3 constant mode, coordinates include motor offset)
M0                   ← pause: install the laser head, then resume
; PHASE 2 / LASER    ← all laser layers (no offset)
```

Powder layers always run before laser layers regardless of their order numbers. With a single device, no M0 is emitted.

---

## 9. Powder layers in depth

### 9.1 How it works

The laser is a "fine pen filling areas" (scanning hatches); powder is a "wide brush in one pass" — line width is controlled directly by amplitude. Motor layers therefore **trace the centerline once** (open paths as-is; closed shapes via skeleton extraction), with the stroke's width rhythm encoded in per-segment `S` values (= amplitude). The powder phase uses **M3 constant mode** instead of M4, so amplitude does not dip during accel/decel and powder flow stays stable.

### 9.2 Width rhythm → amplitude

The stroke style (uniform/taper/nib/brush) defines the design's width rhythm, which is then **normalized**: the thinnest design point maps to the usable amplitude minimum, the thickest to the maximum, interpolated in between. Even tiny design width differences get stretched to the hardware's full expressive range.

- **Min/Max amp %**: linear mapping range when no profile is selected;
- **Contrast γ**: exaggeration. γ>1 makes thin parts thinner (sharper tips); γ<1 fuller. Default 1;
- **Look-ahead (mm)**: powder lags behind amplitude changes; this shifts amplitude changes earlier along the path to compensate. If stroke tips leave blobs, increase it (start at 3 mm).

### 9.3 Component profiles

Every "3D-printed dispenser + powder" combination has different physics (line width range, dead zone, saturation). A **component profile** stores that combination's measured amplitude→width curve; switching profiles in the layer applies the right mapping. "Linear mapping" skips the curve and uses the amp range directly.

### 9.4 Calibration wizard

In a motor layer, click "Calibration wizard…":

1. **Generate test pattern G-code** → download `calib.nc` (8 stepped-amplitude lines, 20%–90%, 30 mm each);
2. Run it with powder loaded;
3. Measure each line's width with calipers and fill the table (**enter 0 for lines with no powder** — treated as dead zone);
4. Save the profile.

> Example result: 30%→1.2 mm, 50%→2.0 mm, 70%→3.1 mm, 80%→3.5 mm, 20%→0 (dead zone). Amplitudes will then only be taken from the 30–80% effective range.

**Quick drift calibration** (daily): hopper level and humidity drift the flow. Draw one line at a reference amplitude (e.g. 60%), enter today's measured width, click the scale button — the whole curve rescales proportionally in seconds.

### 9.5 Purge

After idle time the powder in the line is "asleep" and the first stroke comes out broken. With purge enabled, the arm first goes to a waste location and dispenses for a set duration to establish flow before the real paths.

### 9.6 Motor offset calibration

The nozzle and the laser spot are physically apart. Calibrate once: draw a small cross with the laser at low power → swap to the powder head and dispense a cross at the **same coordinates** → measure the offset between the two centers → enter dX/dY. Powder-phase coordinates are then shifted so both tools hit the same lines on paper. No re-measurement needed unless the tool mount changes.

---

## 10. Material presets

The presets dropdown applies mode/power/feed/passes for a material/thickness/effect in one click. The manager supports create, save-from-layer, delete, JSON import/export, and restoring the built-in library. Data is stored locally in the browser.

---

## 11. Preview & stats

- Wheel to zoom, drag to pan, double-click to reset;
- Dashed lines are travel moves (hidden above 8,000 paths for smoothness);
- Motor layers are drawn with **variable line width** so you can see the powder rhythm;
- Hovering a layer card highlights it;
- Stats: travel before/after optimization, savings, cut length, estimated time, G-code line count.

**Performance**: all heavy computation runs in a background thread — the UI never freezes; rapid parameter changes only compute the latest one. Changing power/feed/passes (non-geometric parameters) hits the cache and rebuilds instantly. Very long G-code is truncated in the textarea; **the downloaded file is always complete**.

---

## 12. G-code structure reference

```gcode
; Generated by LaserGRBL G-code Generator (Fono) v1.8.0
G21            ; mm
G90            ; absolute
G92 X0 Y0      ; (optional) zero here
M8             ; (optional) air
; --- framing ---            (optional) zero-power bounding box
; ============ PHASE 1 / POWDER ============
G0 X-10 Y-10 S0
M3 S600                      ; purge
G4 P2.0
S0
M5
; ===== layer name · POWDER =====
M3
G0 X1.500 Y-2.000 S0         ; coordinates include motor offset
G1 X2.480 Y-1.900 S312 F1500 ; per-segment S = amplitude rhythm
...
M5
M0 ; PAUSE: swap to LASER    ; swap tool, then resume
; ============ PHASE 2 / LASER ============
M4                           ; dynamic engrave mode (M3 for cut)
G0 X0.000 Y10.000 S0
G1 X20.000 Y10.000 S300 F1800
...
M5
G0 X0.000 Y0.000 ; home
```

---

## 13. FAQ

**SVG imports as a single black layer?** Fixed since v1.1 (all color conventions supported). If it still happens, check the SVG really uses more than one stroke color.

**Thick material burns through unevenly?** See §5: switch hatch direction to "Perpendicular to stroke", verify `$32=1` and M4 engrave mode. If still short, add passes rather than endlessly lowering feed.

**Perpendicular fill produced nothing?** The shape is "fat" (not stroke-like); it automatically fell back to fixed-angle hatching — expected behavior.

**Powder blobs at stroke tips?** Increase look-ahead (§9.2).

**First powder stroke is broken?** Enable purge (§9.5).

**Powder and laser lines don't align?** Calibrate the motor offset (§9.6).

**Everything changed after swapping powder/dispenser?** Keep one component profile per combination (§9.3); handle daily drift with quick calibration (§9.4).

**Does gunpowder-burst randomness change every run?** No — it's deterministically seeded; the same file always generates identical G-code.

---

*Maintenance rule: this manual is updated alongside every release, with the version number at the top kept in sync.*
