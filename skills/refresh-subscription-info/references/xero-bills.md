# Xero as an Invoice Source

Xero is a better source of subscription cost than email, and where a user has it you should prefer it. The bills are already normalised, already deduplicated, already attached to a contact per vendor, and often carry line items. You are reading an accounting system's structured record instead of reverse-engineering a marketing team's HTML.

It is not a *complete* source, though, and the gap matters: Xero knows what you paid, and frequently does not know what you paid it *for*. Seats, plan tier, per-user versus flat, and contract end date usually live in the invoice PDF or the email, not in the accounting record. So the right shape is Xero for the money and the dates, email for the shape of the subscription, and the two reconciled in Step 4b of the workflow.

## Bills vs invoices — get this right first

Xero models both directions of money with the same object, separated by a type field:

- **`ACCPAY`** — a **bill**. Money you owe a supplier. This is what this skill wants.
- **`ACCREC`** — a **sales invoice**. Money a customer owes you. Never relevant here, and proposing a subscription cost from one would be a serious error.

How that surfaces depends on which Xero MCP server the user has, and there are two in the wild with different conventions.

### Shape A — the hosted connector (`https://mcp.xero.com`)

Splits the two directions into separate tools, so you never filter by type yourself. Observed tool surface as of September 2026:

| Tool | What it returns |
|---|---|
| `get_bills` | Accounts-payable bills (`ACCPAY`). **This is the one you want.** |
| `get_invoices` | Accounts-receivable only. Asking it for bills returns nothing useful. |
| `get_contacts` | Suppliers and customers, each with a contact ID you can filter bills by. |
| `get_repeating_bills` | Repeating (template) bills, when the organisation uses them. |
| `get_aged_payables` | Outstanding payables summarised by contact. |

### Shape B — the self-hosted official server (`@xeroapi/xero-mcp-server`, run via npx)

Exposes one unified tool and expects you to constrain the type yourself. Relevant tools from its published surface: `list-invoices`, `list-contacts`, `list-credit-notes`, `list-payments`, `list-aged-payables-by-contact`, `list-organisation-details`.

With this shape, **`list-invoices` returns both directions**. Constrain to `ACCPAY` explicitly, and sanity-check the result: if the "vendors" you get back are your own customers, you are reading receivables.

### Don't trust either list — enumerate

Tool names and parameters change. Read the tool surface you actually have, find the one that returns payables, and adapt. The two shapes above are what to expect, not a contract. If you can see only one invoice-shaped tool, assume Shape B and filter by type.

## Four traps, all of them silent

These produce wrong numbers rather than errors, which is what makes them worth writing down.

### 1. The default date window is far too short

The hosted connector's bills tool defaults to roughly **the last month**. The skill's default search window is 18 months, for the good reason that annual contracts dominate SaaS — and an annual vendor's renewal bill is, by definition, almost never in the last month.

**Always pass an explicit start date.** A run that accepts the default window will report "no bill found" for most of the user's annually-billed apps and look like it worked.

### 2. Results are paginated, and the first page looks complete

Around 30 records per page on the hosted connector. Keep paging until you have the whole window. A single page of bills from a busy organisation covers a few weeks.

### 3. Zero-total bills are accounting entries, not free subscriptions

This one bites hard, and the fix is not the obvious one. Where an organisation amortises annual prepayments, a large share of bills come back with a header total of **0.00** and a status of **PAID**, with references shaped like `Prepayment - Annual - INV<number> - <period>`.

They are not empty. Each is a paired double entry: one line posting the month's share to an expense account, and an equal negative line releasing it from a prepayment asset account. The two net to zero at the header, which is why the header is worthless and **the line items are not**.

Two rules, and you need both:

> **Never derive a subscription cost from a bill's header total when that total is 0.00, and never treat a zero total as evidence of a free plan.**
>
> **Do read the line items anyway.** On an amortisation bill the positive line amount is the vendor's cost for that period, which is often exactly the monthly figure you want.

So when you meet a run of zero-total bills, request line items rather than skipping the vendor. Take the positive line, note that it is an amortised period share rather than the invoiced amount, and say so in the proposal. If the line items are also uninformative, widen the window to find the original non-zero bill, and if you cannot, tell the user plainly — *"every <Vendor> bill in this period is a 0.00 amortisation entry; I can read the monthly share off the line items, but the original invoice is outside the window. Worth confirming with whoever does your books."*

