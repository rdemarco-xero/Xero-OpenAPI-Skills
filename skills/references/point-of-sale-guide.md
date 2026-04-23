# Xero POS Integration

## Overview

Point-of-sale integration with Xero follows a strict **API hierarchy**: initial configuration calls build a local mapping of Xero entities (accounts, tax rates, tracking categories) that are referenced on every transaction. Sales flow through **Invoices → Payments**; returns flow through **CreditNotes → Allocations**.

High-volume POS systems should use the **summary invoice pattern** — one invoice per business day grouped by payment type — to stay within Xero's 60 calls/minute rate limit.

---

## API Hierarchy

The guide defines four phases. Execute them in order; do not parallelize across phases or skip setup steps.

### Phase 1 — Initial Setup (once per connected organization)

Run these calls when a user first connects their Xero organization. Cache results locally; do not re-fetch on every sale.

| Priority | Resource | Purpose |
|---|---|---|
| 1 | **Accounts** | Map POS account codes (sales, COGS, tax, cash) to Xero account codes |
| 2 | **BrandingThemes** | Select an invoice branding theme for customer-facing documents |
| 3 | **TaxRates** | Map POS tax types to Xero tax rate configurations |
| 4 | **TrackingCategories** | Map store locations or cost centers to Xero tracking options |
| 5 | **Items** | Sync the product catalog for use as invoice line items |

### Phase 2 — Per-Transaction Flow

Run these calls for every sale, in order.

| Step | Resource | Action |
|---|---|---|
| 1 | **Contacts** | GET to find existing customer; PUT/POST to create or upsert if not found |
| 2 | **Invoices** | PUT with `type: ACCREC` and `status: AUTHORISED` |
| 3 | **Payments** | PUT to apply payment against the invoice |

### Phase 3 — Returns & Refunds

| Step | Resource | Action |
|---|---|---|
| 1 | **CreditNotes** | PUT with `type: ACCREC` to create a credit for the return |
| 2 | **CreditNote Allocations** | PUT to allocate the credit against the original invoice |

### Phase 4 — Bank Reconciliation

| Step | Resource | Action |
|---|---|---|
| 1 | **BankTransactions** | PUT to support bank reconciliation of summary invoices |

---

## Summary Invoice Pattern

For high-volume POS, create **one invoice per business day** rather than one per transaction.

- Reduces API call volume significantly
- Stays within Xero's 60 calls/minute rate limit
- Groups sales by payment type (cash, card, voucher) as separate line items on a single invoice

Each line item on a summary invoice should reference a **TrackingCategory option** (e.g., store location) and a **TaxRate** configured in Phase 1.

---

## Invoice Line Item Types

| Type | Account mapping |
|---|---|
| Regular sale | Revenue account (from Phase 1 Accounts) |
| Gift card sold | Liability account — not a revenue account |
| Gift card redeemed | Applied as a **Payment** against the liability account |
| Voucher sold | Same treatment as gift cards |
| Voucher redeemed | Same treatment as gift card redemptions |

> **Do not net gift card sales and redemptions inside the invoice.** They must be recorded as distinct line items against the correct liability accounts.

---

## Payment Types

Each payment type in your POS must map to a distinct Xero **BankAccount** retrieved in Phase 1. Apply payments against the corresponding account.

| POS payment type | Xero account type |
|---|---|
| Cash | Cash / petty cash account |
| Card (credit/debit) | Merchant services clearing account |
| Gift card | Gift card liability account |
| Voucher | Voucher liability account |

---

## Fees & Adjustments

Merchant fees (e.g., card processing fees) are recorded as a **Purchase Invoice** with `type: ACCPAY` — not as a line item on a sales invoice. This keeps revenue and cost-of-acceptance separate in Xero's reporting.

---

## Endpoint Reference

