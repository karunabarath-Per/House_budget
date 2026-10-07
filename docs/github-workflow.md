# GitHub Workflow

> Status: **PROPOSAL — awaiting approval.**

## Branches

* `main` — always working, tests passing.
* Feature branches, one per area:
  `feature/budget`, `feature/transactions`, `feature/google-drive`,
  `feature/sync`, `feature/backups`, `feature/reports`.
* Flow: branch from `main` → small commits → pull request → CI green →
  merge → delete branch.
* (Work done by Claude Code in this session goes to the session branch
  `claude/laughing-heisenberg-tmgn2b`; it can be renamed/merged as you like.)

## Commit messages

```
feat: add household member management
fix: prevent stale cloud overwrite
test: add transaction merge tests
docs: document google drive setup
chore: update dependencies
```

## CI (`.github/workflows/ci.yml`)

On push and pull request: checkout → Node 22 → `npm ci` → `npm run lint` →
`npm test`. Also a secret scan step (gitleaks, free) as a second line of
defence.

## Security

Never commit: `.env`, Google client secret, OAuth/refresh tokens, real
financial data, private JSON data, exports, receipts, backups, personal
information.

`.gitignore` will include at least:

```
.env
.env.*
!.env.example
node_modules/
*.token.json
google-token.json
client_secret*.json
data/
backups/
exports/
local-backups/
staging/
*.backup.json
receipts/
```

Repository settings to turn on (where available for your plan):
Secret scanning and Push protection (Settings → Code security), and
branch protection on `main` (require CI to pass).

`.env.example` contains placeholders only:

```
PORT=5173
HB_DATA_DIR=
GOOGLE_CLIENT_ID=your-client-id.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=replace-me
GOOGLE_API_KEY=replace-me
GOOGLE_PROJECT_NUMBER=replace-me
```
