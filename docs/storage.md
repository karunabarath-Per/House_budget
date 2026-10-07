# Storage and Local Staging

> Status: **PROPOSAL — awaiting approval.**

## Where things live

| Data | Location | Permanent? |
|---|---|---|
| Household data (config, budget, transactions) | Google Drive "Household Budget" folder | **Yes — source of truth** |
| Backups | Drive `backups/` | Yes |
| Working copy + pending changes | OS app-data directory on each computer | No — temporary staging |
| OAuth refresh token | OS app-data directory, file permission 0600 | Device secret |
| Source code | Private GitHub repo | Code only, never data |
| Browser `localStorage` | Only UI conveniences (e.g. last opened tab) | No |

## App-data directory

| OS | Default |
|---|---|
| Windows | `%LOCALAPPDATA%\HouseholdBudget\` |
| macOS | `~/Library/Application Support/HouseholdBudget/` |
| Linux | `$XDG_DATA_HOME/household-budget/` or `~/.local/share/household-budget/` |

Override with `HB_DATA_DIR` in `.env`. **The server refuses to start if the
resolved directory is inside the repository**, so real data cannot end up in
Git by accident.

## Layout

```
HouseholdBudget/
├── device.json              deviceId, deviceName, selectedAppUserId,
│                            cloud { provider, rootFolderId, manifestFileId,
│                                    connectedAccountHint }
├── secrets/
│   └── google-token.json    refresh token (0600). Delete = disconnect.
├── staging/
│   ├── working/             What the app shows and edits.
│   │   ├── config/*.json
│   │   ├── budget/budget-items.json
│   │   └── transactions/YYYY-MM.json
│   ├── base/                Exact copy of each file as last VERIFIED in the
│   │                        cloud. Needed for three-way merge.
│   ├── sync-state.json      state, per-file known cloud revision, dirty flags,
│   │                        conflicts, lastSyncAt, lastError, restoreGeneration
│   └── conflicts/           One JSON per unresolved conflict
├── local-backups/           Safety copies made when cloud is unreachable
│                            (e.g. before discarding local changes)
└── logs/app.log             Events only — no amounts, notes or names
```

### sync-state.json (per device)

```json
{
  "schemaVersion": 1,
  "state": "LOCAL_CHANGES",
  "restoreGeneration": 0,
  "lastSyncAt": "2026-10-07T19:45:10+05:30",
  "lastCheckedCloudAt": "2026-10-07T19:45:10+05:30",
  "lastError": null,
  "files": {
    "transactions/2026-10.json": {
      "fileId": "<id>",
      "baseRevisionId": "<rev>",
      "baseMd5": "<md5>",
      "dirty": true,
      "pendingCount": 3
    }
  },
  "conflicts": []
}
```

## Write safety

* Every local write is **atomic**: write `file.tmp` → flush → rename over
  the old file. A crash never leaves half a JSON file.
* Every write is validated against the schema first. Invalid data is
  rejected with a clear message.
* Local changes are only cleared after the cloud upload is **verified**.
* If the user tries to disconnect, switch Google account, or restore while
  there are pending changes, the app first writes them to
  `local-backups/` and asks for confirmation.

## Modes

| Mode | When | Status label |
|---|---|---|
| Local only | Phases 2–4, or before Cloud Setup | `LOCAL ONLY — cloud not set up` |
| Cloud | After Cloud Setup | `SYNCED`, `LOCAL CHANGES`, … (see [sync.md](sync.md)) |

In local-only mode the `LocalFolderAdapter` stands in for the cloud (useful
for tests), but the UI makes clear the data exists only on this computer.
