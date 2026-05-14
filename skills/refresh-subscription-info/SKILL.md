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
- Filter OUT clearly non-SaaS receipts (Amazon shopping, ride-share, restaurants, hardware).

### Step 4 — Extract structured data from each candidate

For each candidate invoice, read the body and extract:

- **Vendor** — the SaaS company billing for the service (From-address domain or body header).
- **Invoice date** — when the invoice was issued.
- **Service period** — what the charge covers (e.g. "Jan 1 – Jan 31, 2026").
- **Per-unit cost** — per-user or per-seat cost, as a decimal string (e.g. `"5.00"`).
- **Cost structure** — `user` (per-seat), `flat` (fixed), or `tiered`.
- **Currency** — ISO 4217 code (USD, EUR, etc.).
- **Billing frequency** — `month`, `quarter`, or `year`, inferred from the service period.
- **Total amount + seats** — useful for cross-checking per-unit cost.
- **Contract renewal/end date** — if mentioned ("renews on…", "auto-renews", "contract through…").
- **Plan/tier** — the plan name on the invoice ("Pro", "Business", "Enterprise").

If a candidate's vendor is unclear, has no extractable cost, or otherwise doesn't yield enough structured data, mark it **uncertain** and exclude it from the proposal. Surface it to the user separately: "I saw this invoice but couldn't extract enough to propose an update."

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
   note will be added: "Updated from Slack invoice dated 2026-03-15"

2. Notion
   cost:               $12.00/user/month →  $10.00/user/month
   note will be added: "Updated from Notion invoice dated 2026-03-08"

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
- `notes`: append a short line in the form `"Updated from <Vendor> invoice dated <YYYY-MM-DD>"` so the next person to look at the record can see where these values came from. Read the current notes from Step 1's snapshot and append; don't overwrite.

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
- ❌ Overwriting `notes` instead of appending. Read current notes; append the new line.
- ❌ Proposing changes for apps where no invoice was found, based on "you probably renewed at the same rate". This skill is **invoice-driven**: no invoice → no change.
- ❌ Creating new apps. If an invoice doesn't match a tracked app, surface it as "not tracked" and stop there.

## Version

**v0.1.0** — email-based invoice discovery only. Future versions will add Xero, QuickBooks, Brex, Ramp, and other finance-system sources.

## See also

- [references/invoice-detection.md](references/invoice-detection.md) — full email-search heuristics
- [references/vendor-name-mapping.md](references/vendor-name-mapping.md) — vendor-to-app matching algorithm
- [references/proposal-format.md](references/proposal-format.md) — exact format for the user-facing diff
