# LaserGRBL G-code Generator · User Manual

Applies to: **v1.20.0** | [中文版](MANUAL.zh-CN.md) | This manual is updated with every release; see the [CHANGELOG](../CHANGELOG.md) for revision history.

The tool is a single HTML file that runs directly in a web browser (Chrome or Edge recommended); no installation is required. It is designed for a robot-arm painting workflow in which a vibration motor dispenses gunpowder and a laser ignites it. It can also be used as a general-purpose G-code generator for laser engraving and cutting.

## Glossary

The following terms appear throughout this manual. Refer back to this table as needed:

| Term | Meaning |
|---|---|
| **G-code** | The instruction file executed by the machine. Each line is one command: move to a position at a given output level |
| **.nc file** | The file format for G-code. Download it and run it in LaserGRBL |
| **S value** | The intensity number in a command. For the laser it represents power; for the motor, vibration strength |
| **$30** | A machine setting that defines the full-scale S value. The LaserGRBL default is 1000 |
| **Power% vs. S** | power% = S ÷ $30 × 100. With $30 = 1000, S450 is 45% while S45 is only 4.5%. Check the number of digits when entering values |
| **F value** | Travel speed in millimeters per minute. F600 equals 60 cm per minute |
| **Amplitude** | The vibration strength of the powder motor, also expressed as an S value. Stronger vibration produces a wider powder line |
| **M3 / M4** | Two commands for switching the laser or motor on. M3 holds a constant intensity; M4 adjusts intensity automatically during acceleration and deceleration. Powder dispensing always uses M3; laser engraving normally uses M4 |
| **M0** | The pause command. The machine stops and waits for operator action. Dual-process jobs use it for the tool-head change |
| **purge** | Dispensing powder at a designated position for a few seconds before the actual job, to restore powder flow in the tube |

---

## 1. Quick start

1. Open `gcode_generator.html` in a browser;
2. Select the **job mode** at the top of the page (Laser / Motor / Dual);
3. Drag an SVG file into the import area;
4. Set the parameters for each color layer under "Layer parameters";
5. Wait for the preview to refresh (complex drawings are computed in the background and do not block the interface);
6. Click "Download" to obtain the `.nc` file and run it in LaserGRBL.

> Example: draw a circle with a black outline in Illustrator or Inkscape and export it as SVG. After importing, one black layer appears. Select the "Outline" path mode, set power to 30% and speed to 1800, then download. The resulting G-code traces the circle once.

---

## 2. Importing files (SVG)

Regardless of whether the file was drawn in Illustrator, Figma, or Inkscape, lines of different colors are automatically separated into different layers on import; no special setup is required. The drawing yields one layer per color, and each layer's parameters are set independently.

> Example: to control "strokes that receive powder first" and "laser-only details" separately, draw them in two different colors (for example red and black) in the design application. After import they become two independent layers.

---

## 2.5 Parameter memory & project files

**Automatic memory**: all global settings and every color layer's parameters (power, speed, mode, stroke style, and so on) are stored automatically in the browser on this computer. Settings are restored after a page refresh or on the next day; when an SVG using the same colors is re-imported, each color receives its previously used parameters. An assignment such as "red = construction lines, S550 F600" needs to be set only once.

**Project files**: "Save project" in the import area packs the SVG artwork and all parameters (including powder profiles) into a single downloadable `.json` file; "Load project" restores everything in one step. It is recommended to keep one project file per artwork or parameter recipe, for long-term archiving or transfer to another computer.

> Note: clearing browser data also clears the automatic memory. Save important parameters as project files.

**Project naming**: after clicking "Save project", a dialog asks for a project name (the SVG file name is suggested by default) and the file downloads as `<name>.json`. The name is stored in the project and suggested again on the next save.

**G-code file name**: the downloaded `.nc` file automatically uses the imported SVG's name; `dragon.svg` produces `dragon.nc`. This also applies after loading a project. Calibration patterns always download as `calib.nc`.

---

## 3. Global settings

