---
name: full-codebase-review
description: "Exhaustively review an entire codebase — not a diff or PR, the whole thing. Use this whenever the user asks to 'review the codebase', 'audit my code', 'do a full code review', 'find bugs in my app', 'review my project end-to-end', or anything similar that implies reviewing a repository holistically. Also use when the user wants logic/math bugs found, security audits, performance audits, architecture reviews, or any mix of concerns across a whole project. Dispatches specialized reviewer subagents in parallel (architecture, security, frontend, backend, logic/math, data layer, performance, dependencies, testing, error handling, type safety, concurrency, dead code, observability, compliance) and produces a prioritized findings report. Especially powerful for domain-sensitive code — financial math, retirement calculators, wagering/odds engines, billing, and anywhere correctness bugs are expensive. Anything broader than a single-file or single-PR review, use this skill."
---

# Full Codebase Review

An orchestrated, multi-agent review of an entire codebase. The goal is finding real problems — correctness bugs, security holes, architectural issues, and logic errors in domain math — not decorating the code with style nits.

## Philosophy

**Scale depth to risk, not to line count.** A 10,000-line React app has maybe 200 lines that really matter: auth, money paths, state mutations that persist, user input handlers. Those get line-by-line. A render component gets a skim. Apply budget where it buys you bug-finding.

**Parallel, not sequential.** Dispatch specialized subagents concurrently. Each one loads only the reference file for its concern, keeping context budgets tight. They write findings to disk; the orchestrator stitches them together at the end.

**Streaming, not accumulating.** Findings get written to disk as they're discovered. Never hold the whole review in one context window — it will collapse under its own weight on a real codebase.

**Evidence-based.** Every finding must cite file path and line numbers, include a code snippet, explain the impact, and propose a concrete fix. "Consider refactoring this" is not a finding. "Line 47 of `stripe/webhooks.ts` verifies the signature after parsing the body, which breaks signature validation — here's the fix" is a finding.

**Confidence-scored.** Not every flag is right. Each finding carries a confidence score. Low-confidence findings are still reported but clearly marked so the user can triage.

## Workflow phases

Run these in order. Do not skip phases.

### Phase 0 — Scope & pre-flight

Before touching code, ask the user (if any ambiguity):

1. What's the repo path? (If they're running the skill inside a repo, use CWD.)
2. Any paths to exclude? (common: `node_modules`, `dist`, `.next`, vendored deps, generated code)
3. What kind of software is this? If they don't know or it's obvious, you'll detect it in Phase 1.
4. Are there specific concerns they already have? (e.g., "I'm worried about the payout math", "I think there's a race condition in the escrow flow")
5. Budget guidance: is this a quick pass (1–2 hours of agent-time) or a deep audit (many hours)?

If the user has already given a clear directive ("just review everything, go deep"), skip the interview.

Create a workspace directory: `./.codebase-review-<YYYYMMDD-HHMM>/` at the repo root. All artifacts go here. Structure:

```
.codebase-review-<timestamp>/
├── inventory.md              # What the repo contains
├── stack.md                  # Detected stack + domain
├── plan.md                   # Triage: what gets deep vs skim review
├── findings/                 # One file per agent, findings as JSON lines
│   ├── architecture.jsonl
│   ├── security.jsonl
│   ├── logic-math.jsonl
│   └── ...
├── agent-logs/               # Raw subagent output for debugging
└── REPORT.md                 # Final stitched report
```

### Phase 1 — Inventory

Read `references/inventory.md` for the full procedure. The short version:

- Walk the repo tree. Note: root, top-level directories, file counts, dominant languages.
- Read `package.json`, `requirements.txt`, `Cargo.toml`, `go.mod`, etc. — detect the stack.
- Read any `README.md`, `CLAUDE.md`, `ARCHITECTURE.md` — capture the author's own framing.
- Identify entry points (main, index, app, server, API route handlers).
- Identify critical paths: files touching auth, payments, persistence, external APIs.
- Identify the domain: is this retirement planning, a wagering platform, a SaaS CRM, an e-commerce site? Domain drives which specialized checks run.

