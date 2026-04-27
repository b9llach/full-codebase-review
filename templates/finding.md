# Finding Schema

Every finding written to `findings/<agent>.jsonl` conforms to this schema. One JSON object per line.

## Schema

```json
{
  "id": "AGENT-NNNN",
  "agent": "logic-math",
  "severity": "critical",
  "confidence": 0.95,
  "title": "One-line summary (< 100 chars)",
  "file": "relative/path/from/repo/root.ts",
  "line_start": 42,
  "line_end": 58,
  "code_snippet": "the exact lines from the file",
  "description": "What the bug is, in plain language. 1-3 paragraphs.",
  "impact": "What happens in production if this ships. Be specific.",
  "reproduction": "How to trigger the bug. Can be null for static-analysis findings.",
  "remediation": "Concrete fix. Include code if the fix fits in < 20 lines.",
  "references": ["CWE-89", "OWASP-A03"],
  "tags": ["money", "auth", "race-condition"]
}
```

## Field notes

- **`id`**: Agent prefix + zero-padded sequence. `LM-0001` for the first logic-math finding, `SEC-0042` for the 42nd security finding.
- **`agent`**: Matches the agent name. Used for grouping and filtering.
- **`severity`**: One of: `critical`, `high`, `medium`, `low`, `nit`. See SKILL.md for the rubric.
- **`confidence`**: 0.0 to 1.0. If below 0.3, don't report.
- **`title`**: Scannable. This is what shows in the summary tables.
- **`file`**: Relative to repo root. No leading slash.
- **`line_start`, `line_end`**: Inclusive. For a single-line finding, they're equal.
- **`code_snippet`**: Exact quote from the file. If the finding spans > 30 lines, quote the most relevant portion and note the full range in description.
- **`description`**: Explain the bug. Don't repeat the title. Do explain the mechanism — why is this wrong?
- **`impact`**: Production-facing consequences. "Causes incorrect retirement projections for users over 60" is good. "Could be a bug" is not.
- **`reproduction`**: Step-by-step, if possible. For "this function returns the wrong value when X" findings, give the X. Can be omitted for structural findings.
- **`remediation`**: What to do. Prefer code patches when they're small. For larger fixes, give direction.
- **`references`**: CVEs, CWEs, OWASP IDs, RFC sections, textbook pages, links to authoritative sources. Helps the user learn, not just fix.
- **`tags`**: Free-form labels. Useful for aggregation. Common tags: `money`, `auth`, `pii`, `race-condition`, `n+1`, `accessibility`, `typo-level`.

## Worked examples

### Example 1: Critical — webhook signature bypass

