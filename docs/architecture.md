# Architecture

> Status: **PROPOSAL — awaiting approval** (Phase 0 / First Task).
> No application code exists yet. Open decisions are listed in
> [open-questions.md](open-questions.md).

This document explains how the Household Budget application is put together.
It is written for a beginner developer: every layer has one job, and every
folder says what lives in it.

---

## 1. Current project inspection

| Item | Finding |
|---|---|
| Repository | `karunabarath-Per/House_budget` (GitHub) |
| Branches | `main`, working branch `claude/laughing-heisenberg-tmgn2b` |
| Contents | `README.md` only (one line). No code, no `.gitignore`, no license. |
| Runtime available | Node.js 22, npm 10 |
| Budgeting reel | **Not present in the repository.** The design below is based on the written requirements only. |

So we are starting from an empty project. Nothing has to be migrated.

---

## 2. Technology choices (kept deliberately small)

| Concern | Choice | Why |
|---|---|---|
| Runtime | Node.js 22 LTS | One language (JavaScript) for frontend and backend. |
| Backend | Node + **Express** | Most common, best-documented beginner web server. |
| Frontend | **Plain HTML + CSS + JavaScript ES modules**, no build step | Nothing to compile. Open the file, read it, change it. |
| Validation | **JSON Schema** files + `ajv` | Schemas are both documentation and validation. |
| Tests | Node's built-in test runner (`node --test`) | Zero extra test framework to learn. |
| Google | `googleapis` + `google-auth-library` (official Google packages) | Official, maintained, handles OAuth and PKCE. |
| Database | **None** (JSON files only) | Fixed requirement. |

Why a small backend at all (instead of a browser-only app)?

1. Local staging must live in the operating-system app-data directory,
   **outside** the Git repository. A browser cannot write there; Node can.
2. Google refresh tokens must never sit in browser `localStorage`. The backend
   keeps them in a protected file.
3. Google Drive calls stay in one place (the adapter) instead of being
   scattered across pages.

The backend listens on **127.0.0.1 only** (this computer only), because the
application has no login. See [open-questions.md](open-questions.md) Q2 about
phones.

---

## 3. Final folder structure

```
House_budget/                      (repository root, PRIVATE on GitHub)
│
├── README.md                      What the app is + quick start
├── LICENSE                        "All rights reserved" (private family use)
├── .gitignore                     Blocks .env, tokens, data, backups, exports
├── .env.example                   Placeholders only, never real values
├── package.json                   Scripts: start, dev, test, lint
│
├── frontend/                      Everything the browser loads
│   ├── index.html                 Single page shell
│   ├── css/
│   │   ├── tokens.css             Colours, spacing, radius, fonts (pastel theme)
│   │   ├── layout.css             Sidebar (desktop) / bottom nav (mobile)
│   │   └── components.css         Cards, forms, tables, badges, steps
│   ├── js/
│   │   ├── main.js                Starts the app, picks the first page
│   │   ├── router.js              #/dashboard style hash routing
│   │   ├── api-client.js          The ONLY file that calls the backend (fetch)
│   │   ├── app-state.js           Current app user, cached sync status
│   │   ├── pages/                 One file per page (see §6)
│   │   └── components/            Reusable UI pieces (see §7)
│   └── assets/
│       └── icons/                 Friendly SVG icons for categories
│
├── backend/
│   ├── server.js                  Creates Express app, binds to 127.0.0.1
│   ├── config.js                  Reads .env, resolves app-data directory
│   ├── routes/                    HTTP endpoints → call services (no logic)
│   │   ├── users.routes.js
│   │   ├── household.routes.js
│   │   ├── masters.routes.js
│   │   ├── budget.routes.js
│   │   ├── transactions.routes.js
│   │   ├── reports.routes.js
│   │   ├── sync.routes.js
│   │   ├── backups.routes.js
│   │   ├── import-export.routes.js
│   │   └── google-auth.routes.js
│   ├── services/                  Application services (business rules)
│   │   ├── household.service.js
│   │   ├── master-data.service.js
│   │   ├── budget.service.js
│   │   ├── transaction.service.js
│   │   ├── report.service.js
│   │   ├── upcoming.service.js
│   │   ├── backup.service.js
│   │   ├── import-export.service.js
│   │   ├── cloud-setup.service.js
│   │   └── sync.service.js
│   ├── middleware/
│   │   ├── local-only.js          Rejects non-localhost Host/Origin headers
│   │   └── error-handler.js       Safe error messages, no financial data in logs
│   └── validators/
│       ├── schemas/               *.schema.json, one per data file
│       ├── validate.js            Runs ajv against the schemas
│       └── references.js          Checks categoryId/memberId/userId exist
│
├── storage/
│   ├── data-access.js             The Data Access Layer (getCategories(), …)
│   ├── storage-adapter.md         The adapter contract every backend must meet
│   ├── migrations/                schemaVersion upgrades (v1 → v2 …)
│   ├── local/
│   │   ├── app-data-paths.js      Finds the OS app-data folder
│   │   ├── staging-store.js       working/ + base/ copies, atomic writes
│   │   └── local-folder-adapter.js  "Cloud" adapter backed by a local folder
│   │                                (used in Phases 2–4 and in tests)
│   ├── google-drive/
│   │   ├── google-oauth.js        Authorization-code + PKCE, token storage
│   │   ├── drive-adapter.js       Implements the adapter contract on Drive
│   │   └── drive-layout.js        Folder/file names, manifest handling
│   └── sync/
│       ├── sync-engine.js         Pull → merge → push → verify
│       ├── merge-records.js       Three-way merge for ID-keyed records
│       ├── merge-transactions.js  Transaction-specific rules (tombstones, moves)
│       └── sync-state.js          SYNCED / LOCAL CHANGES / … state machine
│
├── calculations/                  PURE functions, no I/O, shared with browser
│   ├── money.js                   Parse/format/add paise, INR display
│   ├── currency.js                Currency settings (INR first, others later)
│   ├── frequency.js               Annual & monthly planning equivalents
│   ├── budget-totals.js           Totals by category/member/frequency
│   ├── planned-vs-actual.js       Variance
│   ├── schedule.js                Next expected date for upcoming items
│   └── reports.js                 Report aggregations
│
├── tests/
│   ├── unit/                      calculations, merge, validation, migrations
│   ├── integration/               services + local-folder adapter, two users
│   └── fixtures/                  SYNTHETIC data only ("Test Child 1", ₹100)
│
├── docs/                          This documentation
│
└── .github/
    └── workflows/
        └── ci.yml                 Install → lint → test on every push / PR
```

