# Invoice Detection Heuristics

Detailed rules for identifying SaaS invoices in the user's email inbox. The SKILL.md body covers the basics; this file is the deep reference Claude loads when it needs more.

## Search query templates

Adjust to the email-search MCP's syntax. For Gmail-style queries:

```
in:inbox newer_than:90d (subject:invoice OR subject:receipt OR subject:billing OR subject:subscription OR subject:renewal OR subject:"payment confirmation" OR subject:"order confirmation")
```

Refine when too noisy by adding excluded senders the user has explicitly flagged as not-SaaS.

## Signals that boost SaaS-invoice probability

- **From-address patterns:** `billing@*`, `invoices@*`, `no-reply@*`, `accounts@*`, `finance@*`, `payments@*`, `receipts@*`.
- **Known SaaS sender domains** (illustrative, not exhaustive): slack.com, github.com, notion.so, atlassian.com, figma.com, linear.app, vercel.com, supabase.com, mongodb.com, stripe.com (for itself), datadoghq.com, sentry.io, intercom.com, asana.com, monday.com, hubspot.com, salesforce.com, zendesk.com, docusign.com, dropbox.com, box.com, microsoft.com (M365), google.com (Workspace), zoom.us, miro.com, loom.com, calendly.com, lattice.com, gusto.com, rippling.com, deel.com, ramp.com, brex.com, pleo.com, mercury.com.
- **Body content signals:** "Subscription renewed", "Plan: <name>", "Per seat", "Per user", "Invoice number", "PDF receipt attached", "Receipt for your payment", "Auto-renewal scheduled".
- **Recurring sender:** the user has received multiple emails from the same address over time — suggests an ongoing subscription.

## Signals that decrease probability (exclude these)

- **User is the sender** (outbound invoice to a customer) — typically in the Sent folder, or with From-address being the user's own company.
- **One-time purchase receipts** without recurrence — Amazon, eBay, Apple in-app purchases, single-item digital purchases.
- **Non-SaaS B2B receipts** — hardware orders, freelancer/consultant invoices, marketing services (unless those are tracked apps).
- **Notifications dressed as receipts** — "Your free trial is ending" without a real charge.
- **Marketing emails using "subscription"** in a non-billing sense — newsletter subscription confirmations.

## Time window guidance

- **Default: 18 months** — covers every billing cadence (monthly, quarterly, annual) plus enough buffer to catch the most recent annual renewal invoice. Annual contracts dominate in SaaS; a window shorter than 12 months risks missing the latest invoice for half the user's apps.
- **Narrower windows when the user asks:** "this month only", "last quarter", "since the last review" — honor whatever the user specifies. A monthly review of just the most recent month is a legitimate workflow.
- **Wider than 18 months:** rare. Old invoices are usually superseded by more recent ones, and pulling beyond 18 months risks proposing changes based on stale pricing. Only extend if the user explicitly asks (e.g. "go back two years to find the original contract").

## What to do when the inbox is huge

Most real inboxes are noisy. To keep within reasonable LLM context:

- Query in tighter time windows (e.g. last 30 days) and iterate.
- Use the email-search MCP's filter capabilities before reading bodies.
- Read only the first ~1000 characters of each candidate; pull the full body only if structured fields aren't already extracted.

## Full vs incremental invoices

A single vendor often produces multiple invoices in an 18-month window:

- A **full / renewal invoice** at the start of each billing period — the actual subscription charge for the upcoming period.
- One or more **incremental / prorated invoices** for mid-period changes — seat additions, plan upgrades, downgrades-with-credit, vendor-side price adjustments.

The two types carry very different reliability for the fields this skill cares about. Misclassifying a prorated charge as a full charge produces silently-wrong per-user costs in ShiftControl.

### Classification signals

**Incremental invoice tells**:

- Subject line contains: "Seat update", "Plan change", "Adjusted billing", "Prorated charge", "Account update", "True-up", "Mid-cycle adjustment", "Credit memo", "Upgrade confirmation"
- Body text contains: "prorated", "pro-rated", "for the remainder of", "partial billing period", "partial month", "adjustment", "true-up", "added X seats", "upgraded to", "downgraded from"
- Amount is significantly smaller than the vendor's typical invoice in the search window
- Service period is shorter than the billing cycle (e.g. "Mar 15 – Mar 31" inside a monthly subscription, or "Mar – Aug" inside an annual one)
- The invoice falls inside a billing period that already had a renewal invoice from the same vendor

