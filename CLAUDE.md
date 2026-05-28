@AGENTS.md

## Claude Code (umbrella)

- This is an umbrella directory above two independent repos
  (`Kunfupay-Payins-Back/`, `Kunfupay-Payins-Front/`). The umbrella has **no**
  monorepo plumbing — each child repo carries its own `AGENTS.md`, `CLAUDE.md`,
  lockfile, Docker, and CI configuration.
- For any task scoped to one repo, read that repo's `AGENTS.md` first; this file
  only covers cross-repo rules (universal contract, banned terms, PCI, money,
  tenancy, IDs).
- Keep Claude-specific guidance here only when it cannot live in `AGENTS.md`.
