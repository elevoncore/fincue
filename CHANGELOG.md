# Changelog

All notable changes to this project are documented here. Format loosely
follows the "Keep a Changelog" convention (versions in reverse-chronological
order; changes grouped as Added / Changed / Fixed / Removed). Versioning
follows [Semantic Versioning](https://semver.org/) once a `v1.0.0` demo
milestone exists — until then, dates are more meaningful than version
numbers.

## [Unreleased]

### Added

- Initial repository skeleton: monorepo structure (`apps/`, `services/`,
  `packages/`, `docs/`), root tooling config (Turborepo, pnpm workspaces,
  EditorConfig), and the full documentation set (PRD, architecture,
  database, API, security, deployment, ML strategy, developer guide,
  troubleshooting, user guide, gamification strategy, testing strategy,
  glossary).
- Internal AI-context tracking files (`memory.md`, `decisions.md`,
  `roadmap.md`) — git-ignored, used to keep AI pair-programming sessions
  and teammates oriented.

## [0.1.0] — Unreleased (target: end of Phase 0)

Placeholder for the first tagged milestone — "repository skeleton and
tooling complete, no feature code yet." Update this section (and tag the
commit `v0.1.0`) once Phase 0 in `roadmap.md` is marked done.

---

### How to use this file going forward

- Add a bullet under `[Unreleased]` in the same PR that makes the change —
  not as an afterthought before a demo.
- When you cut a milestone (e.g. before a supervisor check-in or the FYP
  midterm demo), rename `[Unreleased]` items into a new dated/versioned
  section and start a fresh `[Unreleased]` section above it.
- Favor user-visible or architecturally significant changes here. Routine
  refactors and dependency bumps can stay in commit history alone.
