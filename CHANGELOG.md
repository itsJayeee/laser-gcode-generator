# Changelog

This project follows Semantic Versioning. A full snapshot of each release is archived under `versions/`.

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
