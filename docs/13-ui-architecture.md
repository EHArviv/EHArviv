# 13 — UI Architecture

Modern, minimal, professional. References: Figma (editor chrome), Linear (speed, keyboard-first), Notion (documents), Stripe Dashboard (data density with calm), AutoCAD (command muscle memory). Lives in `packages/ui` (design system) + `apps/web`.

## 1. Design system foundations

- **Tokens first:** color, spacing (4 px grid), radius, typography, elevation, motion — defined as CSS variables, consumed by Tailwind config. **Dark and light themes are token swaps**, both first-class from day one (designers live in dark; clients get light-default portal).
- **Type:** Inter (UI) + JetBrains Mono (coordinates, IDs, command bar). Full **RTL support** from the foundation: logical CSS properties only (`ms-*/me-*`, no `ml-*/mr-*`), direction-aware icons, Hebrew locale in the test matrix ([doc 01 §4](01-product-strategy.md)).
- **Components:** Radix primitives wrapped with our tokens — buttons, inputs, combobox, dialog, popover, toast, table (virtualized), tabs, tree, date/gantt primitives, command palette (`⌘K`), empty states, skeletons. Storybook as the contract; visual regression snapshots in CI.
- **Accessibility:** WCAG 2.1 AA for all non-canvas UI; canvas gets keyboard tool access, focus management, and a screen-reader object inspector (full canvas a11y is a hard problem — commit to the inspector pattern, not to pretending).

## 2. Application shell

```
┌────────────────────────────────────────────────────────────┐
│ Top bar: org switcher · project switcher · search(⌘K) ·    │
│          presence avatars · notifications · user menu      │
├──────────┬─────────────────────────────────────────────────┤
│ Left nav │  Module surface (Design / BOM / Tasks / Docs …) │
│ (module  │                                                 │
│  rail,   │                                                 │
│  project │                                                 │
│  scoped) │                                                 │
├──────────┴─────────────────────────────────────────────────┤
│ Context bar (selection details / filters, per module)      │
└────────────────────────────────────────────────────────────┘
```

- **Project-centric navigation:** org home (portfolio) → project workspace with module rail: Overview · Design · Coverage · Catalog · BOM & Quotes · Procurement · Tasks · Documents · RFIs · Team · Settings. Modules the plan/role doesn't include simply don't render.
- **Keyboard-first:** `⌘K` command palette reaches every navigation and command; Linear-style shortcut discipline (`G then D` = go to Design). Editor gets the AutoCAD command bar on top of this.
- **URL discipline:** everything deep-linkable (`/p/{project}/design/{drawing}?v=12&sel=CAM-014`) — links are how collaboration actually happens.

## 3. The editor surface (flagship screen)

Figma-pattern chrome around the CAD canvas ([doc 06](06-cad-engine.md)):

- **Left:** tool rail (select/draw/device/cable/zone/measure…) + layers & sheets panel + device tree (grouped by type/zone, syncs selection with canvas).
- **Canvas center:** WebGL viewport; presence cursors; comment pins; status-colored devices; coverage overlay toggles (DORI bands / heatmap / night / k-coverage) as floating layer chips.
- **Right:** contextual inspector — selection properties; for a camera: product card, optics params (height/tilt/lens sliders with live FoV update), per-camera DORI readout, "preview from camera" button; for a zone: requirement editor.
- **Bottom:** command bar + coordinates + snap toggles + units + zoom.
- **Panels are collapsible to a zen canvas**; every panel state persists per user.
- Copilot dock ([doc 12 §1](12-ai-architecture.md)) slides in from the right without covering the inspector.

## 4. Module surface patterns

| Surface | Pattern |
|---|---|
| BOM & Quotes | Virtualized editable table, group sections, rollup footer pinned; margin columns permission-gated; quote builder = document-preview split view |
| Procurement | Pipeline board (Draft → Issued → Ack → Delivering → Done) + PO detail with delivery timeline |
| Tasks | Kanban / Gantt / list / calendar view switcher (state in URL); Gantt uses canvas rendering for large projects |
| Catalog | Faceted search grid with spec filters; product page with assets, price panels (relationship-scoped), "used in N projects" |
| Documents | Tree + grid/list, inline PDF viewer with comments, version drawer |
| Coverage dashboard | Floor heatmap thumbnails, compliance table, findings list — click-through to editor with the finding focused |
| Dashboards | Widget grid (shared widget library), role-gated presets: Executive, Project, Financial, Procurement, Supplier, Coverage, Resource, Risk |

## 5. State & data layer (frontend)

- **Server state:** TanStack Query over the generated SDK ([doc 05 §8](05-api-architecture.md)); realtime events invalidate/patch query caches ("REST mutates, events refresh" — [doc 10 §6](10-collaboration-realtime.md)).
- **Editor state:** Yjs doc + local mirror ([doc 06 §3](06-cad-engine.md)); UI state (panels, tool) in zustand; nothing editor-critical in React state on the hot path.
- **Optimistic UI** for task moves, comments, BOM manual edits with event reconciliation.
- **Route-level code splitting:** the editor bundle (CAD engine + WebGL) loads only on editor routes; dashboards stay light.

## 6. Portal layouts

Client/supplier portals are the **same app, different shell**: simplified top nav (no module rail), brandable (logo/accent color for white-label later), light-theme default, zero jargon labels. Portal screens: status page, drawing viewer (watermark, approved revisions), quote review/approve, RFI inbox, catalog manager (suppliers). Mobile-responsive first — clients approve from phones.

## 7. Field app (PWA)

Installable PWA sharing `packages/ui`: today-list, device checklists (commissioning), plan viewer with pins, camera-first photo capture (auto-pinned via plan tap), issue/snag creation, offline outbox ([doc 10 §7](10-collaboration-realtime.md)). Big touch targets, glove-friendly spacing, aggressive caching.

## 8. Quality gates

Playwright E2E on: editor draw→device→BOM→quote critical path, portal approval flow, RTL rendering snapshot suite, dark/light snapshot suite, keyboard-only navigation audit per release.
