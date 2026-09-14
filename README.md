# ERP → Sunbay Invoice Data Integration Guide

This document specifies the **invoice data Sunbay needs to obtain** from your ERP / source system, and **how that data is exchanged**, so that Sunbay can run debt collection and receivables analytics on your behalf.

It is written for the **IT team building the integration** between the ERP landscape and Sunbay.

The exchange works like this: **you expose a small read-only API over your invoice data, and Sunbay pulls from it** on its own schedule (§4). You implement one endpoint - plus, depending on what §4.3 and §4.5 settle, a PDF endpoint and a single-invoice endpoint - while scheduling, incremental fetching, paging, retries and backfill are all owned by Sunbay.

The **data model (§3) is the core of this document**. Section §4 defines the API contract; §5-§8 cover formats, security, reliability and scheduling. A number of details are deliberately left **open for onboarding (§9)**.

> **On flexibility.** This is the specification to implement, not a loose suggestion. Where a field or detail genuinely cannot be met by your source system, we can align it together during onboarding (§9). Please implement the contract as written and flag the specific points that do not fit.

> **Scope.** This document covers *what data Sunbay needs* and *how it is exchanged*. How Sunbay stores, processes, or acts on the data internally is intentionally out of scope.

---

## Contents

1. [Purpose & Scope](#1-purpose--scope)
2. [How the Integration Works](#2-how-the-integration-works)
3. [Data Model](#3-data-model)
4. [The Invoice API](#4-the-invoice-api)
5. [Data Formats & Conventions](#5-data-formats--conventions)
6. [Security & Authentication](#6-security--authentication)
7. [Reliability](#7-reliability)
8. [Scheduling & Volume](#8-scheduling--volume)
9. [Points to Confirm During Onboarding](#9-points-to-confirm-during-onboarding)

---

## 1. Purpose & Scope

Sunbay automates the collection of overdue receivables (email/SMS reminders and related flows) and provides receivables analytics. To do this, Sunbay needs a continuous feed of your **sales/revenue invoices - both paid and unpaid** - together with the **debtor (customer) data** for each invoice:

- **Unpaid and overdue invoices** drive the collection processes.
- **Paid invoices** power analytics and reporting across your receivables.

Optionally, the **PDF** of an invoice can be exchanged as well - it is needed **only** when invoice documents should be attached to reminder emails (§3.7).

The ERP remains the **system of record**. Sunbay never writes back to the ERP - the integration is read-only from the ERP's perspective.

**In scope for this document**

- The exact data fields Sunbay needs per invoice and per customer (§3).
- The API your side exposes and Sunbay polls (§4).
- Data formats (§5), authentication (§6), reliability rules (§7), scheduling and volumes (§8).

> **If exposing an endpoint is not possible.** Should your network policy rule out any inbound endpoint, tell us during the first technical call - Sunbay also supports a variant in which your system pushes data out to us, and we will share that specification on request.

---

## 2. How the Integration Works

You expose a small, **read-only HTTPS API** over the ERP data (specified in §4). Sunbay calls it on a schedule it controls: incremental polls for new and changed invoices, plus periodic full snapshots for reconciliation.

| | |
|---|---|
| **You implement** | A read-only invoice endpoint, authentication, and a `lastModifiedAt` timestamp bumped on every change in what the API exposes - plus a PDF endpoint (§4.3) and a single-invoice endpoint (§4.4) where those apply. |
| **Sunbay owns** | Scheduling, incremental watermarking, paging, retries, pacing and backfill. |
| **Initiates** | Sunbay, on its own polling schedule (§8). |
| **Tenant identification** | Implicit - the base URL and credentials are client-specific (§6.2). |
| **Cancellations** | Visible as `Cancelled` tombstones in API responses (§4.5). |
| **Backfill / first load** | Sunbay crawls the history through the same endpoint - no separate export needed. |
| **PDFs (if enabled)** | Fetched by Sunbay on demand at each reminder send; no copies are kept (§4.3). |

Why the integration is shaped this way:

- Your side implements **one to three small read-only endpoints** - no scheduler, no retry logic, no outbound delivery pipeline to build and operate.
- Sunbay can adapt scheduling, backfill and pacing without any change on your side.
- Cancellations and corrections propagate naturally: Sunbay simply observes the current state of your data.
- Documents stay authoritative in your system: PDFs are fetched on demand and never stored by Sunbay.
- **Idempotency** is keyed on `invoiceId` (§7) - re-fetching the same invoice is a safe update, never a duplicate.

---

## 3. Data Model

This is the **most important section**. Each invoice exposed to Sunbay should carry the following fields.

An invoice is a JSON object carrying the invoice-level fields at the **top level** plus two **nested objects**: `customer` (§3.4) and `seller` (§3.5). Debtor data is embedded with each invoice (denormalized) - there is no separate customer feed. Field names inside the nested objects are **unprefixed** (`customer.name`, `seller.name`); the complete shape is shown in §3.9.

**Required** column legend: **Yes** = mandatory · **Rec.** = recommended (strongly preferred) · **Opt.** = optional · **Cond.** = conditional.

### 3.1 Invoice identification

| Field | Type | Required | Description |
|---|---|---|---|
| `invoiceId` | string | **Yes** | Stable, globally-unique identifier of the invoice **in the source system**. Used to recognise the same invoice across syncs (deduplication / update key). Must be **stable** - the same invoice must always carry the same id, even after edits. |
| `invoiceNumber` | string | **Yes** | Human-readable invoice number (e.g. `FV/2026/01/0123`). Shown to the debtor in reminders. **Not required to be unique** - see §3.1.3. |
| `documentType` | enum | **Yes** | Kind of document - see §3.1.1. |
| `correctedInvoiceId` | string | **Cond.** | On an adjusting document (`CorrectiveInvoice`, `CreditNote`, `DebitNote`) that adjusts a specific invoice: the `invoiceId` of that invoice. A standalone note that adjusts nothing - late-payment interest, a penalty charge - carries no `correctedInvoiceId` and stands as a receivable of its own. Always point at the original invoice, never at an earlier correction of it. |
| `issueDate` | date | **Yes** | Date the invoice was issued. |
| `dueDate` | date | **Cond.** | Payment due date - when the invoice becomes collectible. Required on every document except a `CreditNote`, where `null` is accepted. |
| `paymentTermDays` | integer | **Opt.** | Payment term in days, if available. |

#### 3.1.1 `documentType` values

| Value | Meaning | Chased by Sunbay? |
|---|---|---|
| `Invoice` | Standard (VAT) invoice | Yes |
| `CorrectiveInvoice` | Corrective / adjustment invoice | Yes |
| `AdvanceInvoice` | Advance payment invoice | Yes |
| `FinalInvoice` | Final invoice | Yes |
| `Proforma` | Pro forma invoice | **No** - not a legal receivable |
| `CreditNote` | Credit note | **No** - a negative document; its open amount offsets what the debtor owes |
| `DebitNote` | Debit note | Yes |

"Yes" means one thing for every kind: the record is chased **when its own `amountOutstanding` is positive** (§3.1.2). Proformas are excluded regardless of amounts, and a credit note never has a positive open amount (a refund larger than the credit note itself would be a bookkeeping error, not a case this contract covers).

> If your source system uses other document kinds, list them during onboarding so we can map them (§9).

#### 3.1.2 Corrections

Adjusting documents (`CorrectiveInvoice`, `CreditNote`, `DebitNote`) are delivered as **separate invoice records**, each with its own `invoiceId`, and - where they adjust a specific invoice - point at it through `correctedInvoiceId`. The original's own amounts (`amountNet` / `amountVat` / `amountGross`) are never rewritten with corrected values; only its open amount moves.

**Every record is its own open item.** `amountOutstanding` on each record - original or adjusting - is **the open amount of that document as your ERP sees it right now**. Sunbay chases every record whose `amountOutstanding` is positive and never recomputes balances from correction chains, which is what keeps the feed safe against double counting: a credit note that has already reduced its original must not reduce it a second time on Sunbay's side. So expose the settled state, not the arithmetic - one rule governs how a not-yet-settled credit is handled, and it follows next.

**No unapplied credit may sit next to a chased document.** Sunbay does not sum records, so any reduction the debtor is entitled to must already be visible in the `amountOutstanding` of the document being chased. If your system clears credit notes against their originals, this happens by itself. If it keeps both as separate open items, the integration layer applies the clearing before exposing the data: the original's `amountOutstanding` is reduced by the credit, and whatever the original cannot absorb stays on the credit note as a negative open amount - a refund your side owes. Without this Sunbay would demand the original's full amount while the offsetting credit sat unread beside it. **Positive** adjustments are the opposite case: a debit note or an increasing correction stands as a receivable of its own, with its own positive `amountOutstanding`, and is chased in its own right. A credit note with no `correctedInvoiceId` - a rebate or goodwill credit tied to no invoice - has nothing to be cleared against: it is exposed as it is, with a negative `amountOutstanding`, and is not reflected in reminders until your ERP allocates it to specific invoices. Sunbay does not allocate credits.

**Amounts on adjusting documents are the document's own amounts, signed.** `amountNet` / `amountVat` / `amountGross` carry the **difference** the document introduces, with its sign: **negative** reduces what the debtor owes, **positive** increases it. If your source system stores a correction as new corrected totals or as "before/after", the API layer derives the signed difference. Do not send absolute values with the direction implied by the document type or by the accounting side of the entry - a `CorrectiveInvoice` goes both ways, so an unsigned amount is not interpretable.

Where it is present, `correctedInvoiceId` is what links the two records: it lets a reminder quote the original invoice number next to the correction, and it groups them for analytics. The referenced original must be reachable (§4.5).

#### 3.1.3 `invoiceNumber` is not a unique key

Only `invoiceId` is unique. One accounting document may reach Sunbay as **several invoice records sharing the same `invoiceNumber`** - for example when the source system keys documents by line or item position, each line carrying its own `invoiceId`, its own amounts and sometimes its own due date.

This is accepted, but it has a visible consequence: the debtor holds one document bearing one number, while Sunbay may be chasing several receivables that all carry it. Tell us at onboarding whether your data works this way (§9), so reminder content and payment matching are set up accordingly.

### 3.2 Amounts & currency

| Field | Type | Required | Description |
|---|---|---|---|
| `currency` | string (ISO 4217) | **Yes** | e.g. `PLN`, `EUR`. |
| `amountNet` | decimal | **Yes** | Net amount. |
| `amountVat` | decimal | **Yes** | VAT amount. |
| `amountGross` | decimal | **Yes** | Gross total of this document. Signed on adjusting documents (§3.1.2). |
| `amountPaid` | decimal | **Yes** | Amount settled against this document by payments (`0` if none). Enables partial-payment handling. **Never clamped** - see overpayments below. |
| `amountOutstanding` | decimal | **Yes** | **The open amount of this document in your ERP, right now.** This is the authoritative value and what collection chases. It is normally `amountGross - amountPaid`, but not always: netting - in either direction - changes it without touching `amountPaid` (§3.1.2), and cancelled documents report `0` (§3.3). |

**Signs.** `amountNet` / `amountVat` / `amountGross` are non-negative on ordinary documents and signed on adjusting documents (§3.1.2). `amountOutstanding` may be **negative on any document** - an overpaid invoice, or a credit note whose refund is still owed. Sunbay chases **only positive** `amountOutstanding`.

`amountPaid` carries the **same sign as the document it belongs to**: a refund paid out against a credit note is a negative `amountPaid`. That way the status rules (§3.3) read the same on both signs.

**Overpayments.** When more is received than is owed on the document, do **not** clamp. `amountPaid` carries the full amount actually received, and `amountOutstanding` goes **negative** by the surplus; `status` is `Paid`. This includes an invoice paid at its face value after a credit was cleared against it (§3.1.2) - the debtor owed less than the face value, so the difference is a surplus like any other. Capping `amountPaid` silently deletes the surplus from the feed and is not acceptable - the overpayment is real information about the debtor's account.

### 3.3 Status & lifecycle

| Field | Type | Required | Description |
|---|---|---|---|
| `status` | enum | **Yes** | `Open` · `PartiallyPaid` · `Paid` · `Cancelled`. |
| `paidDate` | date | **Cond.** | Date the document was fully settled. Required when `status = Paid`. On a document closed by netting rather than by payment (a credit note applied to its invoice) it is the date of that settlement. |
| `isBlockedForCollection` | boolean | **Rec.** | `true` if Sunbay must **not** chase this invoice (dispute, legal hold, internal block). If the source system has no equivalent concept, **omit the field** instead of sending `false` on every record - an omitted field means *no block information is available*, whereas `false` is a positive statement that the invoice may be chased. When the field is absent, Sunbay treats the invoice as chaseable. |
| `lastModifiedAt` | timestamp | **Yes** | When the record **as this API exposes it** last changed (ISO-8601 UTC) - not only edits in the source system, but any change in the exposed fields, including derived ones. Drives incremental fetching (§4.2): **any** change - status, amounts, payments, cancellation, a credit cleared against the record (§3.1.2), correction linkage - must update this timestamp, whether the source record itself changed or the integration layer derived the new value. |

**Status must follow from the amounts.** The two must never contradict each other. The rules below hold for every document kind - the sign of `amountGross` says which direction the document runs in, and `amountOutstanding` is read against it.

| Condition | `status` |
|---|---|
| Nothing settled yet: `amountPaid = 0`, and `amountOutstanding` has the same sign as `amountGross` | `Open` |
| Partly settled: `amountPaid ≠ 0`, and `amountOutstanding` has the same sign as `amountGross` | `PartiallyPaid` |
| Nothing open: `amountOutstanding = 0`, or its sign is opposite to `amountGross` (overpaid) | `Paid` (with `paidDate` set) |
| Document cancelled or voided in the source system | `Cancelled`, whatever the amounts |

A document with `amountGross = 0` - a corrective invoice that changes only descriptive data, say - has nothing to settle: it reports `amountOutstanding = 0` and `status = Paid`, with `paidDate = issueDate`.

If a record nevertheless arrives self-contradictory - `PartiallyPaid` with `amountOutstanding = 0`, say - **the amounts decide** what Sunbay does: only a positive `amountOutstanding` is chased. Two overrides always win regardless of the amounts: `Cancelled` and `isBlockedForCollection = true` both stop collection.

**Cancelled documents.** A cancelled invoice keeps its nominal `amountGross` (the document itself is not rewritten), reports **`amountOutstanding = 0`** and `paidDate = null`, and keeps `amountPaid` truthful - usually `0`, but if a payment was booked against the document before it was voided, that payment stays visible (it is now your ERP's refund to handle). Here `amountOutstanding` is forced to zero regardless of the ledger: nothing is collectible on a cancelled document. The record must stay retrievable as a tombstone (§4.5).

### 3.4 Debtor (customer)

Delivered as a **nested `customer` object** on every invoice. This is the party Sunbay contacts, so the object itself is required.

| Field (inside `customer`) | Type | Required | Description |
|---|---|---|---|
| `id` | string | **Yes** | Stable, unique identifier of the customer in the source system. |
| `name` | string | **Yes** | Debtor name (company or person). |
| `taxId` | string | **Rec.** | Tax identifier (e.g. VAT ID / NIP). One consistent format across the whole feed (§5). |
| `email` | string | **Rec.** | Primary email. Required for email reminders. |
| `emailCcs` | string[] | **Opt.** | **List** of additional CC email addresses. |
| `phone` | string | **Opt.** | Phone number in **international format including the country code**, e.g. `+48512345678`. Required for SMS reminders. |
| `address` | string | **Rec.** | Postal address. |
| `countryCode` | string (ISO 3166-1) | **Yes** | e.g. `PL`. |
| `communicationLanguage` | string (ISO 639-1) | **Opt.** | Preferred language for reminders, e.g. `pl`, `en`, `de`. Sunbay selects the reminder template language per debtor; when absent, the client-wide default is used. |
| `customFields` | object | **Opt.** | Arbitrary key-value pairs at **customer** level (see §3.6). |

### 3.5 Seller & payment

Delivered as a **nested `seller` object** on every invoice. The object is required, because `bankAccount` is.

| Field (inside `seller`) | Type | Required | Description |
|---|---|---|---|
| `name` | string | **Opt.** | Issuing entity name. A single installation may contain several legal entities/sellers. |
| `taxId` | string | **Opt.** | Seller tax identifier. |
| `address` | string | **Opt.** | Issuing entity postal address. Useful for formal reminders and multi-entity installations. |
| `bankAccount` | string | **Yes** | Bank account the debtor should pay into (IBAN/NRB). Included in reminders. |

### 3.6 Custom fields & references

| Field | Type | Required | Description |
|---|---|---|---|
| `customFields` | object | **Opt.** | Arbitrary key-value pairs at **invoice (top) level**. **Custom fields are supported at both invoice and customer level** - send any extra attributes that may be useful (segment, region, contract code, cost centre, ...). |
| `externalReference` | string | **Opt.** | Any additional reference useful for reconciliation. |

### 3.7 PDF (optional)

> **PDFs are optional.** A PDF is needed **only if** you want Sunbay to attach the invoice document to reminder emails. If you do not need attachments, skip PDFs entirely - the integration is fully functional without them.

If PDF attachments are enabled, Sunbay fetches the PDF from your PDF endpoint (§4.3) on demand, each time it sends a reminder with the invoice attached. Sunbay does not keep PDF copies - documents are served from your system on request, and the endpoint must stay available for as long as an invoice is being chased.

One PDF per invoice. PDF specifics (always available? maximum size? PDFs for corrective documents?) are confirmed during onboarding (§9).

### 3.8 Line items (optional)

Invoice line items are **optional but valuable** - they enable richer analytics and more informative reminder content. The invoice is fully usable without them; when PDFs are exchanged, the PDF remains the authoritative document. On an adjusting document the lines carry the same signed differences as the header (§3.1.2).

If provided, `lineItems` is an array where each line carries:

| Field | Type | Required (within a line) | Description |
|---|---|---|---|
| `name` | string | **Yes** | Description of the goods/service. |
| `quantity` | decimal | **Yes** | Quantity. |
| `unit` | string | **Opt.** | Unit of measure, e.g. `pcs`, `hours`. |
| `unitPriceNet` | decimal | **Yes** | Net unit price. |
| `discountPercent` | decimal | **Opt.** | Line discount in percent, e.g. `10.0`. The line totals below are already net of this discount - it is informational. |
| `vatRate` | decimal | **Rec.** | VAT rate in percent, e.g. `23.0`. |
| `vatAmount` | decimal | **Opt.** | VAT amount for the line. |
| `amountNet` | decimal | **Rec.** | Net total for the line. |
| `amountGross` | decimal | **Yes** | Gross total for the line. |

### 3.9 Example invoice object

This is the item shape returned by your API (§4.2).

```json
{
  "invoiceId": "ERP-2026-INV-000123",
  "invoiceNumber": "FV/2026/01/0123",
  "documentType": "Invoice",
  "correctedInvoiceId": null,
  "issueDate": "2026-01-10",
  "dueDate": "2026-01-24",
  "paymentTermDays": 14,

  "currency": "PLN",
  "amountNet": 1000.00,
  "amountVat": 230.00,
  "amountGross": 1230.00,
  "amountPaid": 0.00,
  "amountOutstanding": 1230.00,

  "status": "Open",
  "paidDate": null,
  "isBlockedForCollection": false,
  "lastModifiedAt": "2026-01-25T11:02:14Z",

  "seller": {
    "name": "ACME Sp. z o.o.",
    "taxId": "5213001234",
    "address": "ul. Handlowa 5, 00-002 Warszawa",
    "bankAccount": "PL61109010140000071219812874"
  },

  "customer": {
    "id": "ERP-CUST-10001",
    "name": "Kowalski Handel Sp. z o.o.",
    "taxId": "7010001234",
    "email": "ksiegowosc@kowalski.pl",
    "emailCcs": ["zarzad@kowalski.pl", "biuro@kowalski.pl"],
    "phone": "+48512345678",
    "address": "ul. Przykładowa 12, 00-001 Warszawa",
    "countryCode": "PL",
    "communicationLanguage": "pl",
    "customFields": {
      "segment": "B2B",
      "region": "Mazowieckie"
    }
  },

  "lineItems": [
    {
      "name": "Consulting services - January 2026",
      "quantity": 10.0,
      "unit": "hours",
      "unitPriceNet": 80.00,
      "vatRate": 23.0,
      "vatAmount": 184.00,
      "amountNet": 800.00,
      "amountGross": 984.00
    },
    {
      "name": "License fee",
      "quantity": 1.0,
      "unit": "pcs",
      "unitPriceNet": 200.00,
      "vatRate": 23.0,
      "vatAmount": 46.00,
      "amountNet": 200.00,
      "amountGross": 246.00
    }
  ],

  "externalReference": "SRC-REF-998877",
  "customFields": {
    "costCenter": "CC-204",
    "salesRep": "A. Nowak"
  }
}
```

---

## 4. The Invoice API

This is the contract Sunbay's fetcher will code against. **Host and base path are yours to choose** - a versioned prefix is recommended, e.g. `{baseUrl} = https://api.example.com/sunbay/v1`. Everything below the base URL - paths, parameters, response shapes - should be implemented as specified here; deviations can be discussed during onboarding.

### 4.1 Overview & responsibilities

- You host a **read-only** HTTPS API exposing invoices in the shape defined in §3. Sunbay polls it on an agreed schedule (§8).
- Sunbay handles scheduling, incremental watermarking, paging and retries. Your side only serves data.
- Both paid and unpaid invoices must be exposed; how far back the history goes is agreed during onboarding (§9).

### 4.2 List invoices

```
GET {baseUrl}/invoices?modifiedSince=2026-01-24T09:30:00Z&modifiedUntil=2026-01-24T10:00:00Z&page=1&pageSize=100
Authorization: <see §6.2>
Accept: application/json
```

Query parameters (Sunbay may omit any of them; all must be supported):

| Parameter | Type | Semantics |
|---|---|---|
| `modifiedSince` | ISO-8601 UTC timestamp | Lower bound, **inclusive**: return only invoices with `lastModifiedAt >= modifiedSince`. When omitted, return a **full snapshot**: all `Open` / `PartiallyPaid` invoices, `Paid` / `Cancelled` ones within the agreed history window (§9), and - when route (a) of §4.5 applies - any invoice referenced by `correctedInvoiceId` from a record in the snapshot, regardless of its age. |
| `modifiedUntil` | ISO-8601 UTC timestamp | Upper bound, **exclusive**: return only invoices with `lastModifiedAt < modifiedUntil`. Sunbay sets it to the instant the crawl started, which freezes the result set for the whole crawl. On your side it is one extra condition in the query. |
| `page` | integer | 1-based page number. Default `1`. |
| `pageSize` | integer | Maximum items per page. Default `100`; the server may cap it (suggested cap `500`). |

Response `200 OK`, `application/json`:

```json
{
  "items": [ { ...invoice objects exactly as defined in §3... } ],
  "page": 1,
  "pageSize": 100,
  "totalCount": 1234
}
```

- `items` - full invoice objects (§3.9 shape), field names 1:1.
- `page` / `pageSize` - echo the request. Sunbay walks pages until a page returns fewer than `pageSize` items (or `page * pageSize` reaches `totalCount`).
- `totalCount` - total number of matching invoices. **Recommended** - it lets Sunbay size the crawl; if omitted, Sunbay simply stops when a page returns fewer than `pageSize` items.
- **Ordering:** return results ordered by `(lastModifiedAt, invoiceId)` ascending. Together with `modifiedUntil` this is what makes a crawl deterministic: the matching set is frozen for its whole duration, so paging can neither skip nor repeat a row, and an interrupted crawl resumes on the same page boundaries.
- **Why plain page numbers (not opaque cursors):** they are trivial for you to implement (`LIMIT`/`OFFSET`, `Skip`/`Take`), and bounding the set with `modifiedUntil` makes them safe: a record modified mid-crawl leaves the window instead of shifting position within it, so no row is pushed past a page boundary unread. Only if `modifiedUntil` cannot be implemented does a crawl run against a moving set; then `invoiceId` idempotency (§7) keeps duplicates harmless and the periodic full-snapshot crawl (§4.5) reconciles anything missed - at the cost of a skipped change staying stale until that snapshot.

**Incremental fetching (watermarking).** Sunbay picks a crawl instant `T`, calls with `modifiedUntil = T` and `modifiedSince = previous watermark - small overlap` (a few minutes), and on a completed crawl stores `T` as the new watermark. Bounding the top end is what makes the watermark trustworthy: everything below `T` has been served, so a record modified while the crawl was running is not skipped - it simply belongs to the next one. A full-snapshot crawl carries `modifiedUntil` the same way and seeds the watermark for the incremental polls that follow it. The small overlap re-reads a few records, which is harmless: `invoiceId` idempotency (§7) makes re-processing a safe update. Your obligations: every change in what you expose bumps `lastModifiedAt`, `modifiedSince` is inclusive and `modifiedUntil` exclusive, and timestamps are UTC.

### 4.3 Invoice PDF (optional)

> Implement this endpoint **only if** invoice documents should be attached to reminder emails. If you do not need attachments, skip it - the integration works fully without PDFs.

```
GET {baseUrl}/invoices/{invoiceId}/pdf
```

- `200 OK` with `Content-Type: application/pdf` and the binary document (`Content-Disposition` filename optional).
- `404` when the id is unknown or the PDF is not yet available - Sunbay retries later.
- The path parameter is the URL-encoded `invoiceId`.
- Sunbay does **not** store copies of PDFs. It fetches the PDF from this endpoint **on demand, each time it sends a reminder with the invoice attached** - so the same invoice's PDF may be requested repeatedly over the collection lifecycle. There is no PDF export to build - documents are served straight from your system.
- The endpoint must therefore stay available for as long as an invoice is being chased, not only at the first fetch.
- Size guideline: up to ~5 MB per document (confirmed during onboarding).

### 4.4 Single invoice (recommended)

```
GET {baseUrl}/invoices/{invoiceId}
```

Returns `200 OK` with one invoice object (§3), or `404` if unknown. Used for spot re-fetches and joint debugging.

> Optional in general - **except** when it is the route chosen for reaching corrected originals (§4.5, route **b**). In that case this endpoint is mandatory.

### 4.5 Snapshots, increments, cancellations & corrections

- Regular polls are **incremental** (`modifiedSince`). In addition, Sunbay periodically runs a **full-snapshot** crawl (no `modifiedSince`, still bounded by `modifiedUntil`) to reconcile state - e.g. nightly or weekly (§8).
- **Cancellations must stay visible.** A cancelled or deleted invoice must remain retrievable through the API as a *tombstone*: returned with `status = "Cancelled"` and an updated `lastModifiedAt`. It must **not** silently disappear from results - otherwise Sunbay would keep chasing a debt that no longer exists. If the source system hard-deletes records, the API layer must still expose the tombstone.
- **The corrected original must stay reachable.** Balances do not depend on it (§3.1.2), but an adjusting document is presented and grouped together with the invoice named in `correctedInvoiceId`, so Sunbay must be able to obtain that record. The original is often years old, long paid, and therefore **outside the agreed history window** (§4.2). Two acceptable routes, chosen at onboarding (§9): **(a)** the API layer keeps referenced originals in scope - a document referenced by `correctedInvoiceId` from any record in scope is itself part of the full snapshot regardless of its age or status, and issuing an adjusting document bumps the original's `lastModifiedAt` (correction linkage is a change, §3.3) so it re-enters the next incremental poll; or **(b)** implement the single-invoice endpoint (§4.4), which Sunbay calls to fetch any original it has not seen - in this variant §4.4 is **not optional**.
- **Whatever the route, a changed original must re-enter the feed.** When a credit is cleared against an invoice Sunbay has already seen - by your ERP or virtually by the integration layer (§3.1.2) - its exposed `amountOutstanding` drops, and only the incremental poll can carry that drop to Sunbay. That is an amount change and bumps `lastModifiedAt` (§3.3) in route (b) exactly as in route (a); §4.4 only covers originals Sunbay has never seen.
- Safety net: during full-snapshot reconciliation, open invoices that Sunbay expected in the snapshot but did not receive are flagged and handled per the onboarding agreement (§9). Only records with `lastModifiedAt` before the crawl's `modifiedUntil` count as expected - anything modified during the crawl belongs to the next incremental poll, not to the alarm.

### 4.6 Errors & availability

- Status codes: `400` invalid parameters · `401`/`403` authentication failures · `429` rate limited (with `Retry-After`, which Sunbay honours) · `5xx` server errors, which Sunbay retries with backoff (§7).
- Error bodies: JSON in the form `{ "error": { "code": "...", "message": "..." } }` is recommended, not mandated.
- Responses should complete within ~30 seconds; prefer lowering the page size over risking timeouts.
- Authentication: one of the options in §6.2, chosen per your security policy.
- No hard SLA is required - a brief outage only delays the next successful poll. Data freshness should roughly match the agreed poll interval (§8).

### 4.7 Example exchange

```
GET /sunbay/v1/invoices?modifiedSince=2026-01-24T09:30:00Z&modifiedUntil=2026-01-24T10:00:00Z&page=1&pageSize=100
Authorization: Bearer eyJhbGciOi...
Accept: application/json
```

```json
{
  "items": [
    { ...the invoice object from §3.9... }
  ],
  "page": 1,
  "pageSize": 100,
  "totalCount": 1
}
```

And - only when PDF attachments are enabled (§3.7):

```
GET /sunbay/v1/invoices/ERP-2026-INV-000123/pdf
Authorization: Bearer eyJhbGciOi...

HTTP/1.1 200 OK
Content-Type: application/pdf

%PDF-1.7 ...binary...
```

---

## 5. Data Formats & Conventions

These conventions apply to the payloads your API returns.

| Aspect | Convention |
|---|---|
| **Text encoding** | **UTF-8** for all text and JSON content. ⚠️ Some ERP data sources default to regional encodings (e.g. Windows-125x) - convert to UTF-8. |
| **Dates** | ISO-8601 calendar dates: `YYYY-MM-DD` (e.g. `2026-01-24`). |
| **Timestamps** | ISO-8601 in **UTC** with `Z` (e.g. `2026-01-24T09:30:00Z`). |
| **Decimal separator** | **Dot** (`.`). No thousands separators. ⚠️ Locales using a decimal comma must be normalised (`1230,00` → `1230.00`). |
| **Currency** | ISO 4217 three-letter code. |
| **Phone** | International format with country code, e.g. `+48512345678`. |
| **Booleans** | `true` / `false`. |
| **Whitespace** | **Trim** leading and trailing whitespace from every text value. ⚠️ Watch for **non-breaking spaces (U+00A0)** inside customer names, addresses and bank accounts - they survive a naive trim and break matching and display; replace them with ordinary spaces. |
| **Tax identifiers** | One **consistent** format across the whole feed: either always with the country prefix (`PL5213001234`) or always without (`5213001234`). No spaces, dashes or dots. |
| **Missing values** | Omit the field or send `null` - do not send empty placeholder strings for numeric/date fields. |

---

## 6. Security & Authentication

### 6.1 Transport

- All communication uses **HTTPS / TLS 1.2+**.

### 6.2 Securing your API

You decide how your endpoint authenticates Sunbay; any of the following works, chosen per your security policy during onboarding:

- **API key** - a key you issue to Sunbay, sent in a header (e.g. `x-api-key`). You control issuance and rotation.
- **OAuth 2.0 client credentials** - you provide a token endpoint and a client id/secret; Sunbay exchanges them for short-lived bearer tokens and caches tokens until expiry.
- **Mutual TLS (mTLS)** - Sunbay presents a client certificate you trust.

Additionally:

- **Tenant identification is implicit** - the base URL and the credentials you issue identify the client on both sides, so no tenant identifier is needed in the data itself.
- Sunbay stores the credentials you issue in a secrets store and supports rotation.
- If your API may only be reached from known networks, tell us during onboarding - how Sunbay's fetcher is identified to your perimeter (e.g. source address allow-listing) is agreed there.

### 6.3 Data protection

- Encrypted in transit (TLS) and at rest on the Sunbay side.
- Sensitive values (keys, tokens, certificates) are never logged.
- Credentials are exchanged securely and can be rotated on either side.

---

## 7. Reliability

**Idempotency.** `invoiceId` is the key: re-fetching the same invoice is a safe **update**, never a duplicate. An invoice's `invoiceId` must never change between syncs, edits or retries.

**Sunbay retries**

- Transient failures, timeouts and `5xx` responses are retried with exponential backoff; `429` with `Retry-After` is honoured.
- Brief outages are unproblematic - they only delay the next successful poll.
- Ordering of adjusting documents is handled by Sunbay internally: if a correction appears before its original (e.g. across page boundaries), it is accepted and linked once the original arrives.

**Your obligations**

- Stable identifiers (`invoiceId`, `customer.id`).
- An **inclusive** `modifiedSince` and an **exclusive** `modifiedUntil` filter, with `lastModifiedAt` updated on **every** change in what the API exposes.
- A stable ordering by `(lastModifiedAt, invoiceId)` during a crawl.
- Cancellation tombstones that stay retrievable, and corrected originals that stay reachable (§4.5).

---

## 8. Scheduling & Volume

- Sunbay polls incrementally on an agreed schedule - typically every 15-60 minutes - plus a periodic full snapshot (e.g. nightly or weekly) for reconciliation.
- PDF fetches (if enabled) happen on demand whenever a reminder with an attachment is sent, with bounded concurrency - the same invoice's PDF may be requested more than once over its collection lifecycle.
- Tell us your **rate limits** and maintenance windows - Sunbay stays within them and honours `429` / `Retry-After`.
- Please share expected **daily and peak volumes** (e.g. month-end) so page size and poll frequency can be sized sensibly.

---

## 9. Points to Confirm During Onboarding

> **Note:** these points are for the implementation/rollout phase itself - not something to resolve before reviewing or sharing this document. We work through them together when the integration is being built.

**Data**

1. **Stable identifiers** - does the source system expose a stable, unique id per **invoice** (`invoiceId`) and per **customer** (`customer.id`) that survives edits? What are they?
2. **Corrections - storage** - how does the source system store adjusting documents: as separate documents with their own open amount, or as edits of the original? Does it clear credit notes against their originals, or keep both as open items - in which case the integration layer has to apply the clearing itself? Do unallocated credit notes occur, and who allocates them? (§3.1.2)
3. **Corrections - sign** - can the signed difference be derived reliably? Does the sign in your system follow the accounting side of the entry, and can the same document kind carry both directions? (§3.1.2)
4. **Split documents** - is one accounting document ever delivered as several receivables sharing an `invoiceNumber` (e.g. keyed per line item)? (§3.1.3)
5. **Partial payments** - can `amountPaid` / `amountOutstanding` be provided, or only a binary paid flag?
6. **Overpayments** - can a payment exceed what is owed on a document, and will the surplus be reported rather than clamped? (§3.2)
7. **Cancelled documents** - can the source system produce the agreed shape (nominal `amountGross`, `amountOutstanding = 0`, truthful `amountPaid`)? (§3.3)
8. **Blocking** - does the source system mark invoices that must not be chased (dispute, legal hold)? If not, how should such cases reach Sunbay - or is the field simply omitted? (§3.3)
9. **Document types** - which document kinds exist and which are collectible; mapping of any kinds not listed in §3.1.1. Proformas are never chased (§3.1.1) - should they be sent at all, for analytics?
10. **Multi-company** - can one installation hold several legal entities/sellers? If so, how is the seller disambiguated?
11. **Currencies** - are multi-currency invoices expected?
12. **Formats** - confirm UTF-8, dot decimals, ISO-8601 (including time-zone handling for dates), phone numbers with country code, trimmed text free of non-breaking spaces, and one consistent tax-identifier format (§5).
13. **PDF attachments - yes or no?** Should Sunbay attach invoice documents to reminder emails? Only if yes: PDF availability, maximum size, and PDFs for corrective documents (§3.7).

**API & operations**

14. **Corrected originals** - which route from §4.5 applies: referenced originals kept in scope by the API layer, or the single-invoice endpoint (§4.4)?
15. API base URL, credential exchange and rotation procedure; chosen authentication method (§6.2), plus any network allow-listing needed.
16. Your rate limits and maintenance windows.
17. Initial load depth - how far back paid invoices are exposed (e.g. all open, plus paid within N months).
18. `lastModifiedAt` semantics - which changes bump it, and with what precision? Can the query support the `modifiedUntil` upper bound (§4.2)?
19. Test/sandbox environment availability.
20. Poll schedule - incremental interval and full-snapshot cadence (§8).
21. Expected daily and peak invoice volumes (§8).
