# 01 — Product Strategy

## 1. Positioning statement

> For **security integrators, ELV contractors, and security consultants** who juggle CAD, spreadsheets, camera calculators, and project tools, **Perimeter** is a **collaborative cloud platform** that turns a floorplan into an engineered, costed, procurable, and manageable project — unlike AutoCAD + Excel + JVSG + Procore stitched together, it keeps design, engineering math, money, and execution in **one live model**.

## 2. Who buys and who uses it

### Buyer personas

| Persona | Buys because | Success metric |
|---|---|---|
| **Integrator owner / GM** (10–200 staff) | Quotes go out 5× faster, margins visible per project, fewer redesign disputes | Quote turnaround time, win rate, project margin |
| **Security consultant principal** | Deliverables (coverage reports, DORI compliance) look engineering-grade and are defensible | Billable utilization, report quality |
| **Enterprise security director** (end customer, later) | One system of record for the estate's security design and as-builts | Audit readiness, estate visibility |

### User personas (map to roles in [doc 04](04-permissions-model.md))

| Persona | Daily job in the product |
|---|---|
| **CAD designer / drafter** | Draws floorplans, places devices, manages layers and sheets |
| **Security consultant / engineer** | Camera selection, FoV/DORI engineering, coverage analysis, risk zones |
| **Project manager** | Tasks, Gantt, approvals, RFIs, change requests, client comms |
| **Procurement manager** | BOM → supplier comparison → purchase packages → delivery tracking |
| **Estimator / finance** | Costing, margins, taxes, quote versions |
| **Supplier rep** | Maintains catalog products, answers RFQs, updates lead times |
| **Installer / field tech** | Field app: device checklists, photos, as-built markups |
| **Client stakeholder** | Portal: view drawings, comment, approve revisions and quotes |

## 3. Competitive landscape

| Competitor | Strengths | Exploitable weaknesses |
|---|---|---|
| **AutoCAD (+ Web)** | Industry-standard drafting, DWG | Zero domain intelligence, no lifecycle, per-seat cost, collaboration is file-based |
| **JVSG IP Video System Design** | Best-in-class camera optics math | Desktop, single-user, no BOM/procurement/PM, dated UX |
| **System Surveyor** | Site survey UX, device inventory on plans | No optics engineering, no CAD depth, no procurement |
| **D-Tools Cloud / SI** | Catalog, proposals, integrator workflows | No real design canvas, no camera engineering |
| **Procore** | Construction PM at scale | Generic; no design, no security domain |
| **Monday / Asana** | Flexible PM | No domain model at all |
| **Roundefence-type simulators** | 3D FoV simulation | Point tool; no persistence of value beyond the demo |
| **Ubiquiti Design Center / manufacturer tools** | Free, easy | Locked to one vendor's hardware; we are vendor-neutral |

**Moat sequence:** (1) unified lifecycle → (2) proprietary catalog network effects (suppliers maintain their own data) → (3) project corpus powering AI recommendations → (4) marketplace transaction share.

## 4. Scope by module — brief deltas

The 10 modules in the brief all stand. Strategy adjustments:

1. **CAD module** is built as the substrate for the security module (M1 before M2), marketed as "design canvas," not "AutoCAD replacement," until DWG round-trip ships.
2. **Security module** is the hero feature and demo centerpiece. It must be *better engineering* than JVSG (standards-cited DORI, visibility-polygon occlusion) and *better UX* than everyone.
3. **Catalog** starts as tenant-private + platform-curated seed data; the open supplier marketplace is M8 (network effects need liquidity; don't build the empty marketplace first).
4. **BOM → Quote → Procurement** is one continuous money pipeline ([doc 08](08-catalog-and-bom.md)); quoting was missing from the brief and is now first-class.
5. **PM** integrates with design (a device ↔ an install task) rather than competing with Monday on generic features.
6. **AI** is embedded per-module (optimize placement, draft RFI answers, summarize meetings) rather than a bolted-on chatbot ([doc 12](12-ai-architecture.md)).

### Added scope (missing from the brief)

- Quote/proposal builder with e-signature and versioned revisions
- Field/survey mode (PWA, offline-tolerant, photo pins, snag lists)
- Cable & pathway engineering (runs, containment, PoE budget, switch ports)
- Standards engine (IEC 62676-4 DORI presets, privacy/GDPR masking zones)
- Commissioning checklists and as-built generation
- SSO/SAML/OIDC + SCIM, audit log export (enterprise tier)
- i18n with RTL (Hebrew first-class)

### Explicitly deferred (with triggers)

| Deferred | Trigger to build |
|---|---|
| Full 3D modeling / IFC authoring | Vertical wedge won; enterprise demand for BIM deliverables |
| Native DWG write | ≥20% of deals blocked on DWG round-trip |
| Drones/radar/access-control design | Camera module adoption proven; same geometry engine extends |
| Open marketplace with payments | ≥50 active supplier tenants maintaining catalogs |
| White-label | First committed enterprise/OEM contract |

## 5. Business model

**Multi-tenant SaaS, per-seat with role-based pricing** (design seats cost more than portal seats), plus usage and marketplace layers.

| Plan | Target | Includes | Price posture |
|---|---|---|---|
| **Starter** | Freelancers, tiny integrators | 2 design seats, 5 projects, core CAD + security design, BOM, PDF export | Low flat monthly |
| **Team** | 5–50 person integrators | Unlimited projects, quoting, PM, client portal, DXF import, integrations | Per-seat: designer / PM / field tiers |
| **Business** | 50–200 | Supplier collaboration, approval workflows, dashboards, API access | Per-seat + volume discount |
| **Enterprise** | Consultancies, enterprises, OEM | SSO/SCIM, audit exports, DWG round-trip, white-label, self-host option, SLA | Annual contract |

Additional revenue: **marketplace commission** on transacted POs (M8+), **AI usage metering** above plan quotas, **priority catalog placement** for suppliers (never distorting technical suitability ranking — trust is the moat).

**Free viral loops:** client portal and supplier portal seats are free — every shared project recruits the next tenant.

## 6. North-star & guardrail metrics

- **North star:** weekly active *projects* (a project touched by ≥2 roles in a week — proves the lifecycle thesis).
- Design→quote conversion time; quote→PO conversion; devices placed per week; supplier catalog freshness; portal WAU; NRR.

## 7. Riskiest assumptions (tested earliest)

| Assumption | Test | Milestone |
|---|---|---|
| Integrators will design in a browser canvas | M1/M2 beta with 5 design partners | M2 |
| Camera math credibility wins consultants | DORI report reviewed by 3 external consultants against JVSG output | M2 |
| Live BOM from the drawing changes buying behavior | Track % of quotes generated without exporting to Excel | M3 |
| Suppliers will maintain their own catalog data | 10 suppliers onboarded manually, measure update cadence | M6–M8 |
