# Domain Pack: Multi-tenant SaaS

Extends the `logic-math`, `data-layer`, and `security` agents. Load when the stack detection identifies organizational / workspace / tenant boundaries (presence of `org_id`, `workspace_id`, `tenant_id` columns, multi-seat pricing, SSO integration patterns).

The core concern of multi-tenancy: **no tenant should ever see, modify, or affect another tenant's data.** One cross-tenant data leak destroys the business; everything in this pack flows from that.

## Tenant isolation at the query layer

### The golden rule

Every query against a tenant-scoped table must include a tenant predicate. Period.

Flag:
- Any `SELECT` / `UPDATE` / `DELETE` on a tenant-scoped table without a `WHERE tenant_id = ?` clause
- Tenant predicate applied in the application layer "around" the ORM rather than consistently in the query (easy to miss)
- Any raw SQL without tenant scoping
- `db.user.findById(id)` without tenant context when `User` is tenant-scoped

Best practices to look for (and flag absence of):

1. **Row-Level Security (RLS)** — database enforces the rule, not the application. Postgres RLS with session variables is the gold standard.
2. **Tenant-scoped ORM middleware** — query builder automatically injects tenant predicate.
3. **Schema-per-tenant** — physical isolation; most expensive, least error-prone.
4. **Database-per-tenant** — maximum isolation, operationally complex.

If none of these is in place, flag as high severity.

### Tenant context source

The tenant ID must come from a trusted source — never from the request body or URL path (easy to manipulate).

Trusted:
- Authenticated session (JWT claim, server-verified cookie)
- Subdomain (verified against session's allowed tenants)
- Explicit tenant switch that goes through an authorization check

Not trusted:
- Query parameter (`?tenant=123`)
- Request body field
- Unverified header

Flag any code that reads tenant ID from an untrusted location.

## Cross-tenant contamination vectors

### Search

Flag:
- Full-text search indexes that don't include tenant_id in the index
- Search results filtered by tenant AFTER query (expensive + error-prone; should be in the query)
- Shared Elasticsearch / typesense / algolia instances without per-tenant index or filter contract

### Analytics / reporting

Flag:
- Analytics events missing tenant_id
- Reporting queries that aggregate across tenants (OK for internal dashboards; leak if exposed to users)
- Admin dashboards that display other tenants' data to customer-support users who shouldn't see it

### Shared caches

Flag:
- Cache keys without tenant_id
- Cache instances not partitioned per tenant (one tenant's cache fill can affect another's read)
- Memoization in long-lived processes (`const cache = {}` at module level)

### Shared background processes

Flag:
- Queue workers that process jobs for multiple tenants without isolating state
- A bug in one tenant's job crashing the worker and affecting other tenants
- Jobs enqueued for one tenant but somehow referencing another (happens via leaked context)

### Shared resources

Flag:
- Shared storage bucket without per-tenant prefixes
- Signed URLs for uploads that don't encode the tenant (user could upload to wrong tenant)
- API keys that aren't tenant-scoped (one tenant's key can access another's resources via URL manipulation)

## Permissions and roles

### Hierarchical models

Most multi-tenant SaaS has:
- Tenant (organization / workspace)
- Users within tenants
- Roles within a tenant (owner, admin, member, guest)
- Resources owned by the tenant (or by users within the tenant)

Flag:
- Role checks that don't verify the role is in the current tenant's scope (user is admin in tenant A; tries to access tenant B as an admin — should fail)
- Role check before tenant check (should check tenant access first, then role)
- Roles stored outside the tenant context (global admin flag without explicit authorization boundary)

### Cross-tenant users

Some users belong to multiple tenants. This is legitimate but complicates invariants.

Flag:
- Session without explicit "current tenant" selection
- Switching tenants without a re-auth step (at least a confirm)
- Session cached data not invalidated on tenant switch
- Permissions cached across tenant switches

## Onboarding and invitation flows

Flag:
- Invitation tokens that don't encode the target tenant (user can redirect to a different tenant)
- Invitation tokens that don't expire
- Invitation acceptance that trusts the invitee's claimed tenant_id
- Email-based invites: what if the invited email already has an account in a different tenant? Is the flow clear?

## Billing and plan limits

Flag:
- Seat count enforcement that can be bypassed (invite flow doesn't check, or check is client-side)
- Feature gating based on plan that can be bypassed via URL (user navigates to a page they shouldn't)
- Usage metering without per-tenant limits (one tenant can DoS the shared API resource)
- Prorated billing calculations with off-by-one or midnight-UTC bugs (see fintech pack for date math)

## Data export, import, and deletion

### Export

Flag:
- Tenant data export that includes other tenants' data (bug)
- Export format that leaks schema internals (IDs of other tenants in JOIN data, for example)
- Export that times out on large tenants (should be async with download link)

### Import

Flag:
- CSV import without tenant scoping (rows land in wrong tenant)
- Import that trusts IDs from the file (IDs reference parent tenant's resources)
- Import without size/row limits (resource DoS)

### Tenant deletion

Flag:
- Tenant deletion that doesn't cascade to child records (orphans)
- Tenant deletion that's synchronous and blocks on large tenants
- Deletion that removes tenant row but not data in auxiliary systems (search indexes, caches, third-party integrations like Segment, Intercom)
- No audit log entry for tenant deletion

## Admin / support access

Internal users may need to help customers by accessing their data. This is legitimate but must be tracked.

Flag:
- Support tools that don't log who accessed which tenant
- Impersonation ("Log in as customer") without audit trail and strong authentication
- Impersonation sessions that persist after support person moves on
- No ability for customer to see who has accessed their data

## Configuration and feature flags per tenant

Flag:
- Feature flags applied globally when they should be per-tenant (enterprise customers expect stability)
- Tenant-level config stored in a way that's hard to query / audit
- Secrets stored per tenant without encryption-at-rest keyed by tenant

## Tests for isolation

Flag:
- No test that verifies tenant isolation explicitly (create tenant A + tenant B; perform actions in A; assert nothing in B changed)
- No test for escalation (user of tenant A tries to access tenant B's resources via direct ID reference)
- No test for shared resources (cache, search) staying isolated

A robust suite has an "isolation test" for every endpoint that reads tenant data.

## Process

When reviewing a multi-tenant codebase:

1. First confirm the tenancy model — what's the tenant entity? How is it referenced?
2. Enumerate the tenant-scoped tables.
3. For each, grep every query against them and verify tenant scoping.
4. Walk every API endpoint and check where tenant context comes from.
5. Check cross-cutting concerns: search, cache, jobs, storage.
6. Write findings.

## Severity calibration

Multi-tenant issues lean critical/high because the blast radius is the whole customer base:

- **Critical**: cross-tenant data read possible via any unauthenticated/authenticated path; cross-tenant write possible; missing tenant scoping in a non-trivial query path
- **High**: tenant scope checked only in application layer with a gap visible; support tools without audit
- **Medium**: plan limit enforcement gaps; minor configuration per-tenant issues
- **Low**: nits in tenant onboarding copy
