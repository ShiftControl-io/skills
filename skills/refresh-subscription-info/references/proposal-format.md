# Proposal Format

Exact format for the user-facing proposal in Step 7 of the workflow. Consistency matters — users learn to read the format quickly only if it's the same every time.

## Top-level structure

```
Found invoices for {N} of your {M} ShiftControl apps. Proposed updates:

{1..N: per-app blocks, each labeled with a sequential number for approval reference}

Apps with no invoice in the last {window} days ({K}): {list of names}
Invoices found for apps not in ShiftControl ({J}): {list of vendor names}

Reply "approve N" (e.g. "approve 1, 3, 5") to apply specific changes,
"approve all" to apply all,
or "show details for N" to see the full invoice context for change N.
```

## Per-app block

```
{n}. {App Name}
   {field}:   {current value}   →   {proposed value}
   {field}:   {current value}   →   {proposed value}
   ...
   note will be added: "{note string}"
```

Rules:

- Use a blank line between apps for scannability.
- Right-align the arrow column when reasonable; not required if it complicates rendering.
- Show `(not set)` for currently-null fields rather than blanks.
- Format costs with **currency symbol AND unit**: `$8.00/user/month`, `$300.00/year flat`, `€50.00/user/year`.
- Format dates as ISO 8601 date only: `2026-03-15`.
- Include the `note will be added:` line on every block — showing the user that the change will also write a short context note to the app is part of "no surprises".

## No-change blocks

When you matched an invoice but every field already matches what's in ShiftControl, include it as **confirmation**:

```
{n}. GitHub
   No changes — invoice matches existing record. Last invoice dated 2026-03-12.
```

This is value-add: it tells the user their record IS current, which is itself useful.

## Uncertain / skipped blocks

For invoices you found but excluded, use a separate section AFTER the main proposal:

```
Skipped or uncertain (3):
  - "Atlassian" invoice from 2026-03-08 — could match Jira or Confluence; please clarify.
  - "Adobe Creative Cloud" invoice from 2026-02-28 — no app in ShiftControl matches; should I help you add it?
  - "Generic Vendor LLC" invoice from 2026-03-10 — couldn't extract per-unit cost from the body.
```

## Truncation

If there are MANY proposed changes (>20), show the first 20 with a summary footer:

```
... 8 more apps with proposed changes (reply "show more" to expand)
```

**Don't auto-expand** — it hammers context. Let the user ask.

## Cost-comparison context (optional but recommended)

For each cost change, annotate with a delta-from-current value to help the user evaluate:

```
1. Slack
   cost:  $8.00/user/month  →  $7.00/user/month   (12.5% decrease)
```

Skip if it complicates the rendering. Keep if you can do it cleanly.

## Why this format

Three audiences read it:

1. **The user** — wants to scan quickly, see what's changing, decide.
2. **The agent** — wants unambiguous item numbers to reference in approval messages.
3. **Someone reviewing the record later** — wants to reconstruct what was proposed vs what was approved.

The format serves all three. Don't deviate without good reason.