### 4. A bill total alone never establishes cost structure

A bill for $250.00 with no line-item breakdown is **not** evidence of a flat fee. It is equally consistent with 50 seats at $5.00. ShiftControl ignores a cost with no `costStructure`, so you must fill the field — and Xero, on its own, frequently cannot tell you what to fill it with.

Resolve in this order:

1. **Line items — and read the description, not just the numbers.** Request them where the tool supports it. A line with a quantity of 50 and a unit amount of 5.00 gives you `costStructure: user` and the seat count directly. **But do not trust `quantity` on its own.** On amortisation and journal-style bills the quantity is `1.0` on every line regardless of seats, and the real detail sits in the free-text description: `Team Plan Annual (2 seats) - Amortisation (Jul 2026)` carries the plan name, the seat count and the period, none of which appear in a numeric field. Parse the description for seat and plan language before concluding the record is silent on structure. A quantity of `1.0` means "one line", not "one seat".
2. **The matching email invoice.** The vendor's own invoice usually states seats and plan.
3. **Ask the user.** *"Xero shows $250/month to <Vendor> but no per-seat breakdown — is that per user or a flat fee (per contract)?"*

Never divide a total by a seat count you assumed. If you cannot establish the structure, propose the other fields and leave cost out, saying why.

## What to read off a bill

| Xero field | Maps to | Notes |
|---|---|---|
| Contact (name + ID) | Vendor → app matching | Run it through `vendor-name-mapping.md` exactly like an email sender. |
| Type | Direction filter | `ACCPAY` only. |
| Status | Confidence | `AUTHORISED` and `PAID` are real. `DRAFT` and `SUBMITTED` are provisional — don't propose from them. `VOIDED` / `DELETED` are not charges at all. |
| Date (issued) | Invoice date | Drives the note line and the pick-the-latest logic. |
| Subtotal / Tax / Total | Cost | **Use the tax-exclusive subtotal.** ShiftControl records the subscription price; GST/VAT is a tax position, not a price change. Recording the tax-inclusive total silently inflates every cost in a taxed jurisdiction. |
| Currency code | `costCurrency` | Store the bill's own currency. Never convert — surface it and let the user decide. |
| Line items (description, quantity, unit amount) | Cost structure + seats | The most valuable field on the record when present. |
| Reference / bill number | Note provenance | Useful when the vendor's own invoice number is embedded in it. |

### Repeating bills and aged payables

**Repeating bills**, where the organisation maintains them, are direct evidence of billing cadence — a monthly template is a monthly subscription. Read them when present.

Their **absence proves nothing.** In a real tenant this returned empty across the board, which means only that the organisation doesn't model recurrence in Xero, not that nothing recurs. Never infer cadence from an empty repeating-bills result; fall back to the interval between bills from the same contact.

**Aged payables** is a fast way to see which suppliers currently have outstanding amounts. It's a triage aid, not a source of subscription figures.

### Attachments: detectable, not downloadable

The vendor's own PDF is frequently attached to the bill in Xero, and it is the best single source of seats, plan tier and contract dates. **You cannot fetch it.** The hosted connector exposes fifteen tools and none of them retrieves a file, and the self-hosted official server exposes no attachment tool either. Xero's REST API has an attachments endpoint, so the bytes exist; no MCP server currently in the wild hands them over.

What you *can* do is detect one. The hosted connector returns a `has_attachments` flag — but **only in detailed mode**, so you must request line items to see it. Use it to tell the user something precise instead of something vague:

> <Vendor>'s bill in Xero has the original invoice attached, but no Xero tool can fetch an attachment. If that invoice also came to your inbox I can read it there, otherwise open the bill in Xero and paste the seat count.

Treat `has_attachments: true` on a vendor whose structure you couldn't resolve as a strong hint to go looking in email for the same invoice.

### Credit notes

Informational only, exactly like their email equivalents. A credit note adjusts a balance; it is not a change in per-unit price. Surface it, never propose from it.

## Resellers and distributors in Xero

Everything `invoice-detection.md` says about resellers applies here, and Xero makes it slightly easier: the contact is the reseller, and the line items name the products. Read the line items, produce one proposed update per matched product, and annotate each note with `(billed via <Reseller>)`.

