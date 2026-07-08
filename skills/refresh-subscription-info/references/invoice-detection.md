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

**Cadence is `month` or `year` only — no `quarter`.** ShiftControl's product currently renders monthly and annual cadences; a `quarter` value (which the API technically accepts) does not display correctly. When an invoice is billed quarterly, convert it to the annual equivalent (quarterly amount × 4), store `billingFrequency = "year"`, and record the true cadence in the note (e.g. *"Billed quarterly; stored as annual equivalent"*). Revisit this once the product adds more cadences.

**Cost structure is mandatory whenever a cost is written.** In ShiftControl, a cost with a blank `costStructure` is **not counted in spend totals** — the app silently drops out of the numbers. So treat `cost` + `costStructure` + `costCurrency` as an all-or-nothing set: never propose a bare cost. Full invoices reliably reveal the structure (per-seat line items → `user`, presented as "per user"; a single fixed charge → `flat`, presented as "flat fee (per contract)"; usage brackets → `tiered`). When it's genuinely ambiguous, ASK the user rather than leaving it blank. Always use the plain labels "per user" and "flat fee (per contract)" when talking to the user — the bare word "flat" is unclear — while still storing the `flat` value. This also means a useful standalone audit: any existing app that already has a cost but a blank structure should be flagged for the user to fix, because its spend isn't being counted today.

## Vendor identity beyond the "From" address

The vendor billing the user isn't always the vendor named on the invoice. Two big patterns:

### Vendor renames and acquisitions

When a SaaS company is acquired, billing often moves to the parent. The product name in ShiftControl stays the same; the invoice arrives from a different domain.

| Product | Bills as | Notes |
|---|---|---|
| Slack | Salesforce (`noreply@salesforce.com`, `slackinvoices@salesforce.com`) | Slack was acquired by Salesforce in 2021; invoices migrated. The body still says "Slack" — only the From address changed. |
| Heroku | Salesforce | Same story (Salesforce-Heroku acquisition). |
| Figma | Adobe (post-acquisition close) | Watch for this if/when the acquisition closes. |
| MongoDB Atlas | mongodb.com (no change) | Listed because it's the most common "is this really MongoDB?" question — yes, Atlas billing is direct from mongodb.com. |

When matching to a ShiftControl app, **rely on the body content** (subject line, "Pay <Slack Technologies>" line, product names in line items) over the From address for these cases. Annotate the note with `(billed via <Parent>)` so the user can see why the invoice came from an unexpected sender.

### Resellers (bundled and consolidator billing)

A reseller buys SaaS in bulk and re-bills the customer. The invoice is from the RESELLER; the actual SaaS product is in line items. Examples:

- **ShiftControl itself** — for Google Workspace and JumpCloud customers, ShiftControl bills the customer directly; the customer doesn't see Google or JumpCloud invoices. The ShiftControl invoice line items name the underlying product.
- **CDW, Insight, Carahsoft, Crayon, SoftwareOne** — large IT resellers; one invoice may cover dozens of SaaS line items.
- **Stripe (as consolidator)** — Stripe sometimes bills the customer on behalf of multiple upstream SaaS vendors. Look for "On behalf of <Vendor>" or "Pay <Vendor>" in the body.

When the invoice is from a reseller, the skill:

1. Reads the line items to find which ShiftControl-tracked apps the invoice covers.
2. Creates a SEPARATE proposed update per app (one ShiftControl invoice covering Google Workspace + JumpCloud → two diff blocks in the proposal, one per app).
3. Annotates the note with `(billed via <Reseller>)` for each affected app — e.g. `"Updated from Google Workspace invoice dated 2026-03-15 (billed via ShiftControl)"`.

If a reseller invoice has line items that don't match any ShiftControl app, treat each unmatched line the same way an unmatched standalone invoice would be — surface to the user as "found a Adobe Creative Cloud line on the ShiftControl invoice, but you don't track Adobe in ShiftControl yet."

## Shared billing inboxes (the `invoices@` / `billing@` / `finance@` pattern)

