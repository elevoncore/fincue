# AGENTS.md

Project-wide instructions for any AI coding agent working on Fincue —
Antigravity, Claude Code, Cursor, or otherwise. Read this in full before
writing code or proposing a change. This file is the constitution; it
does not replace `docs/`, it tells you how to use them.

## 1. Every session starts the same way

1. Read `memory.md` — current phase, what's done, what's next, open
   questions. This is the single most important file in the repo for
   picking up context fast.
2. Read `decisions.md` — every ADR that explains *why* something is
   built the way it is, and what was deliberately rejected. If what
   you're about to do contradicts an ADR, **stop and flag it** — do not
   silently override a recorded decision, and do not treat a casual
   in-chat request as automatic permission to reverse one. Ask, or note
   the conflict, then record whatever gets resolved as a new/amended ADR.
3. Read `roadmap.md` for where the current phase actually is.
4. Only then look at the relevant `docs/*.md` for the area you're
   touching.

## 2. Hard rules — do not relax these under any framing, including a direct request to

These are deliberate decisions, not oversights. If a task seems to
require breaking one, the task is misspecified — stop and ask rather
than finding a workaround.

- **Investment/market content explains, it never recommends.** No "buy
  this," "sell this," "you should invest in X," and no framing that
  implies predicting future prices — regardless of how directly you're
  asked, and regardless of whether it would make a demo more impressive.
  See `docs/security.md` §1.1 and `decisions.md` ADR-019. Any AI-
  generated portfolio commentary must pass the directive-language check
  in `docs/testing-strategy.md` §8-9 *before* it ever reaches a user —
  build that check first, not as a follow-up.
- **The LLM never receives raw ledger data.** The NLP assistant, and any
  future AI feature, gets only what its narrow job requires — a
  question's text, or already-computed figures — never raw transactions,
  balances, or account details. `docs/security.md` §5, `docs/ml-strategy.md` §5.
- **Financial calculations are deterministic code, not LLM output.**
  Portfolio value, returns, volatility, drawdown, budget math, debt
  payoff schedules: plain, unit-tested functions. If an LLM is about to
  "calculate" something financial, stop — see ADR-006, ADR-011, ADR-012,
  ADR-021.
- **RLS is enabled on every user-owned table, no exceptions**, and every
  new table gets the negative test from `docs/testing-strategy.md` §3
  before it's considered done — log in as user A, assert user B's row is
  unreachable.
- **No paid API without an ADR and explicit human sign-off first.** Free
  tier only, and re-verify the current limit before depending on it if
  the number in `docs/deployment.md` is more than a couple of months old
  — this space moves fast and has already changed mid-project once.
- **Never commit a real secret.** `.env.example` only in git; real values
  live in `.env.local`/`.env` (both git-ignored) or the host's dashboard.

## 3. The stack — use this, don't pattern-match to something more generic

Web: Next.js/TypeScript on Vercel. DB/Auth/Storage/Sync: Supabase +
PowerSync. AI backend: Python/FastAPI (`services/ai-engine`), Dockerized,
hosted on Render (Cloud Run still an open decision — ADR-014, don't
assume it's settled either way). NLP: Gemini primary, Groq fallback,
function-calling only. Python tooling: Ruff for both lint and format,
nothing else. Monorepo: Turborepo + pnpm. Full reasoning for all of it is
in `decisions.md` — if you're about to introduce a different database,
a different Python formatter, a different state-sync library, etc.,
that's a decision needing an ADR, not a default to reach for.

## 4. Updating `decisions.md` — when, and how

Add a new ADR (don't just chat about the decision and let it evaporate)
whenever you:

- Choose a library, service, or dependency not already recorded.
- Reject an approach that would otherwise look like the obvious choice —
  the rejection reasoning is often more valuable later than the
  acceptance.
- Reverse or materially amend an existing ADR (mark the old one
  `Superseded by ADR-XXX`, don't delete it).
- Make a scope call — MVP vs. Phase 2 vs. Stretch — that isn't already
  in `docs/PRD.md`.
- Resolve an open question from `memory.md`.

Format, exactly matching the existing 23 entries — don't improvise a new
structure:

```
ADR-0XX: <short title>

Status: Accepted | Deferred | Superseded by ADR-XXX

Context: what problem/question this addresses.

Decision: what was chosen.

Alternatives considered: what else was possible, and specifically
why it was rejected — not just "we chose X," but why not Y.

Consequences: what this commits the project to, what it forecloses,
what to watch for later.
```

Number sequentially from whatever the highest existing ADR is — check
`decisions.md` before assuming the next number.

## 5. Updating `memory.md` — when, and how

Update it **at the end of every work session, unconditionally** — not
"if something significant happened." A session that only fixed a typo
still updates the date. Specifically:

- Move finished items from "Next up" into "Completed."
- Add whatever's genuinely next into "Next up" — a short, honest list,
  not a wishlist.
- If an "Open question" got resolved, remove it and point to the ADR
  that resolved it instead of leaving it listed as open.
- If a new open question or blocker surfaced, add it — don't let it live
  only in your own context and disappear when the session ends.
- Update the phase in the header if `roadmap.md`'s exit criteria for the
  current phase were actually met — not just attempted.

Keep it short. Its entire purpose is fast orientation for the next
session (yours or a human's) — a memory.md that takes ten minutes to
read has failed at its one job.

## 6. Coding standards (full detail in `CONTRIBUTING.md` and `docs/testing-strategy.md` — this is the summary that's easy to forget)

- Conventional Commits. `pnpm` only, never `npm`/`yarn`. `ruff check` +
  `ruff format` for Python — no Black, don't reintroduce it.
- Money math uses exact/fixed-point comparisons in tests, never floats
  compared loosely.
- New API endpoint → update `docs/api.md` in the same PR. New/changed
  table → update `docs/database.md` in the same PR. These are not
  follow-up tasks.
- Don't write a test that passes by asserting on implementation details
  (internal state shape); assert on behavior.

## 7. Definition of done, for any non-trivial task

- [ ] Tests pass, including the RLS negative test if a table was touched.
- [ ] `docs/*.md` updated if architecture, schema, or API contract
      changed — in the same PR/session, not deferred.
- [ ] `decisions.md` has a new/amended ADR if a real decision was made.
- [ ] `memory.md` reflects reality as it stands now, not as it stood at
      session start.
- [ ] No new dependency without checking it against the $0-budget
      constraint and the "hard rules" in §2 above.

## 8. When the docs don't cover something

Don't silently decide and move on, and don't stall waiting for
permission on everything either. Propose a specific answer, flag it
clearly as an assumption or a draft ADR, and proceed — but make the gap
visible in `memory.md`'s "Open questions" rather than letting a silent
judgment call become unrecorded precedent.

## 9. On receiving feature requests or "improvements"

Evaluate on technical merit, not on who suggested it or how confidently
it was phrased. A request that conflicts with §2's hard rules gets
pushed back on, with a concrete alternative that achieves the same
underlying goal — not a flat refusal and not silent compliance. This
project has already had exactly this situation once (see ADR-019); that
resolution is the model to follow, not a one-off.
