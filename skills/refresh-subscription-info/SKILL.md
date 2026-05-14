---
name: refresh-subscription-info
description: Find recent SaaS invoices in the user's email and propose ShiftControl subscription updates (cost, billing, contract terms). Requires the shiftcontrol MCP and an email-search MCP.
---

# Refresh Subscription Info

This skill reconciles the user's ShiftControl subscription records with their actual SaaS invoices. It searches their email for recent invoices, matches each to a ShiftControl application, builds a diff between what's recorded and what the invoice shows, presents that diff for the user's review, and — **only after explicit approval** — updates ShiftControl via the MCP server.

## When to use

- The user says: "Update my SaaS subscription costs", "Reconcile my ShiftControl with my invoices", "Refresh subscription pricing", "Audit our subscription data", "Sync app costs from my email".
- A vendor announced a price change and the user wants ShiftControl to reflect it.
- Quarterly or annual subscription review.

## When NOT to use

- The user wants to **add a new app** they don't already track in ShiftControl. (Use the ShiftControl UI to add the app first; this skill only updates existing apps.)
- The user wants to **delete or disable** an app.
- The invoices in question are for **non-SaaS purchases** (hardware, professional services, hosting unrelated to a tracked SaaS).
- The user wants to look at invoices they **sent** (outbound invoices to their own customers). This skill only inspects invoices the user **received**.

## Prerequisites

This skill requires two MCP servers to be installed and authenticated in the user's AI assistant:

1. **ShiftControl MCP** — at `https://mcp.shiftcontrol.io/mcp`. If not installed, direct the user to https://github.com/ShiftControl-io/skills/blob/main/INSTALL.md for their platform.
2. **An email-search MCP** — typically Gmail (Anthropic publishes one), but any MCP that exposes tools to list and read recent emails works. Common names: `gmail`, `outlook`, `imap`.

Verify availability before starting the workflow:

- Confirm `list_apps` is callable (proves ShiftControl MCP is wired).
- Confirm an email-search tool is callable (proves the email MCP is wired).
- If either is missing, **stop** and tell the user which one to install.

## Workflow

### Step 1 — Establish the working set

