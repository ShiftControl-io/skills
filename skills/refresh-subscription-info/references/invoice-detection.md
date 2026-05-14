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

- **Default: 90 days** — typically covers monthly billing cycles plus quarterly cadence.
- **For annual-only contracts:** extend to **400 days** if the user asks for a comprehensive sweep.
- Don't pull older than **18 months** unless the user explicitly requests it — old invoices are usually superseded.

## What to do when the inbox is huge

Most real inboxes are noisy. To keep within reasonable LLM context:

- Query in tighter time windows (e.g. last 30 days) and iterate.
- Use the email-search MCP's filter capabilities before reading bodies.
- Read only the first ~1000 characters of each candidate; pull the full body only if structured fields aren't already extracted.

## Failure modes worth flagging

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
