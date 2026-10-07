# Synchronisation and Conflict Handling

> Status: **PROPOSAL — awaiting approval.**

## Principles

1. Nothing goes to the cloud until someone presses **Sync Now**.
2. Cloud data is **never** silently overwritten.
3. "Synced" is shown **only** after the upload has been verified.
4. Unsynchronised changes are never silently lost.

## Sync states

| State | Meaning | Badge |
|---|---|---|
| `LOCAL_ONLY` | Cloud not set up yet | grey |
| `SYNCED` | No local changes; local copy matched the cloud at the last check (time shown) | green |
| `LOCAL_CHANGES` | Saved on this computer, not yet in the cloud (count shown) | amber |
| `SYNCING` | Sync in progress; editing is paused | blue |
| `SYNC_FAILED` | Last sync did not complete or could not be verified; changes are still safe locally | red |
| `CONFLICT` | Someone else changed the same thing; needs your review | purple |
| `OFFLINE` | Cloud unreachable; "Changes saved locally and waiting for sync." | grey |

### Transitions

```
LOCAL_ONLY ──setup done──► SYNCED
SYNCED ──local save──► LOCAL_CHANGES
SYNCED / LOCAL_CHANGES / SYNC_FAILED ──Sync Now──► SYNCING
SYNCING ──verified, no conflicts──► SYNCED
SYNCING ──conflict found──► CONFLICT
SYNCING ──error / unverified──► SYNC_FAILED
CONFLICT ──all resolved + Sync Now──► SYNCING
any ──network/Drive unreachable──► OFFLINE
OFFLINE ──connection back──► LOCAL_CHANGES (if pending) or SYNCED
         (never auto-uploads; user must press Sync Changes)
```

These transitions are a pure function (`sync-state.js`) with unit tests.

## Version detection

Google Drive gives each file `headRevisionId`, `md5Checksum`, `version`
and `modifiedTime`. We use **`headRevisionId` + `md5Checksum`** as the
identity of the cloud content. `modifiedTime` alone is not trusted
(clock-based, second-level, can tie).

Drive v3 has no "only write if the version is still X" switch. So the app
**checks before writing and verifies after writing** using the revision
history, which catches the rare case where the other person uploaded in the
same few milliseconds.

## Sync Now algorithm

```
syncNow():
  0. Preconditions: cloud connected, online, no other sync running on
     this device. State → SYNCING.

  PULL
  1. List cloud files (id, name, headRevisionId, md5, size).
     Read manifest. If restoreGeneration increased → see "After a restore".
  2. For each file whose revision ≠ our base revision:
       download → validate → migrate older schema
       newer schema → STOP: "Update the app" (SYNC_FAILED)
       invalid JSON → STOP, keep local data, report file name.
  3. For each file: merged = threeWayMerge(base, localWorking, cloud).
  4. If any conflict that cannot be merged safely:
       save non-conflicting merge results to working/,
       write conflicts/, state → CONFLICT, STOP (no upload).
  5. Otherwise base ← cloud copy, working ← merged.

  PUSH (only files where merged ≠ cloud)
  6. Automatic backup if the backup policy says so (e.g. first sync of day).
  7. For each file:
       a. fetch metadata; if headRevisionId ≠ the revision just merged
          → someone synced meanwhile → go back to step 1
          (max 3 rounds, then SYNC_FAILED "Cloud is busy, try again").
       b. upload new content.
  8. VERIFY each uploaded file:
       - file exists and has a new revision
       - downloaded content md5 == md5 of what we uploaded, and it parses
         and validates
       - in revision history, the revision just before ours is the one we
         merged against. If not, another upload slipped in between: its
         content still exists as a Drive revision → download it, merge,
         upload again (back to step 1).
     Any verification failure → SYNC_FAILED
       "Cloud save could not be verified." Local changes stay pending.
  9. base ← verified content; clear dirty flags.
 10. Update manifest (lastSuccessfulSync, file revisions) with the same
     check → write → verify steps.
 11. State → SYNCED.
```

"Get latest" (pull only, steps 1–5) is also available for when you just
want to see the other person's entries.

## Three-way merge

We keep three versions of every file:

* **base** — what the cloud had when this device last synced
* **local** — this device's working copy
* **cloud** — what is in the cloud now

Records are matched **by `id`**, never by position.

### Transactions and other ID-keyed lists (budget items, masters)

| base | local | cloud | Result |
|---|---|---|---|
| — | new | — | keep (added here) |
| — | — | new | keep (added by other person) |
| — | new A | new B, same id | **conflict** (practically impossible with UUIDs) |
| X | X | X′ | take cloud (changed there only) |
| X | X′ | X | take local (changed here only) |
| X | X′ | X′ (identical) | take either |
| X | X′ | X″ (different) | **conflict** — show both, user chooses |
| X | deleted | X | deleted |
| X | X | deleted | deleted |
| X | X′ (edited) | deleted | **conflict** — "edited here, deleted there" |
| X | deleted | deleted | deleted |

Masters are never hard-deleted (only deactivated), so for masters the
delete rows never occur. Duplicate *names* (e.g. both people add "Fuel"
offline) are allowed by merge and flagged as a gentle warning
("2 categories called Fuel — merge them?"), never auto-merged.

### Single-object files (household.json, settings.json)

Changed on one side only → take that side. Changed on both sides →
**conflict**, show both, user chooses. No field-level guessing.

### Conflict review screen

For each conflict: what it was (base), your version, the cloud version, who
changed it (app user) and when. Choices: **Keep mine**, **Keep cloud**, and
for transactions **Keep both** (the "mine" copy gets a new id). Nothing is
uploaded until every conflict is resolved and Sync Now is pressed again.

## Example: two users, same month

```
1. User 1 and User 2 both sync → both have 2026-10.json revision A.
2. User 1 adds "Milk ₹60", syncs → cloud revision B.
3. User 2 (still on base A) adds "Bus ₹40", presses Sync Now.
4. Step 2: cloud is B ≠ base A → "Cloud data changed since your last sync."
   download B.
5. Merge(base A, local A+Bus, cloud A+Milk) → A + Milk + Bus. No conflict.
6. Upload revision C, verify, Synced. Both entries kept, nothing lost.
```

If instead both had edited the **same** transaction differently, step 4
would produce CONFLICT and User 2 chooses.

## After a restore

A restore increments `restoreGeneration` in the manifest. When the other
device sees a higher generation:

* With no local changes → replace base and working with the restored data.
* With local changes → state `CONFLICT`: "Household data was restored by
  User 1 on 7 Oct 2026. You have 4 unsynced changes made before that. Review
  them." Each change can be re-applied or discarded (a local backup is made
  first).

## Offline

If Drive cannot be reached, saving continues locally and the badge shows
**OFFLINE — Changes saved locally and waiting for sync.** When the
connection returns, the app does **not** upload by itself; the user presses
**Sync Changes**, which runs the full algorithm above with all conflict checks.
