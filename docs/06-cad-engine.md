# 06 — CAD Engine

The browser CAD editor: rendering, document model, geometry, tools, and file interop. Lives in `packages/cad-engine` (framework-agnostic TS) + `packages/geometry` (pure math), consumed by React editor routes in `apps/web`.

## 1. Requirements recap & posture

2D drafting with layers, blocks, snapping, grids, dimensions, annotations, object properties, templates; PDF/image/SVG underlays; DXF import first, DWG later; engineered for the security-design workload (hundreds–thousands of devices + coverage overlays on a floorplan), not for 100k-entity mechanical drawings on day one — but with the architecture headroom for it. Future 3D is a planned extension, not a rewrite (§8).

## 2. Rendering architecture

**Decision: custom WebGL2 renderer over a thin abstraction, Canvas2D fallback.**

- Rejected **SVG/DOM** (dies at ~5k nodes), **Konva/Fabric** (retained-mode object models fight CRDT-driven state and coverage overlays), **full engines (Pixi/Three)** as core (we'd use 10% and fight the rest; a thin regl-style wrapper or hand-rolled WebGL2 keeps the pipeline honest). WebGPU is a future backend behind the same interface.

Pipeline per frame (target 60 fps pan/zoom, 30 fps during heavy edits):

```
scene graph (CRDT mirror) → dirty-flag change tracking → spatial index query (viewport)
  → tessellation cache (lines→instanced segments, arcs/text→cached meshes/atlases)
  → draw passes: [underlay raster/vector] [grid] [entities by layer] [coverage overlays]
                 [selection/handles] [presence cursors] [HUD]
```

Key techniques:
- **Instanced rendering** for repeated geometry (device symbols = one mesh, N instances).
- **MSDF text atlas** for crisp labels/dimensions at any zoom.
- **Tile-cached underlays**: PDF pages rasterized server-side into a zoom pyramid (like map tiles); vector PDF content optionally extracted to lines for snapping (§7).
- **Coverage overlays** ([doc 07](07-security-design-module.md)) render as alpha-blended triangulated polygons + a heatmap texture pass — same pipeline, separate passes.
- **Level-of-detail**: text/dimension culling below pixel thresholds; symbol simplification when zoomed out.
- Everything positioned in **world coordinates (f64 in JS, f32 on GPU with camera-relative offsets)** to avoid precision artifacts on large sites.

## 3. Document model — the scene graph

The drawing document is a **Yjs CRDT doc** (collab rationale in [doc 10](10-collaboration-realtime.md)) whose semantic shape is:

```
Drawing
├─ meta: units, scale, grid, north angle
├─ layers:    Y.Map<LayerId, {name, color, lineweight, locked, hidden, order}>
├─ blockDefs: Y.Map<BlockId, {name, nodes: Node[], anchors, params}>   — symbol definitions
├─ sheets:    Y.Map<SheetId, {name, frame, viewports[]}>               — paper-space layouts
└─ nodes:     Y.Map<NodeId, Node>                                      — model space entities
```

```ts
type Node = {
  id: string;              // nanoid; referenced by design.device_instances.scene_node_id
  type: 'line'|'polyline'|'arc'|'circle'|'ellipse'|'spline'|'text'|'mtext'
       |'dimension'|'leader'|'hatch'|'image'|'blockRef'|'device'|'zone'|'cableRun'|'group';
  layer: LayerId;
  transform: [a,b,c,d,tx,ty];          // 2D affine
  geom: TypeSpecificGeometry;          // e.g. polyline: {pts: number[], closed, bulges?}
  style?: Partial<{stroke, fill, lineweight, linetype, textStyle}>;   // overrides layer
  props?: Record<string, unknown>;     // object properties panel
  domain?: {                           // semantic payload for smart entities
    deviceType?: string; productId?: string; params?: CameraParams|…;
  };
  locked?: boolean;
};
```

Design decisions:
- **Flat node map + group nodes**, not a deep tree — flat maps are CRDT-friendly (concurrent edits to different nodes never conflict) and index-friendly. Z-order via fractional `order` keys per layer.
- **Smart entities** (`device`, `zone`, `cableRun`) are first-class node types carrying domain payloads. The realtime service's **semantic extractor** projects them into `design.*` tables ([doc 03 §5](03-database-schema.md)) — this is the bridge from drawing to BOM/PM.
- **Blocks** = definition + `blockRef` instances with per-instance param overrides (device symbols ship as a curated block library; manufacturers can attach blocks to products).
- **Sheets/paper space** hold title blocks and viewports for print/PDF export at scale.
- A **local mirror** (typed arrays + spatial index) shadows the Y.Doc for the renderer; Yjs observers update the mirror incrementally — the renderer never reads Yjs directly.

## 4. Geometry kernel (`packages/geometry`)

Pure, dependency-light, golden-file-tested TS (WASM port per-function if profiling demands):

- Primitives: vec2, affine transforms, segments, arcs (bulge representation, DXF-compatible), ellipses, cubic/quadratic béziers, polylines/polygons.
- Predicates & ops: intersections (all primitive pairs), point-in-polygon, closest-point, offset (for containment/pathways), boolean ops on polygons (Martinez or similar algorithm — also used by coverage), polygon triangulation (earcut) for fills/overlays, convex hull, min bounding box.
- **Visibility polygons** (angular sweep over segment obstacles) — shared core of FoV/occlusion in [doc 07](07-security-design-module.md).
- **Spatial index:** packed R-tree (rebuilt incrementally) for viewport queries, snapping candidates, and hit-testing.
- Units: internal canonical unit = **millimeters**; display conversion (mm/m/cm/in/ft) at the UI edge only.

## 5. Interaction system

- **Tools as state machines** with a shared contract (`onPointerDown/Move/Up/onKey`, transient preview layer, commit → single CRDT transaction = one undo step): select (click/window/crossing), move/copy/rotate/scale/mirror, line/polyline/arc/circle/text, dimension (linear/aligned/angular/radial — associative: dimensions reference node ids and re-measure on change), measure, hatch/zone, device-place, cable-route, block insert.
- **Command bar** (AutoCAD muscle memory): `L`=line, `M`=move, `DI`=dimension…, with numeric coordinate/length/angle entry (`@500<45`).
- **Snapping engine:** endpoint, midpoint, center, intersection, perpendicular, tangent, nearest, grid; ortho + polar tracking. Pipeline: R-tree candidate query around cursor → per-mode snap evaluation → priority resolve → magnetic cursor + glyph. Budget <2 ms/frame at 10k entities.
- **Undo/redo:** Yjs `UndoManager` scoped to local user's transactions (collab-safe — undoing your change doesn't revert others').
- **Selection & properties panel:** multi-select property editing writes one transaction; property schema comes from node type + `domain` (device properties come from the product/category spec schema).

## 6. Layers, templates, reusable content

- Layer states (on/off/lock/print), layer groups, per-viewport overrides (sheet space).
- **Templates** = seed drawings (title blocks, layer standards, device libraries) stored as org-level template documents; new drawing = fork of template snapshot.
- **Org symbol libraries**: curated platform library (cameras, poles, cabinets, racks, sensors per common standards) + org-custom blocks + product-attached blocks from suppliers.

## 7. Import / export pipeline

All conversion runs in `apps/workers` ([doc 02 §8.2](02-system-architecture.md)); results cached by `content_hash`.

| Format | Import | Export | Implementation |
|---|---|---|---|
| **PDF** | Underlay: rasterize pages to tile pyramid (pdfium/mupdf worker); optional vector extraction (lines/text) into a locked "underlay" layer for snapping & scale calibration | Sheets → vector PDF (print pipeline renders scene → PDF primitives) | Worker, native libs in container |
| **PNG/JPEG** | Underlay w/ interactive scale calibration ("click two points, enter distance") | Viewport raster export | Worker + client |
| **SVG** | Parse → scene nodes (paths→polylines/béziers, text) | Scene → SVG (faithful, layer-grouped) | TS, worker |
| **DXF** | **M1.** Parse (own TS parser or vetted lib): entities → nodes, LAYER→layers, INSERT/BLOCK→blockDefs/refs, dimensions→associative where possible; unsupported entities preserved as round-trip blobs | **M3.** Scene → DXF (R2018 text) | TS, worker |
| **DWG** | **Phase 2 (M6+).** Isolated conversion container: ODA Drawings SDK (license) *or* Autodesk APS Design Automation (per-call, cloud) → DXF/scene JSON. Decision deferred to a build-vs-buy spike with real customer files; abstraction: `ConversionProvider` interface | Same path reversed | Native worker or external API |
| **IFC / Revit** | **Phase 3.** IFC via web-ifc (WASM) → floorplan slice extraction for underlays first; full BIM later | — | Worker |

Import UX contract: every import produces a **report** (entities imported/skipped/approximated) — silent lossy imports destroy trust with CAD users.

## 8. Path to 3D

Decisions made now so 3D is an extension:
- The scene graph's `transform`/`geom` are 2D but nodes carry optional `elevation`/`height` (devices already have mount height — the security module is "2.5D" from day one: camera math is fully 3D math projected onto the plan).
- Renderer abstraction is not 2D-specific (camera matrices, passes); a 3D view of the same document (extruded walls from polylines + device frustums) is the first 3D deliverable — visualization before 3D *editing*, which is a much later, separate investment.
- `zones`/walls get optional `height_m` now, so FoV occlusion can go volumetric without schema change.

## 9. Performance budgets & tests

| Metric | Budget |
|---|---|
| Cold open, 5k-entity drawing + PDF underlay | < 2.5 s to interactive |
| Pan/zoom | 60 fps @ 10k entities |
| Snap query | < 2 ms |
| Remote edit → local paint | < 120 ms same-region |
| Memory, 20k entities | < 400 MB tab |

Test strategy: geometry = property-based + golden fixtures; DXF importer = corpus of real-world files (collected from design partners) with snapshot diffs; renderer = pixel-snapshot tests on reference scenes; perf = automated benchmark scenes in CI with regression gates.
