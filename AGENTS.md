# Agent Instructions

This repository stores reusable operational procedures.

Before acting, read `README.md` and the specific procedure document in
the target folder. For the current GMUD procedure, read
`gmud/CODEX_NOCODB_GMUD_JIRA.md`.

Use this structure for each procedure:

```text
<procedure>/
  README.md or CODEX_<NAME>.md
  history/YYYY-MM-DD/registro.md
  history/YYYY-MM-DD/procedimento_usado.md
  .workspace/
  .secrets/
```

Rules:

- Treat `.workspace/` and `.secrets/` as temporary local folders.
- Never commit `.workspace/` or `.secrets/`.
- Prefer QQ secret tooling or official secret stores over local secret
  files.
- Validate destructive or persistent writes with dry-run, rollback, or a
  small scoped test whenever practical.
- Keep backups or exports for data-changing operations when feasible.
- Record each real execution under `history/YYYY-MM-DD/`.
- Do not remove unrelated user files or revert unrelated user changes.
- Keep procedure updates reusable; put one-off details in `history/`.
