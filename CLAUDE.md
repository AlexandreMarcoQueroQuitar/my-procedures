# Claude Code Instructions

Follow the repository conventions in `README.md`.

This repo is organized as a set of operational procedures. Each
procedure has a living document and a `history/YYYY-MM-DD/` execution
record. Temporary inputs, generated files, local backups, and secrets
belong in `.workspace/` and `.secrets/`; those folders must not be
committed and should be cleaned after the execution.

When executing a procedure:

- Read the target procedure first.
- Confirm the environment and persistent targets before writes.
- Use rollback/dry-run validation before committing database or external
  changes.
- Keep deletion filters narrow and explicit.
- Save a concise execution summary and copy of the procedure in
  `history/YYYY-MM-DD/`.
- Leave reusable learnings in the procedure document, not only in chat.

For GMUD/NocoDB work, start with
`gmud/CODEX_NOCODB_GMUD_JIRA.md`.
