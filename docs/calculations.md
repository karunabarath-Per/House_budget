# Calculations

> Status: **PROPOSAL — awaiting approval.** Rounding details depend on Q5.

All formulas live in `calculations/`. They are pure functions (no files, no
network), used by both backend and browser, and fully unit-tested. **No
formula is written anywhere else.**

## Money (`money.js`, `currency.js`)

* Stored as integer minor units: ₹850.75 → `85075`.
* User input is parsed **as text** (`"850.75"` → `85075`), never
  `parseFloat(x) * 100` (that gives 85074.99999…).
* Every result is checked with `Number.isSafeInteger` (safe up to about
  ₹90 lakh crore — far beyond household needs).
* Display uses `Intl.NumberFormat('en-IN', { style: 'currency', currency: 'INR' })`
  → `₹1,23,456.75` (Indian digit grouping).
* Currency settings `{ code, symbol, exponent, locale }` come from
  `settings.json`; nothing assumes 2 decimals or ₹ in code.

## Frequency planning equivalents (`frequency.js`)

> These are **planning equivalents**, not accounting actuals. The UI labels
> them that way.

| Frequency | Annual equivalent |
|---|---|
| Daily | amount × 365 |
| Weekly | amount × 365/7 (= 52.142857…) |
| Monthly | amount × 12 |
| Quarterly | amount × 4 |
| Half-Yearly | amount × 2 |
| Yearly | amount × 1 |

`monthlyEquivalent = annual ÷ 12`

How it is computed without floating point:

```
annualMinor  = roundHalfUp(amountMinor × numerator ÷ denominator)
monthlyMinor = roundHalfUp(amountMinor × numerator ÷ (denominator × 12))
```

Each value is rounded **once**, from the original amount (no rounding of an
already rounded number). Totals = sum of per-item rounded values.

Example: ₹1,000 weekly → annual `100000 × 365 ÷ 7 = 5214285.71…` → ₹52,142.86;
monthly `100000 × 365 ÷ 84 = 434523.80…` → ₹4,345.24.

Leap years: daily stays × 365 (it is a planning figure).

## Budget totals (`budget-totals.js`)

Only **active** items whose date range overlaps the period count. Totals by:
whole household, category (sub-categories roll up to parent), member,
frequency, scope. Produces the wizard's **Estimated Monthly Requirement** and
**Estimated Annual Requirement**.

## Planned vs actual (`planned-vs-actual.js`)

```
actual   = sum(amountMinor of non-deleted transactions in the period)
variance = planned − actual      (positive = under budget)
```

Which "planned" figure a month uses depends on Q3 (monthly equivalent vs
amount due that month).

## Schedule / upcoming (`schedule.js`)

Next expected date from `startDate` + frequency step, within
`startDate…endDate`:

* Month-based steps are computed from the anchor each time, with month-end
  clamping: anchor 31 Jan monthly → 28/29 Feb, 31 Mar, 30 Apr…
* An item is shown as "possibly paid" when a transaction with that
  `budgetItemId` already exists in the current period. Upcoming never
  creates transactions.

## Reports (`reports.js`)

Monthly, Annual, Category, Family Member, Planned vs Actual, Frequency,
Filled By (number of entries, total entered). Services fetch the data;
`reports.js` only aggregates arrays it is given.
