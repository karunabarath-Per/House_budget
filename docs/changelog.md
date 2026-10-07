# Changelog

## Unreleased

* docs: architecture proposal (folder structure, data model, Google Drive
  strategy, OAuth flow, sync & conflicts, backups, calculations, testing,
  GitHub workflow, open questions). No application code yet.
* docs: the app connects to one Google account only (folder owner); the
  folder is shared with the second Google account manually in Drive.
  Removes the cross-account `drive.file` workaround (Picker re-granting,
  pre-created month files).

## Schema migrations

None yet. All data files are `schemaVersion: 1`.
