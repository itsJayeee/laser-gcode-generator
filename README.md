# laser-gcode-generator
# Laser G-code Generator

A **single-file, zero-dependency, fully offline** laser cutting/engraving G-code generator. Drop an SVG into your browser, tune the parameters, download the `.nc` file — nothing to install, and nothing ever leaves your machine.

## Quick start

1. Download `gcode_generator.html` from this repo
2. Open it in any modern browser (Chrome / Edge / Firefox)
3. That's it

Or use it online once GitHub Pages is enabled:

```
https://<your-username>.github.io/<repo-name>/gcode_generator.html
```

## Features

**① Import** — Click or drag in an SVG file; paths are automatically split into layers by color/stroke

**② Size & machine** — Output width (proportional scaling), origin position, rapid-travel G0 speed, and S power ceiling (`$30`)

**③ Per-layer parameters** — Each layer gets independent settings:
- Mode: stroke or fill (with adjustable line spacing and fill angle)
- Power %, feed rate F, number of passes (for cutting through)
- Layer processing order

**④ Material presets** — Load the built-in 40W parameter library with one click; create presets from the current layer and apply them to selected layers; import/export as JSON. Presets persist in browser localStorage between sessions.

**Preview & output** — Toggle between path preview and raw G-code views; path optimization runs automatically at generation time to reduce travel moves; download the result as a `.nc` file with one click.

**Bilingual UI** — Switch between English and Chinese anytime.

## Output format

Generates standard GRBL-style G-code: `G0` rapid moves, `G1` cutting moves with `S` power and `F` feed, and `M3`/`M5` laser on/off. Works with common desktop laser engravers running GRBL-class controllers. Before running, make sure your machine's power ceiling setting (`$30`) matches the S max value configured in the app.

## Privacy

All parsing, computation, and generation happen locally in your browser. Your SVG files are never uploaded anywhere.

## ⚠️ Safety

Lasers pose fire and burn hazards. Test new materials and parameters at low power first, never leave the machine unattended while running, and ensure proper ventilation. Always review generated G-code before running it.

## License

MIT