| Setting | Description |
|---|---|
| Output width (mm) | Actual width of the finished piece. Height is calculated automatically from the drawing's proportions |
| Origin | Bottom-left (all coordinates positive) or centered (origin at the center of the drawing) |
| S max (= the machine's $30) | Must match the `$30` setting inside the machine; otherwise power percentages are incorrect. The LaserGRBL default is 1000. Power and amplitude fields display their corresponding S value in real time |
| Rapid G0 | Speed of pen-up rapid moves, used for time estimation |
| Air pump/fan (M8/M9) | When checked, switches on at the start of the job and off at the end |
| Frame before job | Before the actual job, the machine traces the drawing's bounding box at zero power to verify position and size. With "Place on actual canvas" enabled, the **canvas border** is traced instead; see Section 3.5 |
| Place on actual canvas | Enter the actual canvas size (inch or mm) and position the artwork inside it; see Section 3.5 |
| Start from current position (G92) | Sets the arm's current parked position as the job's starting point. Recommended for large-format work without a fixed start point |
| Motor offset dX/dY | The positional difference between the powder nozzle and the laser spot. Measurement procedure in Section 9.6 |
| Purge before powder | Moves to a designated position and dispenses powder for a few seconds before the actual job; see Section 9.5 |

---

## 3.5 Place on actual canvas (canvas alignment)

Addresses artwork landing outside the canvas or failing to line up with it. When enabled:

1. **Enter the canvas size**: width × height, in inch or mm (for a 10×10 inch canvas, enter 10 and 10);
2. **Choose the artwork position**: "Centered" places the artwork in the middle of the canvas. The artwork can also be **dragged directly in the preview** to any position on the canvas (the change takes effect on release; the position switches to "Custom" and the number fields update automatically). The "Custom" number fields (distances from the canvas's left and bottom edges) are available for precise adjustment;
3. The preview displays a **brown canvas frame** with a marker at the zero corner, showing the artwork's position within the canvas; a **red warning** appears if the artwork extends beyond the canvas;
4. **The coordinate zero is the canvas's bottom-left corner.** Before starting, point the nozzle tip (or laser dot) at that corner and zero the position;
5. With "Frame before job" enabled, the machine first traces the canvas border at zero power and then pauses (M0). Check whether the traced frame coincides with the physical canvas edges: if it does, press resume in LaserGRBL to start the job; if not, press stop, reposition the canvas or re-zero, and run again.

> Tip: if the traced frame comes out tilted, the canvas itself is tilted; move the canvas to coincide with the frame (the frame indicates the actual processing position). If the arm has been bumped, re-point the nozzle at the canvas corner, re-zero, and trace the frame again to confirm.

**Default speeds**: new layers default to F=1000 (laser) / F=3000 (motor); switching device or job mode restores the corresponding default. Previously used colors are restored with their last-used values first.

**Processing order**: the layer list, top to bottom, is the processing order (badges #1, #2, …). Click "Edit order" to enter drag mode (cards fold up and can be dragged), then "Done" to lock. In dual jobs the powder layers always run first; list order determines only the sequence within each phase.

**Select all & range select**: the layer enable column and the test page's export and select columns each provide a "select all" checkbox. Any column of checkboxes supports **Shift range-select**: click one box, hold Shift, and click another; every box in between changes accordingly.

**Other**: after a value is changed, the cursor returns to the same input field (press Tab to move through fields in sequence). There is no "Generate" button; generation is always automatic. The download button is located at the very bottom of the left panel (scroll down to reach it), with a status line beneath it showing "computing Ns…" or "ready".

## 4. Path modes (laser layers)

Each layer selects its own mode:

- **Outline**: traces the drawn outline once. Suitable for cutting and line art.
- **Centerline**: for shapes with stroke width, runs once along the middle. Suitable for converting outlined fonts to single-line engraving.
- **Fill**: sweeps the interior of closed shapes with parallel lines. Settings in Section 5.
- **Thicken**: widens a single line into a stroke with real width, then fills it. Suitable for single-line fonts and handwritten paths. Settings in Section 6.
- **Circles**: draws a circle at intervals along the line; the line itself is not drawn. Circle **size** and **spacing** can each be fixed or random (random results are identical on every generation of the same file). Check "Fill circles" for solid circles, with a **fill density** setting (mm; smaller values produce denser fill).

---

## 5. Fill settings

- **Spacing (mm)**: gap between neighboring sweep lines. Surface engraving typically uses 0.1–0.2; for cutting through thick material, use 0.05 with multiple passes.
- **Sweep direction**:
  - **Fixed angle**: all sweep lines run in one direction; the conventional method.
  - **Perpendicular to stroke**: each sweep line crosses the stroke at a right angle, with length approximately equal to the stroke width. This addresses uneven burn depth on thick material: on long lines the machine cruises at full speed and burns shallow, while short lines burn deeper due to acceleration, so strokes of different orientation receive unequal energy. With perpendicular sweeps, all lines are equal in length and burn depth is uniform. Corners and intersections are handled automatically, without gaps or overlaps.
- **Automatic fallback**: shapes too broad to have stroke form (such as a solid disc) have no extractable centerline; fill automatically reverts to fixed angle in that case.
- **Fill + outline**: when checked, the outline is traced once after filling for a cleaner edge.

> Example (lettering on thick card stock with uneven burn): spacing 0.05, sweep direction "Perpendicular to stroke", S30 F1800, engrave mode (M4), and confirm `$32=1` in the machine.

---

## 6. Thicken mode & stroke styles

The **stroke width** slider sets the base width (mm). The **stroke style** determines how width varies along the stroke:

| Style | Effect | Use |
|---|---|---|
| Uniform | Constant width throughout | Standard thick lines |
| Tapered ends | Width tapers smoothly at the start and end of the stroke, similar to pen entry and exit strokes | Handwritten effect |
| Calligraphic nib (slanted) | Width varies with stroke direction (thin horizontals, thick verticals). The **nib angle** is adjustable: 30° is the customary angle for Western italics; 0° approaches Song-typeface strokes | Calligraphy |
| Brush | Taper and nib effects combined | Signatures, inscriptions |
| Gunpowder burst | Random width variation, scorched spurs along the edges, and scattered sparks. The same file generates identical results every time | Explosive/gunpowder aesthetics |

Fill in thicken mode also supports the "Perpendicular to stroke" direction, and sweep length follows the local width: at tapers, sweep lines shorten automatically and burn depth remains uniform. Closed paths have no stroke start or end, so the taper style degrades to uniform (brush degrades to nib only).

---

## 7. Job modes

Section 0 at the top of the page selects the job mode; the interface shows only the settings relevant to that mode:

| Mode | Behavior |
|---|---|
| **Laser engraving** | All layers are laser; powder-related settings are hidden; material presets available |
| **Vibration motor (powder)** | All layers are powder; material presets hidden (a laser-material concept); motor offset hidden (with a single tool, "Start from current position" suffices); purge available |
| **Dual process (powder, then laser)** | Each layer selects its device individually; all settings visible; the G-code is generated in two phases with an M0 pause for the tool change |

The mode selection is remembered and restored on the next launch.

## 8. Devices & dual process (powder + laser)

Each layer has a **device** option at the top: **Laser** or **Vibration motor (powder)**. The two tools share one control signal, so G-code is generated in the sequence "complete one process, stop for the tool change, run the other":

```
G92 X0 Y0            ← optional: start from current position
(purge)              ← optional
; PHASE 1 / POWDER   ← all powder layers (M3 constant; coordinates include motor offset)
M0                   ← pause: mount the laser head, then press resume in LaserGRBL
; PHASE 2 / LASER    ← all laser layers (coordinates without offset)
```

Powder layers always precede laser layers, regardless of list order. No M0 is produced when only one device is used.

---

## 9. Powder layers in detail

### 9.1 How it works

The laser works like a fine pen filling areas by sweeping; powder works like a broad pen making one stroke, its width controlled directly by amplitude. In **Centerline** mode the motor layer runs once along the middle of each line (open lines as drawn; closed shapes have their centerline extracted automatically), with the thickness rhythm encoded in S values (amplitude) that vary segment by segment. Powder uses **M3 constant mode** rather than M4: M4 would lower the amplitude during acceleration and deceleration, destabilizing the powder flow.

**Powder path modes**: motor layers have four path modes — **Outline** (default: traces the contour once at constant amplitude), **Centerline** (runs along the middle with thickness rhythm), **Fill** (parallel powder lines inside closed shapes at constant amplitude; a powder line is several millimeters wide, so spacing of at least 2 mm is recommended), and **Circles** (powder dots: the arm stops at intervals along the line and vibrates in place for several seconds, letting powder accumulate into a dot; spacing and dwell time can each be fixed or random; dot size depends on dwell time and amplitude and must be determined by testing). Outline, fill, and circles modes use a single "Amplitude %" value.

### 9.2 Thickness rhythm to amplitude

After a stroke style is selected (uniform/taper/nib/brush) to define the design's thickness rhythm, the tool converts it to amplitude automatically: the thinnest point in the design maps to the configured minimum amplitude, the thickest to the maximum, with proportional values in between. Even a small width variation in the design is stretched to the maximum contrast the hardware can express.

**Uniform strokes**: with "Uniform" selected, the line has constant width and the interface shows a single **"Amplitude %"** field used directly in the G-code. Minimum/maximum amplitude, width contrast, and lead distance appear only for variable-width styles (taper/nib/brush).

- **Min/Max amplitude %**: the conversion range used when no component profile is selected;
- **Width contrast**: adjusts the exaggeration of the contrast. Higher values make thin sections thinner (sharper tapers); lower values produce a fuller stroke. Default 1 (no exaggeration);
- **Lead distance (mm)**: powder takes time to fall from the nozzle to the canvas. This parameter issues amplitude changes early by the given distance to compensate. Increase it if powder trails at stroke ends (3 mm is a reasonable starting value).

### 9.3 Component profiles

Each combination of 3D-printed dispenser and powder has different physical characteristics (line width per amplitude, the minimum amplitude that produces flow, the saturation limit). A **component profile** stores the measured amplitude-to-width relationship for one combination; selecting a profile in a layer applies that combination's measured conversion. Selecting "No calibration data" bypasses the curve and converts directly from the amplitude range.

### 9.4 Calibration wizard

Click "Calibration wizard…" in a motor layer:

1. **Generate the test pattern G-code** and download `calib.nc` (8 straight lines with amplitude stepping from 20% to 90%, each 30 mm long);
2. Load powder and run it;
3. Measure each powder line's actual width with calipers and enter the values in the table (**enter 0 for rows that produced no powder**; the tool records that no powder flows below that amplitude);
4. Save the profile.

> Example calibration result: 30%→1.2 mm, 50%→2.0 mm, 70%→3.1 mm, 80%→3.5 mm, 20% entered as 0 (no flow). Under this profile, amplitude values are subsequently confined to the effective 30–80% range.

**Quick drift calibration** (recommended at the start of each working day): powder level and humidity cause output to drift gradually. Draw one line at a reference amplitude (for example 60%), measure its width, enter it under "today's measurement", and click the scale button; the entire curve is corrected proportionally without a full recalibration.

**Save mechanism**: all changes inside the wizard (table values, profile creation/deletion, drift calibration) are drafts; they take effect and participate in G-code generation only after "Save profile" is clicked. Closing the window with unsaved changes prompts for confirmation. An accidentally deleted profile is recovered by closing the window without saving.

### 9.5 Purge

After the machine has been idle, powder in the tube loses flowability and the first stroke would come out incomplete. With purge enabled, the machine first moves to the specified coordinates (choose a waste area outside the artwork) and dispenses for the specified number of seconds to restore flow.

### 9.6 Motor offset measurement

The powder nozzle and the laser spot are at different physical positions; measure the difference once: draw a small cross at low laser power → switch to the powder head and dispense a cross at the **same coordinates** → measure the offset between the two cross centers → enter it as dX/dY. Powder-phase coordinates are then translated automatically so powder lines and laser passes coincide on the canvas. Re-measurement is unnecessary unless the tool heads are removed and remounted.

---

## 10. Material presets

The "Material presets" menu applies the mode/power/speed/passes recorded for a given material/thickness/powder/effect in one step. The preset manager supports creating, saving from the current layer, deleting, JSON import/export, and restoring the built-in library. Data is stored in the browser on this computer.

**Powder field**: presets include a free-form "powder" field (suggested format: manufacturer · color · grain · batch, e.g. "Factory X · black · fine · 2026-07 batch"). Ignition parameters depend on the **combination of powder and canvas** and cannot be derived separately; one preset records one tested combination.

> Note: preset values are reference values until verified by actual testing.

**Drafts and saving**: all changes in the manager (editing, creating, deleting, importing, loading built-ins) are drafts and take effect only after "Save presets" at the bottom is clicked. Closing with unsaved changes prompts for confirmation; accidental edits or deletions are reverted by closing the window without saving. "Export JSON" exports the current draft.

---

## 10.5 Test tab

Two tabs sit at the top of the left panel: **"Create"** (the full SVG workflow) and **"Test"**. Switching to "Test" replaces the left panel with a test workbench. G-code is generated immediately on entry and updates in real time with every change. It is used for material and powder parameter calibration:

**T1 · Layout & steps**: select rows × columns by sliding over a cell grid (similar to inserting a table in a word processor; numbers can also be typed directly, up to 20×20); set line length, row gap, and column gap; select the device (laser/motor). S and F (amplitude % for motor) each have a **row step and a column step**: each cell's value = start + row index × row step + column index × column step; enter 0 for a direction that should not vary. "Fill S / Fill F / Fill amplitude" are three independent buttons; each recalculates only its own variable and does not affect values edited by hand. Laser S is entered as a raw value (bounded by the configured $30) with M3/M4 selectable; motor lines take amplitude % with a single fixed F (M3 is applied automatically).

**T2 · Line list**: each line has its own color, matched between the preview and the list swatches. The checkbox controls **whether the line is exported** (unchecked lines are excluded from the G-code). **Clicking the card selects the line** (the card highlights); Shift-click selects a range; clicking the checkbox or a number field does not change the selection. Each line's S/F (or amplitude %) can be edited individually, with regeneration on every change. **Holding Shift and dragging on the preview canvas draws a selection box**: every line the rectangle touches becomes selected (selected lines are drawn thicker in the preview); a horizontal band selects a row, a vertical band selects a column. Plain dragging still pans the view.

**T3 · Groups**: select several lines, then click "Group selected lines". Groups can be renamed; the group's checkbox toggles export for the whole group; entering S/F/amplitude once in the group applies it to all members; grouped lines are drawn in the group's color in the preview; "Ungroup" restores them. G-code comments include the group name (`; r2c3 [Group 1]: S500 F900`).

Additional notes: the statistics bar (travel/processing length/estimated time/line count) applies here as well (the grid runs in cell order without travel optimization); row 1 is at the bottom (Y=0); each cell is annotated and a copyable parameter table is provided; tab selection, layout, per-line parameters, export checkboxes, and grouping are all remembered automatically; "Download testgrid.nc" is at the bottom of the test panel.

**About dialogs**: all dialogs in the tool (project naming, confirmations, notices) are built into the page and do not use native browser pop-ups, so they are unaffected by the browser's pop-up blocking settings.

## 11. Preview & statistics

- Scroll to zoom, drag to pan, double-click to reset;
- **Green dashed lines** show travel (pen-up rapid) paths; they are hidden automatically above 8000 paths to keep the display responsive;
- Motor layers are drawn with **variable-width lines**, making the powder thickness rhythm directly visible;
- Hovering over a layer card highlights that layer in the preview;
- The statistics bar shows: travel before/after optimization, savings percentage, processing length, estimated time, and G-code line count.

**Performance**: all heavy computation runs in the background and does not block the interface; during rapid consecutive edits, only the latest change is computed. Changes that do not alter path geometry (power, speed, passes) produce near-instant results. Very long G-code is truncated in the text box for display only; **the downloaded file is always complete**.

---

## 12. G-code structure reference

```gcode
; Generated by LaserGRBL G-code Generator (Fono) v1.8.0
G21            ; millimeters
G90            ; absolute coordinates
G92 X0 Y0      ; (optional) start from current position
M8             ; (optional) air assist
; --- framing ---            (optional) zero-power framing pass
; ============ PHASE 1 / POWDER ============
G0 X-10 Y-10 S0
M3 S600                      ; purge
G4 P2.0
S0
M5
; ===== layer name · POWDER =====
M3
G0 X1.500 Y-2.000 S0         ; coordinates include motor offset
G1 X2.480 Y-1.900 S312 F1500 ; varying S = powder thickness rhythm
...
M5
M0 ; PAUSE: swap to LASER    ; change tools, then resume
; ============ PHASE 2 / LASER ============
M4                           ; dynamic mode for engraving (M3 for cutting)
G0 X0.000 Y10.000 S0
G1 X20.000 Y10.000 S300 F1800
...
M5
G0 X0.000 Y0.000 ; home
```

---

## 13. Frequently asked questions

**Only one black layer after importing an SVG?** An issue in old versions, since fixed. If it still occurs, check whether the SVG actually uses only one stroke color.

**Some areas of thick material do not cut through?** See Section 5: set the fill sweep direction to "Perpendicular to stroke", and confirm `$32=1` and M4 for engraving. If it still does not cut through, add passes rather than reducing speed further.

**"Perpendicular to stroke" fill produces no output?** The shape is too broad to have stroke form; the tool has automatically reverted to fixed-angle fill. This is expected behavior.

**Powder accumulates at stroke ends?** Increase the lead distance; see Section 9.2.

**The first powder line is incomplete?** Enable purge; see Section 9.5.

**Powder and laser do not coincide?** Measure the motor offset; see Section 9.6.

**Results change after switching powder or dispenser parts?** Create a component profile for each combination (Section 9.3); use quick drift calibration for day-to-day drift (Section 9.4).

**Does the gunpowder burst style generate different sparks each time?** No. The same file generates identical G-code every time.

---

*Manual maintenance rule: with every release, update the affected sections of this manual and the version number at the top.*
