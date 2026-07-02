# 04 — Permissions Model

## 1. Model overview

Three layers, evaluated in order; **deny wins** at every step:

1. **Org membership + roles (RBAC).** A user's roles in an org grant *permission keys* (e.g. `quote.approve`). Roles are system templates (the 14 below) or org-defined custom roles — both are just bundles of permission keys (`roles` / `role_permissions` in [doc 03 §3](03-database-schema.md)).
2. **Project scoping.** `project_members` narrows or extends access per project. Internal staff may see all projects (`project.view_all`) or only assigned ones. **External users (suppliers, clients, installers from subcontractors) get access *only* through `project_members`**, with a `scope` restricting modules/floors/documents.
3. **Resource guards.** Fine-grained states override roles: a frozen BOM revision is read-only for everyone; an approval in `awaiting_approval` locks its subject; privacy-masked zones hide camera previews from portal viewers.

```
can(user, action, resource) =
  is_member(user, resource.org)                         -- tenancy
  ∧ action ∈ permissions(roles(user, resource.org) ∪ project_roles(user, resource.project))
  ∧ within_scope(user, resource)                        -- floors/modules scope for externals
  ∧ resource_guard(action, resource)                    -- state machines, freezes, privacy
```

Evaluation is implemented once in the `authz` module (`can()` — used by API routes, the realtime service, and workers) and mirrored defensively by Postgres RLS (§6).

## 2. Permission key catalog (by module)

Keys follow `module.action[_qualifier]`. Representative set (full list lives in `packages/schemas/permissions.ts` as the single source of truth):

| Module | Keys |
|---|---|
| org | `org.manage`, `org.members.invite`, `org.members.manage`, `org.roles.manage`, `org.billing.manage`, `org.settings.manage` |
| project | `project.create`, `project.view_all`, `project.view_assigned`, `project.edit`, `project.archive`, `project.members.manage` |
| drawing | `drawing.view`, `drawing.edit`, `drawing.approve`, `drawing.export`, `drawing.version.restore` |
| design | `design.devices.edit`, `design.zones.edit`, `design.cables.edit`, `security.calc.run`, `security.report.export` |
| catalog | `catalog.view`, `catalog.manage`, `catalog.prices.view_cost`, `catalog.prices.manage`, `catalog.publish` |
| bom | `bom.view`, `bom.edit`, `bom.costs.view`, `bom.export` |
| quote | `quote.view`, `quote.edit`, `quote.margins.view`, `quote.send`, `quote.approve_internal`, `quote.approve_client` |
| procurement | `po.view`, `po.edit`, `po.issue`, `po.receive`, `delivery.update` |
| pm | `task.view`, `task.edit`, `task.assign`, `gantt.edit`, `risk.manage`, `cr.create`, `cr.approve`, `meeting.manage` |
| dms | `file.view`, `file.upload`, `file.delete`, `file.sign`, `folder.manage` |
| collab | `comment.create`, `comment.resolve`, `mention.any` |
| portal | `rfi.create`, `rfi.answer`, `submittal.create`, `submittal.decide` |
| ai | `ai.chat`, `ai.actions.apply` |
| dashboards | `dashboard.exec`, `dashboard.finance`, `dashboard.procurement`, `dashboard.project` |
| audit | `audit.view`, `audit.export` |
| system | `platform.admin` (cross-tenant; Perimeter staff only, fully audited) |

## 3. The 14 role templates × permissions matrix

Legend: ✅ full · 👁 view-only · ◐ limited (noted) · — none. Custom roles can deviate; templates ship as defaults.

