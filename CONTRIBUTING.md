# Contributing to Fincue

Thanks for contributing — whether you're a teammate on this FYP or an outside
reviewer. This document keeps collaboration predictable across a small,
fast-moving student team.

## Ground rules

- Be respectful and constructive (see [`CODE_OF_CONDUCT.md`](./CODE_OF_CONDUCT.md)).
- Every change to `main` goes through a pull request — no direct pushes,
  even for "tiny" fixes, once the team is more than one person.
- Prefer small, reviewable PRs over large ones. A PR that touches one
  feature area is easier to review and easier to revert.

## Branching model

We use a lightweight trunk-based model:

```text
main                      # always deployable
├── feat/envelope-budgets
├── fix/receipt-ocr-crash
├── chore/upgrade-turborepo
└── docs/update-api-md
```

Branch prefixes: `feat/`, `fix/`, `chore/`, `docs/`, `refactor/`, `test/`.

## Commit messages

We follow [Conventional Commits](https://www.conventionalcommits.org/):

```text
<type>(<scope>): <short summary>

[optional body]

[optional footer(s)]
```

Types: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`, `perf`.
Scope is the app/package/service touched, e.g. `feat(web): add envelope
rollover toggle` or `fix(ai-engine): correct OCR total parsing regex`.

This matters beyond tidiness: it lets us auto-generate `CHANGELOG.md`
entries and makes `git log --oneline` genuinely useful when writing the
FYP final report.

## Before opening a PR

1. `pnpm lint` and `pnpm test` pass locally (once app code exists — see
   [`docs/testing-strategy.md`](./docs/testing-strategy.md)).
2. If you touched the database schema, update
   [`docs/database.md`](./docs/database.md) and add a migration file (see
   that doc's "Migrations" section).
3. If you touched an API contract, update [`docs/api.md`](./docs/api.md).
4. If you made an architecturally significant decision (new dependency, new
   hosting choice, rejected alternative), add an entry to `decisions.md`
   (git-ignored locally — copy the relevant ADR into your PR description so
   reviewers see the reasoning).
5. Fill in the PR template: what changed, why, how you tested it, and
   screenshots/GIFs for UI changes.

## Code style

- TypeScript/JS: Prettier (config lives in `packages/config` once
  populated) + ESLint. Run `pnpm format` before committing.
- Python (`services/ai-engine`): `ruff` for both linting (`ruff check`)
  and formatting (`ruff format`) — one tool, not two. Ruff's formatter is
  a deliberate Black-compatible drop-in, not a different style.
- Don't fight the formatter — if it disagrees with you, change the config,
  don't hand-format around it.

## Review expectations

- At least one approval required before merging (or a self-review checklist
  if you're a solo contributor at this stage — re-read your own diff after
  a break, not immediately after writing it).
- Reviewers should focus on: correctness, security (especially anything
  touching auth, money math, or PII), and whether the change matches the
  architecture in `docs/architecture.md`.
- It's fine to request changes on scope — "this PR is doing three things,
  can we split it?" is a legitimate review comment.

## Reporting bugs / requesting features

Use GitHub Issues. Bug reports should include: what you expected, what
happened, steps to reproduce, and environment (browser/OS, or "ai-engine
local" vs. "deployed"). Feature requests should reference the relevant
section of `docs/PRD.md` if one exists.

## Security issues

Do **not** open a public issue for a security vulnerability. See
[`SECURITY.md`](./SECURITY.md).
