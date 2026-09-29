# Engineering Principles

This document is about *how* code gets written in Fincue, not *what*
gets built — `docs/PRD.md` and `docs/architecture.md` own that. Read
this before writing any non-trivial module, and re-read it if you
notice yourself reaching for a pattern "because it's best practice"
rather than because this specific piece of code needs it.

**The rule that governs every other rule in this file:** a principle
exists to manage complexity that's actually present. Applying one where
the complexity isn't there yet is not discipline, it's over-engineering
— and over-engineering is a real, well-documented failure mode, not a
safer default than under-engineering. This project has already rejected
several "more sophisticated" options for exactly this reason (no
Airflow/Kafka for a single-user-scale pipeline, classical ML over deep
learning for categorization, no unnecessary microservices — see
`decisions.md` ADR-006, ADR-012, ADR-018). Everything below extends that
same judgment into code-level design, not a different standard.

## 1. SOLID, with Fincue-specific examples — not textbook ones

**Single Responsibility** — a module has one reason to change.
In `services/ai-engine`, this is *why* the router/service/schema split
exists (`services/ai-engine/app/README.md`): a router handles HTTP
concerns, a service holds the actual logic, a schema defines shape.
**Smell to watch for:** a route handler with business logic inline, or
a service function that also happens to format an HTTP response.

**Open/Closed** — open for extension, closed for modification. Already
real in two places: the badge system (`badges.criteria` as structured
JSON, `docs/database.md`) means a new badge is *data*, not new code; the
LLM provider abstraction (`docs/ml-strategy.md` §5, ADR-005) means
adding a third provider shouldn't require touching Gemini/Groq's
existing code. **Apply this forward:** the NLP assistant's bounded
intent set (`sum_by_category`, `compare_periods`, etc.) should be a
registry — intent type → handler function — so adding an intent is
adding an entry, not editing a growing if/elif chain.

**Liskov Substitution** — anything implementing an interface must be
safely swappable for anything else implementing it. Concretely: if
Gemini and Groq both implement "LLM provider," calling code must not
contain `if provider == "gemini": ...` special-casing outside the
provider implementations themselves — that's a Liskov violation
signaling the interface is leaking.

**Interface Segregation** — don't force a caller to depend on things it
doesn't use. Already true of `docs/api.md`'s endpoint shape: narrow,
purpose-specific endpoints (`/v1/categorize`, `/v1/ocr/receipt`), not one
`/v1/process` endpoint branching on a type field.

**Dependency Inversion** — depend on abstractions, not concretions.
`services/ai-engine/app/clients/` (gemini_client.py, groq_client.py,
exchange_rate_client.py) should each implement a shared, narrow
interface that the orchestration code (`assistant_orchestrator.py`,
etc.) depends on — never instantiate `GeminiClient` directly inside
business logic.

## 2. DRY — and the trap inside it

Real DRY in this project: `packages/shared-types` exists specifically so
web and mobile don't maintain two copies of the same entity types. The
color-band thresholds (green/amber/red — `docs/gamification-strategy.md`)
should be one shared function, used by the dashboard, envelope cards,
and Health Score display alike, not reimplemented per component. The
directive-language deny-list (`docs/testing-strategy.md` §9) must be
**one function**, called by both the actual API endpoint and the
automated test that checks it — two copies that could silently drift
apart defeats the entire point of that check.

**The trap:** DRY is about not duplicating *knowledge*, not about
merging code that merely looks similar. Two validation functions that
happen to look alike today but represent genuinely different business
rules (say, a transaction-amount sanity check vs. a market-price sanity
check) should stay separate — merging them creates a false shared
abstraction that gets awkward the moment one rule needs to change and
the other doesn't. If you're not sure, duplication you can later merge
is cheaper to fix than a wrong abstraction you have to untangle.

## 3. KISS / YAGNI — already this project's house style

