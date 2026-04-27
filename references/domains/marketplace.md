# Domain Pack: Marketplace / Two-Sided Platform / Payouts

Extends the `logic-math`, `backend`, and `security` agents. Load when the stack detection identifies distinct buyer and seller roles, commission / split payments, or multi-party payment flows.

The core complexity: money flows between three parties (buyer, seller, platform) with commission, tax, and payouts that must reconcile precisely. Errors here are visible, auditable, and expensive.

## Payment flow correctness

### Commission / split calculation

For every transaction:
```
buyer_pays = item_price + tax + fees
seller_receives = item_price - commission - payment_processing_fee
platform_receives = commission + platform_fees
(tax is held for remittance or passes through)
```

The sum must balance to the penny.

Flag:
- Commission as float percentage (`price * 0.1` produces float imprecision)
- Commission rounded in a way that doesn't preserve the sum (buyer pays X, seller + platform get less than X by rounding residue)
- Fee waterfalls applied in inconsistent order across code paths (does payment fee come out of seller's share or buyer's?)
- Multi-currency transactions without explicit conversion record

### Connected accounts / Stripe Connect

If using Stripe Connect or similar:

Flag:
- Application_fee_amount miscomputed (in cents, not dollars)
- Direct charges vs destination charges vs separate charges + transfers used inconsistently
- Webhook events not subscribed for connected account events (`account.updated`, `payout.failed`)
- No handling for account deauthorization
- No handling for `connected_account_reverse_payout` events

### Payouts to sellers

Flag:
- Payouts triggered before funds settle (user disputes during settlement → platform loses)
- No hold period for new sellers (fraud vector)
- Payout calculations that don't match the sum of individual transaction sellers' shares
- Payouts issued without a corresponding ledger entry
- No reconciliation job for "do our payout records match Stripe's?"
- Failed payouts (bank closed, wrong account) not surfaced to seller

## Two-sided access controls

### Role-based data access

Buyers and sellers see different things about the same entity.

Flag:
- API endpoints that return seller-sensitive data (payout info, margin, internal notes) to buyers
- API endpoints that return buyer PII (email, home address) to sellers without need
- Chat / messaging features that don't enforce role boundaries (seller can see other sellers' messages with buyers)
- Review/rating systems that allow cross-role abuse (seller can leave reviews for other sellers)

### Impersonation and support access

Flag:
- Support tools that can impersonate buyer or seller without audit log
- Session switching that persists state inappropriately
- Internal users with blanket access to all transactions

## Listings and catalog

### Listing integrity

Flag:
- Price / availability checked at browse time but not re-verified at purchase (stale price race)
- Inventory decremented after payment confirmation without atomic reservation
- Listing status changes (delisted, sold) that don't propagate to in-flight transactions

### Search and discovery

Flag:
- Search ranking exploitable (sellers can game visibility)
- No filtering of inactive / problematic sellers
- No bias controls for marketplace fairness (always showing a few sellers hurts platform growth)
- Sponsored / promoted listings not clearly disclosed

## Fraud and trust

### Buyer fraud patterns

Flag:
- No velocity limits on purchases (card testing)
- No check for mismatched shipping/billing addresses on high-risk categories
- No chargeback reserve calculation
- No blocklist / allowlist for BIN / card / email / IP
- Refund policy code that allows "double dip" (refund + chargeback)

### Seller fraud patterns

Flag:
- No KYC for sellers receiving payouts
- No velocity limits on listing creation (stuffing)
- No duplicate listing detection
- No plagiarism / stolen-content detection for listings
- Payouts allowed before first sale has fully settled (instant payout risk)

### Trust and safety

Flag:
- No reporting mechanism for abusive listings / users
- Reports not logged with audit trail
- Banned users can re-register easily (no device / email / payment fingerprinting)

## Disputes and refunds

### Dispute workflow

Flag:
- No mediation step before platform decides (buyer claims, platform instantly sides with them)
- Disputes that don't hold seller payouts until resolved
- No time limit on dispute filing (indefinite liability)
- Refund logic that doesn't reverse the correct ledger entries (buyer gets refund but seller's share isn't clawed back)

### Partial refunds

Flag:
- Partial refund calculations that don't proportionally reduce commission and fees
- Partial refund that leaves the tax portion unchanged (bookkeeping mess)
- Multiple partial refunds summed incorrectly

## Taxes

### Sales tax

Marketplace sales tax is complicated. Most US states require marketplace facilitators to collect and remit.

Flag:
- No sales tax collection logic
- Tax rates hard-coded (rates change; jurisdictions proliferate)
- Origin-based vs destination-based tax treated uniformly (varies by state)
- No nexus tracking (when does platform owe tax in a new state?)
- Missing exempt categories (most states exempt groceries, clothing, etc. differently)
- No API integration with tax service (Avalara, TaxJar, Anrok) at scale

### Income reporting

Flag:
- No 1099-K generation for sellers exceeding thresholds (in 2026: $2,500 federal; some states lower)
- No TIN collection for sellers
- Year-end reporting delayed to the point sellers can't file on time

## International

### Multi-currency

Flag:
- Single currency assumption (everything in USD)
- Currency conversion at display time without locked rate at transaction time
- Currency mismatches in ledger (buyer paid EUR, seller receives USD, but the commission stayed in EUR)
- FX cost absorbed silently by platform when it should be passed through

### Cross-border

Flag:
- No VAT/GST handling for international sales
- No customs / duty disclosure to buyer
- No compliance with Digital Services Act / VAT digital services rules for EU

## Ratings and reputation

Flag:
- Rating systems with no anti-abuse measures (fake reviews, rating bombing)
- Ratings that can be edited without history
- Self-rating possible (seller rates own listing)
- Rating aggregation math that's misleading (a 5-star from 1 review shouldn't outrank 4.8 from 100)
  - Use Wilson lower bound or Bayesian averages

## Notifications

Flag:
- Buyers and sellers notified at different moments (seller still sees "pending" after buyer got confirmation)
- Email notifications sent outside the transaction (sent twice on retry)
- No rate limits on notifications (spam vector for seller abuse)

## Process

For a marketplace codebase:

1. Map the money flow end-to-end: who pays what, who receives what, what the platform keeps.
2. Verify the math balances at every stage.
3. Check every state transition on a transaction: created → paid → fulfilled → settled → disputed? → resolved.
4. Walk role-based access for buyer and seller perspectives.
5. Check payout flows for hold, timing, failure handling.
6. Check dispute flows for fairness, audit, and refund correctness.

## Severity calibration

- **Critical**: payout calculations that don't balance; refund path that doesn't reverse ledger; cross-role data exposure
- **High**: missing KYC for payouts; no hold period for new sellers; dispute workflow that automatically sides with one party
- **Medium**: missing velocity limits; missing sales tax on new jurisdictions; rating manipulation possible
- **Low**: notification spam risk; search ranking quirks
