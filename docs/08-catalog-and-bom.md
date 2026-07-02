# 08 — Catalog & BOM/Quote Engine

The money pipeline: **catalog → design binding → live BOM → costed quote → purchase packages**. Schema in [doc 03 §6–7](03-database-schema.md); permissions (who sees cost vs. margin vs. sell) in [doc 04](04-permissions-model.md).

## 1. Catalog

### 1.1 Data model highlights

- **Category spec schemas:** each `product_category` carries a JSON schema for its specs (`camera.*` requires sensor size, resolution, lens type/range, IR range, power, IP rating…). Validation at write time makes specs *queryable* (`specs.ir_range_m >= 30`) and powers the optics auto-fill in [doc 07 §1](07-security-design-module.md). Curated platform-level schemas; org custom fields allowed in a namespaced extension.
- **Ownership & visibility tiers:** `private` (tenant's own library — day one), `shared` (supplier shares with linked buyer orgs via `org_relationships`), `marketplace` (public, curated — M8).
- **Revisions:** every product edit appends `product_revisions`; projects reference products by id and the BOM freezes resolved data at quote time, so supplier edits never mutate an issued quote.
- **Assets:** images, datasheets, manuals, certificates, and **CAD blocks** (symbol attached to product → placing the product places the right symbol).
- **Pricing separated from product:** `price_lists`/`prices` per supplier↔buyer relationship, currency, validity window, MOQ, lead time, stock snapshot. Multiple suppliers can price the same product — this is what makes supplier comparison possible.

### 1.2 Ingestion & quality

- Bulk import (XLSX/CSV mapping wizard → `/products/import` job with row-level error report); supplier portal CRUD; platform-curated seed data for top security manufacturers (bootstrapping content before marketplace liquidity — [doc 01 §4](01-product-strategy.md)).
- Dedupe by manufacturer+model heuristics with merge review; `lifecycle` flags (EOL products warn in designs that still use them).
- Search: Postgres FTS + spec filters + category tree; ranking by technical fit first — paid placement (M8) is visually labeled and never reorders technical suitability.

## 2. Design ↔ product binding

The core loop that makes "the drawing is the database" real:

1. Designer places a **device** node (generic camera / from product picker / from a product's attached block).
2. Binding sets `device_instances.product_id`; specs flow into camera math instantly ([doc 07](07-security-design-module.md)).
3. **Kit expansion:** products can define an accessory kit (mount, bracket, PSU, license) as `compat.kit[]` → child BOM lines auto-added with the device, editable per instance.
4. **Cable runs** bind to cable products; `length_m` (path + slack factor) drives quantity in meters; PoE budget check: sum of powered-device wattage per switch vs. switch PoE budget → validation finding.
5. Re-binding a device to a different product updates BOM, camera math, and symbol in one transaction; bulk re-bind ("swap all CAM-X to CAM-Y") supported.

## 3. BOM engine

### 3.1 Derivation

`bom_lines.source = 'design'` lines are **projections, not copies**: on `device.placed/updated/removed` and `cable_run.*` events, the BOM module incrementally upserts affected lines (qty = count of instances per product per group; cable qty = summed lengths). Manual lines (`source='manual'`) cover labor, shipping, engineering fees, and anything undrawn. AI-suggested lines ([doc 12](12-ai-architecture.md)) enter as `ai_suggested`, visibly flagged until a human accepts them.

**Completeness checks** (findings, not silent fixes): camera without mount, PoE device without switch port budget, switch without cabinet, cabinet without power, run without cable product — the "missing components" detection from the brief, implemented as deterministic rules over the design graph.

### 3.2 Costing & rollups

Per line: `unit_cost` resolved from the **best applicable price** (buyer-specific list → shared list → list price; FX-normalized to project currency, rate timestamped). Rollups cached in `boms.totals`:

```
cost = Σ qty·unit_cost            sell = Σ qty·unit_sell (from quote pricing)
margin = sell − cost              margin_pct per line / group / project
+ tax (configurable schemes: VAT, US sales tax, per-line class)
+ shipping (manual or rule-based) + installation (labor lines: hours × role rates)
+ contingency %                   ⇒ budget summary
```

### 3.3 Supplier comparison

`GET /projects/{id}/bom/compare` builds a matrix: for each line, every supplier with a valid price → total project cost per supplier mix, plus optimal mixed-supplier selection (min cost s.t. constraints: preferred suppliers, max supplier count, lead-time ceiling). Output feeds PO splitting.

## 4. Quotes & proposals

- Quote **pins a frozen BOM revision** (`boms.status='frozen'`) — design keeps evolving on the live revision; the delta view ("design has changed since Quote R2: +3 cameras, −120 m cable") powers change requests ([doc 09](09-project-management.md)).
- **Pricing layer** stored on the quote: per-line/per-group sell price, discount, margin; margin visibility restricted (`quote.margins.view`).
- **Proposal document**: templated (org branding, cover, scope narrative — AI-drafted, human-edited), line-item or summarized pricing, coverage-report excerpts as appendix. Rendered to PDF by workers.
- **Client flow:** send → portal link (tokenized) → client views (tracked) → approve/reject with comment + e-signature ([doc 11 §4](11-documents-and-portals.md)) → `quote.approved` event → project status advances, PO drafting unlocked.
- Revisions: R1, R2… each pinned to its BOM freeze; diffs rendered between revisions.

## 5. Procurement

- **PO drafting from BOM:** select lines (or accept the comparison optimizer's split) → POs per supplier; lines carry `bom_line_id` lineage, so delivery status flows back to devices ("CAM-014's camera: delivered, mount: backordered").
- Supplier acknowledgment + delivery updates via supplier portal ([doc 11 §5](11-documents-and-portals.md)); warehouse receiving (partial quantities) updates `po_lines.delivered_qty` → device `status` transitions (`planned → ordered → delivered`), visible on the drawing (status-colored devices) and Gantt.
- **Exports:** XLSX/CSV/PDF for BOM, quote, PO; webhook events for ERP sync ([doc 05 §6](05-api-architecture.md)).

## 6. Events (module contract)

Consumes: `device.*`, `cable_run.*`, `product.revised`, `price.updated` (stale-price warnings on open BOMs).
Emits: `bom.updated`, `bom.frozen`, `quote.sent/approved/rejected`, `po.issued/acknowledged/delivered`, `finding.created` (completeness/PoE/stale-price findings).
