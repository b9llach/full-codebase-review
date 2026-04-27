# Report Template

This is the fillable structure for `REPORT.md`. When synthesizing, use this exact structure (see `references/report-format.md` for the rationale).

---

```markdown
# Codebase Review — {REPO_NAME} — {DATE}

> **TL;DR:** {ONE-SENTENCE OVERALL TAKE}. {NUMBER} critical findings, {NUMBER} high, {NUMBER} medium. {FASTEST REMEDIATION WIN}.

---

## Executive summary

{3-5 bullets for a technical founder or principal engineer. No code, no file names.}

- {Overall quality statement — honest, concise}
- {Most serious issue in one sentence}
- {Biggest structural concern if any}
- {Biggest latent risk — the thing that's fine now but will bite later}
- {Estimated time to remediate criticals}

---

## Critical findings

{Full writeup per critical finding. If zero criticals, replace this section with:}

> No critical findings. The codebase handles its current threat surface adequately.

### C1 — {TITLE}

- **File:** `{PATH}:{LINE_START}-{LINE_END}`
- **Severity:** Critical
- **Confidence:** {CONF}
- **Agents:** {LIST}
- **Tags:** {TAGS}

**What's wrong:**

{Plain-language explanation. 1-3 paragraphs.}

**Code:**

```{LANG}
{SNIPPET}
```

**Impact:**

{Production consequences. Be specific — what goes wrong, who is affected, what is the business cost.}

**Reproduction:**

{If applicable, step-by-step. Omit this subsection if not applicable.}

**Fix:**

```{LANG}
{REMEDIATION CODE}
```

{If the fix doesn't fit in code, describe it as prose.}

**References:** {CWEs, OWASP, RFCs, docs}

---

### C2 — {TITLE}

{Same structure as C1}

---

## High findings

{Grouped by agent. Each finding: file:lines, one paragraph, fix direction. Code snippet optional.}

### Security ({N} high findings)

- **`{PATH}:{LINE_START}-{LINE_END}`** — {Title and one-paragraph explanation.} **Fix:** {Concrete direction.}
- **`{PATH}:{LINE_START}-{LINE_END}`** — ...

### Logic & Math ({N} high findings)

- ...

### Backend ({N} high findings)

- ...

{Continue for each agent with high findings. Omit sections for agents without high findings.}

---

## Medium findings

{Single-line format, grouped by agent.}

### Security

- `{PATH}:{LINE}` — {Summary.} **Fix:** {Direction.}
- `{PATH}:{LINE}` — ...

### {OTHER AGENTS}

- ...

---

## Low findings

{Collapsed to a table.}

| File | Line | Agent | Summary |
|------|------|-------|---------|
| `{PATH}` | {LINE} | {AGENT} | {SUMMARY} |
| ... | ... | ... | ... |

---

## Nits

{Summarized. Don't list individually unless the user asked for full nit detail.}

**{N} nit findings** across the codebase — primarily naming consistency, minor comment cleanup, and micro-style issues. See `findings/*.jsonl` filtered by `severity: "nit"` for the complete list.

---

## Findings by file

{For files with 2+ findings. This is the section people open when they're about to edit a file.}

### `{PATH}` ({N} findings)

- **L{LINE}:** [{SEVERITY} — {AGENT}] {TITLE}
- **L{LINE}:** [{SEVERITY} — {AGENT}] {TITLE}
- ...

### `{PATH}` ({N} findings)

- ...

---

## Remediation plan

{Ordered by impact/effort ratio. Each item is a concrete task.}

### This week (~{TIME ESTIMATE})

1. **Fix critical findings** ({LIST OF IDS})
   - {BRIEF DESCRIPTION OF APPROACH}
2. **{NEXT HIGHEST-LEVERAGE ITEM}**

### This sprint (~{TIME ESTIMATE})

1. Address all high findings in {MOST-IMPACTED AGENT} group
2. {OTHER ITEMS}

### This month

1. {MEDIUM FINDINGS SWEEP}
2. {ARCHITECTURAL IMPROVEMENTS}

### When there's time

1. {LOW / NIT SWEEP}
2. {NICE-TO-HAVES}

---

## Observability gaps

{What the team currently cannot see going wrong in production.}

- {EVENT / SYSTEM} has no structured logging
- No metric for {CRITICAL BUSINESS EVENT}
- No alert on {FAILURE MODE}
- Errors in {MODULE} are caught and discarded without reporting
- {etc.}

---

## What's going well

{Two to three bullets about genuine strengths. If nothing specific stands out, omit this section entirely.}

- {STRENGTH}
- {STRENGTH}
- {STRENGTH}

---

## Appendix A: Methodology

- **Repo:** `{PATH}`
- **Commit:** `{SHA}` ({BRANCH})
- **Date of review:** {DATE}
- **Files in scope:** {TOTAL} ({TIER-0 COUNT} deep-reviewed, {TIER-1 COUNT} thorough, {TIER-2 COUNT} skimmed, {TIER-3 COUNT} structural only)
- **Files excluded:** `node_modules`, `dist`, {OTHER EXCLUSIONS}
- **Domain packs applied:** {LIST, OR "none"}
- **Agents run:** architecture, security, frontend, backend, logic-math, data-layer, performance, dependencies, testing, error-handling, type-safety, concurrency, dead-code, observability, compliance
- **Agents skipped:** {LIST WITH REASON, OR "none"}
- **Estimated agent wall-clock time:** {TIME}
- **Findings raw data:** `.codebase-review-{TIMESTAMP}/findings/`

## Appendix B: Limitations

- This is a static review. No code was executed.
- No production-like data was available; claims about query performance are based on schema and query shape, not measured plan costs.
- Subject-matter expertise in {DOMAIN} was applied via reference material. A qualified expert may find additional issues in that area.
- {COUNT} low-confidence findings (confidence < 0.3) were dropped from this report. Full set available in raw findings files.
- {OTHER HONEST LIMITATIONS}

## Appendix C: Finding index

{Quick-reference table of all findings by ID for cross-referencing.}

| ID | Severity | File:Lines | Title |
|----|----------|-----------|-------|
| SEC-0001 | Critical | `{PATH}:{LINES}` | {TITLE} |
| LM-0007 | High | `{PATH}:{LINES}` | {TITLE} |
| ... | ... | ... | ... |

---

*Generated by `full-codebase-review` skill. To re-run specific agents or request walk-throughs of findings, ask.*
```

## Style notes when filling this template

- Replace every `{PLACEHOLDER}` — don't ship a report with unresolved placeholders.
- If a section would be empty, either say so plainly ("No critical findings.") or omit the section. Don't show empty sections.
- Keep titles of findings identical between the Critical/High sections and the Findings-by-file section — users cross-reference.
- The Remediation plan should always end with a time bucket the user can defer ("When there's time") so they don't feel ambushed by urgency.
- No emoji. No celebratory language. No hedging like "some people might consider this a concern."