```json
{
  "id": "SEC-0001",
  "agent": "security",
  "severity": "critical",
  "confidence": 0.95,
  "title": "Stripe webhook verifies signature against parsed body instead of raw body",
  "file": "app/api/stripe/webhook/route.ts",
  "line_start": 8,
  "line_end": 22,
  "code_snippet": "export async function POST(req: Request) {\n  const body = await req.json();\n  const sig = req.headers.get('stripe-signature');\n  const event = stripe.webhooks.constructEvent(\n    JSON.stringify(body),\n    sig,\n    process.env.STRIPE_WEBHOOK_SECRET\n  );\n  // ...",
  "description": "The Stripe webhook handler parses the request body with `req.json()`, then re-stringifies the parsed object and passes that to `stripe.webhooks.constructEvent`. Stripe signatures are computed over the raw request body, so any change to whitespace, key ordering, or numeric representation (e.g., `1` becoming `1.0` via JSON.stringify) will cause signature verification to fail in ways the attacker can exploit — or, worse, if the library is lenient, to succeed against forged payloads.\n\nThis is a known anti-pattern called out in Stripe's docs.",
  "impact": "Legitimate webhooks will intermittently fail verification, leading to missed events (payment confirmations lost). More importantly, an attacker who obtains the raw webhook secret (or who can exploit the parsing differences) could forge events to mark orders as paid, trigger payouts, or confirm subscriptions. Financial fraud is possible.",
  "reproduction": "Send a webhook with `{\"a\": 1}` — internal parsing converts to different JSON representation. Compare the re-stringified signature to the original header.",
  "remediation": "Read the raw body without parsing:\n\n```ts\nexport async function POST(req: Request) {\n  const body = await req.text();\n  const sig = req.headers.get('stripe-signature')!;\n  const event = stripe.webhooks.constructEvent(\n    body,\n    sig,\n    process.env.STRIPE_WEBHOOK_SECRET!\n  );\n  // ...\n}\n```\n\nNote the use of `req.text()` instead of `req.json()`, and passing the raw string directly.",
  "references": [
    "Stripe docs: https://stripe.com/docs/webhooks/signatures",
    "CWE-347: Improper Verification of Cryptographic Signature"
  ],
  "tags": ["money", "webhook", "crypto", "stripe"]
}
```

### Example 2: High — retirement math compounding bug

```json
{
  "id": "LM-0007",
  "agent": "logic-math",
  "severity": "high",
  "confidence": 0.92,
  "title": "Compound interest uses annual rate as periodic rate in monthly projection",
  "file": "lib/retirement/projections.ts",
  "line_start": 34,
  "line_end": 42,
  "code_snippet": "function projectBalance(principal: number, annualRate: number, years: number) {\n  const months = years * 12;\n  let balance = principal;\n  for (let i = 0; i < months; i++) {\n    balance = balance * (1 + annualRate); // <-- BUG: annualRate should be annualRate/12\n  }\n  return balance;\n}",
  "description": "This function compounds the balance each month, but uses the annual rate as the per-period rate. With `annualRate = 0.07` and `years = 30`, this produces `P * 1.07^360` instead of the correct `P * (1 + 0.07/12)^360`. The result is massively inflated — a projection of $100k at 7% for 30 years returns approximately $4.5 quadrillion instead of the correct ~$811k.\n\nLikely this has been masked in testing if the test inputs are small or the rate is tiny.",
  "impact": "Every user-facing retirement projection is wildly incorrect. Users relying on these numbers will be misled about their financial position, which is a core product-integrity failure and a potential regulatory issue (misleading financial information).",
  "reproduction": "Call `projectBalance(10000, 0.07, 30)` and compare to any online compound-interest calculator with the same inputs and monthly compounding.",
  "remediation": "Convert the annual rate to a periodic rate:\n\n```ts\nfunction projectBalance(principal: number, annualRate: number, years: number) {\n  const months = years * 12;\n  const monthlyRate = annualRate / 12;\n  return principal * Math.pow(1 + monthlyRate, months);\n}\n```\n\nAdditionally: note that money should not be stored as `number` — see finding LM-0002 for the broader issue. Convert the API to use decimal.js or integer cents.",
  "references": [
    "Compound interest formula: FV = PV * (1 + r/n)^(n*t)"
  ],
  "tags": ["money", "math", "retirement", "user-facing-number"]
}
```

### Example 3: Medium — missing index

```json
{
  "id": "DL-0003",
  "agent": "data-layer",
  "severity": "medium",
  "confidence": 0.85,
  "title": "No index on foreign key `orders.user_id` despite heavy query use",
  "file": "prisma/schema.prisma",
  "line_start": 45,
  "line_end": 52,
  "code_snippet": "model Order {\n  id        String   @id @default(cuid())\n  userId    String   // no @index\n  total     Int\n  createdAt DateTime @default(now())\n  user      User     @relation(fields: [userId], references: [id])\n}",
  "description": "The `Order.userId` field is not indexed. The application queries orders by userId in multiple places — the dashboard, the account page, and the admin tool. With the default Postgres behavior, these queries will do a sequential scan on the orders table.\n\nAt current size (<10k rows) this is imperceptible. At 1M+ rows, every order-lookup-by-user query will become a significant hotspot.",
  "impact": "Performance degradation that will be invisible until the orders table grows. By the time it's noticeable in production, the table will be large enough that adding the index requires a CONCURRENT operation to avoid blocking writes.",
  "reproduction": "Run `EXPLAIN ANALYZE` on `SELECT * FROM \"Order\" WHERE \"userId\" = 'x'` at any table size — it will show a Seq Scan.",
  "remediation": "Add an index to the Prisma schema:\n\n```prisma\nmodel Order {\n  id        String   @id @default(cuid())\n  userId    String\n  total     Int\n  createdAt DateTime @default(now())\n  user      User     @relation(fields: [userId], references: [id])\n\n  @@index([userId])\n  @@index([userId, createdAt(sort: Desc)]) // if you often query recent orders per user\n}\n```\n\nThe migration will create an index `ON Order(userId)` using CREATE INDEX. If the table is already large, add `CONCURRENTLY` to avoid blocking writes — Prisma does this by default for non-unique indexes in recent versions, but verify.",
  "references": [],
  "tags": ["performance", "database", "index"]
}
```

### Example 4: Low — error handling gap

```json
{
  "id": "EH-0015",
  "agent": "error-handling",
  "severity": "low",
  "confidence": 0.7,
  "title": "Email send failure logged but not reported",
  "file": "lib/email/send.ts",
  "line_start": 18,
  "line_end": 25,
  "code_snippet": "try {\n  await mailgun.send({ to, subject, body });\n} catch (err) {\n  console.error('Email failed', err);\n  // caller not notified; user proceeds as if email sent\n}",
  "description": "Email sending failures are logged to stdout but not reported to an error tracker. The calling code continues as if the email succeeded. For transactional emails (password reset, order confirmation), this is a user-facing failure the team can't diagnose.",
  "impact": "Users who don't receive emails cannot be helped because the team has no visibility. Support burden increases; user frustration increases.",
  "reproduction": "Stub mailgun.send to throw. Call the signup flow. No error in Sentry; user cannot log in because the verification email never arrived.",
  "remediation": "Report the error and optionally retry:\n\n```ts\nimport * as Sentry from '@sentry/nextjs';\n\ntry {\n  await mailgun.send({ to, subject, body });\n} catch (err) {\n  Sentry.captureException(err, { extra: { to, subject } });\n  // consider queueing for retry\n  throw new EmailSendError('Failed to send email', { cause: err });\n}\n```\n\nFor transactional emails specifically, this function should either throw (and the caller handle it) or be backed by a queue with retry.",
  "references": [],
  "tags": ["error-handling", "email", "observability"]
}
```

### Example 5: Nit — variable naming

```json
{
  "id": "ARC-0022",
  "agent": "architecture",
  "severity": "nit",
  "confidence": 0.5,
  "title": "Variable `d` is unclear; consider `durationMs`",
  "file": "lib/retirement/simulate.ts",
  "line_start": 89,
  "line_end": 89,
  "code_snippet": "const d = endTime - startTime;",
  "description": "Single-letter variable name obscures the unit and meaning.",
  "impact": "Readability only.",
  "reproduction": null,
  "remediation": "Rename `d` to `durationMs`.",
  "references": [],
  "tags": ["naming", "readability"]
}
```

## Final guidelines

- **One finding per issue.** Don't bundle multiple unrelated problems into one finding.
- **Write at the severity you're confident in.** Don't mark critical to get attention if the evidence is medium.
- **Avoid false precision.** Confidence of 0.847 is silly; 0.85 or 0.8 is fine.
- **Prefer a code fix over prose.** Showing the fix communicates more than describing it.
- **Reference canonical sources.** When you say "this is a known bad pattern", link to where it's documented as such.
