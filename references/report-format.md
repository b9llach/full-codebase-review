# Phase 4: Synthesis & Report

The report is the product. All the agent work is only as useful as the report makes it. Don't bury the lede; don't list findings without priority; don't hide behind methodology.

## Input

All `findings/*.jsonl` files from `.codebase-review-<ts>/findings/`. Each line is a finding conforming to `templates/finding.md`.

## Procedure

### Step 1 — Ingest and deduplicate

Read all `.jsonl` files. For each finding, compute a fingerprint: `(file, line_start, line_end, agent-category)`. If two findings from different agents target the same lines with semantically overlapping descriptions, merge them:

- Keep the higher severity
- Keep the higher confidence
- List both agents in a new `agents: [a, b]` field
- Take the more specific remediation

### Step 2 — Rank

Sort all findings by a composite score:

```
score = severity_weight * confidence
```

Weights:
- critical = 1000
- high = 100
- medium = 10
- low = 1
- nit = 0.1

### Step 3 — Write the report

Use the structure below exactly. No creative reorganization. People scanning the report for the first time should find the same section in the same place every time.

## Report structure

```markdown
# Codebase Review — <repo name> — <date>

## Executive summary

Three to five bullets. Written for a technical founder, not a line engineer.

- "N critical findings require attention before next deploy."
- "Most serious issue: <one-sentence description of worst problem>."
- "Overall code quality: <honest assessment — solid, mixed, concerning, etc.>"
- "Biggest structural concern: <one-sentence, if any>."
- "Time to remediate criticals: <rough estimate>."

If there are zero critical findings, say so plainly: "No critical findings. The codebase is in good shape for its stage."

## Critical findings (fix before next deploy)

For each critical, a full writeup. No collapsing. No "see appendix."

### C1: <Title>
- **File:** `path/to/file.ts:42-58`
- **Severity:** Critical
- **Confidence:** 0.95
- **Agents:** security, logic-math

**What's wrong:**
[Plain-language explanation]

**Code:**
```ts
[snippet]
```

**Impact:**
[What happens in production]

**Reproduction:**
[Steps, if applicable]

**Fix:**
```ts
[patch or clear remediation]
```

**References:** CWE-89, OWASP A03

---

### C2: ... [etc for every critical]

## High findings

Group by agent. Each finding gets: file:lines, one-paragraph description, fix direction. Full code snippets optional — if a finding is obvious from its description, skip the snippet.

### Security (N high findings)
- `lib/auth/session.ts:120-135` — <description>. Fix: <direction>.
- ...

### Logic & Math
- `lib/retirement/compound.ts:23` — <description>. Fix: <direction>.
- ...

### [other agents with high findings]

## Medium findings

Single-line format. File:lines — summary — fix direction. Grouped by agent.

### Security
- `app/api/users/route.ts:45` — Missing rate limit on signup endpoint. Fix: add IP-based limiter.
- ...

## Low findings

Collapsed to a table:

| File | Line | Agent | Summary |
|------|------|-------|---------|
| ... | ... | ... | ... |

## Nits

A single summary: "X naming issues, Y formatting inconsistencies, Z comment cleanups. See `findings/nits.jsonl` for the full list if desired."

Do not list individual nits in the report unless the user asked for them explicitly.

## Findings by file

For each file that had two or more findings, a mini-section listing them. This is the section people open when they're about to edit a specific file and want to batch fixes.

### `lib/retirement/monte-carlo.ts` (4 findings)
- L23: [Critical — Logic] Compound interest formula uses nominal instead of real returns
- L67: [High — Logic] Monte Carlo runs only 100 simulations, insufficient for retirement planning
- L89: [Medium — Performance] Simulation is single-threaded; could parallelize
- L112: [Low — Observability] No logging of simulation parameters

### `app/api/stripe/webhook/route.ts` (3 findings)
- ...

## Remediation plan

Ordered by impact/effort ratio. Each item is a concrete ticket-sized task.

1. **[This week]** Fix critical findings C1–C4 (est. ~4 hours)
   - C1: ...
   - C2: ...
2. **[This sprint]** Fix high findings in security group (est. ~1 day)
3. **[This month]** Address top 3 architectural concerns
4. **[When there's time]** Sweep medium findings by file

## Observability gaps

What the team currently cannot see going wrong in production. This is its own section because these aren't "bugs" but they make bugs 10x harder to diagnose:

- No structured logging in <file/module>
- No metric for <critical business event>
- No alert on <failure mode>
- Errors swallowed without reporting in <file>

## What's going well

A brief section — two or three bullets — about genuine strengths. Not performative encouragement; real observations. This calibrates the rest of the report.

- "Type coverage is excellent — strict mode throughout."
- "Input validation is consistent across API boundaries."
- "Test coverage on the billing module is thorough."

If there's nothing genuinely going well, skip this section rather than inventing things.

## Appendix: methodology

- **Repo:** <path>, commit <sha>, <N files total>
- **Agents run:** architecture, security, frontend, backend, logic-math, data-layer, performance, dependencies, testing, error-handling, type-safety, concurrency, dead-code, observability, compliance
- **Domain packs:** fintech
- **Files deep-reviewed (Tier 0):** N
- **Files thoroughly reviewed (Tier 1):** N
- **Files skimmed (Tier 2):** N
- **Files structural only (Tier 3):** N
- **Runtime:** N minutes of wall-clock time; ~M subagent invocations
- **Excluded:** node_modules, dist, generated, <any user-requested exclusions>

## Appendix: limitations

Honest acknowledgments:
- "This is a static review; no tests were executed."
- "Agents did not run against a production-like database, so query performance claims are theoretical."
- "Domain expertise in <specialty> was applied via reference material; a subject-matter expert may find additional issues in that area."
- "<N> low-confidence findings were dropped (confidence < 0.3). Full set available in raw findings files."
```

## Stylistic rules for the report

- **Write like a senior engineer doing a favor for a peer.** Not like a consultant billing by the insight.
- **Be specific. Always name files and lines.** "There are issues in the auth code" is not a finding.
- **Don't hedge when you're sure.** "This is a bug" beats "this may potentially be a concern."
- **Don't overclaim when you're not sure.** Confidence < 0.7 findings should hedge appropriately.
- **Use code fences for code.** Always. Syntax-highlight when known.
- **Prefer active voice.** "The handler skips signature verification" > "Signature verification is skipped by the handler."
- **No apology. No throat-clearing.** Start with the findings.
- **End with what's next.** The remediation plan should leave the user knowing what to do tomorrow morning.
