# Domain Pack: Fintech / Retirement / Tax

Extends the `logic-math` agent. Load alongside `agents/logic-math.md` when the stack detection identifies retirement planning, tax software, lending, or any financial-math-heavy code.

The standing assumption: a wrong number here costs the user real money (wrong projection, wrong tax, wrong payout). Apply extra skepticism.

## General financial arithmetic

### Money representation (hard rule)

- Money is **never** a `number` / `float` / `double`.
- Use integer minor units (cents, satoshis) OR a decimal library (`decimal.js`, `big.js`, Python `Decimal`, Postgres `numeric`).
- Two floats sum: `0.1 + 0.2 === 0.3` is false. This alone disqualifies floats for money.

Flag every occurrence of float money. Treat it as critical.

### Rounding

- Every rounding step must be explicit about direction and intent.
- For financial math, banker's rounding (half-to-even) is the most common default — reduces systematic bias.
- Rounding should happen at display or persistence boundaries, not repeatedly through a calculation.

Flag:
- `.toFixed(2)` in calculation paths (it's a string operation)
- `Math.round` without comment
- Rounding intermediate results
- Rounding direction that differs across similar code paths

### Currency and localization

- All money operations implicitly have a currency. Mixing currencies without conversion is a bug.
- Display formatting: use `Intl.NumberFormat` with locale and currency; don't hard-code `$`.

## Compound interest

### The formulas (keep these handy)

Discrete compounding:
```
FV = PV * (1 + r/n)^(n*t)
```
Where `r` is annual rate, `n` is compounds per year, `t` is years.

Continuous compounding:
```
FV = PV * e^(r*t)
```

APR → APY:
```
APY = (1 + APR/n)^n - 1
```

Periodic payment (ordinary annuity):
```
FV = PMT * ((1 + r)^n - 1) / r
PV = PMT * (1 - (1 + r)^-n) / r
```

For any code implementing these, verify the code matches the formula exactly. One misplaced `+ 1` silently breaks the whole projection.

### Common bugs

Flag:
- APR treated as APY (or vice versa) — compounding frequency ignored
- Monthly rate used where annual rate is expected (or vice versa) — look for `rate` or `r` being used directly in `(1 + r)^n` when `r` should be `rate/12`
- `Math.pow(1 + r, n)` vs `(1 + r) ** n` — same math, but if `r` is NaN or negative, watch behavior
- Off-by-one in the `n` exponent (number of periods, inclusive vs exclusive)
- Future value calculated correctly but present value using the wrong sign convention
- Annuity-due vs ordinary annuity — differ by one period of compounding

## Inflation

### Real vs nominal consistency

A projection is either in real (today's) dollars or nominal (future dollars). Pick one and stick with it.

Flag:
- Contributions in nominal dollars, withdrawals in real dollars (or any mix)
- Display value labeled "in today's dollars" but the calculation was in nominal
- Inflation rate applied inconsistently (sometimes to contributions, sometimes not)
- Inflation compounded alongside returns when one is already net of the other

### Inflation rate assumptions

Flag:
- Hard-coded inflation rate without rationale (2%? 3%? 3.2%? — document the source)
- No way for the user to adjust the assumption
- Inflation modeled as a constant (real-world inflation varies)

## Monte Carlo simulations

### Simulation quality

Flag:
- Fewer than 1000 simulations (common in naïve implementations; gives noisy output)
- Results reported to 4 decimal places from a 100-simulation run (false precision)
- No random seed handling (results non-reproducible, which is bad for debugging)
- Same seed across all runs (misleadingly deterministic)
- Simulations run inside the request path (slow; should be async / cached)

### Assumptions

Flag:
- Returns drawn from normal distribution without mentioning fat tails (equity returns have kurtosis; normal underestimates extreme outcomes)
- Returns and volatility assumptions hard-coded without documentation of source (CAPE-based? Historical?)
- Correlation between asset classes ignored (90% bonds + 10% stocks isn't 0.9 * bond vol + 0.1 * stock vol)
- Rebalancing not modeled (portfolio drifts; assumed allocation ≠ actual)
- Fees not subtracted (real returns are net of fees, not gross)
- Taxes not modeled in tax-deferred vs taxable accounts

### Sequence-of-returns risk

The big one. A portfolio earning an average 7% has very different outcomes if the bad years hit during withdrawals vs during accumulation.

Flag:
- Projections that use average return only (no variability = no sequence risk)
- "Success rate" reporting without defining success clearly (90% success = 10% of simulated retirees run out of money — is this acceptable?)
- Withdrawal strategies that don't adapt to market conditions (4% rule, no guardrails)

## Withdrawal strategies

### 4% rule

Original Bengen: 4% of initial balance, adjusted for inflation each year, over 30 years.

Flag:
- "4% rule" implementations that take 4% of current balance (different rule — "constant percent")
- Fixed 4% without inflation adjustment
- Applied to portfolios with non-standard asset mix (Bengen assumed 50/50 to 75/25 stock/bond)
- Applied to retirement horizons >30 years without increasing safety margin

### Dynamic strategies

Flag:
- "Guardrails" logic that isn't tested at the boundaries (portfolio exactly at the upper/lower band)
- Ratcheting up-only without a ratchet down (asymmetric, optimistic bias)
- No drawdown limits

## Tax calculations (US specifically)

### Tax brackets

Flag:
- Hard-coded bracket numbers without a year attribution (2024 brackets used in a 2026 calculation)
- No indexing of brackets (brackets are adjusted annually for inflation)
- Marginal rates applied as effective rates (wrong for total-tax calculation)
- Effective rates computed wrong (should be total tax / total income, not an average of marginal rates)
- Boundary bugs: for a 2026 single filer, the 22% bracket starts at $48,475. What does income of exactly $48,475 pay? Brackets are inclusive on the lower bound.

### 2026 federal income tax (single, for reference)

| Rate | On income over |
|------|---------------|
| 10%  | $0            |
| 12%  | $12,150       |
| 22%  | $49,475       |
| 24%  | $105,700      |
| 32%  | $201,775      |
| 35%  | $256,225      |
| 37%  | $640,600      |

(Verify against current IRS publications — these are illustrative; bracket amounts change.)

### Capital gains

Flag:
- Long-term vs short-term not distinguished (different rates; short-term is ordinary income)
- Qualified dividends treated as ordinary income (they're taxed at LTCG rates)
- Wash-sale rule not modeled in tax-loss harvesting logic
- Cost basis tracked per lot vs averaged (matters for capital gains calculation)
- Net Investment Income Tax (3.8% NIIT on certain income above thresholds) missing

### Social Security taxation

This one trips everyone up.

The provisional income formula:
```
Provisional income = AGI + tax-exempt interest + 0.5 * SS benefits
```

Then the taxability of SS:
- Below lower threshold ($25k single / $32k married): 0% taxable
- Between thresholds: up to 50% taxable
- Above upper threshold ($34k single / $44k married): up to 85% taxable

These thresholds are **not** indexed for inflation (intentionally, to creep more taxpayers into taxability over time).

Flag:
- Simplification like "SS is 85% taxable" without implementing the tiered calculation
- Provisional income missing the half-of-SS component
- Thresholds indexed (they shouldn't be)

### Required Minimum Distributions (RMDs)

Post-SECURE 2.0:
- RMDs begin at age 73 (for anyone turning 72 after 2022)
- Rises to age 75 in 2033
- Calculated using IRS Uniform Lifetime Table (or Joint Life if spouse is >10 years younger)
- First RMD can be delayed to April 1 of year following age 73 (but then two RMDs in that year)

Flag:
- Old "age 70.5" logic (pre-SECURE)
- Missing the age-75 transition in 2033
- Flat divisor (e.g., dividing by life expectancy at 73 forever) instead of IRS table
- Roth IRAs subject to RMDs in the code (they're not during the original owner's lifetime)
- Roth 401(k) RMDs not respected (pre-SECURE 2.0 they were required; now they're not, starting 2024)

### Contribution limits (2026, verify current)

- 401(k) elective deferral: $24,500
- 401(k) catch-up (age 50+): $8,000
- 401(k) super catch-up (age 60-63, SECURE 2.0): $11,250 (higher of $10k or 150% of regular catch-up)
- IRA: $7,500
- IRA catch-up (age 50+): $1,100
- HSA: $4,400 self / $8,750 family; $1,000 catch-up (age 55+)

Flag:
- Hard-coded limits without the year
- No catch-up logic by age
- Super catch-up (ages 60-63) missing entirely
- Employer match included in the elective deferral limit (it's a separate limit, $76,500 total contribution in 2026)

### Roth conversions

Flag:
- Five-year rule missing (Roth conversions have their own 5-year clock per conversion)
- Pro-rata rule missing (if user has pretax IRA balances, conversions are taxed pro-rata)
- Backdoor Roth without checking other pretax IRA balances
- Medicare IRMAA surcharge impact not modeled (Roth conversion increases MAGI, can jump premium tiers)

## Interest rates and loans

### Amortization

Standard amortization formula:
```
P = L * (r * (1+r)^n) / ((1+r)^n - 1)
```

Where `L` is loan amount, `r` is periodic rate, `n` is number of periods, `P` is periodic payment.

Flag:
- Payment calculation using APR directly without converting to monthly (`r` should be `APR/12`)
- Principal/interest split per payment computed wrong (it's `P * (current_balance * r / P)` not a fixed ratio)
- Extra payments applied to principal without reducing the remaining schedule correctly
- Last payment not adjusted for rounding residue (loans never pay off to exactly zero without a final correction)

### APR vs APY for loans

- APR: simple interest rate for the period
- APY/EAR: effective annual rate accounting for compounding

Consumer loans display APR by regulation (Truth in Lending Act). Flag displays labeled APR that actually compute APY.

## Date-related

### Retirement age math

Flag:
- Age in years computed as `(now - dob) / (365 * 86400 * 1000)` — ignores leap years, drifts
- Retirement date computed as DOB + 65 years using date-fns `addYears` (correct) vs manual `setFullYear(y + 65)` (handles Feb 29 correctly on most platforms, but verify)
- "Year you retire" computed without handling birthday during year

### Years of service

Flag:
- Integer years by subtraction of year components only (ignores partial year)
- Rounding direction on partial years matters for vesting / benefit calculations

## Specific product checks

### 401(k) loans

Flag:
- Loan limits: lesser of $50,000 or 50% of vested balance — not always correctly computed
- Payment terms: 5 years max (25 for primary residence) — sometimes violated
- Default on separation: outstanding balance treated as distribution — missing in some flows

### HSA

Flag:
- Triple tax advantage (contribution deductible, growth tax-free, withdrawals for medical tax-free) — the third leg often missed
- Non-medical withdrawals before age 65: 20% penalty plus ordinary income — sometimes modeled as just income
- After age 65: non-medical withdrawals treated like traditional IRA (income tax but no penalty)
- Excess contribution penalty (6% per year until withdrawn)

### Annuities

Flag:
- Annuity income taxability depends on whether premiums were pretax or after-tax — often simplified incorrectly
- Surrender charges not modeled
- Guaranteed minimum rates vs projected rates confused

## Process

When reviewing a retirement/fintech codebase, apply this checklist IN ADDITION TO the general `logic-math` checklist. Many of these bugs don't fail tests because tests often use the same (incorrect) logic to compute expected values — independently verify against external sources (IRS publications, textbook formulas, reference implementations).

A good technique: for any formula used, write it out as a comment above the code. Does the code match the comment? Does the comment match a real source?