Write `inventory.md` and `stack.md` with your findings.

### Phase 2 — Triage

Read `references/triage.md` for the full procedure. Produce `plan.md` that specifies, for each agent, which files get deep treatment vs skim. Rationale matters: a future reviewer should be able to read `plan.md` and understand why attention was allocated this way.

**Deep treatment** = line-by-line review, read every function, trace data flow, check invariants.

**Skim treatment** = structural read, flag obvious issues, don't go line-by-line unless something jumps out.

Hard rules for what always gets deep treatment:
- Anything handling money (charges, refunds, payouts, balances, escrow)
- Authentication and authorization flows
- Session/token management
- Cryptographic code
- Webhook handlers (especially payment webhooks)
- Database migrations
- Anything with "admin", "internal", or elevated privileges
- Domain-core math (retirement projections, odds calculations, rate tables)
- Data export / deletion (GDPR / CCPA paths)

### Phase 3 — Dispatch agents in parallel

The core of the review. Spawn the agents listed below as parallel subagents using the Task tool. Each subagent:

1. Reads its reference file from `references/agents/<name>.md`.
2. Reads the triage plan and inventory.
3. Performs its specialized review.
4. Writes findings to `.codebase-review-<ts>/findings/<name>.jsonl` (one JSON finding per line).
5. Writes a summary to `.codebase-review-<ts>/agent-logs/<name>.md`.

**Standard agents (always run):**

| Agent | Reference file | Focus |
|-------|---------------|-------|
| architecture | `references/agents/architecture.md` | Module boundaries, layering, coupling, cohesion, abstraction levels |
| security | `references/agents/security.md` | OWASP Top 10, auth, secrets, input validation, injection, CSRF, SSRF |
| frontend | `references/agents/frontend.md` | React/Next.js correctness, hydration, accessibility, state, re-renders |
| backend | `references/agents/backend.md` | API design, error handling, idempotency, input validation, transactions |
| logic-math | `references/agents/logic-math.md` | **Correctness bugs in domain math.** Rounding, off-by-one, boundaries, unit confusion |
| data-layer | `references/agents/data-layer.md` | Schema, indexes, N+1, query patterns, migrations, integrity |
| performance | `references/agents/performance.md` | Algorithmic complexity, hot paths, caching, bundle size, memory |
| dependencies | `references/agents/dependencies.md` | CVEs, outdated deps, unused deps, license issues, supply chain |
| testing | `references/agents/testing.md` | Coverage gaps, test quality, missing edge cases, flaky patterns |
| error-handling | `references/agents/error-handling.md` | Resilience, retries, timeouts, swallowed errors, failure modes |
| type-safety | `references/agents/type-safety.md` | TS strictness, `any` leaks, runtime validation, type assertions |
| concurrency | `references/agents/concurrency.md` | Race conditions, deadlocks, order-of-operations, atomicity |
| dead-code | `references/agents/dead-code.md` | Unused exports, orphaned files, stale TODOs, duplication |
| observability | `references/agents/observability.md` | Logs, metrics, traces, debuggability, alertability |
| compliance | `references/agents/compliance.md` | PCI/HIPAA/GDPR/CCPA/SOC2 as relevant |

**Domain agents (conditionally run based on Phase 1 detection):**

| Detected domain | Load |
|----------------|------|
| Financial / retirement / tax software | `references/domains/fintech.md` |
| Wagering / gaming / odds / sportsbook | `references/domains/gaming-wagering.md` |
| Multi-tenant SaaS | `references/domains/saas-multitenant.md` |
| IoT / telemetry / GPS / time-series | `references/domains/iot-telemetry.md` |
| Marketplace / two-sided / payouts | `references/domains/marketplace.md` |
| Healthcare / PHI | `references/domains/healthcare.md` |

Domain packs extend the `logic-math` agent's prompt with domain-specific checks. For example, if this is retirement software, the fintech pack adds checks for: compounding formula correctness, tax bracket boundary handling, SECURE 2.0 catch-up contribution rules, Monte Carlo assumption documentation, real-vs-nominal consistency, and sequence-of-returns risk modeling.