The direction check still applies to distributors that both sell to you and bill through you. If a line item describes software your organisation resells rather than consumes, filter it out and list it in the skipped section.

## Reconciling Xero against email

When both sources cover the same vendor in the window, don't pick one — use each for what it's good at.

| Field | Prefer | Why |
|---|---|---|
| Cost (per-unit) | Xero, when line items give quantity × unit amount; otherwise email | Xero is the accounting truth; email carries the breakdown when Xero doesn't. |
| Currency | Xero | It's the posted currency. |
| Invoice date | Xero | Normalised and unambiguous. |
| Billing frequency | Xero (interval between bills, or a repeating template) | Bill history over 18 months shows cadence directly. |
| Cost structure | Xero line items → email → ask | See trap 4. |
| Total seats | Xero line-item description, else email or PDF | Often present as text ("Team Plan Annual (2 seats)") even when no numeric field carries it. |
| Plan / tier | Email or PDF | Not an accounting concept. |
| Contract end date | Email or PDF | Not an accounting concept. |

**When the two disagree on an amount, say so rather than silently preferring one.** A mismatch is usually informative — a proration, a partial credit, a currency conversion, or a bill posted to the wrong contact:

> Xero has <Vendor> at $480 on 2026-03-01; the emailed invoice for the same period says $520. Difference could be a credit applied at the accounting end. Which should I record?

## Connecting Xero

Ask before assuming. If no Xero tool is present, the user may still use Xero — the connector simply isn't installed. Ask once: *"Do you use Xero? Your bills there are a cleaner source than email, and connecting it takes a minute."* If the answer is no, proceed email-only and don't ask again.

If they want to connect, give them the instructions for **their** environment:

| Environment | How to connect Xero |
|---|---|
| **Claude Desktop / claude.ai** | Settings → Connectors → Add custom connector. URL `https://mcp.xero.com/mcp`. Authorise in the browser on first use. Xero also appears in the built-in connector directory on some plans — check there first. |
| **Claude Code** | `claude mcp add xero --transport http https://mcp.xero.com/mcp`, then trigger any Xero tool to run OAuth. |
| **Cursor** | Settings → MCP → Add new MCP Server → URL `https://mcp.xero.com/mcp`. Restart when prompted. |
| **Windsurf** | Settings → MCP Servers → add the same URL, restart. |
| **Cline (VS Code)** | Settings → MCP Servers → add server `xero` with the same URL. |
| **Continue.dev** | Add an entry under `experimental.modelContextProtocolServers` in `~/.continue/config.json` with `transport: {type: "http", url: "https://mcp.xero.com/mcp"}`. |
| **ChatGPT** | MCP support varies by plan and is still rolling out. If unavailable, the delegated-collection path (`delegated-collection.md`) is the practical route — have someone with a Xero-capable client collect and hand back a file. |
| **Any other MCP client** | Add a remote HTTP MCP server at `https://mcp.xero.com/mcp`; it authorises via OAuth in the browser. If the client only supports local servers, the official self-hosted alternative is `npx -y @xeroapi/xero-mcp-server@latest` with Xero Custom Connection credentials — Shape B above. |

Two things to tell them regardless of environment: the connection is read-only for this skill's purposes (it never writes to Xero), and Xero authorises **per organisation** — if they run several entities, they need to pick the right one during OAuth, and the bills you see are only that entity's.

## Anti-patterns

- ❌ Reading receivables and proposing them as costs. Check the type, or use the payables-specific tool.
- ❌ Accepting the default date window. It is about a month; you need eighteen.
- ❌ Stopping at the first page of results.
- ❌ Deriving a cost from a zero-total bill, or reading a run of them as "these apps are free now".
- ❌ Recording the tax-inclusive total as the subscription cost.
- ❌ Inferring `costStructure: flat` from a bill with no line items.
- ❌ Reading `quantity` as a seat count. On journal-style bills it is `1.0` regardless; the seats are in the description text.
- ❌ Skipping a zero-total bill's line items. The header nets to zero; the lines carry the period cost.
- ❌ Promising the user you can open a PDF attached to a Xero bill. You can see that it exists; you cannot fetch it.
- ❌ Inferring "no recurring subscriptions" from an empty repeating-bills result.
- ❌ Converting currency. Surface the bill's own currency and let the user decide.
- ❌ Proposing from a `DRAFT` or `VOIDED` bill.
