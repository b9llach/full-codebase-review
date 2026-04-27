# Domain Pack: Gaming / Wagering / Skill-Based Competition

Extends the `logic-math` agent. Load when the stack detection identifies any wagering, betting, odds-based gaming, or skill-based-competition platform.

Two assumptions drive this pack:

1. **Users are adversarial.** At least some of them will actively try to exploit the system. Every shortcut, every assumption, every trust boundary is a target.
2. **Money bugs are unrecoverable.** Wrong payouts get disputed, chargebacked, or pooled into systematic extraction. Defensive paranoia is the baseline, not a stretch goal.

## General wagering principles

### Money handling

All the rules from `domains/fintech.md` apply: integer minor units or decimals, never float. This pack adds:

- Balance changes must be atomic with the event that triggered them (wager placed, wager settled, refund issued). Never split across non-transactional writes.
- Every balance change must have a ledger entry: what happened, when, by whom, how much, resulting balance. No destructive updates.
- A balance anywhere in the system should equal the sum of its ledger entries. This invariant should have a test and ideally a production reconciliation job.

Flag:
- Balance stored without a corresponding ledger / transactions table
- Ledger writes that aren't in the same transaction as the balance mutation
- Ledger entries without a unique event reference (can't trace back to the wager)

### Escrow and custody

When two users wager against each other, both stakes must be held atomically. The funds are neither user's — they belong to the escrow account — until settlement.

Flag:
- Funds "held" by setting a flag on the user's balance instead of moving to an escrow account (a balance bug could expose held funds)
- Escrow balance that doesn't have its own ledger
- Escrow reconciliation missing — nothing verifies that total_escrow = sum_of_held_wagers
- Hold release that happens before settlement confirms (user could spend held funds)

Canonical pattern:

```
place_wager(user, amount):
  transaction:
    assert user.balance >= amount
    user.balance -= amount
    escrow.balance += amount
    ledger: user -amount (reason: "wager X placed")
    ledger: escrow +amount (reason: "wager X escrow")
    wager.status = "open"
```

Every piece of this must be atomic.

## Odds and payout math

### Payout invariants

For any wager matched between two sides, the total payout to the winner(s) should equal the total escrowed stakes minus the house cut, to the cent. No rounding residue lost.

Flag:
- Payouts computed with floats (see money rules)
- Payouts computed as `stake * odds` with rounding that doesn't preserve the sum
- House cut applied inconsistently (percentage of stake vs percentage of payout)
- Ties/pushes: is the stake returned? Is the house cut still taken? (Usually: stake returned, no cut — verify the code)

### Odds representations

Wagering apps deal with multiple odds formats:
- **American / moneyline**: +200 means "win $200 on $100 stake"; -150 means "stake $150 to win $100"
- **Decimal**: 3.00 means total return (stake + profit) is 3x stake
- **Fractional**: 2/1 means profit is 2x stake

Flag:
- Conversion between formats done lossy (float imprecision)
- Inconsistent representations stored in different tables (some American, some decimal)
- Display showing one format but math using another without explicit conversion

### Implied probability and overround

For any two-sided market:
```
implied_prob(home) + implied_prob(away) + house_margin = 1.0 + overround
```

The difference from 1.0 is the house edge. Flag:
- Markets with overround < 0 (implies negative house edge — book could be exploited)
- Overround inconsistent across similar markets
- Edge calculations that ignore the effect of stake sizes on outcomes

## Settlement

### Settlement correctness

Flag:
- Settlement that doesn't validate the event outcome against an authoritative source
- Settlement that can be triggered by the user (must be server-authoritative only)
- Settlement without an audit trail of "by whom / from what source / when"
- Reversal of settlement — if the outcome is contested and changed, is there a rollback path?
- Settlement on an event before it's considered final (e.g., during overtime vs after OT ends)

### Partial settlement / cashout

If the app supports cashout / partial settlement:

Flag:
- Cashout price calculation without a documented formula
- Cashout available at a price higher than fair value (user is being overpaid — house loses)
- Cashout available at a price significantly lower than fair (user ripoff — legal exposure)
- Race: user can submit a cashout request while settlement is already happening (double-payment risk)

## Race conditions (the big one)

Wagering systems have more critical races than almost any other domain. Special attention:

### Matching races

When two users want to take the same open wager:

Flag:
- Matching via read-then-update without atomic claim
- Multiple instances of the matcher service running without a distributed lock
- Wager marked "matched" before funds are debited from the acceptor
- Wager marked "matched" in the DB but the corresponding balance mutation failed

Preferred pattern: `UPDATE wagers SET matched_by = ?, status = 'matched' WHERE id = ? AND status = 'open'`, check affected rows.

### Simultaneous cash-out and settlement

If a user hits cashout at the same moment the server begins settling the event:

Flag:
- No lock on the wager during state transitions
- Both paths (cashout, settle) taking different code paths that can write to the same fields
- Idempotency missing — second operation creates duplicate ledger entries

### Deposit-and-wager races

User deposits, then immediately wagers, then (separately) the deposit is reversed (chargeback, ACH return):

Flag:
- No holds on deposited funds before settlement window
- Chargebacks not modeled in balance updates (user withdraws, then chargeback succeeds, negative balance)
- ACH holds (typically 3-5 business days) not enforced for new users

### Refund races

On cancellation or event voiding:

Flag:
- Refund path that doesn't check current wager status (could refund an already-settled wager)
- Refund that runs before the settlement rollback completes
- Multiple refund triggers (admin UI + webhook + cron) without coordination

## Fair play and integrity

### Server authority

Everything that matters must be computed server-side. Client can only display and request.

Flag:
- Client-computed balance displayed as source of truth (should be `GET` from server after every change)
- Game state held only in client memory (any skill-based gaming needs anti-cheat; match outcomes must come from an authority)
- Odds displayed to client locked in at display time (user clicks "accept odds 2.5"; server accepts without re-validating current odds — can be exploited by stale data)

### Cheating resistance (for skill-based games)

For skill-based wagering platforms:

Flag:
- Match outcomes reported by one player only (should be corroborated, replayed, or from game API)
- No anti-bot measures (CAPTCHA on wager, unusual-pattern detection)
- Account sharing not detectable (multiple IPs, device fingerprint changes)
- Collusion detection missing (same IP playing against itself through different accounts)
- Sandbagging detection missing (user throws low-stakes matches to lower rating, then wins high-stakes)

### Verification sources

Flag:
- Game API integrations that don't verify response signatures (could be spoofed)
- Reliance on a single data source for settlement (should cross-reference)
- Caching of event status that could be stale during rapid events

## Deposits and withdrawals

### Deposit flows

Flag:
- Deposit credited before processor webhook confirms (could reverse; user has already wagered)
- No per-user deposit velocity limits (sudden surge = high fraud risk)
- No minimum / maximum deposit amounts
- Currency mismatches at deposit boundaries

### Withdrawal flows

Flag:
- Immediate withdrawal of freshly-deposited funds (classic laundering pattern; should have a hold period)
- No KYC check before large withdrawal
- Withdrawal destination (bank account, third-party wallet, etc.) not verified
- Withdrawal limits per day / week absent

## KYC / AML / identity

### Identity verification

Flag:
- Identity verification only at signup (should revalidate for large transactions, long dormancy, jurisdiction change)
- Age verification only at signup (state laws often require re-verification per session / per wager)
- No check against OFAC / sanctions lists
- No monitoring for structuring (multiple deposits just under reporting threshold)

### Self-exclusion

Flag:
- No self-exclusion feature (legally required in most US states)
- Self-exclusion that can be reversed by the user without a cooling period
- No cross-property exclusion (user excluded in one game but can play another on same platform)
- No check against state self-exclusion lists (many states maintain central registries)

## Geolocation and jurisdiction

### Location verification

Different jurisdictions allow different wagering types. Verification must happen at each relevant action.

Flag:
- Geolocation only at signup (user moves; legal status changes)
- IP-only geolocation (easily spoofed with VPN)
- No geolocation verification at the moment of wager placement (required in most real-money states)
- Integration with a third-party geolocation vendor treated as trusted without signature verification

### Jurisdiction rules

Flag:
- Rules hard-coded for one jurisdiction without a mechanism for others
- New state launches requiring code changes (should be config)
- Different minimum ages per state not modeled
- Different allowed wager types per state not enforced server-side

## Promotions and bonuses

### Bonus abuse

Every sign-up bonus is a target for fraud.

Flag:
- Bonus usable without deposit match requirement
- Bonus funds mixed with real funds in the same balance (can't distinguish withdrawable from bonus)
- Bonus that can be withdrawn without playthrough (rollover) requirement
- Playthrough calculation bugs (wagering on low-edge markets to cheaply clear rollover)
- Multiple accounts per user to claim signup bonuses repeatedly — link detection missing

### Referral bonuses

Flag:
- Referral bonuses awarded before referred user has meaningful activity (signup + $1 deposit → bonus)
- Self-referral possible (same user refers themselves via different account)
- Referral chains (A refers B refers C refers A) paying out

## Tax reporting

In the US, gambling winnings are reportable income:
- W-2G for certain thresholds (varies by game: $1,200+ on slots/bingo; $600+ on sweepstakes/lottery with 300:1 odds; etc.)
- 1099-MISC for cumulative winnings above $600
- Withholding (24% federal) required above certain thresholds

Flag:
- No W-2G / 1099 generation logic
- Net winnings calculated wrong (tax law: each win is reportable; losses can offset only if itemizing)
- No TIN (SSN / ITIN) collection for users exceeding thresholds
- Withholding not implemented where legally required

## Data integrity and audit

### Audit trail

Flag:
- Balance changes without audit trail
- Wager modifications without audit trail (who changed this? when? why?)
- Settlement changes / reversals without audit trail
- Admin actions not logged

### Reconciliation

Flag:
- No daily reconciliation job that verifies balance integrity
- No reconciliation of house P&L against expected edge
- No reconciliation of payment processor vs internal ledger

## Process

Apply this pack alongside general `logic-math` for any wagering codebase. Be especially paranoid about:

1. Every function that moves money
2. Every state transition on a wager
3. Every moment where a user's action can race with a system action (settlement, cashout, cancellation)
4. Every bonus / promotion code path (lowest-trust users interact with these)
5. Every admin tool that can manually adjust balances or states

Spend extra time here. A missed race condition in a standard CRUD app is an inconvenience; a missed race condition in a wagering app is a six-figure liability.