### Phase 4 — Synthesis

Read all `findings/*.jsonl` files. Read `references/report-format.md` for the full synthesis procedure. Produce `REPORT.md` with:

1. Executive summary (3–5 bullets, non-technical)
2. Critical findings (must-fix, with reproductions where possible)
3. Findings grouped by severity: Critical, High, Medium, Low, Nit
4. Findings grouped by agent (so the user can focus on one concern at a time)
5. Findings grouped by file (so the user can fix one file at a time)
6. Prioritized remediation plan (ordered by impact/effort)
7. Observability gaps (what the user can't currently see going wrong in production)
8. Appendix: methodology, agents that ran, files deep-reviewed vs skimmed

Deduplicate findings across agents — two agents may find the same issue from different angles; merge them and credit both.

### Phase 5 — Present

Present `REPORT.md` to the user. Lead with the critical findings, not the methodology. If there are zero criticals, say so plainly. Offer to:
- Walk through any finding in detail
- Generate patches for specific findings
- Create tickets / a TODO file the user can work through
- Re-run individual agents with more depth

## Finding schema

Every finding written to `findings/*.jsonl` conforms to this schema:

```json
{
  "id": "AGENT-NNNN",
  "agent": "logic-math",
  "severity": "critical | high | medium | low | nit",
  "confidence": 0.0-1.0,
  "title": "One-line summary",
  "file": "relative/path/from/repo/root.ts",
  "line_start": 42,
  "line_end": 58,
  "code_snippet": "the exact lines quoted from the file",
  "description": "What the bug is, in plain language",
  "impact": "What happens in production if this ships",
  "reproduction": "How to trigger the bug, if applicable",
  "remediation": "Concrete fix — code if possible, otherwise direction",
  "references": ["CWE-89", "OWASP-A03", "RFC-7519 §4.1", etc],
  "tags": ["money", "auth", "race-condition"]
}
```

**Severity rubric:**

- **Critical** — exploitable security hole, data loss, money lost/stolen, crash on common path, legal/compliance violation. Fix before shipping anything else.
- **High** — bug that will hit real users in common scenarios; serious architectural issue that compounds over time.
- **Medium** — bug in uncommon path, noticeable correctness issue, significant tech debt.
- **Low** — minor bug, small optimization, style issue with real downstream impact.
- **Nit** — style, naming, minor cleanup. Collapse nits by category in the final report; don't list each individually unless asked.

**Confidence rubric:**

- **0.9–1.0** — definitely a bug, I have the reproduction in hand.
- **0.7–0.9** — strong evidence this is a bug, but I haven't run the code.
- **0.5–0.7** — this looks wrong, could be intentional.
- **0.3–0.5** — smells off. Worth a human eye.
- **Below 0.3** — don't report.

## What this skill is not

- **Not a linter.** ESLint / ruff / clippy already exist. Don't duplicate them.
- **Not a style guide enforcer.** Formatting, naming conventions, and opinion-based preferences belong in CI config, not a review.
- **Not a refactor bot.** Findings propose fixes; the user decides what to apply.
- **Not a rubber stamp.** If a codebase is solid, say so. Don't invent findings to look thorough.

## Running modes

The user can invoke the skill three ways:

- **`full`** — all agents, all phases, domain packs activated. The default.
- **`focused:<agent>`** — run only the named agent (e.g., `focused:security`, `focused:logic-math`). Skip synthesis unless multiple `focused:` are stacked.
- **`quick`** — skip deep triage, dispatch a lean subset (security, logic-math, critical-path correctness only). Good for a sanity check.

If the user's invocation is ambiguous, default to `full`.

## Reference files

Read these as you progress through the phases. Don't load them all upfront.

- `references/inventory.md` — Phase 1 procedure
- `references/triage.md` — Phase 2 procedure
- `references/report-format.md` — Phase 4 procedure
- `references/agents/*.md` — one per agent; loaded by the subagent itself, not the orchestrator
- `references/domains/*.md` — domain-specific extensions for `logic-math`
- `templates/finding.md` — the finding schema with worked examples
- `templates/report.md` — final report structure

Good hunting.