Rules that keep it maintainable:

* `frontend/` never imports from `backend/` or `storage/`.
* `routes/` contain no business logic; they call one service and return JSON.
* Only `storage/google-drive/` knows that Google Drive exists.
* `calculations/` never reads files or the network, so it is trivially testable
  and the browser can import the same files (served read-only at
  `/calculations/`). No formula is written twice.
* **No real data under the repository.** At start-up the server refuses to
  run if its data directory is inside the repository folder.

---

## 4. Architecture diagram

```
 ┌──────────────────────────────────────────────────────────────────────┐
 │  BROWSER (http://127.0.0.1:5173)                                     │
 │                                                                      │
 │   Pages ──► Components                                               │
 │     │                                                                │
 │     └──► api-client.js  (fetch /api/...)        calculations/*.js    │
 │                │                                 (display helpers)   │
 └────────────────┼─────────────────────────────────────────────────────┘
                  │ HTTP, localhost only
 ┌────────────────▼─────────────────────────────────────────────────────┐
 │  NODE BACKEND                                                        │
 │                                                                      │
 │   routes/  ──►  APPLICATION SERVICES  ──►  calculations/ (pure)      │
 │                 budget, transaction, report, backup, sync …          │
 │                        │                                             │
 │                        ▼                                             │
 │                 DATA ACCESS LAYER  (storage/data-access.js)          │
 │                 getCategories(), saveTransactions(month), …          │
 │                 validates every read & write                         │
 │                        │                                             │
 │          ┌─────────────┴──────────────┐                              │
 │          ▼                            ▼                              │
 │   LOCAL STAGING STORE           SYNC ENGINE                          │
 │   (working copy + base copy)    pull / 3-way merge / push / verify   │
 │   OS app-data folder                  │                              │
 │                                       ▼                              │
 │                          STORAGE ADAPTER (interface)                 │
 │                          ┌────────────┴─────────────┐                │
 │                          ▼                          ▼                │
 │                 GoogleDriveAdapter         LocalFolderAdapter        │
 │                 (Phase 5+)                 (Phases 2–4, tests)       │
 └──────────────────────────┼───────────────────────────────────────────┘
                            ▼
                ┌──────────────────────────┐
                │ GOOGLE DRIVE             │
                │ "Household Budget" folder│  owned by Google Account 1
                │ (permanent source of     │  = the ONLY account the app
                │  truth)                  │  connects with. Shared by hand
                └──────────────────────────┘  with Google Account 2 (Drive
                                              access only, not via the app)
```

