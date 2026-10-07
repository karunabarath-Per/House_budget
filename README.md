# Household Budget

A personal household budgeting web application for private family use.

* Two application users (User 1, User 2) with a simple "who is entering data?"
  selector — **no login**.
* Budget plans separate from actual expenses; expenses stored as one JSON
  file per calendar month.
* Google Drive as permanent storage, with manual, conflict-safe sync,
  temporary offline staging, and automatic + manual backups.
* INR first, other currencies possible later.

## Status

**Design phase.** The architecture proposal is in [`docs/`](docs/) and is
waiting for approval. No application code yet.

Start with:

1. [docs/architecture.md](docs/architecture.md) — folder structure, diagram, pages, components
2. [docs/open-questions.md](docs/open-questions.md) — decisions needed from you
3. [docs/data-model.md](docs/data-model.md) — JSON file formats
4. [docs/google-drive-setup.md](docs/google-drive-setup.md) — Drive layout and OAuth
5. [docs/sync.md](docs/sync.md) — sync and conflict handling
6. [docs/storage.md](docs/storage.md) — local staging
7. [docs/backups.md](docs/backups.md) — backup, restore, import, export
8. [docs/calculations.md](docs/calculations.md) — money and frequency maths
9. [docs/testing.md](docs/testing.md) — testing strategy
10. [docs/github-workflow.md](docs/github-workflow.md) — branches, CI, secrets

## Data safety

Real financial data, backups, exports, `.env` files and Google tokens must
**never** be committed to this repository.
