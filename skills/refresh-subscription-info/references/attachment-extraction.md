# Reading PDF Invoices from Email Attachments

Many SaaS vendors put the numbers in a PDF and leave the email body nearly empty. GitHub, JumpCloud, Zoom, Cloudflare and Salesforce-billed Slack all do it. Skipping those invoices means skipping some of the largest line items in a typical SaaS bill, so it is worth doing properly.

This file is the procedure. Load it when Step 4 of the workflow hits an invoice whose body lacks the figures and whose message carries a PDF.

## First: which capability do you have?

Attachment handling varies more between email MCPs than almost anything else. Three tiers, in descending order of convenience:

| Tier | What the email MCP offers | What you do |
|---|---|---|
| 1 | A dedicated attachment tool returning **content or a download URL** (e.g. Superhuman's `get_attachment`) | Call it, read the PDF, extract fields. Nothing else in this file applies. |
| 2 | No attachment tool, but `get_message` accepts a **raw / full-MIME format** (the Gmail connector's `messageFormat: "RAW"`) | Follow the RAW procedure below. Size-gated. |
| 3 | Neither — filenames only, no raw format | You genuinely cannot read the PDF. Offer manual entry (Step 6a) and say so plainly. |

Determine the tier by looking at the tools you actually have, not by assuming. The presence of an `attachmentId` field in a message payload is **not** evidence you can fetch the attachment — the Gmail connector returns attachment IDs while exposing no tool that accepts one.

## The RAW procedure (tier 2)

### Step 1 — Gate on size before you fetch anything

**Never call RAW unconditionally.** RAW returns the entire message as one base64 string. On a message carrying a multi-megabyte attachment that string is hundreds of thousands of tokens, and a single call can exhaust the context window and end the session.

The size signal is the message's `sizeEstimate`, which comes back free from the thread search and from the cheap metadata formats. Note carefully: **the per-attachment metadata carries `filename`, `id` and `mimeType` but no size**, so the message-level estimate is the only number you get. A message with one PDF is mostly that PDF, which makes the estimate a good proxy.

```
if no PDF part in the attachment metadata      -> skip, nothing to read
if the body already yielded cost + period      -> skip, no need to spend the call
if sizeEstimate is unavailable                 -> skip, mark uncertain (you cannot gate safely)
if sizeEstimate > 150_000                      -> skip, mark uncertain, TELL THE USER
otherwise                                      -> fetch RAW
```

**If the connector gives you no size signal at all, do not call RAW.** Some email MCPs report neither a message size nor a per-attachment size; on those you cannot gate, and an ungated fetch is exactly the call that ends a session. Treat the invoice as uncertain, tell the user the figure is in an attachment you can't safely open, and offer manual entry.

The 150 KB threshold is a judgement call, not a documented limit. Rationale: the RAW response runs about **1.3× the `sizeEstimate`**, because the whole MIME message — attachment base64 included — is base64-encoded a second time into the response. A 35 KB message returns a comfortable ~48 KB string. A 2.3 MB message returns something in the region of 3 MB. Keep the constant in one place so it can be raised if context budgets grow.

**Truncation is the dangerous failure mode.** A partially returned RAW response still looks like a successful call and still decodes — into a corrupt PDF. Never treat "the call returned" as "the call succeeded"; validate (Step 4 below).

### Step 2 — Fetch and decode the outer layer

Call `get_message` with the raw format and the message ID. What comes back is the complete RFC 2822 message in a single field, encoded with the **URL-safe base64 alphabet** — `-` and `_` in place of `+` and `/`. Decoding it with a standard base64 decoder produces garbage or an error, so decode accordingly (`base64.urlsafe_b64decode` in Python, `Buffer.from(raw, "base64url")` in Node).

The result is plain MIME text: headers, then boundary-delimited parts.

### Step 3 — Walk the MIME tree and pull the PDF part

Use a real MIME parser. In Python that is `email.message_from_bytes()` then `.walk()`. **Do not regex for base64 blocks** — multipart boundaries nest (a typical invoice email is `multipart/mixed` wrapping a `multipart/alternative` body plus the attachment), and a regex will happily return the wrong part.

Select parts where:

- `Content-Type` is `application/pdf`, **and**
- `Content-Disposition` is `attachment` — not `inline`. Inline parts are usually logos and tracking pixels.

When a message carries several PDFs, match the part's `filename` against the filename you saw in the attachment metadata so you extract the right one.

Then decode that part's payload from **standard** base64 (the inner layer uses the ordinary alphabet; only the outer envelope is URL-safe). Two encodings, two alphabets, in one call — this is the step people get wrong.

### Step 4 — Validate before you trust

Run all three checks against the decoded bytes. Any failure means mark the invoice uncertain and stop — do not attempt to read the content.

```
bytes[:5] == b"%PDF-"            header present
b"%%EOF" in bytes[-2048:]        trailer present, file not truncated
len(bytes) > 1000                not a stub
```

A valid PDF from this path starts with `%PDF-1.x` and ends with `%%EOF`. If the trailer is missing, the RAW response was truncated somewhere and every number you extract is suspect.

### Step 5 — Extract the text, then reuse the logic you already have

Write the validated bytes to a temporary file and extract text — `pdfplumber` handles invoice tables better than `pypdf`, so prefer it and fall back if unavailable. Feed the extracted text into the **existing** Step 4 field extraction. A PDF is just another source of the same fields; nothing downstream changes.

If text extraction returns less than roughly 50 characters, the PDF is almost certainly a scan. **Do not attempt OCR** — mark it uncertain and move on.

### Step 6 — Carry the provenance forward

A PDF-derived figure has been through one more parsing step than a body-derived one, and is correspondingly likelier to be wrong. Say so in the proposal, so the user can spot-check those preferentially:

```
3. <Vendor>
   cost:  $12.00/user/month  →  $14.50/user/month
   source: PDF attachment "Invoice 12345.pdf"
```

## When you skip one, say why

Silently dropping an oversized or unreadable attachment is the worst outcome available — the user believes the app was checked when it wasn't. Name it:

> I found a <Vendor> invoice where the cost is only in a PDF attachment that's too large for me to open (2.3 MB). I can update the contract date from the body, but the per-unit cost needs a manual check — paste the figure and I'll record it.

## Anti-patterns

- ❌ Calling RAW without checking `sizeEstimate` first. One oversized call can end the session.
- ❌ Treating a RAW response as successful because it returned. Validate the header and the `%%EOF` trailer before parsing.
- ❌ Decoding the outer payload with a standard base64 decoder. It is URL-safe; use the matching decoder.
- ❌ Regex-scraping the MIME text for base64 blocks instead of using a MIME parser. Boundaries nest; you will pick up the wrong part.
- ❌ Treating an `attachmentId` as proof you can fetch the attachment. Check for a tool that actually accepts one.
- ❌ Proposing a cost from a PDF that yielded almost no extractable text. A scanned invoice that parses to whitespace is a failure, not an empty invoice.
- ❌ Running OCR. Out of scope; mark uncertain instead.
