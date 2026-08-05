# LaserGRBL G-code Generator · User Manual

Applies to: **v1.19.1** | [中文版](MANUAL.zh-CN.md) | This manual is updated with every release; see the [CHANGELOG](../CHANGELOG.md) for history.

This tool is **one HTML file** — open it in a browser (Chrome/Edge recommended) and it just works, nothing to install. It was built for a robot-arm painting workflow: a vibration motor that dispenses gunpowder plus a laser that ignites it, working as a pair. It also works as a regular laser engraving/cutting G-code generator.

## A few words to know first

These terms come up throughout the manual. Come back to this table whenever one is unclear:

| Term | Meaning |
|---|---|
| **G-code** | The instruction file the machine reads — one command per line, telling the arm "go here, at this power" |
| **.nc file** | A G-code file. You download it and feed it to LaserGRBL |
| **S value** | The "strength" number in a command. For the laser it means power; for the motor it means vibration strength |
| **$30** | A setting inside the machine that says what a "full-strength" S value is. LaserGRBL's default is 1000 |
| **Power% vs. S** | power% = S ÷ $30 × 100. With $30=1000, S450 = 45% but S45 is only 4.5% — watch the digits |
| **F value** | Travel speed in mm per minute. F600 = 60 cm per minute |
| **Amplitude** | How hard the powder motor shakes (also written as an S value). Harder shake = thicker powder line |
| **M3 / M4** | Two ways of switching the laser/motor on. M3 = constant strength; M4 = the machine auto-adjusts strength while speeding up and slowing down. Powder always uses M3; laser engraving usually uses M4 |
| **M0** | The "pause and wait for a human" command. Dual jobs use it to stop so you can swap tool heads |
| **purge** | Before real powder work starts, shake out powder for a few seconds off to the side to wake up powder that has settled in the tube |

---

## 1. Quick start

1. Double-click `gcode_generator.html` to open it;
2. Pick the **Job mode** at the top (Laser / Motor / Dual);
3. Drag an SVG file into the import area;
4. Set the parameters for each color layer under "Layer parameters";
5. Wait for the preview to refresh (complex drawings are computed in the background — the screen never freezes);
6. Click "Download" to get a `.nc` file and run it in GRBL.

**Simplest possible example**: draw a circle with a black outline in Illustrator or Inkscape, export as SVG, drop it in → one black layer appears → path mode "Outline" → power 30%, speed 1800 → download. The G-code traces the circle once.

---

## 2. Importing files (SVG)

Whether you drew your file in Illustrator, Figma, or Inkscape, as long as lines have different colors, they will automatically become separate layers when imported — no special setup needed. However many colors are in the drawing, that's how many layers you get, each with its own parameters.

> Example: to control "strokes that get powder first" and "laser-only details" separately, just draw them in two colors (say red and black) in your design app. After import they are two independent layers.

---

## 2.5 Parameter memory & project files

**Auto-memory**: all global settings and every color layer's parameters (power/speed/mode/stroke style, etc.) are remembered automatically by the browser on this computer. Refresh the page or come back tomorrow and everything is restored; re-import an SVG that uses the same colors and each color gets its last-used parameters back — habits like "red = construction lines S550 F600" are set once and stick.

**Project files**: "Save project" in the import area packs the **SVG artwork + every parameter (including powder profiles)** into one downloadable `.json` file; "Load project" restores everything in one click. Keep one file per artwork or per parameter recipe — they can be archived long-term or moved to another computer. Note: clearing browser data wipes the auto-memory, so save a project file for anything important.

