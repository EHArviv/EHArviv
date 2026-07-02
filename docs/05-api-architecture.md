# 05 — API Architecture

## 1. Surfaces

| Surface | Protocol | Consumers | Purpose |
|---|---|---|---|
| **REST API** | HTTPS + JSON, OpenAPI 3.1 | Web app, field PWA, customer integrations | All CRUD + commands |
| **Realtime** | WebSocket (two channel types) | Web app, field PWA | CRDT doc sync; live events (presence, notifications, BOM updates) |
| **Webhooks** | HTTPS POST, HMAC-signed | Customer systems, ERP connectors | Outbound domain events |
| **Export endpoints** | HTTPS (presigned) | Users, ERP | XLSX/CSV/PDF/DXF artifacts produced by workers |

One API serves first-party and third-party clients — the web app has no private backdoor (API-first principle, [doc 02 §1](02-system-architecture.md)).

## 2. Conventions

- **Base:** `https://api.perimeter.app/v1/…`. URI-versioned major (`/v1`); additive changes are non-breaking; breaking changes ⇒ `/v2` with ≥12-month overlap.
- **Resources:** plural nouns, shallow nesting only where ownership is intrinsic: `/projects/{id}/drawings`, then flat: `/drawings/{id}`.
- **Commands** (state transitions) are POST subresources, not PATCH-with-magic-status: `POST /quotes/{id}/send`, `POST /pos/{id}/issue`, `POST /drawings/{id}/versions`.
- **IDs:** UUIDs; human codes (`Q-2026-0113`) are display attributes.
- **Pagination:** cursor-based (`?cursor=…&limit=50`, default 50, max 200) with `next_cursor` in the envelope.
- **Filtering/sorting:** whitelisted params per resource (`?status=in_progress&assignee=me&sort=-created_at`).
- **Sparse fields / expansion:** `?expand=product,assignee` for common joins; no arbitrary GraphQL-style traversal (rejected: GraphQL — authz complexity across 14 roles outweighs client flexibility; `expand` covers the real cases).
- **Errors:** RFC 9457 problem+json: `{type, title, status, detail, errors[{path,message}], request_id}`.
- **Idempotency:** `Idempotency-Key` header honored on all POST commands (stored 24 h).
- **Concurrency:** `If-Match` ETags on PATCH for non-CRDT resources; CRDT resources never conflict by construction.
- **Validation:** zod schemas in `packages/schemas` are the single source for request/response types, OpenAPI generation, and client SDK types.

## 3. Authentication & authorization

| Credential | Used by | Form |
|---|---|---|
| Session | Web/PWA users | Short-lived JWT (15 min) in memory + httpOnly rotating refresh cookie (14 d); `org_id` claim = active org; org switch mints a new token |
| API key | Integrations | `per_live_…` bearer key, hashed at rest, org-scoped, permission-key-scoped subset, rotatable |
| SSO | Enterprise users | SAML/OIDC → same session JWTs; SCIM for provisioning |
| Webhook signature | Outbound | `X-Perimeter-Signature: t=…,v1=HMAC-SHA256(secret, t + body)` |

Every request resolves `(user|key, org, permissions)` once in middleware; handlers declare required permission keys declaratively; the `authz.can()` check and RLS session context ([doc 04](04-permissions-model.md)) are applied automatically.

**Rate limits:** token bucket in Redis per key/user+org: default 600 req/min REST, burst 100; export/AI endpoints metered separately against plan quotas. `429` + `Retry-After`.

## 4. Endpoint catalog (representative, per module)

### Identity & org
```
POST   /auth/login | /auth/refresh | /auth/logout
GET    /me                          — profile, orgs, permissions
POST   /orgs                        GET/PATCH /orgs/{id}
GET    /orgs/{id}/members           POST /orgs/{id}/invitations
GET    /roles                       POST /roles     PATCH /roles/{id}
```

### Projects & structure
```
GET/POST /projects                  GET/PATCH/DELETE /projects/{id}
POST   /projects/{id}/members       PATCH /project-members/{id}       — incl. external scope
GET/POST /projects/{id}/sites       …/buildings, …/floors
```

### Drawings & design
```
GET/POST /projects/{id}/drawings    GET/PATCH/DELETE /drawings/{id}
POST   /drawings/{id}/versions      — freeze snapshot (label, from live doc)
GET    /drawings/{id}/versions      POST /drawings/{id}/versions/{n}/restore
POST   /drawings/{id}/export        — {format: pdf|dxf|svg|png, sheet, scale} → job
GET    /drawings/{id}/devices       — semantic projection (read-only; writes go through CRDT)
GET/POST /projects/{id}/zones       GET/POST /projects/{id}/cable-runs
POST   /imports                     — {file_id, target: drawing|underlay|catalog} → job
GET    /jobs/{id}                   — status of any async job
```

### Security engineering
```
POST   /drawings/{id}/coverage/compute        — full recompute → job (incremental runs live client-side)
GET    /drawings/{id}/coverage                — polygons, heatmap tiles, blind spots, per-zone DORI results
POST   /drawings/{id}/coverage/report         — PDF/DOCX engineering report → job
POST   /design/optimize                       — AI+solver placement suggestions (doc 12) → job
GET    /lens-calculator?sensor=…&lens=…&dist=…  — stateless optics math
```