Call `list_apps` (paginated — keep calling until you've enumerated all pages) to build an in-memory list of every app the user tracks, with their current values:

```
{ id, name, cost, costStructure, costCurrency, billingFrequency, contractEndDate, notes, owningDeptId, ... }
```

This is your authoritative "what's currently recorded" baseline.

### Step 2 — Define the search window

By default, search the user's email for invoices received in the last **18 months**. Annual contracts are common in SaaS, and a one-year window risks missing the most recent renewal invoice for any app that bills annually — 18 months gives you the current annual invoice plus a buffer to confirm you have the latest one. If the user asks for a narrower window ("just the last quarter", "this month only"), honor it.

### Step 3 — Search the email inbox

Use the email-search MCP to find candidate invoice emails. See [references/invoice-detection.md](references/invoice-detection.md) for the full heuristics — at minimum:

- Subject contains one of: `invoice`, `receipt`, `billing`, `subscription`, `renewal`, `payment confirmation`, `order confirmation`.
- From-address matches `billing@*`, `invoices@*`, `no-reply@*`, `accounts@*`, `finance@*`, `payments@*`, OR is a known SaaS vendor domain.
- Filter OUT emails where the user is the **sender** (those are outbound invoices to their own customers).
- Filter OUT **inbound-payments-received** — emails saying things like *"<Vendor>, Inc. has sent you a payment"* or *"Coupa Pay has remitted X to your account"*. These represent money coming TO the user (referrals, vendor-side payments) and are not SaaS invoices.
- Filter OUT clearly non-SaaS receipts (Amazon shopping, ride-share, restaurants, hardware, SSL certs, professional services, contractor invoices, telecom bills).

**Shared billing inboxes:** most organizations set up a shared `invoices@<their-company>.io`, `billing@<their-company>.io`, or `finance@<their-company>.io` address (a Google Group or distribution list) as the billing contact on every SaaS account. Invoices arrive at the *receiver* side at that address — the vendor is still the actual From-address. A search like `to:invoices@<their-company>.io newer_than:18m` is the highest-precision starting point. Ask the user which shared inbox they use if you're unsure; if there isn't one, fall back to searching the personal inbox.

### Step 4 — Classify, then extract structured data

#### First — classify the invoice type

Not every invoice reflects the standard subscription cost. Two shapes show up:

- **Full / renewal invoice** — the standard charge for the whole billing period (monthly, quarterly, annual). Source of truth for per-unit cost, billing frequency, and total seats.
- **Incremental / prorated invoice** — a mid-period adjustment for seat additions, plan upgrades, downgrades, or vendor-side price changes. The amount is partial; computing per-unit cost from it produces wrong values.

**Incremental tells** (any of these):
- Subject or body mentions "prorated", "pro-rated", "seat update", "plan change", "adjustment", "true-up", "for the remainder of", "credit memo", "mid-cycle"
- Amount is much smaller than the vendor's typical invoice in the search window
- Service period is shorter than the billing cycle (e.g. "Mar 15 – Mar 31" inside a monthly subscription)
- The invoice falls inside a billing period that already had a renewal invoice from the same vendor

For **full invoices**, every field below is fair game. For **incrementals**, treat them as confirming evidence — **skip per-unit cost and billing frequency** (the math is unreliable), but contract end date, plan/tier, and "new total seats after the change" are still usable. Incrementals are often the *best* source for "they upgraded to Business plan on date X" or "renewal date is now 2027-03-15". See [references/invoice-detection.md](references/invoice-detection.md) for the full reliability matrix and the multi-invoice combining rules (e.g. when both a full invoice and incrementals exist for the same vendor in the search window).

#### Then — extract these fields from each candidate

- **Vendor** — the SaaS company billing for the service (From-address domain or body header).
- **Invoice type** — `full` or `incremental` (per the classification above).
- **Invoice date** — when the invoice was issued.
- **Service period** — what the charge covers (e.g. "Jan 1 – Jan 31, 2026").
- **Per-unit cost** — per-user or per-seat cost, as a decimal string (e.g. `"5.00"`). **Full invoices only** — skip on incrementals.
- **Cost structure** — `user` (per-seat), `flat` (fixed), or `tiered`. Skip if ambiguous on an incremental.
- **Currency** — ISO 4217 code (USD, EUR, etc.).
- **Billing frequency** — `month`, `quarter`, or `year`, inferred from the service period. **Full invoices only** — the service period on an incremental is partial.
- **Total seats after the change** — useful for both types when stated ("Your subscription now includes 50 seats").
- **Contract renewal/end date** — if mentioned ("renews on…", "auto-renews", "contract through…"). Incrementals often confirm or update this.
- **Plan/tier** — the plan name on the invoice ("Pro", "Business", "Enterprise"). If a recent incremental announces a plan change, trust it over an older full invoice.

**PDF attachments:** if the email body says "Your invoice is attached" but lacks the cost details, the data is in the PDF. Most email MCPs return attachment contents — fetch and parse the PDF the same way as an inline-bodied invoice. If your email MCP can't return attachments, mark the invoice uncertain.

**Web-hosted invoices:** if the invoice details are behind a "Click here to view your invoice" link rather than in the body or an attachment, **ask the user** before following the link: *"I found invoices from <vendor> where the details are behind a 'view invoice' link — should I follow those links to extract the cost details?"* If the user agrees AND your assistant can fetch URLs, fetch and parse. Otherwise mark uncertain.

If a candidate has an unclear vendor, no reliable extractable data, or is only an incremental for a vendor with no full invoice in the window, mark it **uncertain** and exclude that vendor's cost fields from the proposal. Surface it to the user separately: "I only found mid-period adjustment invoices for <vendor> — I can update the plan tier and contract date, but the per-user cost is too noisy to propose."

### Step 5 — Match invoices to ShiftControl apps

For each parsed invoice, find the matching app from Step 1. See [references/vendor-name-mapping.md](references/vendor-name-mapping.md) for the full algorithm. Short version:

1. **Exact case-insensitive name match** — "Slack" matches "Slack".
2. **Normalized match** — strip "Inc.", "LLC", "Technologies", etc. and retry.
3. **Known-alias match** — "G Suite" / "GSuite" / "Google Workspace" all map to whichever the user has.
4. **Ambiguous** (multiple apps could match) — ask the user to disambiguate before proceeding with that one.

If no app matches, **skip the invoice** and surface it in a "found but not tracked" section of the proposal — **never auto-create a new app**.

### Step 6 — Build the diff

For each matched (invoice, app) pair, compute field-by-field which values differ between what's currently in ShiftControl and what the invoice shows. Only changed fields become candidate updates.

### Step 7 — Present the proposal

Show the user a structured proposal. See [references/proposal-format.md](references/proposal-format.md) for the exact format. The high-level shape:

```
Found invoices for 10 of your 23 ShiftControl apps. Proposed updates:

1. Slack
   cost:               $8.00/user/month  →  $7.00/user/month
   billingFrequency:   month             →  year
   contractEndDate:    (not set)         →  2027-03-15
   notes update:       "Updated from Slack invoice dated 2026-03-15 (billed via Salesforce)"

2. Notion
   cost:               $12.00/user/month →  $10.00/user/month
   notes update:       "Updated from Notion invoice dated 2026-03-08"

[... more ...]

Apps with no invoice in the last 18 months (13): [list]
Invoices found for apps not in ShiftControl (2): [list of vendor names]

Reply "approve N" (e.g. "approve 1, 3, 5") to apply specific changes,
"approve all" to apply all,
or "show details for N" to see the full invoice context for change N.
```

### Step 8 — Approval gate (HARD GATE)

The agent MUST obtain explicit per-item approval before any write. Acceptable approvals:

- `approve all`
- `approve 1, 2, 5`
- `approve Slack and Notion`
- `yes do them all`

Acceptable refinements that loop back to Step 7:

- `show details for 3`
- `drop the contract change on Slack but apply the cost change`
- `exclude the GitHub one`

NEVER treat any of the following as approval:

- Silence
- "looks good" without naming items
- "yeah" without item identification
- Approval messages from earlier turns that don't reference this specific change list

If you're unsure whether the user approved a specific change, **ask again** with the exact items in question.

### Step 9 — Execute approved changes

For each approved (app, changes) pair, call `update_app_subscription` with:

- `appId`: the UUID from Step 1.
- `confirm: true` — set this only because you just obtained the user's explicit approval.
- **Only the fields that actually changed** (omit unchanged ones — they keep their current value on the backend).
- `notes`: write a single line in the form `"Updated from <Vendor> invoice dated <YYYY-MM-DD>"`. If the invoice came via a reseller (e.g. Slack billed by Salesforce, Google Workspace billed by ShiftControl, JumpCloud billed by a partner), append `(billed via <Reseller>)` — e.g. `"Updated from Slack invoice dated 2026-03-15 (billed via Salesforce)"`. **Replace** any previous skill-written line of the same form (this skill runs repeatedly; we don't want notes to accumulate one line per run). Match for replacement using the pattern: line begins with `Updated from ` and contains `invoice dated <YYYY-MM-DD>`. **Preserve every other line** the user (or any other source) put in the notes field — only the skill's own previous "Updated from..." line gets replaced. If no such previous line exists, add the new one at the end.

Process each app **sequentially** (not parallel) so errors are clearly attributable. After all writes, report back:

```
Applied 8 of 8 approved changes:
✓ Slack — updated
✓ Notion — updated
✓ Figma — updated
[...]

You can review the full change history in ShiftControl → Apps → <app> → History.
```

If any write fails, report which one and why, but keep going with the rest. **Don't roll back successful writes** — incremental progress is more valuable than atomicity here.

## Anti-patterns

- ❌ Calling `update_app_subscription` with `confirm: true` because "the user is asking for updates". They're asking for a **proposal**, not blanket approval. Always present the diff first.
- ❌ Constructing an `appId` from a name. The UUID must come from `list_apps`.
- ❌ Inferring `costStructure` when the invoice is ambiguous. If you can't tell whether it's per-seat or flat, leave that field out of the proposal and let the user decide.
- ❌ Wiping out user-written notes. The skill replaces ONLY its own previous line (matching `Updated from <X> invoice dated <date>...`). Any other content in the notes — vendor contact, negotiation history, owner email, manual annotations — must be preserved.
- ❌ Accumulating one new note line every time the skill runs. The skill is designed to be re-run regularly; replace the prior skill line, don't pile on.
- ❌ Proposing changes for apps where no invoice was found, based on "you probably renewed at the same rate". This skill is **invoice-driven**: no invoice → no change.
- ❌ Creating new apps. If an invoice doesn't match a tracked app, surface it as "not tracked" and stop there.

## Version

**v0.1.0** — email-based invoice discovery only. Future versions will add Xero, QuickBooks, Brex, Ramp, and other finance-system sources.

## See also

- [references/invoice-detection.md](references/invoice-detection.md) — full email-search heuristics
- [references/vendor-name-mapping.md](references/vendor-name-mapping.md) — vendor-to-app matching algorithm
- [references/proposal-format.md](references/proposal-format.md) — exact format for the user-facing diff