**Full / renewal invoice tells**:

- Subject: "Invoice", "Subscription renewed", "Receipt for your payment", "Auto-renewal", "Annual subscription"
- Body shows a service period equal to the full billing cycle (full month / full quarter / full year)
- Amount aligns with `per-unit cost × seats × billing-frequency`
- No "prorated" / "adjustment" / "credit" language

### Field-extraction reliability matrix

| Field | Full invoice | Incremental |
|---|---|---|
| Vendor | ✅ reliable | ✅ reliable |
| Invoice date | ✅ reliable | ✅ reliable |
| Per-unit cost | ✅ reliable | ❌ **DO NOT use** — prorated math will be wrong |
| Cost structure (user / flat / tiered) | ✅ reliable | ⚠️ OK if explicitly stated |
| Currency | ✅ reliable | ✅ reliable |
| Billing frequency | ✅ derivable from service period | ❌ service period is partial |
| Total seats (after this invoice) | ✅ reliable | ✅ reliable when stated ("Your subscription now includes N seats") |
| Contract end date | ✅ reliable when stated | ✅ reliable when stated — incrementals often confirm renewal date |
| Plan / tier | ✅ reliable | ✅ **best source** — plan changes are exactly what incrementals announce |
| Net seat change (+3, -5) | — not on full invoices | ✅ uniquely available on incrementals |

### Combining multiple invoices from the same vendor

If you find both a full invoice AND incrementals for the same vendor in the search window:

1. Use the **full invoice** as the source for per-unit cost, billing frequency, currency, cost structure.
2. Use the **most recent incremental** for current seat count (more up-to-date than the full invoice).
3. Use either type for contract end date — both reliably state it when present; prefer the most recent mention.
4. If they disagree on plan tier (full says "Pro", a later incremental says "upgraded to Business"), trust the most recent incremental.

If you find **only incrementals** for a vendor in the search window, treat it as low-confidence:

- Skip the per-unit cost field in the proposal for that vendor.
- Still propose plan tier and contract end date updates if you found them.
- Surface a note in the proposal: *"Only mid-period adjustment invoices for <vendor> in the window — I can update plan tier and contract date, but the per-user cost is too noisy to propose."*

### Annual-vs-monthly cost normalization

A separate but adjacent gotcha: full invoices can be quoted at different cadences. An annual invoice may show `$84/user/year`, which is `$7/user/month billed annually`. The skill normalizes:

- Invoice says `$84/user` over a 1-year service period → store `cost = "7.00"`, `billingFrequency = "year"`, proposal shows it as `"$7.00/user/month billed annually"`.
- Invoice says `$8/user` over a 1-month service period → store `cost = "8.00"`, `billingFrequency = "month"`.

ShiftControl stores per-unit cost AND billing frequency separately, so the normalization always preserves both pieces. Don't multiply / divide cost without also updating billingFrequency.

## Other failure modes worth flagging

- **Vendor consolidator** — emails from Stripe billing the user on behalf of MANY upstream SaaS vendors. The "From" is Stripe but the actual vendor is in the body. Look for "On behalf of" or "Pay <vendor>".
- **Reseller invoicing** — organizations that buy SaaS through a reseller (CDW, Insight, Carahsoft, etc.). Vendor on the invoice is the reseller; the actual SaaS product is in line items.
- **Currency conversion** — invoice in USD but user's org defaults to EUR/SGD. Surface the invoice currency; don't auto-convert. Let the user decide whether to store in invoice currency or org default.
- **PDF-only invoices** — the email body is "Your invoice is attached" with no structured data; the actual numbers are in the PDF. v0.1.0 does NOT parse PDF attachments — mark these as uncertain.
- **Web-hosted invoices** — "Click here to view your invoice" with the actual numbers behind a link. v0.1.0 does NOT follow these links — mark these as uncertain.

## What this skill does NOT do (yet)

These are explicitly out-of-scope for v0.1.0 and tracked for v0.2.0+:

- Parse PDF attachments
- Follow links to web-hosted invoices
- Integrate with Xero, QuickBooks, Brex, Ramp, or other finance systems
- Multi-currency normalization
- Automatic vendor → ShiftControl-app addition (creating apps from invoices)
