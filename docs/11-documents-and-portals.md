# 11 — Document Management & External Portals

## 1. DMS core

Schema in [doc 03 §9](03-database-schema.md); storage flows in [doc 05 §4](05-api-architecture.md) (presigned PUT/GET, never proxied through the API).

- **Structure:** org-level and project-level folder trees; system folders auto-provisioned per project (Drawings, Contracts, Photos, Permits, Reports, Invoices) — convention over configuration, fully renameable.
- **Files:** versioned (`file_versions`), content-hash deduped, kind-tagged, previewable (worker-generated thumbnails; PDF page previews; image EXIF/GPS retained for field photos — a photo taken in the field app pins itself to the plan).
- **Generated artifacts** (quotes, coverage reports, exports) are first-class files with provenance metadata linking back to their source (quote id + revision) — the DMS is the single place clients and auditors look.
- **Retention & legal hold:** per-org policy hooks (enterprise tier), soft-delete with trash window, hard purge job.

## 2. Ingestion intelligence (workers)

Pipeline on upload (per-kind): thumbnail → text extraction (native PDF text, else **OCR** — Tesseract baseline, cloud OCR adapter for quality) → `files.ocr_text` (FTS) → chunk + embed into `embeddings` ([doc 03 §11](03-database-schema.md)) → optional **AI summary** (`files.ai_summary`) and entity extraction (invoice numbers, permit dates) as suggested metadata. Federated `GET /search` spans files (OCR), products, tasks, comments, drawings-by-device-label.

## 3. Digital signatures

- V1: built-in click-to-sign (drawn/typed signature, signer identity, timestamp, IP, document hash) — legally adequate for approvals in most B2B contexts, stored in `signatures` with the signed PDF rendition frozen as a new file version.
- Enterprise: adapter interface for qualified e-signature providers (DocuSign/local eIDAS providers) where statutory signatures are required. Same `signatures` table, `provider` column.

## 4. Client portal

A scoped, simplified surface for Customer-role users ([doc 04 §4](04-permissions-model.md)) — same app, portal layout ([doc 13 §6](13-ui-architecture.md)):

- **View:** approved drawing revisions (watermarked viewer, no export unless granted), project status page (phase progress, milestones, upcoming work), dashboards (client-safe: no costs/margins/supplier identities).
- **Act:** comment on drawings (client feedback loop), **approve/reject drawing revisions and quotes** (with e-signature capture), raise change requests, download shared reports.
- **Access:** invited via email → lightweight account; every portal view/download is audit-logged (consultants bill for this visibility).
- Quote approval links are tokenized for the signer but always land in an authenticated session (no anonymous approvals).

## 5. Supplier portal

For Supplier-role externals, scoped by `project_members.scope` + `org_relationships`:

- **Catalog self-service:** CRUD own products, bulk price/stock/lead-time updates (the data freshness the whole BOM engine depends on — make this effortless: XLSX round-trip, API keys for their ERP).
- **RFQ / quoting:** see only BOM groups shared with them; submit line pricing + lead times → flows into supplier comparison ([doc 08 §3.3](08-catalog-and-bom.md)). Sealed until the buyer opens (no supplier sees competitors).
- **Order flow:** acknowledge POs, update delivery status/tracking, upload delivery notes.
- **Product revisions & submittals:** push product updates; respond to technical queries.

## 6. RFI & submittal workflows

Construction-standard structured Q&A, because email is where project knowledge goes to die:

- **RFI:** numbered, question + references (drawing pin, spec section, photo), assignee (internal or external), due date, status (`open → answered → closed`), answer becomes part of the project record; overdue RFIs escalate to the PM dashboard and risk suggestions ([doc 09 §4](09-project-management.md)).
- **Submittal:** supplier/installer submits product data or shop drawing for approval → review workflow (`approvals`) → approved/approved-as-noted/rejected; approved submittals link to the BOM line and device instances (the auditable chain from "what was designed" to "what was actually installed").

## 7. Notifications & digests

All portal-facing events (revision shared, approval requested, RFI assigned, PO issued) notify in-app + email with deep links; external users get branded email (white-label ready). Digest batching per user preference; unsubscribe never silences contractual events (approval requests).