Reading the diagram top to bottom:

1. **UI** shows data and collects input. It never touches files or Google.
2. **Application services** apply rules ("an expense needs a valid category",
   "create a backup before restore").
3. **Data Access Layer** is the single set of `get…/save…` methods. Pages and
   services do not know where data lives.
4. **Local staging** holds the working copy so the app is fast and works
   offline. It is **not** the permanent store.
5. **Sync engine** moves changes between staging and the cloud safely.
6. **Storage adapter** is the plug: Google Drive today, something else later.

### Data Access Layer methods

```
getHousehold()            saveHousehold(household)
getAppUsers()             saveAppUser(user)
getFamilyMembers()        saveFamilyMember(member)
getCategories()           saveCategory(category)
getPaymentMethods()       savePaymentMethod(method)
getFrequencyTypes()
getSettings()             saveSettings(settings)
getBudgetItems()          saveBudgetItems(items)
getTransactions(month)    saveTransactions(month, file)
listTransactionMonths()
createBackup(trigger)     listBackups()          restoreBackup(backupId)
syncFromCloud()           syncToCloud()          getSyncStatus()
```

All `save…` methods write to **local staging** and mark the file as having
local changes. Only `syncToCloud()` talks to the cloud.

### Storage adapter contract (what Google Drive must implement)

```
listFiles()                      → [{ path, fileId, revisionId, md5, size, modifiedTime }]
getFileMetadata(path)            → { fileId, revisionId, md5, size, modifiedTime }
readFile(path)                   → { content, metadata }
createFile(path, content)        → metadata
updateFile(path, content)        → metadata
listRevisions(path)              → [{ revisionId, modifiedTime }]
readRevision(path, revisionId)   → content
trashFile(path)                  (move to trash, never permanent delete)
checkAccess()                    → { canRead, canWrite }
```

A future backend (OneDrive, a NAS, a database) only needs to implement these.

---

## 5. Identity model (three separate concepts)

| Concept | Stored where | Used for | Ever stored in a record? |
|---|---|---|---|
| **Google Account** | Device token file (outside repo) | Authorising Drive access only. **One account (the folder owner) for every computer**; Account 2 only gets the folder shared in Google Drive | **No** |
| **Application User** (`user-01`, `user-02`) | `config/app-users.json` | "Filled By" / "Updated By" | Yes: `filledByUserId`, `createdBy`, `updatedBy` |
| **Household Member** (Self, Spouse, Child 1 …) | `config/family-members.json` | "For whom" an expense/budget is | Yes: `memberId` (optional) |

The selected application user is **identification, not security**. It is
remembered per device and can be changed from the header at any time.

---

## 6. Page map

Hash routes keep things simple (no server routing needed).

| # | Page | Route | Purpose |
|---|---|---|---|
| 1 | User Selection / Startup | `#/start` | "WHO IS ENTERING DATA?" → User 1 / User 2 → Continue. Shown on every app start, last choice pre-selected. If cloud not set up, offers Cloud Setup. |
| 2 | Dashboard | `#/dashboard` | Monthly Planned, Actual Spent, Remaining/Variance, Annual Requirement, frequency summary, planned vs actual, category & member breakdown, recent transactions, upcoming, sync state. |
| 3 | Budget Setup (wizard) | `#/budget-setup/1` … `/7` | Seven steps; ends with Estimated Monthly & Annual Requirement. |
| 4 | Budget Plan | `#/budget` | Edit budget items directly (table + cards), no wizard needed. |
| 5 | Actual Expenses | `#/expenses` | List + search/filter + **Add Expense** + edit/delete. |
| 6 | Upcoming | `#/upcoming` | Expected budget items by next expected date (no fake transactions). |
| 7 | Reports | `#/reports/:type` | Monthly, Annual, Category, Family Member, Planned vs Actual, Frequency, Filled By. |
| 8 | Household | `#/household` | Household name + household members. |
| 9 | Masters | `#/masters/:type` | Categories (with sub-categories), Payment Methods, Frequency Types. |
| 10 | Cloud Sync / Backup | `#/cloud` | Sync status, Sync Now, conflicts to review, backups list, Create/Restore Backup. |
| 11 | Settings | `#/settings/:section` | Application Users, Household Members (link), Currency, Categories/Payment Methods/Frequency (links), Cloud Storage, Backup Settings, Data Export, Data Import, Sync Status, Application Information. |

