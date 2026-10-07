# Open Questions (need your decision)

Only questions that change behaviour, money figures or data safety are listed.
Each has a recommendation. Nothing below has been decided silently.

## Q1. Google Drive scope and how Account 2 accesses files ⚠ (blocks Phase 5)

**Why it matters:** with `drive.file`, Google Account 2's copy of the app
cannot see files that Google Account 1's copy created later (for example a
new month's file or a backup), even inside the shared folder. Details in
[google-drive-setup.md](google-drive-setup.md#the-drivefile-problem-needs-your-decision--q1).

* **A (recommended):** keep `drive.file`. Pre-create the year's 12 month files;
  when needed, show "N cloud files need your permission" → one Picker
  multi-select. Verified with a two-account test at the start of Phase 5.
* **B:** full Drive scope. Simpler, but the app can access all files in both
  Drives, and you must reconnect every 7 days (unless you pay for Google's
  security assessment).
* **C:** only Account 1 ever authorises. Breaks the agreed ownership model.

## Q2. Which devices, and do you need phone access?

**Why it matters:** there is no app login. If the server is reachable from
the home Wi-Fi, anyone on that network could read and change your finances.

* **A (recommended):** each person runs the app on their own computer,
  reachable only from that computer (`127.0.0.1`). Mobile layout still
  works for small windows; phone use waits for "future hosting".
* **B:** one shared computer used by both (one Google account connected,
  user selector picks who is entering).
* **C:** phones over home Wi-Fi now. Needs at least a device PIN or
  network restriction, which goes beyond "no authentication" — your call.

Also helpful: Windows, macOS or Linux? (Only affects setup docs.)

## Q3. What does "Monthly Planned" mean on the dashboard?

**Why it matters:** a ₹12,000 yearly insurance is ₹1,000/month "on average",
but ₹12,000 in the month it is due. Variance looks very different.

* **A (recommended):** headline = **monthly equivalent** (smoothed planning
  figure), plus a second line **"Due this month"** (cash-flow, from the
  schedule). Remaining/Variance shown against the monthly equivalent, with
  the due-this-month figure beside it.
* **B:** headline = due this month only.
* **C:** monthly equivalent only.

## Q4. Annual period: calendar year or Indian financial year?

**Why it matters:** Annual reports, "Annual Requirement" and yearly totals.

* **A:** Calendar year (Jan–Dec) — matches the monthly file naming.
* **B:** Financial year (Apr–Mar) — matches Indian tax/insurance cycles.
* **Recommendation:** make it a setting (`yearStartMonth`), default
  **January**; choose B if you think in financial years.

## Q5. Rounding

**Why it matters:** totals can differ by a few paise depending on method.

Recommended (please confirm):
* Weekly factor exactly **365/7** (your 52.142857 is that number rounded;
  difference is under ₹0.01 per ₹1 lakh a year).
* Round **half-up to the nearest paisa, once per budget item**; totals are
  sums of item values.
* Show paise in forms/tables; show **whole rupees** on big dashboard cards.

## Q6. Amounts: expenses only? Refunds?

**Why it matters:** "Remaining" and all totals.

* **Recommended for Version 1:** expenses only, amounts must be > 0. No
  income tracking. A refund is recorded later as a separate type (not now).
* Alternative: allow negative amounts as refunds now.

## Q7. Record the Google account on each transaction?

Your identity example lists "Google Account: Google Account 2" next to a
transaction.

* **Recommended:** do **not** store Google account emails in records (keeps
  the three identities separate and avoids personal data in files). Drive's
  own file history and the manifest's `lastSuccessfulSync` already show
  which device/account uploaded.
* Alternative: store an account label (not email) per transaction.

## Q8. Confirm documented defaults

Tell me if you disagree with any:

1. Automatic backups: keep newest **30**; manual backups never auto-deleted;
   old ones go to Drive trash, never permanently deleted.
2. Daily automatic backup on first sync of the day.
3. Import mode default **Merge** (Replace-all available, works like restore).
4. Master data (categories, members, users, payment methods) is
   **deactivated, never deleted**.
5. Frontend in plain JavaScript (no React/Vue, no build step); backend
   Node + Express.
6. LICENSE = "All rights reserved" (private family software, not open source).
7. Forms save locally; header **Sync Now** uploads. (No separate global
   "Save Changes" button, because each form already saves.)
8. Category hierarchy limited to two levels (category → sub-category).

## Note

The budgeting reel mentioned in the brief was not included in the
repository or the message, so the visual design is based on the written
description. Screenshots can be added to `docs/` later if you want closer
alignment (they are not personal data).
