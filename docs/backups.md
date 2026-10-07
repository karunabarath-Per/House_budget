# Backups, Restore, Import and Export

> Status: **PROPOSAL — awaiting approval.**

## Backup file

One self-contained JSON file in Drive `backups/`:

```
backup-20261007-194500-manual.json
backup-20261007-200110-before-restore.json
```

```json
{
  "schemaVersion": 1,
  "kind": "household-budget-backup",
  "backupId": "uuid",
  "createdAt": "2026-10-07T19:45:00+05:30",
  "createdByUserId": "user-01",
  "trigger": "manual",
  "appVersion": "0.1.0",
  "householdId": "6f1c…uuid",
  "recordCounts": {
    "appUsers": 2, "familyMembers": 4, "categories": 13,
    "paymentMethods": 6, "frequencyTypes": 6, "budgetItems": 27,
    "transactionMonths": 10, "transactions": 412
  },
  "contentSha256": "<hash of the data block>",
  "data": {
    "config": { "household": {}, "appUsers": {}, "familyMembers": {}, "categories": {},
                "paymentMethods": {}, "frequencyTypes": {}, "settings": {} },
    "budget": { "budgetItems": {} },
    "transactions": { "2026-10": {}, "2026-09": {} }
  }
}
```

The manifest is **not** included (it holds Drive IDs that change);
restore rebuilds it.

## When backups happen

| Trigger | Default | Can be turned off? |
|---|---|---|
| Manual — **Create Backup** button | — | — |
| Before restore | on | No |
| Before import | on | No |
| Before bulk/destructive operations (e.g. delete a month, reset data) | on | No |
| First successful sync of each day | on | Yes |

Backups are uploaded and **verified** (re-download, checksum) like any
other file. A failed backup **blocks** the operation it was protecting.

If the cloud is unreachable, the safety backup goes to the device's
`local-backups/` and destructive cloud operations (restore) are not offered.

## Retention (documented default — please confirm, Q8)

* **Manual backups: never deleted automatically.**
* **Automatic backups: keep the newest 30.** Older automatic ones are moved
  to Drive **trash** (recoverable from Drive for 30 days), never
  permanently deleted.
* Retention runs only after a new backup has been verified.
* Configurable in Settings → Backup Settings.

## Restore

```
[Restore Backup]
1. List backups: date, trigger, created by, size.
2. Choose one → show date, size, record counts, checksum OK ✓.
3. Validate the whole file (schema + references). Invalid → stop.
4. Create safety backup of current cloud data ("before-restore") — verified.
5. Confirm: "Replace ALL current household data with the backup from
   3 Oct 2026? A safety backup was saved as …"
6. Restore: write each file (check-before-write, verify-after-write),
   rebuild manifest, increment restoreGeneration.
7. Validate restored cloud data by re-reading it.
8. Mark this device SYNCED; the other device will be told on its next sync
   (see sync.md "After a restore").
```

Pending local changes on the restoring device are saved to
`local-backups/` first and the user is asked whether to discard them.

## Export

* **JSON export** → same structure as a backup with `"kind":
  "household-budget-export"`, saved to Drive `exports/` or downloaded to the
  computer (user chooses). Downloaded exports must never be put in the repo.
* Exporters live behind one interface (`exportTo(format)`), so CSV and Excel
  can be added later without touching the rest.

## Import (JSON)

```
1. Choose file → parse. Malformed JSON → reject whole file.
2. Validate schema version (migrate older; reject newer).
3. Validate every record and every reference.
4. Preview: "412 transactions, 27 budget items… 3 rejected (reasons…)"
5. Choose mode: Merge (default) or Replace all.
6. Safety backup (verified).
7. Confirm → apply to local working copy → LOCAL CHANGES → Sync Now.
```

* **Merge**: new IDs added; identical IDs skipped; same ID with different
  content listed as conflicts for review.
* **Replace all**: behaves like a restore.
* If the file as a whole is invalid nothing is imported (no partial import).
  Individually invalid records are listed as rejected and only imported if
  the user confirms the rest.
