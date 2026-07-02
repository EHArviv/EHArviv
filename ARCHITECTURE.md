# Perimeter — Platform Architecture

*A unified cloud platform for designing, engineering, costing, procuring, and delivering physical-security and technology projects: browser CAD + camera engineering + catalog/BOM/quotes + project management + client/supplier collaboration + AI — in one live model.*

**Status:** Architecture phase complete, awaiting review. No implementation code exists yet — by design ([why](docs/00-executive-summary.md)).

> This design set currently lives on a branch of the profile repository. First implementation step is moving it to a dedicated product repo ([doc 14 §5](docs/14-roadmap-and-risks.md)).

## Read in this order

| # | Document | One-liner |
|---|---|---|
| 00 | [Executive Summary](docs/00-executive-summary.md) | The vision, the pushback, the decisions register — start here |
| 01 | [Product Strategy](docs/01-product-strategy.md) | Personas, competition, wedge strategy, business model |
| 02 | [System Architecture](docs/02-system-architecture.md) | C4 diagrams, modular monolith + workers, tenancy, tech stack |
| 03 | [Database Schema](docs/03-database-schema.md) | ERDs + DDL for every domain |
| 04 | [Permissions Model](docs/04-permissions-model.md) | 14 roles × modules matrix, external scoping, Postgres RLS |
| 05 | [API Architecture](docs/05-api-architecture.md) | REST conventions, endpoint catalog, realtime protocol, webhooks |
| 06 | [CAD Engine](docs/06-cad-engine.md) | WebGL renderer, CRDT scene graph, geometry kernel, DXF/DWG/PDF interop |
| 07 | [Security Design Module](docs/07-security-design-module.md) | FoV, DORI, occlusion, heatmaps, optimization — the differentiator |
| 08 | [Catalog & BOM](docs/08-catalog-and-bom.md) | Product model, design↔product binding, BOM→quote→PO pipeline |
| 09 | [Project Management](docs/09-project-management.md) | Tasks, Gantt/CPM, design-linked work, change control |
| 10 | [Collaboration & Realtime](docs/10-collaboration-realtime.md) | Yjs CRDT, presence, versions, offline/field mode |
| 11 | [Documents & Portals](docs/11-documents-and-portals.md) | DMS, OCR, signatures, client & supplier portals, RFIs |
| 12 | [AI Architecture](docs/12-ai-architecture.md) | Tool-use assistant, RAG, proposed-action gate, guardrails |
| 13 | [UI Architecture](docs/13-ui-architecture.md) | Design system, app shell, editor chrome, portals, field PWA |
| 14 | [Roadmap & Risks](docs/14-roadmap-and-risks.md) | Milestones M0–M8, risk register, scalability stages |

## The platform in three sentences

1. **The drawing is the database** — a camera placed on the floorplan *is* a catalog product, a BOM line, an install task, and a commissioning record; nothing is copy-pasted between tools.
2. **Deterministic engineering, honest AI** — optics/DORI/costing/scheduling are typed, tested engines; AI orchestrates, explains, and proposes, behind a human-approval gate.
3. **Win the security-design vertical first** — better engineering than JVSG, better collaboration than everyone, then expand outward toward general CAD, marketplace, and enterprise.

## Key decisions at a glance

Security-design wedge before general CAD · own CRDT scene graph as native format (DXF/PDF first, DWG via conversion worker) · modular monolith + realtime service + workers · single Postgres with RLS multi-tenancy · Yjs for collaboration · WebGL2 canvas · TypeScript end-to-end in a pnpm/Turborepo monorepo · REST + OpenAPI + webhooks · Claude-powered assistant with tool-use over internal APIs.

Full decision register with rationale: [doc 00 §4](docs/00-executive-summary.md).
