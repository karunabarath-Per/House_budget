# Google Drive Strategy and OAuth Flow

> Status: **PROPOSAL — awaiting approval.** Contains a scope issue that
> needs a decision before Phase 5: see "The `drive.file` problem" below and
> [open-questions.md](open-questions.md) Q1.

## Ownership model

```
Google Account 1 ──owns──► "Household Budget" folder (My Drive, not a Shared Drive)
                                │
                                └── shared with Google Account 2 as "Editor" (writer)
```

* Google accounts only **authorise Drive access**. They are not app users.
* There is exactly **one** household dataset. Account 2 never creates a
  second one; setup for Account 2 only *finds* and *verifies* the existing
  folder.
* The app only works inside this folder and files it created or the user
  selected. It never browses or edits other Drive content.

## Folder / file layout

```
Household Budget/
├── config/
│   ├── app-manifest.json
│   ├── household.json
│   ├── app-users.json
│   ├── family-members.json
│   ├── categories.json
│   ├── payment-methods.json
│   ├── frequency-types.json
│   └── settings.json
├── budget/
│   └── budget-items.json
├── transactions/
│   └── 2026-10.json  (created when the first expense of a month is synced)
├── backups/
│   └── backup-20261007-194500-manual.json
└── exports/
    └── export-20261007-195000.json
```

No other files are created. Files are found by the IDs recorded in
`app-manifest.json`, not by name (Drive allows two files with the same name).

## OAuth flow (authorization code + PKCE)

The Node backend runs the flow. The browser never sees the refresh token.

```
Browser                    Local backend (127.0.0.1)              Google
   │ click "Connect Google Drive"   │                                 │
   │───────────────────────────────►│ create state + PKCE verifier    │
   │                                │ (code_challenge = S256(verifier))
   │◄──── redirect to Google ───────│                                 │
   │──────────────────────────────────────────────────────────────────►│
   │        consent screen: "Household Budget wants to see, edit,     │
   │        create … only the specific Google Drive files you use     │
   │        with this app"   (scope: drive.file)                      │
   │◄────── redirect to http://127.0.0.1:5173/oauth/callback?code&state
   │───────────────────────────────►│ check state                     │
   │                                │ exchange code + verifier ───────►│
   │                                │◄──── access + refresh token ─────│
   │                                │ save token file (0600) in       │
   │                                │ OS app-data dir, NOT the repo   │
   │◄────── "Cloud connected ✓" ────│                                 │
```

Details:

* Google Cloud project with an OAuth client of type **Web application**,
  redirect URI `http://127.0.0.1:5173/oauth/callback`. Client ID and secret go
  in `.env` (never committed); `.env.example` has placeholders.
* Parameters: `response_type=code`, `code_challenge_method=S256`,
  `access_type=offline`, `scope=https://www.googleapis.com/auth/drive.file`,
  random `state` checked on return. No implicit flow.
* Libraries: Google's official `google-auth-library` (it generates PKCE
  pairs). The Google Identity Services *browser* code-client is not used for
  the exchange because it does not let us supply a PKCE verifier; doing the
  exchange on the local backend is the standard Google "web server" flow
  with PKCE added.
* **Google Picker** runs in the browser. It needs a short-lived access token,
  which the backend hands to the localhost page on request (never the refresh
  token), plus an API key restricted to `http://127.0.0.1:5173` and the
  Cloud project number as App ID.
* "Disconnect Google Drive" revokes the token and deletes the token file.
  Pending local changes are kept and the user is warned.
* OAuth consent screen: "External", with the two Google accounts as test
  users. `drive.file` is a **non-sensitive** scope, so the app can be moved
  to "In production" status without Google's restricted-scope security
  review. That matters: in "Testing" status Google expires refresh tokens
  after 7 days, which would force both people to reconnect every week.

## The `drive.file` problem (needs your decision — Q1)

`drive.file` gives the app access only to files that **this Google account**
either created with the app or picked in Google Picker. Files are not
shared between the two accounts' grants just because they sit in a shared
folder.

What that means here:

| Situation | With `drive.file` |
|---|---|
| Account 1 creates the folder and files during setup | ✓ Account 1 can use them |
| Account 2 picks the folder + all files in Picker during its setup | ✓ Account 2 can use those files |
| Later, Account 1's app creates `transactions/2026-11.json` | ✗ **Account 2's app cannot see it** until Account 2 picks it in Picker |
| Account 2's app cannot see it, so creates its own `2026-11.json` | ✗ two files with the same name → must be detected and merged |
| Backups created by Account 1 | ✗ invisible to Account 2 until picked |

I have not yet been able to test this with two real accounts; Phase 5
starts with a small **two-account spike** to confirm the exact behaviour.

Options:

* **A (recommended): keep `drive.file`, design around it.**
  * Setup creates all 12 transaction files of the year up front (and next
    year's in December), so new months rarely need new files.
  * The manifest lists every file ID. When a device finds a listed file it
    cannot open, the status shows **"2 cloud files need your permission"**
    and a button opens Picker at the Household Budget folder with
    multi-select; the user selects all and confirms.
  * Duplicate-name detection merges accidental duplicates.
  * Cost: occasional extra Picker step for Account 2 (roughly once a year,
    plus when restoring a backup made by the other account).
* **B: request full Drive access** (`https://www.googleapis.com/auth/drive`).
  Everything "just works", but the app could read/edit **every file** in both
  Google Drives, and because it is a restricted scope the consent screen
  would stay in "Testing" status → reconnect every 7 days (or go through
  Google's paid security assessment).
* **C: Account 2 does not authorise at all; both devices use Account 1.**
  Contradicts the ownership model in the requirements. Not recommended.

Per the requirements I have **not** chosen a broader scope.

## First-time setup — Account 1 (owner)

```
Cloud Storage Setup
1. Connect Google Drive        → OAuth flow above
2. Authorize application        → consent for drive.file
3. Select Household Budget folder → "Create new folder" (app creates it,
                                    so it has access) or Picker for an
                                    existing EMPTY folder
4. Verify access                → create/read/delete a probe file
5. Initialize application files → sub-folders, seed config files,
                                  manifest, current-year month files
6. Confirm successful setup     → Cloud connected ✓  Folder found ✓
                                  Files verified ✓
Then: instructions to share the folder with Account 2 as Editor in
Google Drive (done by the person in Drive, not by the app).
```

If local data already exists (Phases 2–4), step 5 asks whether to upload it
as the initial cloud data (with a local backup first).

## First-time setup — Account 2

```
1. Connect Google Drive
2. Authorize application
3. Select the shared Household Budget folder (Picker → "Shared with me",
   then multi-select all its files, Option A)
4. Verify writer access   → read manifest, check householdId,
                            check capabilities.canEdit on each file
5. Load existing application data → download, validate, store as base
                                    + working copy
6. Cloud connected ✓  Folder found ✓  Files verified ✓
```

If the selected folder has no manifest, setup stops: "This is not a
Household Budget folder. Ask the owner to share the correct folder." It
never initialises a second dataset from Account 2.

## Configuration storage

The chosen root folder ID, manifest file ID and connected-account email (for
display only) are stored in the device's `device.json` in the OS app-data
directory. Nothing Drive-specific is hard-coded.
