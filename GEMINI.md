# Gemini Instructions

Use `README.md` as the source of truth for repository conventions.

This repository contains operational procedures, not application source
code. Work inside the relevant procedure folder and preserve the pattern:

```text
<procedure>/
  procedure document
  history/YYYY-MM-DD/registro.md
  history/YYYY-MM-DD/procedimento_usado.md
  .workspace/
  .secrets/
```

Operational rules:

- `.workspace/` and `.secrets/` are local-only temporary folders.
- Do not commit generated working files, local backups, CSV inputs, or
  secrets.
- Validate database or external writes before applying them.
- Record real executions in `history/` with enough context to audit and
  repeat the work.
- Update the living procedure when an execution teaches a reusable rule.

For the GMUD process, use
`gmud/CODEX_NOCODB_GMUD_JIRA.md`.
