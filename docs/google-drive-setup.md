# Google Drive Strategy and OAuth Flow

> Status: **PROPOSAL — awaiting approval.**
> Decision recorded (2026-10-07): **the app connects to ONE Google account
> only** (the folder owner). The owner shares the folder with the second
> Google account manually in Google Drive; the app never uses the second
> account.

## Ownership and access model

```
Google Account 1 (owner)
   │
   ├── owns "Household Budget" folder (My Drive, not a Shared Drive)
   │
   ├── is the ONLY account the app is authorised with
   │      └── every computer running the app connects as Account 1
   │
   └── shares the folder with Google Account 2  (done by hand in Google Drive)
              └── Account 2 can open the folder in Google Drive itself
                  (view the files, download backups/exports).
                  The app does NOT connect as Account 2.
```

* The Google account only **authorises Drive access**. It is not an app user.
  User 1 and User 2 both pick themselves in the app's user selector; who
  filled each record is recorded as an **application user**, regardless of
  which Google account is connected.
* There is exactly **one** household dataset, created once by setup.
* The app only works inside this folder, on files it created. It never
  browses or edits other Drive content.

### Why one account

With the narrow `drive.file` permission, Google gives an app access only to
files that the **same Google account** created with the app (or picked in
Google Picker). If two different accounts each authorised the app, files
created through one account would be invisible to the other, needing extra
"grant access" steps and duplicate-file handling. With one connected
account:

* `drive.file` covers **every** file the app creates, on every computer.
* No broader Drive scope is needed.
* No Picker re-granting, no duplicate month files from invisible files.

### Sharing with Account 2 — Viewer or Editor?

This is set by the owner in Google Drive, not by the app.

| Permission | Account 2 can… | Risk |
|---|---|---|
| **Viewer (recommended)** | open/download files and backups in Drive | none to app data |
| Editor | also change, rename, delete or add files in the folder by hand | a hand-edited or deleted JSON file can break sync; files Account 2 *adds* are invisible to the app (`drive.file`) |

The app is safe either way: it validates every file it reads and
refuses to sync a corrupt file, and backups allow recovery. Editor is
only needed if Account 2 must manage the files directly in Drive.

### Consequences to be aware of

* On a second computer, the person setting it up signs in to Google **as
  Account 1** once during setup (Account 1's password or approval is needed
  then). After that the computer keeps its own refresh token.
* Drive's own "modified by" history will always show Account 1. Who entered
  each record is visible in the app (`filledByUserId` / `updatedByUserId`).
* Removing access: in Account 1's Google settings → Security → Third-party
  access, remove "Household Budget" (disconnects all computers), or use
  "Disconnect" inside the app on one computer.
* Two computers using the same Google account can still edit at the same
  time, so all conflict detection in [sync.md](sync.md) still applies.

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
Month files are created on demand; because every computer uses the same
Google account, every computer sees them.

## OAuth flow (authorization code + PKCE)

The Node backend runs the flow. The browser never sees the refresh token.

```
Browser                    Local backend (127.0.0.1)              Google
   │ click "Connect Google Drive"   │                                 │
   │───────────────────────────────►│ create state + PKCE verifier    │
   │                                │ (code_challenge = S256(verifier))
   │◄──── redirect to Google ───────│                                 │
   │──────────────────────────────────────────────────────────────────►│
   │        sign in as Google Account 1 (owner)                       │
   │        consent: "see, edit, create … only the specific Google    │
   │        Drive files you use with this app"   (scope: drive.file)  │
   │◄────── redirect to http://127.0.0.1:5173/oauth/callback?code&state
   │───────────────────────────────►│ check state                     │
   │                                │ exchange code + verifier ───────►│
   │                                │◄──── access + refresh token ─────│
   │                                │ save token file (0600) in       │
   │                                │ OS app-data dir, NOT the repo   │
   │◄────── "Cloud connected ✓" ────│                                 │
```

Details:

* Google Cloud project (created under Account 1) with an OAuth client of
  type **Web application**, redirect URI
  `http://127.0.0.1:5173/oauth/callback`. Client ID and secret go in `.env`
  (never committed); `.env.example` has placeholders.
* Parameters: `response_type=code`, `code_challenge_method=S256`,
  `access_type=offline`, `scope=https://www.googleapis.com/auth/drive.file`,
  random `state` checked on return. No implicit flow.
* Libraries: Google's official `google-auth-library` (it generates PKCE
  pairs). The Google Identity Services *browser* code-client is not used for
  the exchange because it does not let us supply a PKCE verifier; doing the
  exchange on the local backend is Google's standard "web server" flow with
  PKCE added.
* **Account check:** after sign-in the backend reads the connected account's
  Drive identity (`about.get` → user email, `drive.file` allows this). On
  every computer after the first, it must match the owner recorded at setup;
  otherwise: "This computer is signed in to a different Google account.
  Connect with the Household Budget owner account." The email is stored only
  in local `device.json` for this check and display, never in household
  records.
* **Google Picker** is only used if the owner wants to choose an existing
  empty folder instead of letting the app create one. It needs a short-lived
  access token (handed by the backend to the localhost page, never the
  refresh token), an API key restricted to `http://127.0.0.1:5173`, and the
  Cloud project number as App ID.
* "Disconnect Google Drive" revokes this computer's token and deletes the
  token file. Pending local changes are kept and the user is warned.
* OAuth consent screen: "External", Account 1 as test user. `drive.file` is
  a **non-sensitive** scope, so the app can be switched to "In production"
  status without Google's restricted-scope security review. That matters:
  in "Testing" status Google expires refresh tokens after 7 days, forcing a
  reconnect every week.

## First-time setup — first computer

```
Cloud Storage Setup
1. Connect Google Drive          → OAuth flow above, signed in as Account 1
2. Authorize application          → consent for drive.file
3. Select Household Budget folder → "Create new folder" (recommended; the
                                    app creates it, so it has access) or
                                    Picker for an existing EMPTY folder
4. Verify access                  → create / read / trash a probe file
5. Initialize application files   → sub-folders, seed config files, manifest
6. Confirm successful setup       → Cloud connected ✓  Folder found ✓
                                    Files verified ✓
Then: on-screen instructions for sharing the folder with Google Account 2
in Google Drive (Share → add Account 2 → Viewer, or Editor if needed).
```

If local data already exists (Phases 2–4), step 5 asks whether to upload it
as the initial cloud data (with a local backup first).

## Setup — another computer (same household)

```
1. Connect Google Drive          → sign in as Account 1 (owner)
2. Authorize application
3. Find Household Budget folder  → the app looks up its own manifest
                                    (files created by this app + account
                                    are visible); no Picker needed
4. Verify access                  → manifest householdId, read/write check
5. Load existing application data → download, validate, store as base +
                                    working copy
6. Cloud connected ✓  Folder found ✓  Files verified ✓
```

If more than one Household Budget manifest is found, the user picks one and
the app shows each one's creation date and record counts. Setup on another
computer never creates a second dataset.

## Configuration storage

The root folder ID, manifest file ID and owner account email (for the
account check) are stored in the device's `device.json` in the OS app-data
directory. Nothing Drive-specific is hard-coded in source.
