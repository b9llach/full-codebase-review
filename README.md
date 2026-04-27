# full-codebase-review

A [Claude Code](https://claude.com/claude-code) skill that reviews your **entire codebase** — not a diff, not a PR, the whole repo — and produces a prioritized findings report.

It dispatches fifteen specialized reviewer agents in parallel (security, logic/math, data-layer, concurrency, performance, and more), auto-loads additional checks when the codebase matches a known domain, and writes findings to disk as it goes so it can review codebases far larger than fit in a single context window.

Built for the situations PR-scoped review tools can't handle: end-to-end sanity checks before a release, pre-launch hardening passes, audits of inherited or unfamiliar codebases, and any time you want a holistic answer to "what's actually wrong with this repo" rather than a per-diff opinion.

## Quick start

```bash
git clone https://github.com/<you>/full-codebase-review ~/.claude/skills/full-codebase-review
```

Then in any Claude Code session at the root of a repo:

```
Please run a full codebase review.
```

That's it. The skill auto-activates on phrases like "review the codebase", "audit my code", "find bugs in my app", etc.

## What you get

The skill creates `.codebase-review-<timestamp>/` at the repo root containing:

| File | What it is |
|------|-----------|
| `REPORT.md` | **The deliverable.** Executive summary → critical findings in full → high/medium/low → findings grouped by file → prioritized remediation plan. |
| `inventory.md` | What's in the repo, stack detected |
| `stack.md` | Frameworks, languages, applicable domain packs |
| `plan.md` | Triage: which files got deep review vs skim |
| `findings/*.jsonl` | Raw findings, one JSON line per finding, per agent |
| `agent-logs/*.md` | Per-agent summary of what they looked at |

Every finding includes a file path, line range, code snippet, impact statement, reproduction steps, and a concrete fix.

## What it reviews

### Standing agents (always run)

| Agent | Focus |
|-------|-------|
| **architecture** | Module boundaries, dependency direction, coupling/cohesion |
| **security** | OWASP Top 10, secrets, input validation, webhooks, file uploads |
| **frontend** | React correctness, Next.js App Router, hydration, a11y, forms |
| **backend** | API design, transactions, idempotency, rate limiting, error shapes |
| **logic-math** | *The main bug-finder.* Money representation, rounding, units, boundaries, NaN propagation, date math, compound formulas |
| **data-layer** | Schema, indexes, migrations, multi-tenancy, transactions, integrity |
| **performance** | Complexity, caching, bundle size, hot paths, memory |
| **dependencies** | CVEs, version pinning, unused deps, license compliance |
| **testing** | Coverage gaps on critical paths, test quality, missing edge cases |
| **error-handling** | Swallowed errors, retries, timeouts, circuit breakers |
| **type-safety** | Strictness, `any` leaks, runtime validation at boundaries |
| **concurrency** | Race conditions, distributed locks, queue idempotency |
| **dead-code** | Unused exports, orphaned files, stale TODOs, duplication |
| **observability** | Structured logging, metrics, alerting, audit trails, PII-in-logs |
| **compliance** | GDPR / CCPA / HIPAA / PCI / SEC where relevant |

### Domain packs (auto-activated)

| Pack | Activates when |
|------|---------------|
| **fintech** | Stack handles money, retirement projections, tax math, loans, brokerage |
| **gaming-wagering** | Odds, escrow, settlement, KYC/AML, geolocation, tax reporting |
| **saas-multitenant** | Tenant isolation at the query layer, role hierarchies, billing limits |
| **iot-telemetry** | Device data, geospatial correctness, GPS quality, time-series ingest |
| **marketplace** | Commission/split math, payouts, disputes, sales tax, ratings |
| **healthcare** | PHI handling, HIPAA audit trails, clinical correctness, BAAs |

## Run modes

```
Run a full review of this codebase. Pay extra attention to the retirement math.
→ full mode, all agents, domain pack auto-loaded

Run just the security and logic-math agents on this repo.
→ focused mode

Do a quick review — security and critical-path bugs only.
→ quick mode (lean subset, faster)
```

## Design principles

- **Scale depth to risk.** Money paths, auth code, and domain math get line-by-line. Render components get a skim. Budget goes where bugs hide.
- **Parallel, not sequential.** Agents run concurrently in subagents with tight context budgets.
- **Stream findings.** Agents write to disk as they work — the orchestrator never holds the whole review in one context window. Reviews codebases that would otherwise overflow.
- **Evidence-based.** Every finding has file:line, code, impact, repro, fix.
- **Confidence-scored.** Not every flag is right. Each finding carries a confidence; low-confidence flags are reported but marked.

## What it isn't

- Not a linter. ESLint, ruff, clippy exist.
- Not a style enforcer.
- Not a refactor bot.
- Not a rubber stamp. If the code is good, it says so.

## Requirements

- [Claude Code](https://claude.com/claude-code) (skills work in Claude Code, not on the web app)
- Bash; `tree` and `cloc` are preferred but not required (skill has fallbacks)
- Optional, for deeper analysis: `madge`, `ts-prune`, `knip`, `depcheck`, `jscpd`, `npm audit`, `pip-audit`, `cargo audit`

## Known limitations

- **Static review only** — the skill doesn't execute tests or run the code.
- **Performance findings are shape-based, not measured.** For hot-path confirmation, run load tests.
- **Domain correctness relies on reference material.** A subject-matter expert may catch edge cases the skill misses.
- **Very large monorepos** may need scoping — the skill works best at one logical product at a time.

## Customizing

- **Add a new agent:** create `references/agents/<name>.md` following the existing patterns, then add it to the dispatch list in `SKILL.md`.
- **Add a new domain pack:** create `references/domains/<name>.md` and add the detection signal to `references/inventory.md` Step 5.
- **Tighten review for your stack:** add a `CLAUDE.md` at your repo root listing what matters most; the orchestrator reads it in Phase 1.

## Repo layout

```
full-codebase-review/
├── SKILL.md                        # Orchestrator — Claude reads this first
├── README.md                       # You are here
├── references/
│   ├── inventory.md                # Phase 1: stack & domain detection
│   ├── triage.md                   # Phase 2: which files get deep review
│   ├── report-format.md            # Phase 4: synthesis rules
│   ├── agents/                     # 15 standing reviewer agents
│   └── domains/                    # 6 domain packs
└── templates/
    ├── finding.md                  # JSONL schema for findings
    └── report.md                   # Final REPORT.md template
```

## License

MIT. Adapt freely.