This isn't new guidance, it's naming what's already been decided
repeatedly: the simplest technique that meets the accuracy bar
(`docs/ml-strategy.md`'s stated guiding principle), rule-based checks
before ML where a rule is genuinely sufficient (duplicate-charge
detection, subscription price-creep — both deliberately *not* ML,
`docs/ml-strategy.md` §3), and scheduled jobs via `pg_cron`/GitHub
Actions instead of a dedicated orchestration platform. **Apply this
forward:** before adding an abstraction (a new base class, a new config
layer, a new "manager" object), ask whether there are actually two-plus
concrete cases needing it right now — one case doesn't justify a
generalized abstraction, it justifies a specific implementation.

## 4. Design patterns actually used here — and the deliberate boundary around them

Five patterns are genuinely load-bearing in this codebase. This list is
meant to be close to exhaustive for this project's size — reaching for a
pattern not on this list should prompt a "does this code actually need
it" pause, not automatic adoption.

| Pattern | Where | Why it earns its place |
|---|---|---|
| **Strategy** | LLM provider (Gemini/Groq), currency API (primary/fallback), OCR engine consideration (ADR-015) | Multiple interchangeable implementations of the same job, genuinely swapped at runtime or by config — not hypothetically. |
| **Adapter** | Every external API wrapper in `services/ai-engine/app/clients/` (Plaid, Alpha Vantage, Groq, Gemini) | Isolates Fincue's internal shapes from each vendor's actual API shape, so a vendor's breaking change touches one file. |
| **Chain of Responsibility (fallback chains)** | Categorization: ML model → rule-based fallback. Currency: primary API → fallback API. LLM: Gemini → Groq | Each of these is "try this, fall back to that" — implement it as an explicit ordered chain, not three separately-written if/else fallback blocks with slightly different shapes. |
| **Registry (a lightweight Factory)** | NLP assistant intent handlers, badge criteria evaluators | A lookup from a type/key to a handler, so new entries are additive — see the Open/Closed example above, same underlying pattern. |
| **Repository (lightweight)** | `services/ai-engine`'s database access | Not a full ORM abstraction — just enough that every query touching a user-owned table goes through one place, making the "ai-engine must always manually scope by `user_id`" rule (`docs/security.md` §4) structurally checkable in one location instead of hoped-for everywhere it queries. |

**Deliberately not used, and don't introduce without a real reason
recorded as an ADR:** Observer/pub-sub (Supabase Realtime already covers
the "notify on change" need — see `docs/architecture.md` §4.2), a full
abstract-factory hierarchy (nothing here has more than 2-3
implementations of anything), deep inheritance of any kind (prefer
composition — small services/hooks calling each other over class
hierarchies, which is also just idiomatic React and idiomatic modern
Python).

## 5. A few more principles worth naming explicitly

- **Idempotency.** The keep-alive heartbeat write (`INSERT ... ON
  CONFLICT DO UPDATE`, ADR-017) is the clean current example — safe to
  run twice, same result. PowerSync now owns this for ledger sync
  specifically (`docs/architecture.md` §5), but the *principle* still
  applies to every new endpoint you write that could plausibly be
  retried (network hiccup, double-tap on a slow connection) — design for
  "called twice = same result" by default, not as an afterthought.
- **Fail-safe defaults.** RLS denies by default. `requires_review`
  defaults to flagging low-confidence OCR rather than silently trusting
  it. The directive-language check *rejects* on failure rather than
  passing through on error. When a new feature has an uncertain case,
  default to withholding/flagging, not to trusting.
- **Principle of least privilege.** RLS for user-facing access; the
  ai-engine's service-role connection is the one deliberate exception,
  and it's exactly why that exception gets the extra scrutiny in
  `docs/security.md` §4. Extend this to credentials generally — a new
  service gets only the API keys/scopes it actually needs, not a shared
  "does everything" key.
- **Testability as a design signal.** If you can't unit-test "categorize
  this merchant string" without spinning up a live database and a real
  network call to Gemini, that's not a testing problem, it's a coupling
  problem — the function is doing more than one job. This is usually the
  fastest way to *notice* a Single Responsibility violation in practice,
  more useful day-to-day than reciting the definition.
- **Explicit over implicit.** `docs/api.md`'s consistent error envelope
  (`{"error": {"code", "message", "request_id"}}`) is this principle
  applied once, everywhere — a new endpoint that invents its own error
  shape is a regression, not a stylistic choice.

## 6. A principle-driven review checklist

Run this on any non-trivial new module, yours or a human's:

- [ ] Could I unit-test this without a live database or network call? If
      not, what's coupled that shouldn't be?
- [ ] If I swapped one implementation for another here (a provider, a
      data source), would calling code need to change? If yes, the
      abstraction is leaking.
- [ ] Is there a second, slightly-different copy of this logic
      elsewhere? If yes, is it the *same rule* (merge it) or a
      *coincidentally similar* rule (leave it separate)?
- [ ] Does this introduce a new pattern, base class, or config layer for
      a single current use case? If yes, that's premature — implement
      the concrete case, generalize when a second real case shows up.
- [ ] Does this new endpoint/job behave correctly if called twice?

## 7. Where this applies less strictly, on purpose

Fincue is a demo-scale FYP, not a system serving millions of users —
`docs/architecture.md` §9 already says this plainly about
infrastructure, and the same honesty applies to code structure. The bar
is **correct, testable, and clear** — not maximally abstracted, not
"enterprise" for its own sake. A 40-line function that does one obvious
thing plainly does not need to become a strategy-pattern-backed
pluggable pipeline. If a principle in this document and the instinct to
just ship the simple version are in tension for a piece of throwaway or
genuinely one-off code, say so in the PR description and move on —
don't let this document become the reason a two-week feature takes six.
