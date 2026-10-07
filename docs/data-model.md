# Data Model (JSON schemas)

> Status: **PROPOSAL — awaiting approval.**

Every persisted JSON file has a top-level `"schemaVersion"`. The formal
JSON Schema files will live in `backend/validators/schemas/`. This page
shows each structure with an example and its rules.

## General rules

| Rule | Detail |
|---|---|
| IDs | Seeded master data uses readable stable IDs (`cat-food`, `pm-upi`, `freq-monthly`, `user-01`). Anything created in the app gets a **UUID v4** (`crypto.randomUUID()`), so two people working offline can never create the same ID. Array position is never an ID. |
| Money | Integer **minor units** (paise). Field names end in `Minor`: `amountMinor: 85075` = ₹850.75. Must be a safe integer. |
| Dates | `YYYY-MM-DD` (ISO 8601 calendar date). |
| Timestamps | ISO 8601 with offset, e.g. `2026-10-06T19:30:00+05:30`. |
| Who | `createdBy` / `updatedBy` / `filledByUserId` hold **application user IDs**, never Google accounts. |
| Masters are never hard-deleted | Categories, members, users, payment methods are **deactivated** (`active: false`). Old transactions keep valid references, and sync never has to reconcile a deleted master. |
| Transactions use tombstones | A deleted transaction stays in the file with `deleted: true` so the other device learns about the delete during sync. Tombstones are hidden from the UI and totals. |
| Unknown fields | Rejected on import; preserved-and-warned on cloud read (so a newer app version's data is not silently destroyed — see schema versions below). |

### Schema versions

* The app knows the highest `schemaVersion` it supports per file type.
* **Older version** → run migrations in `storage/migrations/` step by step
  (v1→v2→v3), validate, keep the original as part of the next backup.
* **Newer version** (someone updated the app on the other computer) → stop,
  show "This data was saved by a newer version of the app. Update this app
  before syncing." Never reinterpret, never overwrite.
* Each migration is documented in [changelog.md](changelog.md) and has a test.

---

## config/app-manifest.json

Describes the cloud layout. Written only by setup and at the end of a
successful sync.

```json
{
  "schemaVersion": 1,
  "app": "household-budget",
  "householdId": "6f1c…uuid",
  "dataVersion": 1,
  "restoreGeneration": 0,
  "createdAt": "2026-10-07T10:00:00+05:30",
  "folders": {
    "root": "<drive-folder-id>",
    "config": "<id>", "budget": "<id>", "transactions": "<id>",
    "backups": "<id>", "exports": "<id>"
  },
  "files": {
    "config/household.json":        { "fileId": "<id>", "schemaVersion": 1, "lastWrittenRevisionId": "<rev>", "lastWrittenMd5": "<md5>" },
    "budget/budget-items.json":     { "fileId": "<id>", "schemaVersion": 1, "lastWrittenRevisionId": "<rev>", "lastWrittenMd5": "<md5>" },
    "transactions/2026-10.json":    { "fileId": "<id>", "schemaVersion": 1, "lastWrittenRevisionId": "<rev>", "lastWrittenMd5": "<md5>" }
  },
  "lastSuccessfulSync": {
    "at": "2026-10-07T19:45:10+05:30",
    "byAppUserId": "user-02",
    "deviceId": "dev-…uuid",
    "deviceName": "Study laptop"
  },
  "lastRestore": null
}
```

* `dataVersion` — increments when the overall data format changes.
* `restoreGeneration` — increments on every restore. A device whose local
  changes were made against an older generation must review them (see
  [sync.md](sync.md)).
* Each device additionally keeps its **own** "last known versions" in local
  staging; those are what conflict detection uses. The manifest copies are
  informational and used for verification.
* Drive folder/file IDs are stored here and in device settings, **never in
  source code**.

## config/household.json

```json
{
  "schemaVersion": 1,
  "householdId": "6f1c…uuid",
  "name": "Our Household",
  "notes": "",
  "createdAt": "…", "createdBy": "user-01",
  "updatedAt": "…", "updatedBy": "user-01"
}
```

## config/app-users.json

```json
{
  "schemaVersion": 1,
  "users": [
    { "id": "user-01", "name": "User 1", "displayName": "User 1", "active": true },
    { "id": "user-02", "name": "User 2", "displayName": "User 2", "active": true }
  ]
}
```
No passwords, emails, roles or tokens. Users can be added/renamed later.

## config/family-members.json

```json
{
  "schemaVersion": 1,
  "members": [
    { "id": "mem-self",  "name": "Self",    "relationship": "self",   "active": true, "notes": "" },
    { "id": "3b9e…uuid", "name": "Child 1", "relationship": "child",  "active": true, "notes": "" }
  ]
}
```
`relationship` ∈ `self | spouse | child | parent | sibling | other`.
Unlimited members. The wizard step 1 creates them; nothing assumes a count.

## config/categories.json

```json
{
  "schemaVersion": 1,
  "categories": [
    { "id": "cat-food", "name": "Food", "parentId": null, "icon": "food", "color": "peach", "active": true, "sortOrder": 20 },
    { "id": "8a1d…uuid", "name": "Groceries", "parentId": "cat-food", "icon": "basket", "color": "peach", "active": true, "sortOrder": 1 }
  ]
}
```
Seeded top-level categories: Home, Food, Transport, Children, Education,
Personal, Health, Financial, Investment, Debt, Lifestyle, Utilities, Other.
Rules: `parentId` must exist; maximum two levels (category → sub-category);
no cycles. Names are data, never hard-coded in HTML.

## config/payment-methods.json

```json
{
  "schemaVersion": 1,
  "paymentMethods": [
    { "id": "pm-cash", "name": "Cash", "active": true, "sortOrder": 1 },
    { "id": "pm-upi",  "name": "UPI",  "active": true, "sortOrder": 2 }
  ]
}
```
Seeded: Cash, UPI, Debit Card, Credit Card, Bank Transfer, Other.

## config/frequency-types.json

```json
{
  "schemaVersion": 1,
  "frequencyTypes": [
    { "id": "freq-daily",       "name": "Daily",       "periodsPerYear": { "numerator": 365, "denominator": 1 }, "schedule": { "unit": "day",   "every": 1 }, "system": true, "sortOrder": 1 },
    { "id": "freq-weekly",      "name": "Weekly",      "periodsPerYear": { "numerator": 365, "denominator": 7 }, "schedule": { "unit": "day",   "every": 7 }, "system": true, "sortOrder": 2 },
    { "id": "freq-monthly",     "name": "Monthly",     "periodsPerYear": { "numerator": 12,  "denominator": 1 }, "schedule": { "unit": "month", "every": 1 }, "system": true, "sortOrder": 3 },
    { "id": "freq-quarterly",   "name": "Quarterly",   "periodsPerYear": { "numerator": 4,   "denominator": 1 }, "schedule": { "unit": "month", "every": 3 }, "system": true, "sortOrder": 4 },
    { "id": "freq-half-yearly", "name": "Half-Yearly", "periodsPerYear": { "numerator": 2,   "denominator": 1 }, "schedule": { "unit": "month", "every": 6 }, "system": true, "sortOrder": 5 },
    { "id": "freq-yearly",      "name": "Yearly",      "periodsPerYear": { "numerator": 1,   "denominator": 1 }, "schedule": { "unit": "month", "every": 12 }, "system": true, "sortOrder": 6 }
  ]
}
```
Factors are stored as fractions so no binary floating point is involved.
`system: true` frequencies may be renamed but their factors are locked
(changing them would silently change every budget). Weekly uses `365/7`
(= 52.142857…) — see open question Q5.

## config/settings.json

```json
{
  "schemaVersion": 1,
  "currency": { "code": "INR", "symbol": "₹", "exponent": 2, "locale": "en-IN" },
  "timezone": "Asia/Kolkata",
  "yearStartMonth": 1,
  "backupPolicy": {
    "beforeRestore": true,
    "beforeImport": true,
    "beforeBulkDelete": true,
    "dailyOnFirstSync": true,
    "keepAutomaticCount": 30,
    "deleteManualAutomatically": false
  }
}
```
`beforeRestore` / `beforeImport` are always on (the UI shows them locked).
`yearStartMonth` depends on open question Q4.

## budget/budget-items.json

```json
{
  "schemaVersion": 1,
  "budgetItems": [
    {
      "id": "c0a8…uuid",
      "name": "School fees",
      "categoryId": "cat-education",
      "memberId": "3b9e…uuid",
      "scope": "individual",
      "frequencyId": "freq-monthly",
      "plannedAmountMinor": 1800000,
      "startDate": "2026-06-10",
      "endDate": null,
      "active": true,
      "notes": "",
      "createdBy": "user-01", "createdAt": "…",
      "updatedBy": "user-02", "updatedAt": "…"
    }
  ]
}
```
* `scope` ∈ `common | individual | commitment | lifestyle` — matches wizard
  steps 2–5. `memberId` is **required** when `scope = individual`, optional
  otherwise.
* `startDate` is also the **schedule anchor** used to compute the next
  expected date (e.g. yearly insurance due every 15 March).
* `plannedAmountMinor` is the amount **per occurrence** of its frequency.
* A planned amount change mid-year = end-date the old item, add a new one,
  so history stays honest.

## transactions/YYYY-MM.json (one file per calendar month)

```json
{
  "schemaVersion": 1,
  "month": "2026-10",
  "transactions": [
    {
      "id": "e4b2…uuid",
      "transactionDate": "2026-10-06",
      "amountMinor": 85000,
      "categoryId": "cat-food",
      "budgetItemId": "c0a8…uuid",
      "memberId": "3b9e…uuid",
      "paymentMethodId": "pm-upi",
      "notes": "Vegetables",
      "filledByUserId": "user-02",
      "createdAt": "2026-10-06T19:30:00+05:30",
      "updatedByUserId": "user-02",
      "updatedAt": "2026-10-06T19:30:00+05:30",
      "deleted": false
    }
  ]
}
```
Differences from the example in the requirements, and why:

* `amount: 850` → `amountMinor: 85000` — paise integer, unambiguous name.
* Added `updatedByUserId` — knows who last edited, not only who first entered.
* Added `deleted` (+ `deletedAt`, `deletedByUserId` when true) — tombstone so
  deletions sync correctly.

Rules:

* `transactionDate` must fall inside `month`.
* `amountMinor` > 0 (see open question Q6 about refunds).
* `categoryId` required; `budgetItemId`, `memberId`, `paymentMethodId`
  optional but, if present, must reference an existing record.
* `filledByUserId` required, defaults to the selected app user, editable
  before saving.
* Changing the date to another month = tombstone in the old file with
  `"movedToMonth": "2026-11"` + the same `id` added to the new file.

## backups/backup-YYYYMMDD-HHMMSS-<trigger>.json

See [backups.md](backups.md). One self-contained file with all config,
budget and transaction files plus record counts and a SHA-256 checksum.

## Local device files (never in the cloud, never in Git)

See [storage.md](storage.md): `device.json`, `sync-state.json`, token file.
