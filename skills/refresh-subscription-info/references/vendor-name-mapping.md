# Vendor → App Matching

How to map a vendor name extracted from an invoice to an app in the user's ShiftControl tenant.

## Matching algorithm (try in this order)

For each parsed invoice, try matches in priority order. **Stop at the first success.**

### 1. Exact case-insensitive match

Invoice vendor name lowercased + trimmed `==` ShiftControl app name lowercased + trimmed. **Highest confidence.**

- "Slack" matches "Slack"
- "GitHub" matches "github"

### 2. Normalized match

Strip common corporate suffixes from both sides, then exact match. Suffixes to strip (case-insensitive):

```
Inc., Inc, Ltd., Ltd, Limited, LLC, LLC.,
PLC, Pty Ltd, Pty. Ltd., GmbH, S.A., S.A.S.,
Technologies, Software, Software Inc.,
Corporation, Corp., Co., Holdings,
", Inc.", ", LLC", ", Ltd.", ", Inc"
```

- "Slack Technologies, LLC" → "Slack" matches the user's "Slack"
- "Notion Labs, Inc." → "Notion Labs" then strip "Labs" if needed (don't over-strip — "Labs" is significant for some products like "Notion Labs" vs hypothetical "Notion Cloud")

### 3. Known-alias match

Hard-coded canonical aliases for products with multiple market names:

| Aliases | Canonical |
|---|---|
| "G Suite", "GSuite", "Google Workspace", "Google Apps" | Whichever name the user has |
| "Office 365", "Microsoft 365", "M365" | Whichever name the user has |
| "GitHub", "GitHub Enterprise", "GitHub Team" | GitHub (if user has only one GitHub entry) |
| "Atlassian Jira", "Jira", "Jira Software", "Jira Cloud" | Treat each line item separately if user has multiple Atlassian apps |
| "Atlassian Confluence", "Confluence", "Confluence Cloud" | Same — separate match |
| "Notion", "Notion Labs" | Notion |
| "Figma", "Figma Inc." | Figma |
| "Adobe Creative Cloud", "Adobe CC", "Creative Cloud" | Adobe Creative Cloud |

### 4. Domain-based match

If the invoice's From-address domain matches a known vendor, accept that match even if the company name in the body is different.

- Email from `billing@slack.com` → "Slack" regardless of body header
- Email from `accounts@google.com` with "Google Workspace" in body → Google Workspace

Build a domain → product mapping as you go: when you successfully match a vendor by name, remember its domain for the rest of the session.

**Acquired-product domain rebinds** — the From-domain doesn't always match the product because the parent company sometimes takes over billing post-acquisition. The pattern varies per acquisition and per account:

- Some vendors keep their original billing domain post-acquisition (e.g. Slack frequently continues to bill from `feedback@slack.com` even after the Salesforce acquisition).
- Others migrate to the parent (e.g. some Heroku accounts now bill from `noreply@salesforce.com`).
- A few vendors split — some account types stay on the original domain while enterprise accounts move to the parent's billing.

**Don't assume the rebind has happened**; read the body for the product name. If you DO see a clear rebind (the From-domain is one vendor and the body talks about a different product), match to the product named in the body and annotate the note with `(billed via <ParentCompany>)` so the next reader understands the From-address.

### Observed vendor → email-sender patterns

A non-exhaustive catalog of patterns observed in real ShiftControl inboxes. The skill should recognize these but not be limited to them.

| Product | Typical From | Subject pattern | Format |
|---|---|---|---|
| Slack | `feedback@slack.com` | "<Org>, your plan has renewed", "<Org>, you're all set" | Inline body + "View Receipt" link |
| Notion | `team@mail.notion.so`, vendor-templated | "Updates to your Notion invoice" (incremental), "Notion reminder: You will be charged in N days" (pre-renewal) | Inline body |
| GitHub | `noreply@github.com` | "GitHub Invoice <INV-ID> - <Org>" | PDF attached |
| Granola | `notifications@mail.granola.ai` (and Stripe template) | "Your receipt from Granola #<NUMBER>" | Stripe-template inline |
| Framer | (Stripe-template sender) | "Your receipt from Framer B.V. #<NUMBER>" | Stripe-template inline |
| Anthropic | `invoice+statements@mail.anthropic.com` | "Your receipt from Anthropic, PBC #<NUMBER>" | Stripe-template inline |
| 1Password | (varies) | "Your 1Password invoice (<Org>)" | Inline body |
| JumpCloud | (varies; often forwarded by an internal alias) | "Invoice <NUMBER> from JumpCloud" | PDF attached |
| Cloudflare | (varies) | "Your invoice is attached" OR "Your Cloudflare purchase confirmation" | PDF or "View invoice" link |
| Zoom | (varies) | "Payment Processed for <NUMBER>" | PDF attached |
| AWS | `invoicing@aws.com`, `aws-globalreceivables@email.amazon.com` | "Amazon Web Services Billing Statement", "AWS Marketplace Statement" | PDF |

### Stripe-template receipts are a recognizable shape

Several vendors (Granola, Framer, Anthropic, others) use Stripe's billing platform, which produces a recognizable email template:

- Subject: `"Your receipt from <Vendor> #<NUMBER>"`
- Body opens with the vendor name and an invoice/receipt number
- A "View receipt" or "Download invoice" link is usually present
- Charge amount and period are inline

When you see this template, the vendor in the subject IS the vendor you're matching — the From-address may be the vendor's own domain or a Stripe relay.

### 4a. Reseller line-item matching

When the invoice is from a reseller (ShiftControl-as-reseller, CDW, Insight, Carahsoft, Stripe-as-consolidator), the reseller's name is on the envelope but the products are in line items. Match each line item separately:

1. Read the line items in the body.
2. For each line item, run matching rules 1-3 above against the line-item text (e.g. "Google Workspace Business Plus — 50 seats") against the user's app list.
3. Create one proposed update per matched line item.
4. Annotate each note with `(billed via <Reseller>)` — e.g. `"Updated from Google Workspace invoice dated 2026-03-15 (billed via ShiftControl)"`.

If a reseller line item doesn't match any ShiftControl app, surface it as a not-tracked entry separately (same path as a standalone unmatched invoice).

### 5. Fuzzy match

Edit distance ≤ 2 on normalized strings, but only for app names ≥ 6 characters (avoids false matches on short names like "Box" → "Bot").

- "Slacck" (typo) → Slack — accept with MEDIUM confidence
- "Box" → "Bot" — REJECT (too short)

MEDIUM confidence matches go into the proposal with a "fuzzy match — please verify" annotation.

### 6. No match

Surface as **"found invoice for X but no matching app in ShiftControl — should I help you add it?"** but do NOT auto-create. v0.1.0 does not create apps.

## Disambiguation

If multiple ShiftControl apps could match the same invoice (e.g. user has both "Atlassian Jira" and "Atlassian Confluence" and the invoice says "Atlassian"), pause and ask the user:

```
The invoice from Atlassian could match two apps in your ShiftControl:
  - Atlassian Jira (currently $10/user/month)
  - Atlassian Confluence (currently $5/user/month)

The invoice line items show:
  - Jira Software Cloud: 50 seats × $7.50/month
  - Confluence Cloud: 50 seats × $5.50/month

Should I apply each line item to its respective app? (yes / no / let me decide each)
```

## Quality threshold for confidence

| Match level | Confidence | Action |
|---|---|---|
| Exact / normalized / known-alias / domain | HIGH | Include in proposal automatically |
| Fuzzy (edit distance ≤ 2) | MEDIUM | Include with "verify" annotation |
| No match | — | Surface separately, don't propose changes |

## Why not LLM-judge every match

LLM-based matching is appealing but flaky and non-deterministic. The deterministic rules above produce a smaller, higher-quality match set. If the rules don't match, the right move is **"ask the user"**, not "have the LLM guess harder". Asking leaves a clear record of why each match was made; guessing doesn't.