The Cloud Storage setup flow (see [google-drive-setup.md](google-drive-setup.md))
lives under `#/settings/cloud/setup` and is offered from Startup the first time.

### Header (every page except Startup)

```
[≡]  Household Budget        ● LOCAL CHANGES (3)  [Sync Now]   User 1 ▼
```

### Navigation

* **Desktop (≥ 900 px):** left sidebar with all pages.
* **Mobile (< 900 px):** bottom bar with Dashboard · Expenses · **＋ Add** ·
  Budget · More. "More" opens the rest.

### Save / Sync buttons (proposed arrangement)

* Each form has its own **Save** button → writes to local staging
  (status becomes `LOCAL CHANGES (n)`). Nothing is lost if the browser closes.
* Header **Sync Now** → pull, merge, push, verify (see [sync.md](sync.md)).
* **Create Backup** sits on the Cloud Sync / Backup page and in Settings.

---

## 7. Component structure

Every component is a plain JavaScript function that returns a DOM element.
No framework. Example: `createMoneyCard({ label, amountMinor, tone })`.

```
frontend/js/components/
├── app-shell.js          Header + sidebar/bottom-nav + content area
├── sidebar-nav.js        Desktop navigation
├── bottom-nav.js         Mobile navigation
├── user-switcher.js      "User 1 ▼" dropdown in the header
├── sync-badge.js         Coloured state pill + pending count
├── money-card.js         Big rounded card with a prominent ₹ value
├── stat-row.js           Row of money cards (dashboard KPIs)
├── category-card.js      Pastel card with icon (wizard + breakdowns)
├── step-header.js        Large section number "01" + title (wizard)
├── step-progress.js      1–7 progress indicator
├── data-table.js         Sortable table, becomes stacked cards on mobile
├── filter-bar.js         Date range, category, member, filled-by … + Reset
├── form-fields.js        Money input, date input, select-from-master helpers
├── expense-form.js       Add/Edit Expense form
├── budget-item-form.js   Add/Edit budget item form
├── master-list.js        Generic editor for masters (name, active, order)
├── bar-chart.js          Simple SVG bars (planned vs actual, breakdowns)
├── conflict-card.js      Shows "mine vs cloud" for one conflict
├── modal.js              Dialog container
├── confirm-dialog.js     "Are you sure?" with details (restore/import)
├── empty-state.js        Friendly "nothing here yet" panel
└── toast.js              Short success/error messages

frontend/js/pages/
├── startup.page.js           ├── household.page.js
├── dashboard.page.js         ├── masters.page.js
├── budget-setup.page.js      ├── cloud.page.js
├── budget-plan.page.js       ├── settings.page.js
├── expenses.page.js          └── cloud-setup.page.js
├── upcoming.page.js
└── reports.page.js
```

Every page exports one function: `render(container, context)`, where
`context` holds the current app user and the api client.

### Visual style (inspired by, not copied from, the reel)

* Rounded cards (16 px radius), soft pastel accent per category group.
* Strong headings; large two-digit step numbers in the wizard ("01"–"07").
* Money values are the largest text on a card.
* Category cards with simple icons.
* Professional neutral background; no social-media branding.

---

## 8. Development phases (summary)

| Phase | Scope |
|---|---|
| 1 | Skeleton, Git hygiene, README, local start-up |
| 2 | User selector, household members, master data |
| 3 | Budget model, frequency calculations, setup wizard |
| 4 | Local transaction storage (monthly files), dashboard |
| 5 | Google Cloud project, OAuth, Drive adapter, folder selection |
| 6 | Cloud save/load, manual sync, sync status |
| 7 | Conflict detection, two-user merge tests |
| 8 | Backups, restore, import/export |
| 9 | Reports, UI refinement, responsive design |
| 10 | Full testing, security review, docs, release prep |

**First implementation target (after approval):** Phase 1–3 items listed in
the requirements (skeleton, start-up, user selector, household members,
categories, payment methods, frequency master, budget items, calculation
module, tests). Phases 2–4 use the `LocalFolderAdapter`; the status shows
**LOCAL ONLY — cloud not set up** so nobody mistakes it for cloud storage.
Only synthetic test data should be entered until Phase 6 is done.

Related documents:
[data-model.md](data-model.md) ·
[google-drive-setup.md](google-drive-setup.md) ·
[storage.md](storage.md) ·
[sync.md](sync.md) ·
[backups.md](backups.md) ·
[calculations.md](calculations.md) ·
[testing.md](testing.md) ·
[github-workflow.md](github-workflow.md) ·
[open-questions.md](open-questions.md)
