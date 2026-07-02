# 09 — Project Management Module

PM that competes on **design integration**, not on out-generic-ing Monday: tasks link to devices and drawings, schedules react to procurement lead times, change control links quotes to scope. Schema in [doc 03 §8](03-database-schema.md).

## 1. Work model

- **Tasks** with subtasks (one level via `parent_id`), assignee, dates, estimates, status machine (`todo → in_progress → blocked → review → done`), fractional-key ordering for Kanban.
- **Design linkage:** a task may reference a `device_instance` or `drawing`. Generators create work from design: "create install tasks for all 46 cameras on Floor 2, grouped by zone" → tasks carrying device links; completing the install task (with field photo evidence) advances the device status (`delivered → installed`), which recolors the drawing and feeds the coverage-vs-installed dashboard.
- **Milestones** group tasks and gate phases (Design approved → Procurement → Installation → Commissioning → Handover). Default phase template per project type; fully editable.
- **Checklists** on tasks (commissioning checklists are generated per device type from templates — the commissioning feature from [doc 01 §4](01-product-strategy.md)).

## 2. Views

| View | Notes |
|---|---|
| **Kanban** | Status columns (or custom fields later); swimlanes by assignee/milestone; drag = `PATCH {status, position}` |
| **Gantt / Timeline** | Bars from `starts_on/due_on`; dependency arrows; drag-to-reschedule cascades through dependents (preview then commit); baseline snapshots for slippage view |
| **List / table** | Grouping, saved filters, bulk edit |
| **Calendar** | Due dates, milestones, meetings |
| **Workload** | Per-assignee load from estimates vs. capacity |

All views are projections of the same task set; realtime via event channel ([doc 05 §5](05-api-architecture.md)).

## 3. Scheduling engine (CPM)

`GET /projects/{id}/schedule` computes on read (cached, event-invalidated):

- Forward/backward pass over the dependency DAG (FS/SS/FF/SF + lag) → ES/EF/LS/LF, total float, **critical path** flags; cycles rejected at write time.
- Working calendar (org holidays, workdays) applied.
- **Procurement-aware:** a task linked to devices whose PO has `expected_delivery` later than the task's early start gets a **material-constraint warning** ("Install Floor 2 cannot start before Mar 14 — switches arrive Mar 12 + staging"). This coupling is the differentiator; generic PM tools cannot do it.
- Duration estimation assist: AI suggests estimates from labor norms (devices × install-hours by type) — flagged, human-confirmed ([doc 12](12-ai-architecture.md)).

## 4. Risk, issues, meetings

- **Risk register:** probability × impact (1–5) grid, owner, mitigation, status; heat-matrix dashboard widget; AI drafts risk suggestions from project signals (stale RFIs, slipping critical tasks, single-supplier concentration) — created as drafts for human triage.
- **Issues:** lightweight defects/blockers, linkable to tasks/devices/drawings; field app creates issues with photo + plan pin (snag list).
- **Meetings:** agenda, attendees, notes; AI meeting summarization ([doc 12](12-ai-architecture.md)) produces minutes + extracted action items proposed as tasks (accept/edit/reject).

## 5. Approvals & change control

- **Generic approval workflow** (`approvals`): subject (drawing version, quote, submittal, CR, document), ordered steps, per-step approvers (role or named users), decision + comment; pending approval locks the subject ([doc 04 §5](04-permissions-model.md)). Notifications + portal surfaces for client-side steps.
- **Change requests:** the bridge between scope and money. CR captures description, origin (client ask, site condition, design gap), then *estimating* attaches a delta-BOM/mini-quote (built with the same BOM tooling on a CR scratch revision), schedule impact (delta days, affected tasks); approval applies the deltas: merges lines into the live BOM, issues quote revision, shifts schedule — all as one audited transaction.
- **Version history everywhere:** drawings have CRDT versions ([doc 10](10-collaboration-realtime.md)); quotes/BOMs have revisions; tasks and CRs have field-level activity trails from `activity_events`.

## 6. Project dashboard (module-owned)

Widgets fed by materialized rollups ([doc 03 §13](03-database-schema.md)): phase progress, critical path summary, budget vs. committed (POs) vs. actual, coverage compliance %, open RFIs/CRs/risks, upcoming milestones, recent activity. Executive/financial/procurement dashboards compose the same widget library across projects ([doc 13 §5](13-ui-architecture.md)).
