# Testing Strategy

> Status: **PROPOSAL — awaiting approval.**

Runner: Node's built-in `node --test`. Command: `npm test`. CI runs it on
every push and pull request.

## Layers

| Layer | What | Where |
|---|---|---|
| Unit | Pure functions: money, frequency, totals, variance, schedule, reports, merge, sync state machine, validators, migrations | `tests/unit/` |
| Integration | Services + Data Access Layer + `LocalFolderAdapter` in a temporary directory; full sync between two simulated devices | `tests/integration/` |
| Drive contract | Same adapter-contract tests run against a fake in-memory Drive (CI) and, manually, against a real test folder (never in CI, never real data) | `tests/integration/adapter-contract.test.js` |
| Manual UI checklist | Startup, wizard, add expense, sync badge, mobile layout | `docs/testing.md` checklist (Phase 9) |

## Required test areas

| Area | Example cases |
|---|---|
| Frequency | each frequency; weekly 365/7 rounding; zero; very large safe amounts |
| Money | parse `"850.75"`, `"850.7"`, `"1,234"`, reject `"abc"`, `"-5"`, 3 decimals; INR formatting `₹1,23,456.75` |
| Budget totals | inactive excluded; date range; sub-category roll-up |
| Planned vs actual | deleted transactions excluded; variance sign |
| Transaction merge | every row of the merge table in sync.md; move between months |
| Conflict detection | stale base revision detected; race caught by revision check |
| Two users | **User 1 and User 2 on two simulated devices add, edit and delete independently, then sync in both orders — final cloud data identical, nothing lost** |
| Backup | counts and checksum correct; verification failure blocks the operation |
| Restore | safety backup made first; invalid backup rejected; restoreGeneration handled on the other device |
| Validation | malformed JSON, bad dates, unknown references, wrong month, unsafe integers |
| Schema migration | v1 → v2 fixture; newer version refused |
| Sync states | every transition in sync.md; "Synced" never shown after failed verification |

## Test data

Only synthetic fixtures (`"Test Member A"`, round amounts). No real names,
amounts or exports — ever — in `tests/`.
