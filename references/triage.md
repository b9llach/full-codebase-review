# Phase 2: Triage

The purpose of triage is to allocate attention. On a real codebase you cannot line-by-line every file — you'd burn context and still run out before finding the actual bugs. Triage is the decision about *where* to go deep.

## Principle

**The code that matters is the code where bugs cost real money or real trust.** Everything else gets a skim. Be explicit about this allocation so a future reviewer can argue with the call.

## Tiering

Every source file falls into one of four tiers:

### Tier 0 — Deep line-by-line (every line read, every function traced)

Automatic inclusion:
- Money paths: charges, refunds, payouts, balance mutations, escrow, wagers
- Authentication: login, signup, session, JWT creation/verification, password reset, OAuth callbacks, SAML
- Authorization: middleware, permission checks, role checks, tenant isolation
- Cryptography: signing, HMAC verification, encryption, key derivation
- Webhook handlers that mutate state (payment webhooks especially)
- Database migrations (each migration reviewed on its own, plus the cumulative shape)
- Domain-core math: retirement projections, odds engines, tax calculations, pricing, rate tables
- Admin endpoints and internal tools
- Data export / deletion (GDPR / CCPA / account deletion)
- File upload handlers (type validation, size limits, storage destination)
- Any file explicitly flagged by the user

### Tier 1 — Thorough review (every function skimmed, complex logic read in full)

- API handlers that don't move money or change auth state
- Business logic modules (non-money)
- Background jobs / queue consumers
- External API integrations (outbound calls that aren't money)
- Complex utilities (date math, string parsers, number formatters)
- Cache layers and invalidation logic
- Feature flags / experiment frameworks (controlling production behavior)

### Tier 2 — Skim (structural read, flag obvious issues)

- Standard CRUD handlers
- UI components with non-trivial logic
- Forms with client-side validation
- Standard React hooks
- Configuration modules
- Email / notification sending

### Tier 3 — Structural only (presence check, no line-by-line)

- Pure presentational components (render only)
- Static pages
- Design system primitives
- Test fixtures and mocks
- Generated code (flag if it's being hand-edited; otherwise skip)
- Vendored code
- Documentation files

## Procedure

1. For every source file identified in Phase 1, assign a tier.
2. Produce a count: how many files at each tier?
3. If Tier 0 is more than ~50 files on a repo of typical size, flag this — it's a sign the app has sprawling money/auth surface area, which is itself a finding.
4. For each **agent**, decide which tiers it examines:

| Agent | Tier 0 | Tier 1 | Tier 2 | Tier 3 |
|-------|--------|--------|--------|--------|
| architecture | skim | skim | sample | presence only |
| security | deep | deep | targeted skim | no |
| frontend | N/A | deep (FE tier 1) | deep (FE tier 2) | skim |
| backend | deep | deep | skim | no |
| logic-math | **exhaustive** | deep on math-touching files | no | no |
| data-layer | deep (migrations, models) | deep (query code) | skim | no |
| performance | targeted (hot paths) | targeted | no | no |
| dependencies | N/A (lockfile analysis) | — | — | — |
| testing | deep (does this tier have tests?) | deep | sample | no |
| error-handling | deep | deep | skim | no |
| type-safety | deep | deep | skim | no |
| concurrency | deep (any shared state) | targeted | no | no |
| dead-code | full-repo static analysis | — | — | — |
| observability | deep (what's logged/traced) | skim | no | no |
| compliance | targeted (PII/PHI touchers) | targeted | no | no |

"N/A" = agent doesn't care about that tier.

## The `logic-math` special case

This agent gets extra rigour because logic bugs are the most expensive bugs to find after they ship. For Tier 0 files that include numerical calculation:

- Read every arithmetic line
- Check operator precedence (people write `a + b * c` meaning `(a + b) * c` constantly)
- Check unit consistency (ms vs s, cents vs dollars, percent vs decimal)
- Trace the type of every numeric variable from input to output
- Check boundary behavior: what happens at 0? 1? negative? very large? NaN? Infinity?
- Check rounding: where does it happen, which direction, does it favor one party?
- If money: is it stored and computed as integer cents or decimal? Never float.
- If dates: timezone, DST, leap day, leap second, year boundaries
- If compound formulas: verify against a known-good reference (Wikipedia, textbook, spec)
- If Monte Carlo: number of simulations, assumption documentation, random seed handling

Load the relevant domain pack from `references/domains/` for an extended checklist.

## Write the plan

Output to `.codebase-review-<ts>/plan.md`:

```markdown
# Triage plan

## File tiering

### Tier 0 — Deep line-by-line (N files)
- `app/api/stripe/webhook/route.ts` — Stripe webhook handler, mutates balance
- `lib/auth/session.ts` — session creation and validation
- `lib/retirement/monte-carlo.ts` — core simulation math
- ...

### Tier 1 — Thorough (N files)
- `lib/email/templates/*.ts`
- `app/api/users/route.ts`
- ...

### Tier 2 — Skim (N files)
[list or "all of <directory> except Tier 0/1"]

### Tier 3 — Structural only (N files)
[typically: all of components/ui, pages that are static, etc.]

## Agent assignments

For each agent, which tier gets what treatment. Paste the table above, adjusted for this codebase.

## Special notes
- Domain pack: fintech (retirement) — logic-math agent runs extended Monte Carlo / tax bracket / compounding checks
- User specifically flagged: retirement projection math in `lib/retirement/`
- CI already enforces: prettier, ESLint → don't duplicate these in findings
```

With `plan.md` written, proceed to Phase 3 (dispatch agents).