Many organizations set up a **shared billing inbox** — typically a Google Group or distribution list at an address like `invoices@<company>.io`, `billing@<company>.io`, or `finance@<company>.io`. They then configure each SaaS subscription's billing-contact email to point at that address, so all renewal / invoice mail lands in one place where finance can see and act on it.

This means the **To-address** is the shared inbox, not the From-address. The From-address remains the actual vendor (`feedback@slack.com`, `noreply@github.com`, `team@mail.notion.so`, etc.). The skill should treat this pattern as the default, not the exception — most ShiftControl customers will run a shared billing inbox.

### How shared inboxes appear via email-search MCPs

How the email surfaces in the search depends on how the user reads the group's mail:

- **Reading the group's own mailbox directly** (if the email-search MCP authenticates against `invoices@<company>.io` as a mailbox): From-address is the vendor, exactly as expected.
- **Reading a personal mailbox that's a group member**: Gmail's API sometimes shows the GROUP address as the sender for messages delivered through the group. The actual vendor is still recoverable from the email's standard headers (Reply-To, Sender, body From-line, "Your receipt from <Vendor>" subject), but a naive From-header check can mislead.
- **Manual forwards from a personal inbox**: occasionally someone forwards a stray invoice into the group address. From is the forwarder; the original vendor is in the forwarded body.

### Practical guidance

- **Don't rely solely on the From-header to identify the vendor.** Cross-check the subject, the body's vendor name / logo, the Reply-To, and PDF attachment filenames (e.g. `Salesforce_Invoice_36849376.pdf` makes the product context obvious even when the body is sparse).
- If the From-address is the user's own org alias (`invoices@<their-company>.io`) and the body header / subject names a recognizable SaaS vendor, treat the body as authoritative.
- A search like `to:invoices@<their-company>.io newer_than:18m` is a high-precision starting point — the user has explicitly routed billing to that address, so almost every result is invoice-shaped (with the false-positive classes documented above still applying).

## False positives — outbound and unrelated emails

The dedicated invoice mailbox typically collects more than just SaaS subscription invoices. Filter these OUT:

- **Outbound payments / referrals received** — emails where the user's organization is RECEIVING money rather than paying. Common phrases: `"<Vendor>, Inc. has sent you a <amount> payment"`, `"You received a payment of <amount>"`, `"Coupa Pay has remitted <amount> to your account"`. Workato's referral payments to ShiftControl partners are a textbook example. These look structurally like invoices but represent the OPPOSITE direction of money.
- **Customer payments to YOU** — emails from your own billing system (Stripe, Sequence HQ, etc.) confirming that one of YOUR customers paid YOU. Detect by: the From-domain belongs to the user's billing platform AND the body references *receiving* a payment from a customer.
- **Professional services** — accountant fees, lawyer invoices, contractor / freelancer invoices. These are real money out the door but are NOT SaaS subscriptions. Detect by: vendor doesn't appear in any tracked-app list and the body describes services rather than software access.
- **One-time purchases** — SSL certificates, domain registrations, hardware, marketing services. No recurring subscription relationship.
- **Telecom and utilities** — SIM cards, internet, electricity, office costs.
- **Bank, tax, regulatory** — bank statements, tax filings, government billings.

These all share a recognizable pattern: vendor doesn't match any tracked ShiftControl app, OR the email describes a one-off transaction rather than a subscription. When in doubt, surface in the "found but not tracked" section of the proposal rather than guessing.

## Distributors and resellers — direction matters

In addition to the major resellers (CDW, Insight, Carahsoft, Crayon, SoftwareOne) listed earlier, real inboxes commonly include:

- **Ingram Micro** (`Imcloudservicedesk.hk@cloud.im`, also `*@ingrammicro.com`) — a large IT **distributor**. Subjects: `"Invoice <REGION>SI<HKSGUS>0000<NUMBER>"`, `"Credit Memo"`, `"Payment ... has been received"`, `"Your Account ... Was Put on Credit Hold"`.
- **AWS Marketplace** (`invoicing@aws.com`) — third-party SaaS subscriptions billed through AWS. Subject contains `"AWS Marketplace Statement"` or `"AWS Marketplace Billing Statement"`. Line items name the underlying SaaS product.