### Catalog
```
GET    /products?category=&q=&specs.ir_range_m[gte]=30&sort=…
POST   /products                    GET/PATCH /products/{id}
GET    /products/{id}/revisions     POST /products/{id}/assets
GET/POST /price-lists               PUT /price-lists/{id}/prices        — bulk upsert (CSV/XLSX import via /imports)
POST   /products/import             — supplier bulk import → job w/ row-level error report
```

### BOM, quotes, procurement
```
GET    /projects/{id}/bom           — live revision w/ rollups
POST   /projects/{id}/bom/lines     PATCH/DELETE /bom-lines/{id}        — manual lines only; design lines are derived
POST   /projects/{id}/bom/freeze    — new frozen revision
GET    /projects/{id}/bom/compare?suppliers=a,b,c                       — supplier comparison matrix
POST   /projects/{id}/quotes        GET/PATCH /quotes/{id}
POST   /quotes/{id}/send | /approve | /reject                           — client approval via portal token
POST   /quotes/{id}/export          — {format: pdf|xlsx|csv} → job
POST   /projects/{id}/pos           POST /pos/{id}/issue | /acknowledge
POST   /pos/{id}/deliveries         — receive/partial-receive
```

### PM
```
GET/POST /projects/{id}/tasks       PATCH /tasks/{id}      — incl. {position} for kanban
PUT    /tasks/{id}/dependencies     GET /projects/{id}/schedule          — computed CPM: ES/EF/LS/LF, critical flags
GET/POST /projects/{id}/milestones | /risks | /issues | /change-requests | /meetings
POST   /change-requests/{id}/submit | /approve | /reject
POST   /approvals                   POST /approvals/{id}/decide
```

### DMS, collab, portal
```
POST   /files/presign               — presigned PUT {name,mime,size} → {upload_url, file_id}
POST   /files/{id}/complete         GET /files/{id}/download            — presigned GET
GET/POST /projects/{id}/folders     POST /files/{id}/versions | /sign
GET    /search?q=…&types=file,product,task,comment                      — federated search
GET/POST /…/{subject}/comments      POST /comments/{id}/resolve
GET    /notifications               POST /notifications/read
GET/POST /projects/{id}/rfis        POST /rfis/{id}/answer
GET/POST /projects/{id}/submittals  POST /submittals/{id}/decide
```

### AI
```
POST   /ai/conversations            POST /ai/conversations/{id}/messages   — SSE stream
POST   /ai/actions/{id}/apply       — human-approved execution of a proposed action (doc 12 §5)
```

### Dashboards & audit
```
GET    /dashboards/executive | /projects/{id}/dashboard | /finance | /procurement | /coverage
GET    /audit-log?actor=&action=&from=&to=          GET /audit-log/export → job
```

## 5. Realtime protocol

Two WebSocket channel types on `apps/realtime`:

**A. Document channels** — `wss://rt.perimeter.app/doc/{drawing_id}`
Binary Yjs sync protocol (sync step 1/2, update, awareness). Auth: one-time ticket from `POST /drawings/{id}/rt-ticket` (avoids long-lived JWT in query strings). Awareness carries presence: cursor, selection, active tool, viewport.

**B. Event channel** — `wss://rt.perimeter.app/events` (one per client)
JSON envelope `{type, subject, payload, ts}`; client subscribes to topics it can read (`project:{id}`, `user:{id}`): notifications, comment created, BOM updated, job finished, task moved. Server-filtered by the same `can()` checks.

Reconnect: exponential backoff; document channels resync via state vectors (no missed-message problem — CRDT); event channel replays from a Redis stream cursor (`Last-Event-Id` analog).

## 6. Webhooks

```
POST /webhooks            {url, events: ["quote.approved","po.issued","delivery.received",
                           "task.status_changed","drawing.version_created","rfi.answered",…]}
```

- At-least-once delivery from the outbox drain; retries with exponential backoff for 24 h; auto-disable after sustained failures (notify admin).
- Payloads carry `event_id` (dedupe), `org_id`, subject snapshot; consumers fetch fresh state via REST if needed.
- ERP integration path: webhooks + `GET /quotes/{id}/export?format=csv` + PO endpoints cover the brief's "export to ERP" without building per-ERP connectors prematurely; dedicated connectors (SAP/Netsuite/Priority) become marketplace apps later.

## 7. Async job pattern

All heavy operations return `202 {job_id}`; `GET /jobs/{id}` → `{status: queued|running|done|failed, progress, result:{file_id|url}, error}`. Clients also get push completion on the event channel. Jobs are org-scoped rows; results in object storage via presigned GET.

## 8. SDKs & docs

- OpenAPI spec auto-generated from zod route schemas; published at `api.perimeter.app/docs` (Scalar/Redoc UI).
- TypeScript SDK generated from the spec (`@perimeter/sdk`) — the web app itself consumes it, guaranteeing the public SDK is first-class.
