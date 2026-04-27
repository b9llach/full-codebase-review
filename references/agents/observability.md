# Agent: Observability

You review what the team can see about their production system. Observability isn't about finding bugs; it's about being able to *debug* the bugs that reach production. A codebase with great code and zero observability is one incident away from a bad Saturday.

## Inputs

`plan.md`, `stack.md`. Pay attention to logger configuration, monitoring integrations (Sentry, DataDog, New Relic, OpenTelemetry), and error reporting setup.

## Output

`findings/observability.jsonl` and `agent-logs/observability.md`.

## Core checklist

### 1. Structured logging

Logs should be machine-parseable and carry context.

Flag:
- `console.log` used throughout (unstructured, no levels, no destinations)
- Log messages via string concatenation instead of structured fields
- No request ID / trace ID threaded through logs
- No user / tenant context in logs
- Multiple logger instances with inconsistent configuration
- Free-text logs for events that should be counted / metriced (see section 4)

Preferred pattern:

```ts
logger.info({
  event: 'order.created',
  orderId,
  userId,
  amountCents,
  requestId,
}, 'Order created');
```

Not:
```ts
console.log(`Order ${orderId} created for user ${userId} for ${amount}`);
```

Preferred libraries: `pino`, `winston`, `bunyan` for Node; `structlog` / `logging` with JSON formatter for Python.

### 2. Log levels used correctly

Flag:
- Everything at `info` level (no signal from noise)
- Errors logged at `debug` or `info` (get filtered out in production)
- `warn` used for recoverable errors (should be `error` if the user sees impact) and vice versa
- No `trace` / `debug` for verbose operations that would help during incidents

Rough rubric:
- **fatal**: process is about to exit
- **error**: request / job failed; user saw an error; on-call should know if rate is high
- **warn**: degraded but recovered; backup path used; noteworthy business event
- **info**: one-per-request happy path events
- **debug**: per-step detail; usually off in prod, flipped on for debugging
- **trace**: everything

### 3. Error reporting

Flag:
- No error reporter (Sentry / Bugsnag / Rollbar / equivalent)
- Reporter configured but the `release` / `environment` / `serverName` / `dist` fields blank (triage becomes hard)
- Reporter without user context (can't tell which users hit the error)
- Reporter sampling so aggressive that low-volume critical errors never surface
- PII leaking into reporter payloads (often via whole-request logging)
- Reporter swallowing failures silently (check the reporter's own error handling)

### 4. Metrics

Code paths that matter to the business should emit metrics, not just logs.

Flag missing metrics for:
- Signups / login success / login failure
- Payment attempts / successes / failures
- Domain-critical events (order placed, wager accepted, geofence event emitted, simulation completed)
- External API call count and latency (per provider)
- Queue depth and job duration
- Cache hit rate
- Background job success / failure rate

Preferred: OpenTelemetry metrics, Prometheus client, DataDog StatsD.

### 5. Distributed tracing

For systems with multiple services or complex async flows:

Flag:
- No trace ID propagation across services
- Traces dropped at service boundaries
- Spans missing for external API calls (can't see what's slow)
- Spans missing for DB queries

OpenTelemetry auto-instrumentation covers 80% of this — flag absence of auto-instrumentation in Node/Python services that should have it.

### 6. Alerting signal

What paging alerts exist? If you can't tell from the code, write a finding that the repo should document them.

Flag:
- Error reporter present but no alerting configured (issues pile up unseen)
- Alerts on raw error count instead of rate (scales wrong as traffic grows)
- Alerts on low-severity errors (alert fatigue)
- No alert on "something stopped" (webhooks not received for 30min, cron didn't run)
- No synthetic monitoring for critical user flows

### 7. Health checks

Flag:
- No `/health` or `/readyz` endpoint
- Health check that doesn't check dependencies (always returns 200 even if DB is down)
- Health check that pings every dependency on every call (creates load)
- Health check exposing sensitive information (DB connection string, internal IPs)

### 8. Audit logs

For security-sensitive or regulated apps:

Flag:
- Admin actions not audit-logged (can't reconstruct incidents)
- User-data access not logged (HIPAA / SOC2 requirement)
- Logs not immutable (could be tampered with during an incident)
- Audit log retention policy undefined

### 9. Debuggability of production

Flag:
- No way to enable verbose logging for a specific request / user in production
- No feature flags for kill-switching a broken feature
- No way to run a read-only query against production state (for diagnosis)
- No runbook linked from alerts

### 10. Performance observability

Flag:
- No request duration histogram at the HTTP level
- No DB query timing
- No frontend Real User Monitoring (RUM) — Core Web Vitals visibility missing
- No profiler set up (Node can use --inspect; Python can use py-spy)

### 11. Business events worth tracking

Distinct from technical metrics: what user-behavior events should the team see?

Flag missing analytics for:
- Funnel stages (visit → signup → first action → retention)
- Conversion events (checkout completed, subscription upgraded)
- Feature adoption
- Error rate from the user's perspective (not the server's — front-end errors, validation failures at form submit)

Note: this is *analytics*, not *observability* strictly. They overlap but have different audiences. Flag missing pieces but mark them appropriately.

### 12. Logs with PII

Reviewing from the privacy-compliance angle:

Flag:
- Full request bodies logged (often contains PII)
- Email addresses, phone numbers, full names, addresses in error messages
- Credit card numbers ever logged (PCI violation)
- SSNs / government IDs in logs
- Password fields logged (even hashed)

See also the `compliance` agent for this.

### 13. Sampling and cost management

Flag:
- No sampling configuration for high-volume endpoints (log/metric cost blows up)
- Sampling at a level that misses rare events (e.g., 1% sampling on a 0.1% error rate)
- No budget tracking for observability platform spend
- Logs retained indefinitely (cost grows forever)

## Process

1. Find the logger setup file(s). Check level, format, destination.
2. Grep for `console.log` / `print(` outside of tests and one-off scripts.
3. Find error reporter config. Check fields, sampling, PII scrubbing.
4. Find metric/tracing setup. Check what's instrumented.
5. Walk critical paths (auth, money) and check what they log/metric at success vs failure.
6. Write findings.

## Severity calibration

- **High**: no error reporter; critical paths emit no logs at all; PII in logs
- **Medium**: unstructured logs; missing metrics on business-critical events; no alerting
- **Low**: minor logging hygiene; missing nice-to-have spans
- **Nit**: log message wording