**Project naming**: "Save project" asks you to name the project (defaults to the SVG's file name) and downloads `<name>.json`; the name is stored in the project so the next save suggests it again.

**G-code file name**: the downloaded `.nc` automatically uses the imported SVG's name — `dragon.svg` produces `dragon.nc`. Works after loading a project too. Calibration patterns still download as `calib.nc`.

---

## 3. Global settings

| Setting | What it does |
|---|---|
| Output width (mm) | Real width of the finished piece; height follows the drawing's proportions automatically |
| Origin | Bottom-left (all coordinates positive) or centered (origin in the middle of the drawing) |
| S max (= the machine's $30) | Enter the same number as the `$30` setting inside your machine, or the power percentages will be wrong. LaserGRBL default is 1000. Power/amplitude boxes show their matching S value live |
| Rapid G0 | Speed of fast pen-up moves, used for time estimates |
| Air pump/fan (M8/M9) | When checked, turns on at the start and off at the end |
| Frame before job | Before the real job, the machine traces the drawing's outer box at zero power so you can check position and size. With "Place on actual canvas" on, it traces the **canvas border** instead, see §3.5 |
| Place on actual canvas | Enter your real canvas size (inch or mm) and position the artwork inside it, see §3.5 |
| Start from current position (G92) | Treats wherever the arm is **parked right now** as this job's starting point. A must for large canvases where the start point isn't fixed |
| Motor offset dX/dY | The powder nozzle and the laser spot aren't in the same place — enter the difference, see §9.6 |
| Purge before powder | Shake out powder at a chosen spot for a few seconds before the real job, to wake up the powder flow, see §9.5 |

---

## 3.5 Place on actual canvas (alignment helper)

Solves "the drawing lands outside / doesn't line up with my canvas". When enabled:

1. **Enter the canvas size**: width × height, in inch or mm (a 10×10 inch canvas is just 10 and 10);
2. **Choose artwork position**: "Centered" puts it in the middle; "Custom" takes distances from the canvas's left and bottom edges;
3. The preview draws a **brown canvas frame** with a dot at the zero corner, so you see exactly where the artwork sits; a **red warning** appears if it sticks out;
4. **Coordinate zero = the canvas's bottom-left corner.** Point the nozzle tip (or laser dot) at that corner and zero before starting;
5. With "Frame before job" on, the machine first traces the canvas border at zero power and then **pauses (M0)** — if the traced frame matches your physical canvas, hit resume in LaserGRBL; if not, hit stop, move the canvas or re-zero, and run it again.

> Tip: if the traced frame comes out tilted, the canvas is sitting tilted — move the canvas to match the frame (the machine will draw exactly where the frame went). Bumped the arm? Re-point the nozzle at the canvas corner, re-zero, trace the frame again to confirm.

**Default speeds**: new layers default to F=1000 (laser) / F=3000 (motor); switching device or job mode restores that device's default. Colors you've used before still get their last-used values back first.

**Processing order**: the layer **list, top to bottom, is the processing order** (#1, #2 badges). Click "Edit order" to enter drag mode (cards fold up and can be dragged), "Done" to lock. In dual jobs the powder layers always run first; list order only decides the sequence inside each phase.

**Select all & range select**: the layer enable column and the test page's export/select columns each have a "select all" box; any column of checkboxes supports **Shift range-select**: click one, hold Shift, click another, and everything in between follows.

**Other**: after changing a value the cursor returns to the same box (press Tab to hop through them); there is no "Generate" button (generation is always automatic), the download button sits at the very bottom of the left panel (scroll down to reach it), with a status line under it showing "computing Ns…" or "ready".

## 4. Path modes (laser layers)

Each layer picks its own:

- **Outline**: trace the drawn outline once. Good for cutting and line art.
- **Centerline**: for thick-stroke shapes, run once along the middle. Good for turning outlined fonts into single-line engraving.
- **Fill**: sweep the inside of closed shapes full of parallel lines. Settings in §5.
- **Thicken**: widen a single line into a stroke with real width, then fill it. Good for single-line fonts and handwriting. Settings in §6.
- **Circles**: draw a circle every so often along the line; the line itself is not drawn. Circle **size** and **spacing** can each be fixed or random (random results come out identical on every generation of the same file); check "Fill circles" for solid circles, with a **fill density** (mm — smaller numbers fill more solidly).

---

## 5. Fill settings

- **Spacing (mm)**: gap between neighboring sweep lines. Surface engraving usually 0.1–0.2; to cut deep into thick material use 0.05 plus multiple passes.
- **Line direction**:
  - **Fixed angle**: all lines run the same way — the traditional method.
  - **Across the stroke**: every line cuts straight across the stroke, so each is only about as long as the stroke is wide. **This is the fix for "thick material burns through in some places but not others"** — long lines cruise at full speed mid-line and burn shallow, short lines never get up to speed and burn deep, so strokes pointing different ways heat unevenly. With "across the stroke" every line is equally short and everything burns equally deep. Corners and crossings are handled automatically — no gaps, no double-burn.
- **Fallback**: shapes that are too "fat" to look like a stroke (a solid disc, say) have no middle line to find, and quietly fall back to fixed-angle fill.
- **Fill + outline**: when checked, traces the outline once after filling, for a cleaner edge.

> Example recipe (lettering on thick card stock that used to burn unevenly): spacing 0.05, line direction "across the stroke", S30 F1800, engraving mode (M4), and make sure the machine has `$32=1`.

---

## 6. Thicken mode & stroke styles

The **stroke width** slider sets the base width (mm). The **stroke style** decides how width changes along the stroke:

| Style | Effect | Use for |
|---|---|---|
| Uniform | Same width start to finish | Standard thick lines |
| Tapered ends | Smoothly thins out at the start and end, like a pen entering and leaving the paper | Handwritten feel |
| Calligraphy (angled) | Width follows stroke direction (thin horizontals, thick verticals); the **nib angle** is adjustable: 30° is the classic Western italic angle, 0° feels like a Song-dynasty typeface | Calligraphy |
| Brush | Tapering + nib combined | Signatures, inscriptions |
| Gunpowder burst | Width wobbles randomly + short scorch spikes on the edges + sparks scattered around. The same file produces exactly the same result every time — it won't come out different on the next run | Explosive/gunpowder aesthetics |

Thicken-mode fill also supports the "across the stroke" direction, and the short lines follow the local width — where a stroke tapers, the lines shorten with it, so the burn stays even. Closed paths have no start or end, so tapering automatically becomes uniform (brush becomes pure nib).

---

## 7. Job modes

Section 0 at the top picks this job's mode; the screen only shows settings that mode actually uses:

| Mode | Behavior |
|---|---|
| **Laser engraving** | All layers are laser; powder settings are hidden; material presets available |
| **Powder (vibration motor)** | All layers are powder; material presets hidden (those are a laser concept); motor offset hidden (with a single tool, "start from current position" is all you need); purge available |
| **Dual (powder, then laser)** | Each layer picks its own device; everything visible; the G-code comes out in two phases with an M0 pause between them for the tool swap |

The mode is remembered and restored next time.

## 8. Devices & dual jobs (robot arm: powder + laser)

Each layer has a **Device** choice at the top: **Laser** or **Vibration motor (powder)**. Both tools share one control line, so the G-code follows a "finish one tool, pause, swap, run the other" flow:

```
G92 X0 Y0            ← optional: start from current position
(purge)              ← optional
; PHASE 1 / POWDER   ← all powder layers (M3 constant, coordinates include motor offset)
M0                   ← pause: mount the laser head, press resume in LaserGRBL
; PHASE 2 / LASER    ← all laser layers (no offset)
```

Powder layers always come before laser, regardless of list order. With only one device in use, no M0 appears.

---

## 9. Powder layers in detail

### 9.1 How it works

The laser is like a "fine pen coloring in" (sweeping back and forth to fill an area); powder is a "thick pen, one stroke" — the line's width is controlled directly by the amplitude. In **centerline** mode a motor layer runs once along the middle of each line (open lines are followed as-is; closed shapes get their middle line found automatically), and the thickness rhythm is encoded in the S value (=amplitude) changing segment by segment along the way. Powder uses **M3 constant mode**, not M4: M4 would drop the amplitude during speed-ups and slow-downs and make the powder flow unsteady.

**Powder path modes**: motor layers have four path modes — **Outline** (default: trace the drawn outline once, constant amplitude), **Centerline** (the middle-line behavior above, with thickness rhythm), **Fill** (parallel powder lines inside closed shapes, constant amplitude; a powder line is several mm wide, so spacing ≥2mm is recommended), and **Circles** (powder dots: the arm stops at points along the line and keeps vibrating for a few seconds so powder piles up into a dot; spacing and dwell time can each be fixed or random, and dot size depends on dwell time and amplitude — measure to find out). Outline/fill/circles modes use a single "Amplitude %" value.

### 9.2 Thickness rhythm → amplitude

Pick a stroke style (uniform/tapered/nib/brush) to set the design's thickness rhythm; it is then converted to amplitude automatically: the thinnest point in the design gets your minimum amplitude, the thickest gets the maximum, and everything between is scaled proportionally. So even if the design's width variation is subtle, it gets stretched to the biggest contrast the hardware can show.

**Uniform strokes**: with "uniform" the whole line is one width, so only a single **"Amplitude %"** box is shown and the G-code uses it directly; min/max/width-contrast/lead-distance only appear for varying-width styles (tapered/nib/brush).

- **Min/max amplitude %**: the amplitude range used when no component profile is selected;
- **Width contrast**: exaggerates or flattens the contrast. Higher → thin spots get thinner (sharper taper); lower → everything fuller. Default 1 (no exaggeration);
- **Lead distance (mm)**: powder takes a moment to fall from the nozzle to the canvas, so this sends amplitude changes **early** by that distance to cancel the delay. If stroke ends trail powder, increase it (start from 3mm).

### 9.3 Component profiles

Every "3D-printed dispenser part + powder" combination has its own personality (how coarse the flow is, how small a shake produces nothing at all, how big a shake maxes out). A **component profile** stores the measured "amplitude → line width" relationship for one combination; switch profiles in the layer and the conversion uses that set's real measurements. Choosing "no calibration data" skips the curve and just uses the amplitude range directly.

### 9.4 Calibration wizard

On a motor layer, click "Calibration wizard…":

1. **Generate test pattern G-code** → downloads `calib.nc` (8 straight lines stepping up from 20% to 90% amplitude, 30mm each);
2. Load powder and run it;
3. Measure each powder line's real width with calipers and type it into the table (**rows that produced no powder get 0** — the tool remembers "amplitudes this small produce nothing");
4. Save the profile.

> Example result: 30%→1.2mm, 50%→2.0mm, 70%→3.1mm, 80%→3.5mm, 20% gets 0 (no powder). From then on, this profile only ever uses amplitudes inside the working 30–80% range.

**Quick drift calibration** (use at the start of each day): powder level and humidity slowly change the flow. Draw one line at a reference amplitude (say 60%), measure its width, type it into "today's measurement", press the scale button — the whole curve is corrected proportionally. Ten seconds, no full re-run.

**Saving**: everything in the wizard (table values, creating/deleting profiles, drift calibration) is a draft until you press "Save profile" — only then does it take effect and enter G-code generation. Closing with unsaved changes asks for confirmation. Deleted a profile by mistake? Just close the window and it rolls back.

### 9.5 Purge

After the machine sits idle, the powder in the tube has "gone to sleep", and the first stroke comes out patchy. With purge on, before the real paths the arm automatically goes to a spot you choose (pick a waste area outside the picture) and shakes powder out for a set number of seconds to wake up the flow.

### 9.6 Measuring the motor offset

The nozzle and the laser spot sit in different physical places, so measure once: draw a small cross with the laser at low power → swap to the powder head and dispense a cross at the **same coordinates** → measure how far apart the two cross centers are → enter it as dX/dY. From then on the powder phase's coordinates shift automatically and powder lines land exactly on the laser's lines. No need to re-measure unless a tool head is removed and remounted.

---

## 10. Material presets

The "Material presets" dropdown applies mode/power/speed/passes for a given material/thickness/powder/effect in one click. The preset manager supports creating, saving from the current layer, deleting, JSON import/export, and restoring the built-in library. Data is remembered by the browser on this computer.

**Powder field**: presets have a free-text "powder" field (suggested format: maker·color·grain·batch, e.g. "ACME·black·fine·2026-07 batch"). Ignition settings depend on the **powder × canvas** combination and can't be worked out separately — one preset records one tested combination. As always: preset numbers are only estimates until you've measured them yourself.

**Draft & save**: every change in the manager (editing, creating, deleting, importing, loading built-ins) is a **draft** until you press "Save presets" at the bottom. Closing with unsaved changes asks for confirmation — made a mistake? Just close the window and it rolls back. "Export JSON" exports the current draft.

---

## 10.5 The Test tab

Two tabs sit at the top of the left panel: **"Create"** (the full SVG workflow) and **"Test"**. Switching to "Test" turns the whole left panel into a test workbench that **generates G-code the moment you enter**, then updates live with every change. Use it for material/powder calibration:

**T1 · Layout & steps**: pick rows × columns by sweeping over a grid of cells, just like inserting a table in Google Docs (or type numbers directly, up to 20×20); set line length / row gap / column gap; pick the device (laser/motor). S and F (amplitude% for motor) each have a **row step and a column step**: each cell's value = start + row × row-step + column × column-step; set a direction's step to 0 if you don't want it to change. "Fill S / Fill F / Fill amplitude" are three separate buttons — each recalculates only its own variable and never touches values you've edited by hand. Laser S is entered as a raw value (capped by your $30 setting), M3/M4 selectable; motor lines take amplitude% each, with one fixed F (M3 automatic).

**T2 · Line list**: every line gets its own color, and the swatches in the list match the preview exactly. The checkbox = **whether to export that line** (unchecked lines stay out of the G-code); **clicking the card itself = select** (the card highlights), Shift-click range-selects, and clicking a checkbox or number box does not select; each line's S/F (or amplitude%) is editable, and edits regenerate instantly. **Hold Shift and drag on the preview = box select**: every line the rectangle touches gets selected (selected lines draw thicker in the preview); drag a flat strip to grab a row, a tall strip to grab a column. Plain dragging still pans.

**T3 · Groups**: select some lines → press "Group selected lines". Groups can be renamed; a group's checkbox toggles export for the whole group at once; type S/F/amplitude once on the group and it applies to every line in it; grouped lines share the group's color in the preview; "Ungroup" reverts. G-code comments carry the group name (`; r2c3 [Group 1]: S500 F900`).

Also: the stats bar at the top (travel/work length/time estimate/line count) works here too (the grid runs in cell order, no travel optimization); row 1 is at the bottom (Y=0); every cell gets a comment plus a copyable parameter table; tab choice, layout, per-line values, export checkboxes and groups are all remembered automatically; "Download testgrid.nc" sits at the bottom of the test panel.

**About dialogs**: every dialog in the tool (project naming, confirmations, notices) is drawn by the page itself, not by the browser, so the browser's "block dialogs" setting can never swallow them.

## 11. Preview & stats

- Scroll to zoom, drag to pan, double-click to reset;
- **Green dashed lines** are travel moves (fast pen-up moves); hidden automatically on very heavy drawings (over 8000 paths) to stay smooth;
- Motor layers are drawn as **variable-width lines**, so you see the powder thickness rhythm directly;
- Hovering over a layer card highlights that layer in the preview;
- The stats bar shows: travel before/after optimization, savings, work length, estimated time, G-code line count.

**Performance**: all heavy computation happens in the background — dragging sliders never freezes the screen; rapid changes only compute the latest one. Changing only power/speed/passes (nothing that alters the toolpath shape) gives near-instant results. Very long G-code is shown truncated in the text box, but **the downloaded file is always complete**.

---

## 12. G-code structure reference

```gcode
; Generated by LaserGRBL G-code Generator (Fono) v1.8.0
G21            ; millimeters
G90            ; absolute coordinates
G92 X0 Y0      ; (optional) start from current position
M8             ; (optional) air pump
; --- framing ---            (optional) zero-power outline pass
; ============ PHASE 1 / POWDER ============
G0 X-10 Y-10 S0
M3 S600                      ; purge
G4 P2.0
S0
M5
; ===== layer name · POWDER =====
M3
G0 X1.500 Y-2.000 S0         ; coordinates include motor offset
G1 X2.480 Y-1.900 S312 F1500 ; S changes segment by segment = thickness rhythm
...
M5
M0 ; PAUSE: swap to LASER    ; swap tool, press resume
; ============ PHASE 2 / LASER ============
M4                           ; engraving mode (M3 for cutting)
G0 X0.000 Y10.000 S0
G1 X20.000 Y10.000 S300 F1800
...
M5
G0 X0.000 Y0.000 ; home
```

---

## 13. FAQ

**My SVG imports as one single black layer?** An old-version problem, long fixed. If it still happens, check whether the SVG really only uses one stroke color.

**Thick material won't burn through in places?** See §5: switch fill line direction to "across the stroke", make sure the machine has `$32=1` and engraving uses M4. If it still won't cut, add passes rather than slowing down forever.

**"Across the stroke" fill comes out empty?** The shape is too "fat" to look like a stroke; the tool already fell back to fixed angle (this is normal).

**A blob of powder at the end of strokes?** Increase the lead distance (§9.2).

**The first powder line is patchy?** Turn on purge (§9.5).

**Powder and laser don't land in the same place?** Measure the motor offset once (§9.6).

**Changed powder or printed parts and everything behaves differently?** Make one component profile per combination (§9.3); handle day-to-day drift with quick calibration (§9.4).

**Does the gunpowder burst style spark differently every run?** No — the same file generates exactly the same G-code every time.

---

*Manual policy: every release updates the relevant sections of this manual and the version number at the top.*
