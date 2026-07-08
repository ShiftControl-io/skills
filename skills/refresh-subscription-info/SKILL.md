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
- The user wants to **delete or disable** an app. (This skill can flag an app as an *archive candidate* — see Step 6b — but never archives or deletes anything itself.)
- The invoices in question are for **non-SaaS purchases** (hardware, professional services, hosting unrelated to a tracked SaaS).
- The user wants to look at invoices they **sent** (outbound invoices to their own customers). This skill only inspects invoices the user **received**.

## Prerequisites

This skill requires two MCP servers to be installed and authenticated in the user's AI assistant:

1. **ShiftControl MCP** — at `https://mcp.shiftcontrol.io/mcp`. If not installed, direct the user to https://github.com/ShiftControl-io/skills/blob/main/INSTALL.md for their platform.
2. **An email-search MCP** — typically Gmail (Anthropic publishes one), but any MCP that exposes tools to list and read recent emails works. Common names: `gmail`, `outlook`, `imap`, `superhuman`.

Verify availability before starting the workflow:

- Confirm `list_apps` is callable (proves ShiftControl MCP is wired).
- Confirm an email-search tool is callable (proves the email MCP is wired).
- **Check attachment capability.** Many invoices carry the actual figures in a PDF attachment, not the email body (GitHub, JumpCloud, Zoom, Salesforce-billed Slack, etc. — see references/invoice-detection.md). Whether you can read those depends on the email MCP:
  - If the email MCP exposes an attachment tool that returns file **content or a download URL** (e.g. Superhuman's `get_attachment`), you can read PDF invoices — see Step 4.
  - If the email MCP returns attachment **filenames only** (the default Anthropic Gmail connector does this), you cannot read PDF invoices. Note this up front and tell the user which invoices you'll have to skip or ask them to provide manually. Do not silently miss them.
- If either MCP is missing, **stop** and tell the user which one to install.

## Workflow

### Step 1 — Establish the working set

Call `list_apps` (paginated — keep calling until you've enumerated all pages) to build an in-memory list of every app the user tracks, with their current values:

```
{ id, name, cost, costStructure, costCurrency, billingFrequency, contractEndDate, notes, owningDeptId, ... }
```

This is your authoritative "what's currently recorded" baseline.

**Data-integrity audit (do this now, before searching email).** As you enumerate apps, flag any app that has a **cost set but a blank `costStructure`**. In ShiftControl a cost with no cost structure is **not counted in spend totals** — the app silently drops out of the numbers even though a cost is recorded. Collect these into a "needs cost structure" list and surface them in the proposal (Step 7) so the user can fix them, even for apps you find no new invoice for. If you can infer the structure with high confidence from an invoice, propose it; if not, ask the user (see Step 4, cost structure). Never leave a costed app with a blank structure once you've touched it.

### Step 2 — Define the search window

By default, search the user's email for invoices received in the last **18 months**. Annual contracts are common in SaaS, and a one-year window risks missing the most recent renewal invoice for any app that bills annually — 18 months gives you the current annual invoice plus a buffer to confirm you have the latest one. If the user asks for a narrower window ("just the last quarter", "this month only"), honor it.

### Step 3 — Search the email inbox(es)

Use the email-search MCP to find candidate invoice emails. See [references/invoice-detection.md](references/invoice-detection.md) for the full heuristics — at minimum:

- Subject contains one of: `invoice`, `receipt`, `billing`, `subscription`, `renewal`, `payment confirmation`, `order confirmation`.
- From-address matches `billing@*`, `invoices@*`, `no-reply@*`, `accounts@*`, `finance@*`, `payments@*`, OR is a known SaaS vendor domain.
- Filter OUT emails where the user is the **sender** (those are outbound invoices to their own customers).
- Filter OUT **inbound-payments-received** — emails saying things like *"<Vendor>, Inc. has sent you a payment"* or *"Coupa Pay has remitted X to your account"*. These represent money coming TO the user (referrals, vendor-side payments) and are not SaaS invoices.
- Filter OUT clearly non-SaaS receipts (Amazon shopping, ride-share, restaurants, hardware, SSL certs, professional services, contractor invoices, telecom bills).

**Search across every mailbox you can reach, not just one.** Invoices for different apps often land in different places:

- **Shared billing inboxes** — most organizations route SaaS billing to a shared `invoices@<company>`, `billing@<company>`, or `finance@<company>` alias (a Google Group or distribution list). A search like `to:invoices@<company> newer_than:18m` is the highest-precision starting point. Ask the user which shared inbox(es) they use if you're unsure.
- **Individual owners** — some invoices are addressed to a specific person (`jordan@`, `chichen@`, the app's owner), not the shared alias. If ShiftControl records an owner for an app, or the user names likely recipients, search those addresses too.
- **Name the gap when you can't reach a mailbox.** The email MCP only sees the authenticated user's own and shared mailboxes — it cannot read a colleague's private inbox. When an app has no invoice in the mailboxes you *can* see and its invoice likely lives in one you *can't* (e.g. billed to a specific person), say so explicitly rather than reporting "not found": *"Equals may be billed to chichen@ — I can't see that mailbox. Forward the invoice or tell me the cost and I'll record it."*

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
- **Cost structure** — `user` (per user / per-seat) or `flat` (a **flat fee** for the whole account — i.e. **per contract**, not per user); `tiered` for usage brackets. Always describe these to the user as **"per user"** vs **"flat fee (per contract)"** — the bare word "flat" is unclear. The value stored in ShiftControl is still `user` / `flat` / `tiered`. **This field is MANDATORY whenever you propose a cost — see the rule below.**
- **Currency** — ISO 4217 code (USD, EUR, etc.).
- **Billing frequency** — `month` or `year`, inferred from the service period. **Full invoices only.** Do NOT emit `quarter` — see the cadence rule below.
- **Total seats after the change** — useful for both types when stated ("Your subscription now includes 50 seats").
- **Contract renewal/end date** — if mentioned ("renews on…", "auto-renews", "contract through…"). Incrementals often confirm or update this.
- **Plan/tier** — the plan name on the invoice ("Pro", "Business", "Enterprise"). If a recent incremental announces a plan change, trust it over an older full invoice.

**Cost structure is a MUST-fill field (do not leave it blank).** ShiftControl does not count a cost that has no cost structure — the app drops out of spend totals. So the cost record is **all-or-nothing**: whenever you propose a `cost`, you MUST also set `costStructure` and `costCurrency`. Never write a bare cost.
- If the invoice makes the structure clear (per-seat line items → `user`; a single fixed charge → `flat`, i.e. a flat fee per contract; usage brackets → `tiered`), set it.
- If it's genuinely ambiguous, **ask the user** — do not skip the field and do not guess silently: *"Is Notion charged per user, or a flat fee (per contract)? ShiftControl needs this to count the spend."*

**Billing cadence — `month` or `year` only.** ShiftControl's product currently supports monthly and annual cadences. Even though the API technically accepts `quarter`, a quarterly value doesn't render correctly in the product, so **do not emit `quarter`**. When an invoice is billed quarterly, convert it to the annual equivalent (quarterly amount × 4), set `billingFrequency = "year"`, and record the real cadence in the note: *"Billed quarterly; stored as annual equivalent."*

**Multi-plan apps.** Some apps run more than one plan at once — e.g. Figma with an annual base plan plus monthly seat overflow. ShiftControl holds a single cost + structure + cadence per app, so you cannot represent both lines faithfully. When you detect this, do NOT silently pick one: surface it in the proposal — *"Figma has two billing lines (annual base + monthly overflow). I can record one; which should be the tracked cost, or should I record the combined effective monthly?"* — and let the user decide.

**PDF attachments — read them when you can.** If the email body says "Your invoice is attached" (or simply lacks the figures) but a PDF is present, the data is in the PDF. Handle it per the attachment capability you checked in Prerequisites:
1. If the email MCP exposes an attachment tool that returns content or a download URL (e.g. `get_attachment`), fetch it, read the PDF, and extract the same fields as an inline invoice. (Most assistants read PDFs natively; if you get a URL, fetch and parse it.)
2. If the email MCP returns filenames only, you cannot read the PDF. Tell the user plainly: *"GitHub's amount is in a PDF attachment this email connector won't hand me. Paste the figure, forward the invoice, or connect an attachment-capable email tool."* Then treat the field as user-provided (Step 6a) or leave the app unchanged — never invent a number.

**Web-hosted invoices:** if the details are behind a "Click here to view your invoice" link, **ask the user** before following it: *"I found invoices from <vendor> where the details are behind a 'view invoice' link — should I follow those links to extract the cost details?"* If the user agrees AND your assistant can fetch URLs, fetch and parse. Otherwise mark uncertain.

If a candidate has an unclear vendor, no reliable extractable data, or is only an incremental for a vendor with no full invoice in the window, mark it **uncertain** and exclude that vendor's cost fields from the proposal. Surface it to the user separately.

### Step 5 — Match invoices to ShiftControl apps

For each parsed invoice, find the matching app from Step 1. See [references/vendor-name-mapping.md](references/vendor-name-mapping.md) for the full algorithm. Short version:

1. **Exact case-insensitive name match** — "Slack" matches "Slack".
2. **Normalized match** — strip "Inc.", "LLC", "Technologies", etc. and retry.
3. **Known-alias match** — "G Suite" / "GSuite" / "Google Workspace" all map to whichever the user has.
4. **Ambiguous** (multiple apps could match) — ask the user to disambiguate before proceeding with that one.

If no app matches, **skip the invoice** and surface it in a "found but not tracked" section of the proposal — **never auto-create a new app**.

### Step 6 — Build the diff

For each matched (invoice, app) pair, compute field-by-field which values differ between what's currently in ShiftControl and what the invoice shows. Only changed fields become candidate updates. Remember the cost-structure rule: if you're changing `cost`, ensure `costStructure` and `costCurrency` are part of the same proposed update.

### Step 6a — Apps with no invoice: offer manual entry

Some tracked apps won't have a findable email invoice at all — the vendor bills by card or portal only (e.g. HReasily), the invoice went to a mailbox you can't see, or the figure was in a PDF you couldn't read. For these, do NOT propose a change and do NOT zero the cost. Instead, offer the user a direct path: *"I couldn't find an invoice for HReasily. Tell me the cost, whether it's per user or a flat fee (per contract), the currency, and the cadence, and I'll record it in ShiftControl."* If the user provides the values, treat them exactly like extracted invoice values — show the before→after diff and write them back via `update_app_subscription` behind the same approval gate (Step 8/9). Note them as `"Updated from user-provided figures on <date>"`.

### Step 6b — Free, stopped-paying, and archive candidates

Classify each tracked app into one of: *still paying* / *now free* / *stopped paying* / *unknown*.

- **Now free** (e.g. the plan is genuinely free, or included at no cost in another plan): do NOT infer "free" merely from the absence of an invoice — that's how a real cost gets wrongly zeroed. **Confirm with the user** first: *"Attio looks like it's on a free plan now — set its cost to zero?"* When ShiftControl exposes a dedicated "mark as free" capability, use it; until then, on confirmation set `cost = "0.00"` with `costStructure` still set and a note explaining the free plan.
- **Stopped paying** (the user no longer uses/pays for the app, e.g. a cancelled ChatGPT plan): surface as an **archive candidate** in a dedicated section of the proposal — *"You indicated you stopped paying for ChatGPT — consider archiving it in ShiftControl."* **Never archive or delete anything yourself** (archiving isn't in this skill's MCP surface); just flag it for the user to action in the UI.
- **Free-then-paid transition** (e.g. a first-year-free plan that has now started charging, like Granola): detect the first paid invoice after the free period and propose the new cost + a note recording the transition.

### Step 7 — Present the proposal

Show the user a structured proposal. See [references/proposal-format.md](references/proposal-format.md) for the exact format, including the sections for **cost-structure fixes** (from the Step 1 audit), **archive candidates** (Step 6b), and **no-invoice / manual-entry** apps (Step 6a). The high-level shape:

```
Found invoices for 10 of your 23 ShiftControl apps. Proposed updates:

1. Slack
   cost:               $8.00/user/month  →  $7.00/user/month
   costStructure:      user  (unchanged — kept so the cost still counts)
   billingFrequency:   month             →  year
   contractEndDate:    (not set)         →  2027-03-15
   notes update:       "Updated from Slack invoice dated 2026-03-15 (billed via Salesforce)"

2. Notion
   cost:               $12.00/user/month →  $10.00/user/month
   notes update:       "Updated from Notion invoice dated 2026-03-08"

Needs cost structure (won't count in spend until fixed) (2):
  - Documenso: cost $9.00 recorded but structure is blank — per user or flat fee (per contract)?
  - Aspire: cost $5.00 recorded but structure is blank — per user or flat fee (per contract)?

Archive candidates (1): ChatGPT — you said you stopped paying.
Apps with no invoice found (3): [list] — reply with a cost to record any of them.
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
- `for Framer, estimate the annual cost from the proration` — the **opt-in proration estimate**. By default the skill does NOT derive a price from a prorated invoice. But if the user explicitly asks, compute the annualized figure from the prorated amount and billing period, propose it clearly **labelled as an estimate** (`"~S$652/year, estimated from a prorated invoice"`), and only write it on approval.

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
- **Only the fields that actually changed** (omit unchanged ones — they keep their current value on the backend), **except** that whenever `cost` is in the change set you MUST also send `costStructure` and `costCurrency` so the cost is counted.
- `notes`: write a single line in the form `"Updated from <Vendor> invoice dated <YYYY-MM-DD>"`. If the invoice came via a reseller (e.g. Slack billed by Salesforce, Google Workspace billed by ShiftControl), append `(billed via <Reseller>)` — e.g. `"Updated from Slack invoice dated 2026-03-15 (billed via Salesforce)"`. For user-provided figures use `"Updated from user-provided figures on <date>"`. **Replace** any previous skill-written line of the same form (this skill runs repeatedly; we don't want notes to accumulate one line per run). Match for replacement using the pattern: line begins with `Updated from ` and contains `invoice dated <YYYY-MM-DD>` or `user-provided figures on`. **Preserve every other line** the user (or any other source) put in the notes field — only the skill's own previous "Updated from..." line gets replaced. If no such previous line exists, add the new one at the end.

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
- ❌ **Writing a `cost` without a `costStructure`.** A blank cost structure means ShiftControl ignores the cost entirely — the app silently drops out of spend totals. If you can't tell whether it's charged per user or as a flat fee (per contract), **ask the user**; never leave the field blank and never write a bare cost. (This reverses earlier guidance — cost structure is now mandatory, not optional.)
- ❌ Emitting `billingFrequency: "quarter"`. The product doesn't render it; convert quarterly invoices to the annual equivalent and note the real cadence.
- ❌ Inferring "free" from a missing invoice. Absence of an invoice is NOT evidence of a free plan. Confirm with the user before zeroing any cost.
- ❌ Wiping out user-written notes. The skill replaces ONLY its own previous "Updated from..." line. Any other content — vendor contact, negotiation history, owner email, manual annotations — must be preserved.
- ❌ Accumulating one new note line every time the skill runs. Replace the prior skill line, don't pile on.
- ❌ Proposing changes for apps where no invoice was found, based on "you probably renewed at the same rate". This skill is **invoice-driven**: no invoice → offer manual entry (Step 6a), don't guess.
- ❌ Creating new apps, or archiving/deleting apps. If an invoice doesn't match a tracked app, surface it as "not tracked". If an app is no longer paid for, surface it as an archive candidate — never act on either yourself.

## Version

**v0.2.0** — cost structure is now a mandatory field (blank structure = cost ignored by ShiftControl) with a data-integrity audit of existing apps; PDF invoices read via attachment-capable email tools with a graceful Gmail fallback; manual cost entry for apps with no email invoice; free / stopped-paying / archive-candidate handling; multi-mailbox search with explicit gap-naming; opt-in proration estimate; multi-plan-app surfacing; `quarter` cadence mapped to annual until the product supports it.

**v0.1.0** — email-based invoice discovery only.

## See also

- [references/invoice-detection.md](references/invoice-detection.md) — full email-search heuristics
- [references/vendor-name-mapping.md](references/vendor-name-mapping.md) — vendor-to-app matching algorithm
- [references/proposal-format.md](references/proposal-format.md) — exact format for the user-facing diff