**Critical distinction — distributor invoices flow in both directions:**

Large distributors like Ingram Micro work as a two-way channel. The same email account can receive:

1. **Invoices for SaaS the user's organization consumes** (the org is the listed customer; this is a real subscription cost to reconcile).
2. **Invoices for SaaS the user's organization SELLS THROUGH the distributor** to its own downstream customers (the org is the channel / partner, not the consumer; the listed customer is a third party).

Only #1 should produce a proposed update in ShiftControl. #2 is a billing transaction on the user's own product, not a SaaS subscription they consume.

**Detection signal — the "Bill To" / "Customer" field:**

Parse the invoice (body or PDF) for the customer-side fields:

- `Bill To: <Org Name>`
- `Customer: <Org Name>`
- `Sold To: <Org Name>`
- `Account: <Org Name>` (less reliable — sometimes the distributor's own customer ID)

If the listed customer matches the user's organization → real subscription → proceed.
If the listed customer is someone else (a downstream company name unfamiliar to the user) → outbound channel sale → **filter out**. Surface in a "channel / resale invoices skipped" section of the proposal so the user knows you saw them but intentionally didn't act.

Credit Memos and payment-received notifications also fall under this filter — they're typically not subscription cost changes either way. Mention them in the skipped section so the user can confirm.

Credit Memos and payment-received notifications, even for direct consumption: don't propose changes from these. They modify a balance, not a per-unit cost. Surface as informational only.

## Other failure modes worth flagging

- **Currency conversion** — invoice in USD but user's org defaults to EUR/SGD. Surface the invoice currency; don't auto-convert. Let the user decide whether to store in invoice currency or org default.
- **PDF invoices** — many SaaS vendors send the invoice details as a PDF attachment with only a short summary in the email body ("Your invoice is attached"). Whether you can read the PDF depends on the email MCP's attachment capability, which varies a lot — **check it before assuming**:
  - **Returns attachment content or a download URL** (e.g. Superhuman's `get_attachment`, which returns a short-lived download URL for PDFs): fetch it, read the PDF (most assistants read PDFs natively; if you get a URL, fetch and parse it), and extract the same fields (vendor, amounts, service period, seats, plan tier) as you would from an inline body. This is the path that lets you read GitHub/JumpCloud/Salesforce-billed PDF invoices.
  - **Returns attachment filenames only** — the default Anthropic Gmail connector does this: it gives you the filename but not the bytes or a link. In this case you genuinely **cannot** read the PDF. Do not guess the amount. Tell the user which invoices are affected and offer the manual-entry path, or suggest connecting an attachment-capable email tool (Superhuman is verified to work). This is not a skill bug — it's a connector limitation.
  - Note: often the figure is *also* in the email body (GitHub receipts list the amount in the body text), so always parse the body thoroughly first; fall to the PDF only when the body lacks the numbers.
- **Web-hosted invoices** — "Click here to view your invoice" with the actual numbers behind a link rather than in the body or attachment. **Ask the user before following the link** — most will say yes, some prefer not to follow links from their inbox. Phrasing: *"I found invoices from <vendor> where the details are behind a 'view invoice' link — should I follow those links to extract the cost?"* If the user agrees and your assistant has a web-fetch capability, fetch and parse. Otherwise mark uncertain.

## What this skill does NOT do (yet)

These are explicitly out-of-scope for v0.1.0 and tracked for v0.2.0+:

- Integrate with Xero, QuickBooks, Brex, Ramp, or other finance systems (v0.2.0 adds these as alternate sources alongside email)
- Multi-currency normalization (skill surfaces the invoice currency; user decides whether to store in invoice currency or org default)
- Automatic vendor → ShiftControl-app addition (skill never creates apps from invoices; only updates existing ones)
