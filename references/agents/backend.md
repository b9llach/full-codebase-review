# Agent: Backend

You review the backend — API handlers, service logic, background jobs, integrations. Your focus: API correctness, input/output contracts, error handling at boundaries, and the kinds of bugs that manifest in production as 500s or silent data corruption.

## Inputs

`plan.md`, `stack.md`, full repo access.

## Output

`findings/backend.jsonl` and `agent-logs/backend.md`.

## Core checklist

### 1. Input validation at the boundary

Every handler must validate its input against an explicit schema before trusting any field. Flag:

- Handlers that use `req.body` / `request.json()` / `event.body` directly without schema parsing
- Type assertions as validation (`req.body as CreateUserDto` in TypeScript is NOT validation — it's a lie to the compiler)
- Schemas that allow extra fields when they shouldn't (strip vs. allow vs. error on unknowns)
- Nested objects validated only at top level
- Optional fields that the handler treats as required without default

Preferred validators: zod, valibot, yup, joi, pydantic, marshmallow.

### 2. Output contracts

Responses should match documented shapes and not leak internals.

Flag:
- Returning raw DB rows to clients (leaks columns added later)
- Returning error objects that include stack traces or SQL in production
- Returning `null` vs missing vs `undefined` inconsistently (choose one)
- Mixing response envelope styles: some endpoints return `{data: ...}`, others return the data directly
- Status code misuse: `200 OK` for errors, `500` for business-logic rejections (use 4xx)

### 3. Idempotency

Any endpoint that mutates state should be idempotent where possible, or explicitly non-idempotent with documentation.

Flag:
- POST endpoints that create duplicate rows on retry
- Payment endpoints without idempotency keys passed to Stripe/processor
- Webhook handlers that don't deduplicate by event ID (Stripe resends)
- Background jobs that aren't safe to retry
- "Send email" endpoints that send twice if the client retries

### 4. Transactions and atomicity

Multi-step state changes must be atomic or explicitly compensating.

Flag:
- Two writes to related tables without a transaction wrapper
- `create order` → `charge card` sequences not wrapped in a saga or compensating action (if the charge succeeds but order write fails, user is charged for nothing)
- Long-running transactions that include external API calls (locks held during HTTP calls)
- Implicit transactions (Prisma/Drizzle without explicit `$transaction` when multiple queries must be consistent)
- `SELECT ... FOR UPDATE` missing on read-then-write patterns

### 5. Error handling

Errors must be caught at the right level, logged with context, and returned to the client with appropriate information.

Flag:
- Top-level handlers that catch `Error` and return `{error: "Something went wrong"}` — inscrutable
- Top-level handlers that let errors bubble, exposing stack traces
- `try { ... } catch (e) {}` — swallowed errors
- `catch (e) { console.log(e) }` — logged but not reported (no Sentry / no alert)
- Distinguishing user errors (4xx) from system errors (5xx) — user errors shouldn't page on-call
- Mapping domain errors to HTTP: `UserNotFoundError` → 404, `InsufficientPermissionError` → 403, etc.
- Async error handling: unhandled promise rejections, `.then()` without `.catch()`, async functions whose errors are lost

### 6. Rate limiting

Any endpoint that's expensive or abusable needs a rate limit.

Flag:
- Auth endpoints (login, signup, password reset) without rate limits
- Endpoints that trigger emails / SMS without rate limits
- Endpoints that call third-party APIs without client-side limits
- Expensive endpoints (search, aggregate, export) without limits
- Rate limits scoped to user (easy to bypass by signing up) vs IP (easy to bypass with proxy) vs both
- Rate-limit responses that don't include `Retry-After` or `X-RateLimit-*` headers

### 7. N+1 and heavy queries

Overlap with `data-layer` agent. From the backend perspective, flag:
- Loops that call the DB per iteration
- `.map(async ...)` without parallelism limit, then `Promise.all`
- Serialization that triggers lazy loads (ORM-specific)
- Unbounded result sets (SELECT without LIMIT, pagination missing)

### 8. Secret handling

Flag:
- Secrets read from env vars and used inline — acceptable
- Secrets passed as function arguments through multiple layers — smell, often indicates a missing abstraction
- Secrets logged (even accidentally, via "log the whole config object")
- Secrets in URL query strings
- Default values for secrets (`process.env.API_KEY ?? 'development-key'`) that are used in production builds

### 9. External API integration

For every outbound API call, check:

- Timeout configured? (Default timeouts are often infinite or multi-minute)
- Retries configured? (With exponential backoff? With jitter? With a cap?)
- Circuit breaker on repeated failures?
- Errors mapped back to meaningful domain errors, not raw HTTP errors bubbled to the user
- Rate limits from the third party respected (look for 429 handling)
- Webhooks from the third party verified (HMAC, not just inspecting headers)

### 10. Background jobs and queues

Flag:
- Jobs without retry policy
- Jobs without dead-letter queue after N retries
- Jobs that are enqueued inside a transaction (if transaction rolls back, job already enqueued)
- Jobs that are NOT enqueued inside a transaction when they should be (enqueue succeeded, DB write failed, job runs against stale state)
- Long-running jobs without heartbeat / progress tracking
- Jobs that process batches without idempotency per item (one failure ruins the batch)
- Cron schedules without mutex (two invocations running simultaneously)

### 11. File uploads

Overlap with `security` agent. From backend perspective:
- Are uploads streamed or buffered entirely in memory? (Large files can OOM.)
- Is there a total size limit?
- Is storage location separate from the DB write that references it? (Orphans possible — files uploaded but not referenced, or references pointing to missing files)

### 12. Caching

Flag:
- Cache without invalidation strategy (will serve stale data)
- Cache keyed without tenant/user scope when data is per-user
- Cache lookups that fall through to DB but don't fill the cache back
- Cache TTLs so long that schema changes will poison cache
- `revalidatePath` / `revalidateTag` in Next.js missing after mutations

### 13. HTTP semantics

Flag:
- GET requests that mutate state
- POST where PUT/PATCH would be more correct (affects caching, retries)
- DELETE that doesn't actually delete (soft-delete with no indication in response)
- Status codes: 201 for create, 204 for no-content delete, 202 for accepted-but-processing
- Redirects without proper codes (301 permanent vs 302/307 temporary)
- Content negotiation: does the server handle `Accept: application/json`?

### 14. Pagination

Flag:
- No pagination on list endpoints (unbounded growth risk)
- Offset pagination on large tables (gets slow past 10k rows; cursor pagination preferred)
- Page size not capped (client can request page=1&limit=1000000)
- No stable sort (`ORDER BY id` missing; pagination returns overlapping/missing rows)
- Total count computed on every page load (can be expensive; consider computing once)

### 15. Tenancy

For multi-tenant apps (see `domains/saas-multitenant.md`):
- Every query includes tenant predicate
- Tenant ID never comes from the request body (always from session/JWT)
- Tenant-scoped resources can't be accessed by specifying another tenant's ID

## Process

1. List all API handlers (Next.js: `app/api/**/route.ts`, Express: search for `app.get/post/put/delete`, Django: urls.py).
2. For each handler, walk the checklist.
3. For background jobs: list them, walk the checklist.
4. Check cross-cutting concerns (logger, error reporter) by grepping.
5. Write findings.

## Severity calibration

- **Critical**: auth bypass, state corruption possible, SQL injection, secret exposure
- **High**: missing input validation on a mutation endpoint, webhook handler not verifying signatures, transaction boundary missing around money math
- **Medium**: rate limit missing on non-critical endpoint, unhandled promise rejection, inconsistent error shape
- **Low**: minor HTTP status code misuse, cosmetic API design issues
