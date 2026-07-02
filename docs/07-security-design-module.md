# 07 — Security Design Module

The differentiator. Turns camera placement from drawing icons into **defensible optical engineering**: FoV, DORI, occlusion, coverage, lighting, and optimization — computed deterministically in `packages/security-calc` (+ `packages/geometry`), rendered live on the CAD canvas, and exported as engineering reports.

## 1. Camera model

A placed camera = `device` node ([doc 06 §3](06-cad-engine.md)) whose `domain.params` + bound product specs define:

```ts
type CameraParams = {
  position: {x: number; y: number};   // plan, mm
  heightM: number;                    // mount height
  panDeg: number;                     // azimuth
  tiltDeg: number;                    // downward tilt
  lens: {focalMm: number | {min: number; max: number}; set?: number};  // fixed | varifocal
  sensor: {widthMm: number; heightMm: number; resH: number; resV: number}; // from product specs
  ir?: {rangeM: number};              // from product specs
  minSceneIlluminationLux?: number;   // day/night sensitivity, from specs
  privacyMasks?: Polygon[];
};
```

Product binding auto-fills sensor/lens/IR from catalog specs ([doc 08](08-catalog-and-bom.md)); unbound "generic camera" uses editable assumptions clearly flagged in reports.

## 2. Optics math (deterministic core)

**Angles of view** from sensor + focal length:

```
hFov = 2·atan(sensorWidth / (2·focal))      vFov = 2·atan(sensorHeight / (2·focal))
```

**Ground-plane FoV footprint (2.5D):** the view frustum (apex at camera height `h`, axis at tilt `t`) intersected with a target plane. We compute at **target height** (default 1.0–1.6 m configurable — detecting *people*, not floor texture): near/far edges from `h`, `t`, `vFov`; lateral spread from `hFov`; yielding the classic trapezoid, clipped by:

1. **Max useful distance** for the active requirement (from pixel density, below) — not just geometric horizon.
2. **Occlusion** — visibility polygon from the camera position against obstacle segments (walls from the drawing, `zone` obstacles with `height_m` ≥ sightline height at that distance; the 2.5D check compares obstacle height against the ray's height at the obstacle — cheap and surprisingly accurate).
3. **Privacy masks** — subtracted and reported (GDPR zones).

**Pixel density** at distance `d` (the number consultants sell):

```
ppm(d) = resH / widthAtDistance(d)   where  widthAtDistance(d) = 2·d·tan(hFov/2)
```

**DORI (IEC 62676-4)** thresholds, the standards engine's default preset:

| Level | Requirement | Typical use |
|---|---|---|
| Detect | ≥ 25 px/m | Something is present |
| Observe | ≥ 63 px/m | Characteristics (clothing color) |
| Recognize | ≥ 125 px/m | Known person recognizable |
| Identify | ≥ 250 px/m | Unknown person identifiable (evidence) |
| LPR preset | ≥ ~600–800 px/m on plate | License plates (regional presets) |

Each camera's footprint is rendered as **nested DORI bands** (identify → detect, color-coded). Requirement `zones` ([doc 03 §5](03-database-schema.md)) declare a needed level; compliance = zone polygon ⊆ union of camera regions meeting that level.

**Distance-for-quality inverse** (lens selection): given a target at distance `d` and required ppm, solve required focal length; the lens picker suggests fixed lenses / varifocal settings from the bound product line and flags "no lens on this camera achieves Identify at 40 m."

**Night vision / IR:** night footprint = day footprint ∩ disc(IR range) for IR-equipped cameras; for non-IR, min-illumination spec vs. ambient lighting zones (lighting analysis v1 = user-declared lux zones on the drawing; photometric simulation is a later phase). Day and night coverage are **separate computed layers** toggled in the UI.

## 3. Coverage aggregation & analysis

Computed per drawing (client-side incremental for the edited camera; worker for full recompute — [doc 05 §4](05-api-architecture.md)):

- **Coverage union** per DORI level (polygon boolean union).
- **Blind-spot detection:** requirement zones minus coverage → gap polygons, ranked by area × zone criticality; each gap becomes an actionable finding ("Gap G3: 14 m² in Zone 'Loading Dock' below Observe").
- **Redundancy/overlap map:** k-coverage (areas seen by ≥2 cameras) — required for high-security zones and camera-failure resilience analysis.
- **Heatmap:** raster grid (0.25–1 m cells) of max ppm per cell → texture overlay ([doc 06 §2](06-cad-engine.md)); doubles as the export image.
- **Dynamic target simulation:** a path polyline + target speed animates a moving subject; output = timeline of which cameras see it at which DORI level, and handoff gaps ("target unobserved for 4.2 s between CAM-07 and CAM-12"). Deterministic sampling along the path — no physics engine needed.
- **Multi-camera comparison:** side-by-side metrics table for candidate products on the same position (coverage area per DORI level, night coverage, cost) — feeds BOM decisions.
- **Camera preview:** synthetic perspective view from the camera (walls extruded, target dummy at distance markers) — the sales-demo feature; WebGL view reusing the same math.

## 4. Risk analysis & standards engine

- **Risk zones** with likelihood/impact grading roll up into a coverage-vs-risk matrix: highest-risk uncovered areas first.
- **Standards presets** (pluggable rule packs): IEC 62676-4 DORI defaults; regional LPR presets; org-custom rule packs ("casino floor: Identify + 2× redundancy"). Each computed result cites the rule pack + parameters in the report — *defensibility is the product*.
- **Compliance checklist** auto-derived per project: every requirement zone × rule → pass/fail/margin.

## 5. Optimization & AI assistance

Two distinct engines, honestly separated:

1. **Deterministic solver** (`security-calc`): greedy + local-search placement over candidate positions (wall-mounted along selected walls, pole positions) maximizing weighted zone coverage per cost. Transparent objective, reproducible. Suggests: reposition/re-tilt/re-lens existing cameras (gradient-free local search over pan/tilt/focal), and "add camera here" candidates for gaps.
2. **AI layer** ([doc 12](12-ai-architecture.md)): explains solver output in plain language, translates natural-language requirements into zones/rule packs ("make the parking lot LPR-compliant at both gates"), recommends products given constraints, drafts the narrative sections of reports. **The AI never invents coverage numbers** — it calls the solver/calculator tools and cites their output.

## 6. Reports & exports

Worker-generated ([doc 05 §7](05-api-architecture.md)), branded, versioned artifacts:

- **Coverage report** (PDF/DOCX): per-floor heatmaps, DORI band maps, camera schedule table (position, height, product, lens, expected ppm at zone boundaries), blind-spot findings, compliance checklist, assumptions appendix (target height, rule pack, lighting).
- **Camera schedule** (XLSX/CSV) — feeds straight into BOM/commissioning.
- **Risk analysis report**; **client-facing summary** (portal-visible, cost-free version).

## 7. Future device classes (architecture note)

Radar, LiDAR, PIR, access control, fences, drones all reduce to the same pattern: *emitter/sensor with a parametric coverage geometry + product specs + rule packs*. `security-calc` is structured as `CoverageProvider` plugins (camera is the first provider), so these are content + math additions, not architecture changes.

## 8. Validation strategy

- Golden tests: fixture cameras with hand-computed FoV/ppm values; cross-checked against JVSG outputs and manufacturer lens calculators (Axis/Hanwha) for the same inputs — published tolerance ≤1%.
- External review: 3 practicing security consultants validate M2 reports before the module is marketed as standards-compliant ([doc 01 §7](01-product-strategy.md)).
- Property tests: coverage monotonicity (more cameras ⇒ never less coverage), occlusion soundness (covered point must have unobstructed sightline).