Endpoints directly referenced in the [Xero POS integration guide](https://developer.xero.com/documentation/guides/how-to-guides/how-to-integrate-my-pos-system-with-xero), with links to their `operationId` in the [Xero OpenAPI specification](https://github.com/rdemarco-xero/Xero-OpenAPI-Skills/blob/main/skills/references/xero_accounting.yaml).

| Endpoint | Method | operationId | Phase |
|---|---|---|---|
| `/Accounts` | GET | [`getAccounts`](https://github.com/rdemarco-xero/Xero-OpenAPI-Skills/blob/main/skills/references/xero_accounting.yaml#L41) | 1 — Setup |
| `/BrandingThemes` | GET | [`getBrandingThemes`](https://github.com/rdemarco-xero/Xero-OpenAPI-Skills/blob/main/skills/references/xero_accounting.yaml#L3726) | 1 — Setup |
| `/TaxRates` | GET | [`getTaxRates`](https://github.com/rdemarco-xero/Xero-OpenAPI-Skills/blob/main/skills/references/xero_accounting.yaml#L20661) | 1 — Setup |
| `/TrackingCategories` | GET | [`getTrackingCategories`](https://github.com/rdemarco-xero/Xero-OpenAPI-Skills/blob/main/skills/references/xero_accounting.yaml#L21137) | 1 — Setup |
| `/Items` | GET | [`getItems`](https://github.com/rdemarco-xero/Xero-OpenAPI-Skills/blob/main/skills/references/xero_accounting.yaml#L9780) | 1 — Setup |
| `/Items` | PUT | [`createItems`](https://github.com/rdemarco-xero/Xero-OpenAPI-Skills/blob/main/skills/references/xero_accounting.yaml#L9857) | 1 — Setup |
| `/Contacts` | GET | [`getContacts`](https://github.com/rdemarco-xero/Xero-OpenAPI-Skills/blob/main/skills/references/xero_accounting.yaml#L4172) | 2 — Transaction |
| `/Contacts` | PUT | [`createContacts`](https://github.com/rdemarco-xero/Xero-OpenAPI-Skills/blob/main/skills/references/xero_accounting.yaml#L4383) | 2 — Transaction |
| `/Contacts` | POST | [`updateOrCreateContacts`](https://github.com/rdemarco-xero/Xero-OpenAPI-Skills/blob/main/skills/references/xero_accounting.yaml#L4645) | 2 — Transaction |
| `/Invoices` | PUT | [`createInvoices`](https://github.com/rdemarco-xero/Xero-OpenAPI-Skills/blob/main/skills/references/xero_accounting.yaml#L8238) | 2 — Transaction |
| `/Invoices` | POST | [`updateOrCreateInvoices`](https://github.com/rdemarco-xero/Xero-OpenAPI-Skills/blob/main/skills/references/xero_accounting.yaml#L8605) | 2 — Transaction |
| `/Payments` | PUT | [`createPayment`](https://github.com/rdemarco-xero/Xero-OpenAPI-Skills/blob/main/skills/references/xero_accounting.yaml#L13468) | 2 — Transaction |
| `/CreditNotes` | PUT | [`createCreditNotes`](https://github.com/rdemarco-xero/Xero-OpenAPI-Skills/blob/main/skills/references/xero_accounting.yaml#L6295) | 3 — Returns |
| `/CreditNotes/{CreditNoteID}/Allocations` | PUT | [`createCreditNoteAllocation`](https://github.com/rdemarco-xero/Xero-OpenAPI-Skills/blob/main/skills/references/xero_accounting.yaml#L7548) | 3 — Returns |
| `/BankTransactions` | PUT | [`createBankTransactions`](https://github.com/rdemarco-xero/Xero-OpenAPI-Skills/blob/main/skills/references/xero_accounting.yaml#L1685) | 4 — Reconciliation |

---

## Critical Notes

### Contacts
For anonymous walk-in customers, a generic Contact (e.g., `"POS Customer"`) is acceptable. Use `updateOrCreateContacts` (POST) to upsert by name — this prevents duplicate contact creation on repeated calls.

### Invoice Status
Always create invoices with `status: AUTHORISED`. Creating in `DRAFT` requires an additional update call before a payment can be applied, doubling your API calls per transaction.

### Payments
A Payment requires three fields resolved from earlier phases:

| Field | Source |
|---|---|
| `InvoiceID` | Returned by the preceding `createInvoices` call |
| `AccountCode` | BankAccount code from Phase 1 Accounts mapping |
| `Date` + `Amount` | From POS transaction data |

### Rate Limits
Xero enforces **60 API calls per minute** per connected organization. The summary invoice pattern is the primary strategy for staying within this limit at scale. If you exceed the limit, Xero returns `HTTP 429`; implement exponential backoff before retrying.

### Reports in Xero
Summary invoices feed directly into Xero's standard reports (Profit & Loss, GST/VAT). Ensure your account code mapping in Phase 1 is correct — remapping account codes after transactions have been posted requires voiding and re-creating invoices.