| Permission area | Sys Admin¹ | Company Admin | Project Mgr | Engineer | CAD Designer | Security Consultant | Procurement | Finance | Warehouse | Supplier² | Installer² | Customer² | Auditor | Viewer |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Org & members | ✅ | ✅ | — | — | — | — | — | — | — | — | — | — | — | — |
| Billing | ✅ | ✅ | — | — | — | — | — | ✅ | — | — | — | — | — | — |
| Projects create/edit | ✅ | ✅ | ✅ | ◐ edit | ◐ edit | ◐ edit | 👁 | 👁 | 👁 | — | — | — | 👁 | 👁 |
| Drawings edit | ✅ | ✅ | ◐ | ✅ | ✅ | ✅ | — | — | — | — | ◐ markup | — | — | — |
| Drawings view/export | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | 👁 | — | — | ◐ scoped | 👁 scoped | 👁 approved revs | 👁 | 👁 |
| Drawing approve | ✅ | ✅ | ✅ | ◐ | — | ✅ | — | — | — | — | — | ◐ client approve | — | — |
| Security calc & reports | ✅ | ✅ | 👁 | ✅ | ◐ run | ✅ | — | — | — | — | — | 👁 reports | 👁 | — |
| Catalog manage | ✅ | ✅ | — | ◐ suggest | ◐ suggest | ◐ suggest | ✅ | — | — | ✅ own products | — | — | — | — |
| Cost prices view | ✅ | ✅ | ✅ | ◐ | — | ◐ | ✅ | ✅ | — | ◐ own | — | — | 👁 | — |
| BOM edit | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ◐ | — | — | — | — | — | — | — |
| Quote edit/margins | ✅ | ✅ | ✅ | — | — | — | ◐ | ✅ | — | — | — | — | 👁 | — |
| Quote send / client-approve | ✅ | ✅ | ✅ send | — | — | — | — | ◐ | — | — | — | ✅ approve | — | — |
| PO issue / receive | ✅ | ✅ | ◐ | — | — | — | ✅ | ◐ approve | ✅ receive | ◐ acknowledge | — | — | 👁 | — |
| Delivery updates | ✅ | ✅ | 👁 | — | — | — | ✅ | — | ✅ | ✅ own POs | — | — | — | — |
| Tasks / Gantt | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ◐ | — | ◐ | ◐ own tasks | ✅ own tasks | 👁 status | 👁 | 👁 |
| Risks / CRs | ✅ | ✅ | ✅ | ◐ create | ◐ create | ◐ create | ◐ | ◐ | — | — | ◐ report issue | ◐ request CR | 👁 | — |
| DMS upload/delete | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ◐ | ◐ scoped | ◐ photos | ◐ scoped | — | — |
| DMS view | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ◐ scoped | ◐ scoped | ◐ scoped | 👁 all | 👁 |
| Comments / RFIs | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | — | — |
| AI assistant | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ◐ | — | ◐ | — | — | — |
| Dashboards | ✅ all | ✅ all | ✅ project | ◐ | ◐ | ◐ coverage | ✅ procurement | ✅ finance | ◐ | ◐ own | — | ✅ client | 👁 all | 👁 |
| Audit log | ✅ | ✅ | — | — | — | — | — | — | — | — | — | — | ✅ export | — |

¹ **System Admin** = platform staff (cross-tenant, break-glass, every action audited + alerted). **Company Admin** is the top role *within* a tenant.
² Supplier, Installer (subcontractor), and Customer are **external**: they exist only via `project_members` with `scope`, never as full org members of the tenant. "Scoped" = limited to the floors/folders/modules listed in their scope JSON. Supplier variants from the brief (quote-only vs pricing vs delivery-updates) are custom roles derived from the Supplier template by removing keys.

## 4. Scoping semantics for external users

`project_members.scope` example for a supplier invited to quote on networking only:

```json
{
  "modules": ["bom.view", "rfi", "comments"],
  "bom_groups": ["network"],
  "floors": ["<floor-uuid>"],
  "hide": ["cost_prices_of_others", "margins", "client_identity"]
}
```

Hard rules enforced by resource guards regardless of scope:
- Suppliers never see other suppliers' prices, internal costs, or margins.
- Clients never see cost prices, margins, or supplier identities (unless tenant opts in).
- Externals never see the org member directory beyond participants of their own threads.
- Portal users see only **approved** drawing revisions unless explicitly granted draft access.

## 5. Approval workflows as permission overlays

Approvals ([doc 03 §8](03-database-schema.md)) lock their subject while pending: `quote.awaiting_approval` blocks `quote.edit` even for Company Admin (they must withdraw the approval first — audited). Client approval of a quote or drawing revision is executed by a Customer-role user and captured with timestamp, IP, and optional e-signature.

## 6. Postgres RLS strategy

RLS is the **backstop**, not the primary evaluator (business rules like margin-hiding live in `can()`; RLS guarantees tenancy + project scoping even if application code has a bug).

Per request, `core-api` opens a transaction and sets:

```sql
select set_config('app.user_id', :user_id, true);
select set_config('app.org_id',  :org_id,  true);   -- active org context
```

Canonical policies:

```sql
-- 1) Tenancy: applied to every org-scoped table
create policy org_isolation on projects
  using (org_id = current_setting('app.org_id')::uuid);

-- 2) Project scoping for users who are not org-wide viewers
create policy project_scope on tasks
  using (
    org_id = current_setting('app.org_id')::uuid
    and (
      exists (select 1 from org_members m
              join member_roles mr on mr.member_id = m.id
              join role_permissions rp on rp.role_id = mr.role_id
              where m.user_id = current_setting('app.user_id')::uuid
                and m.org_id = tasks.org_id
                and rp.permission = 'project.view_all')
      or exists (select 1 from project_members pm
                 where pm.project_id = tasks.project_id
                   and pm.user_id = current_setting('app.user_id')::uuid)
    )
  );

-- 3) External users connect with org context = the *tenant's* org they were invited into,
--    but their rows are reachable only via project_members (policy 2), and column-level
--    protection (margins, costs) is done by API serializers, not SQL.
```

- Membership/permission lookups are cached per-connection via a `security definer` helper function marked `stable`, keeping policy overhead ~one indexed probe.
- Workers and the realtime service use the same `set_config` discipline; a dedicated `migrator` role bypasses RLS; the app role cannot.
- **CI isolation tests:** a test suite forges every context combination and asserts cross-tenant reads/writes return zero rows / fail — a red build is a release blocker.

## 7. Auditing

Every grant/revoke, role change, external invitation, approval decision, export, and break-glass access writes to `audit_log` (append-only, partitioned). Auditor role gets read + export of their own org's log; enterprise tier streams it via webhook/SIEM export (the user persona here — security engineers — will demand this).
