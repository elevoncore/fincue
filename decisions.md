# Architecture Decision Records (ADRs)

Format: **Status** / **Context** / **Decision** / **Alternatives considered
(and why rejected)** / **Consequences**. Add a new ADR for every
architecturally significant choice — don't edit old ones after the fact
except to mark them `Superseded by ADR-XXX`; the history of _why_ we
changed our minds is as valuable as the current state.

---

## ADR-000: Build this as a monorepo skeleton + docs first, no app code

**Status:** Accepted

**Context:** The team wanted an exhaustively researched, defensible
architecture before writing feature code, given the scope of the original
brief and a strict $0 budget where wrong infrastructure choices are
expensive to unwind later (migrating a database or LLM provider mid-build
costs real time a small FYP team doesn't have to spare).

**Decision:** Produce the full repository structure, tooling config, and
documentation set (this file included) as a standalone deliverable before
any `apps/`, `services/`, or `packages/` code is written.

**Alternatives considered:**

- _Start coding immediately, document as we go._ Rejected: for a project
  this feature-rich, under-planning the ML/hosting stack risks discovering
  a free-tier dead end (e.g., a service that turns out not to be free at
  the scale needed) after real code already depends on it.

**Consequences:** Phase 1 (see `roadmap.md`) starts with zero technical
debt but also zero running code — this is a deliberate trade, not an
oversight. It also means every doc in `docs/` should be treated as a
living plan, not a historical record — update them as implementation
teaches the team something the plan got wrong.

**A note on licensing:** this ADR number is also where we flag that the
`LICENSE` file's MIT choice is a **default, not a verified-safe decision**
— check your university's FYP/IP policy before publishing the repository
publicly.

---

## ADR-001: Turborepo + pnpm workspaces for the monorepo

**Status:** Accepted

**Context:** The project needs `apps/web`, a future `apps/mobile`, the
`services/ai-engine`, and shared `packages/*` to live together with
sensible build caching and dependency management.

**Decision:** Turborepo (task orchestration/caching) + pnpm workspaces
(package management), using Turborepo's current `tasks` key in
`turbo.json` (the `pipeline` key was renamed in Turborepo 2.0 — don't
copy older tutorials that still show `pipeline`).

**Alternatives considered:**

- _Nx_ — more powerful (generators, more plugins) but a steeper learning
  curve for a small student team, and Turborepo's simpler model pairs more
  naturally with the Vercel-hosted Next.js frontend (same parent company,
  first-class integration). Rejected for this project's scale, not
  rejected as "worse" in general.
- _No monorepo tooling, just separate repos._ Rejected: the whole
  "decoupled backend supports a future mobile app" architecture (see
  `docs/architecture.md`) depends on `packages/shared-types` being trivial
  to share — separate repos would need a published npm package and
  version-bumping ceremony that's disproportionate for a small team.
- _Yarn/npm workspaces instead of pnpm._ Rejected: pnpm's content-addressable
  storage is faster and stricter about phantom dependencies, both of which
  are just better defaults for a project multiple people (or multiple AI
  sessions) will touch over time.

**Consequences:** Everyone (and every AI session) must use `pnpm`, not
`npm`/`yarn`, or the lockfile and workspace resolution breaks. Enforced via
`packageManager` in root `package.json`.

---

## ADR-002: Next.js on Vercel for the web frontend

**Status:** Accepted

**Context:** Need a web frontend framework and a $0 host for it.

**Decision:** Next.js (React), deployed to Vercel's Hobby (free) tier.

**Alternatives considered:**

- _Cloudflare Pages as primary host_ — genuinely unlimited free bandwidth
  (vs. Vercel's 100 GB/month), but Next.js's full feature set (ISR,
  certain middleware behaviors) is natively supported on Vercel and needs
  an adapter (`@cloudflare/next-on-pages` or OpenNext) on Cloudflare,
  adding friction. **Kept as the documented fallback** if Vercel's Hobby
  bandwidth or its non-commercial-use terms become a problem — see
  `docs/deployment.md`.
- _SvelteKit / Remix / plain Vite+React_ — all viable and all free to
  host; Next.js was chosen for ecosystem maturity, first-party Vercel
  integration, and because most free ML/AI integration tutorials and the
  team's existing familiarity (assumption — confirm with your actual
  team) skew toward the Next.js/React ecosystem, reducing implementation
  risk.

**Consequences:** Vercel Hobby's cron jobs are capped at 2/day — scheduled
jobs (subscription price-creep checks, health-score computation) must run
elsewhere (Supabase `pg_cron` or a scheduled GitHub Actions workflow — see
`docs/deployment.md` §4). Vercel Hobby's terms restrict commercial use;
fine for an FYP demo, revisit if this becomes a real product.

---

## ADR-003: Supabase as the primary backend-as-a-service

**Status:** Accepted

**Context:** Need a database, authentication, and file storage, on $0,
with minimal custom backend code for standard CRUD.

**Decision:** Supabase (managed Postgres + Auth + Storage + Realtime +
Row-Level Security), free tier.

**Alternatives considered:**

- _Neon (serverless Postgres) + a separate auth solution (Auth.js/Clerk)._
  Neon's free tier has a real advantage — it scales to zero and wakes
  automatically, with **no weekly manual "unpause" step**, unlike
  Supabase's free projects, which pause after 7 days of API inactivity and
  must be restored from the dashboard. This was a close call. **Rejected
  for the primary choice** because Supabase's bundled Auth + Storage + RLS
  removes an entire layer of custom backend code (a real productivity win
  for a small team with a hard deadline), and the 7-day-pause problem has
  a cheap mitigation (a scheduled keep-alive ping — see
  `docs/deployment.md`). **Documented as the fallback** if the pause
  behavior becomes a real problem during development.
- _Firebase (Firestore)_ — genuinely free-tier-friendly and avoids the
  pause issue entirely, but Firestore's NoSQL document model is a poor fit
  for this app's relational data (transactions ↔ accounts ↔ categories ↔
  budgets, with real joins for reporting) and it lacks Postgres's
  `pgvector`/RLS/SQL ecosystem this project leans on. Rejected primarily
  on data-model fit, not price.
- _PlanetScale (MySQL)_ — PlanetScale discontinued its free "Hobby" tier
  in 2024; even if a free tier has since returned in some form, Postgres's
  richer extension ecosystem (RLS, `pgcrypto`, `pg_trgm`, `pg_cron`, all
  used elsewhere in this stack) makes Postgres the better fit regardless.
  Rejected.

**Consequences:** Set up the keep-alive mitigation (`docs/deployment.md`
§2) early, not as an afterthought right before a demo. The `ai-engine`
connects via the Postgres service role (bypassing RLS) for its
cross-user scheduled jobs — this makes manual `user_id` scoping in
`ai-engine` code a critical, must-review security boundary (see
`docs/security.md` §4).

---

## ADR-004: A decoupled Python/FastAPI `ai-engine`, hosted on Render

**Status:** Accepted — the decoupled-service architecture itself is
settled; the specific hosting provider is refined by **ADR-014**, which
deliberately reopened and deferred the Render-vs-Cloud-Run choice. Read
this ADR for _why a separate service exists at all_, and ADR-014 for
_where it currently runs_.

**Context:** The AI/ML features (OCR, categorization, anomaly detection,
forecasting, NLP assistant orchestration) are Python-ecosystem-first, and
need to be usable by both the web app now and a mobile app later.

**Decision:** A standalone FastAPI service in `services/ai-engine`,
hosted on Render's free Web Service tier as the default; Hugging Face
Spaces and Google Cloud Run's "Always Free" tier documented as
alternatives.

**Alternatives considered:**

- _Do everything in Next.js API routes / Vercel serverless functions._
  Rejected: fighting the platform for Python-ML workloads (Vercel's
  Python function support and execution-time limits are a poor fit for
  OCR/embedding inference), and it would re-couple AI logic to the web
  app's deploy lifecycle, defeating the "shared by web and future mobile"
  goal.
- _Self-host an LLM/ML stack on a GPU instance._ Rejected outright on
  budget grounds — no free tier researched provides a persistent free GPU
  suitable for production inference (Hugging Face's free "ZeroGPU" is
  queued, demo-oriented capacity, not a backend a real app can depend on
  for request-time inference).
- _Hugging Face Spaces as the primary host, not just an alternative._
  Genuinely appealing (2 vCPU/16 GB RAM free), but research turned up
  **conflicting recent signals** about whether HF's free persistent
  Docker/Gradio compute is being tightened — see the dated caveat in
  `docs/deployment.md`. Render's terms were unambiguous at time of
  writing, so it's the safer default; HF Spaces remains worth
  re-evaluating at implementation time.
- _Google Cloud Run "Always Free."_ A genuinely durable free allowance
  (unlike a time-boxed trial credit), but requires a billing account with
  a card on file, which Render does not. Kept as a fallback for teams
  comfortable with that trade-off.

**Consequences:** Render's free tier spins down after ~15 minutes of
inactivity (cold start ~30–60s) — acceptable for a demo, documented in
`docs/troubleshooting.md` so it isn't mistaken for a bug. **Do not use
Render's free Postgres** for this project — it expires 30 days after
creation; the database lives in Supabase (ADR-003), Render is compute-only
here.

---

## ADR-005: Gemini (primary) + Groq (fallback) for the NLP assistant LLM

**Status:** Accepted

**Context:** The natural-language assistant needs an LLM to parse
questions into structured intents (see ADR-011), on $0 budget.

**Decision:** Google's Gemini API (via AI Studio's free tier) as primary,
Groq's free tier as a fast, low-latency fallback/alternative.

**Alternatives considered:**

- _Self-hosted open-source LLM (Llama/Phi/Gemma via Ollama)._ Rejected for
  the same reason as ADR-004's self-hosted-GPU option: no free tier
  researched offers persistent GPU compute suitable for this, and running
  a usable LLM on free CPU-only compute is too slow for an interactive
  assistant.
- _OpenAI / Anthropic paid APIs._ Explicitly excluded by the $0 budget
  constraint — not evaluated further.
- _OpenRouter free models / Cloudflare Workers AI free tier._ Both
  genuinely viable free options surfaced in research (OpenRouter: ~50
  free requests/day across 25+ models; Cloudflare Workers AI: ~10,000
  "Neurons"/day). Not chosen as primary only because Gemini's published
  free daily quota is larger for this project's expected single-user
  query volume, and Groq's speed is a better complement for a live demo
  than a second aggregator. Worth revisiting if Gemini's terms change.

**Consequences:** The `ai-engine`'s LLM client code (see
`services/ai-engine/app/README.md`) must be provider-agnostic behind a
thin interface, so swapping primary/fallback (or adding a third provider)
is a config change, not a rewrite — this was a deliberate implementation
requirement, not just a nice-to-have.

---

## ADR-006: Classical ML (embeddings + classifier) for categorization, not a fine-tuned transformer

**Status:** Accepted

**Context:** Need to categorize merchant text automatically, on $0 budget
and CPU-only free hosting.

**Decision:** `sentence-transformers/all-MiniLM-L6-v2` embeddings +
Logistic Regression/XGBoost, with a rule-based fallback layer underneath.
Full reasoning and citations in `docs/ml-strategy.md` §1.

**Alternatives considered:**

- _Fine-tune a transformer (e.g., DistilBERT) end-to-end._ A real prior-art
  project (`fin-classifier` on Hugging Face) did exactly this. Not chosen
  as the default because it needs GPU time to train (Google Colab's free
  GPU tier is usable but time-boxed and not guaranteed) and is heavier to
  serve on free CPU-only hosting than an embeddings+classical-head
  pipeline, for a problem (short, templated merchant strings) that doesn't
  clearly need the extra model capacity. Documented as a credible Phase 2
  upgrade if evaluation shows the simpler approach underperforming.
- _Pure rule-based/keyword matching, no ML at all._ Rejected as the _only_
  layer (kept as the fallback layer) — it doesn't generalize to merchants
  outside the hand-written keyword list, and the brief specifically calls
  for "smart categorization," which a static keyword list doesn't
  credibly demonstrate for an FYP evaluation.

**Consequences:** Model accuracy must be evaluated with real
precision/recall/F1 and a confusion matrix (see `docs/testing-strategy.md`
§4) — an unevaluated model isn't a credible ML claim in the final report.

---

## ADR-007: PaddleOCR for receipt OCR

**Status:** Accepted

**Context:** Need to extract merchant/date/total from photographed
receipts, on $0 budget.

**Decision:** PaddleOCR (open source, Apache 2.0), server-side in
`ai-engine`, evaluated against the SROIE dataset.

**Alternatives considered:**

- _Tesseract.js, client-side._ Kept as a documented fallback/offline mode
  (privacy advantage: image never leaves device), but general benchmarking
  shows PaddleOCR handling varied receipt layouts more robustly, and
  there's no deployment reason to prefer a client-side engine when OCR
  already happens server-side in `ai-engine`.
- _Paid cloud OCR (Google Cloud Vision, Azure Computer Vision) free
  tiers._ Both have real free allowances, but require a billing-enabled
  cloud account and add an external paid-service dependency the $0-budget
  constraint is meant to avoid entirely — rejected in favor of a fully
  open-source, self-hosted (within `ai-engine`) approach with no usage cap
  to worry about.

**Consequences:** OCR accuracy is bounded by input photo quality — this is
disclosed as a known limitation (`docs/ml-strategy.md` §2,
`docs/troubleshooting.md`), not something to over-promise in the demo.

---

## ADR-008: `@fawazahmed0/currency-api` for exchange rates

**Status:** Accepted

**Context:** Need live exchange rates for the multi-currency ledger, $0
budget, ideally no API key/signup friction.

**Decision:** `@fawazahmed0/currency-api` (free, unlimited, no key, 200+
currencies, CDN-hosted) as primary; `open.er-api.com` as a fallback.

**Alternatives considered:**

- _exchangerate-api.com's standard free tier_ — requires signup and caps
  at ~1,500 requests/month, unnecessary friction and a real cap when a
  key-free, uncapped alternative exists.
- _Open Exchange Rates_ — requires an App ID (signup) even for free
  personal use; same reasoning as above.

**Consequences:** Both chosen providers update rates daily (not
real-time) — fine for a personal finance app (nobody needs
tick-by-tick FX rates for expense tracking), but document this
limitation if precision matters for a specific feature later.

---

## ADR-009: Investment/account sync via Plaid, scoped to Phase 2/Stretch

**Status:** Accepted

**Context:** The brief asks for "sync your investment portfolios." Real
bank/brokerage linking is a materially different (and riskier) problem
than everything else in this project.

**Decision:** If attempted at all, use Plaid's Sandbox (free, unlimited,
mock data) for the primary demo, and optionally Plaid's Trial plan
(free, up to 10 real Production Items, auto-approved for most US/CA
developers as of a policy change taking effect April 15, 2026) for a
real-data demo. Manual entry of holdings is the MVP default; sync is
Phase 2/Stretch. See `docs/PRD.md` §6 and `docs/ml-strategy.md` §6.

**Alternatives considered:**

- _Custom bank-scraping (screen-scraping login credentials)._ Rejected
  outright — a serious security and terms-of-service risk with no
  redeeming budget advantage over Plaid's genuinely free Sandbox tier.
- _SnapTrade (investment-focused aggregator)._ A reasonable alternative
  worth a closer look if Plaid's investment-product coverage or
  geographic restrictions (Trial plan is US/CA-only) don't fit your
  situation — not deeply evaluated here; flagged for whoever picks up
  Phase 2.

**Consequences:** Don't let this feature's complexity threaten the MVP
timeline — `docs/PRD.md` explicitly tags it Phase 2/Stretch so the team
isn't tempted to treat "real bank sync" as required for a passing demo.

---

## ADR-010: Supabase Auth + WebAuthn for authentication and biometric lock

**Status:** Accepted

**Context:** Need account authentication plus a "locked behind biometrics
even if the device is unlocked" experience.

**Decision:** Supabase Auth for identity (bundled with the database
choice, ADR-003); WebAuthn (platform authenticators) as an additional
local re-authentication gate on web; `expo-local-authentication` as the
mobile equivalent (Phase 10+).

**Alternatives considered:**

- _Clerk_ — a strong dedicated auth product with a generous free tier, but
  redundant with Supabase Auth (which is already free and bundled) and
  would add a second identity system to keep in sync with RLS's
  `auth.uid()` model. Rejected as unnecessary complexity given Supabase
  was already chosen.
- _Custom-rolled auth (bcrypt + JWT from scratch)._ Rejected — reinventing
  session/token handling for a finance app is pure downside risk with no
  budget benefit over a free, audited, managed provider.

**Consequences:** Biometric lock is a **local re-authentication gate**,
not a replacement for the underlying Supabase session — implementers must
not treat "WebAuthn succeeded" as equivalent to "user is logged in" (see
`docs/security.md` §3).

---

## ADR-011: Function-calling / bounded-intent design for the NLP assistant, not text-to-SQL or open chat

**Status:** Accepted

**Context:** The assistant needs to answer natural-language questions
about a user's own financial data, safely.

**Decision:** The LLM parses a question into one of a fixed set of
structured intents (see `docs/ml-strategy.md` §5); the `ai-engine`
executes a corresponding fixed, parameterized query; the LLM never
receives ledger data.

**Alternatives considered:**

- _LLM-generated SQL executed directly (text-to-SQL)._ Rejected — an
  injection and data-exfiltration risk disproportionate to the value of
  supporting fully open-ended questions, especially for a finance app.
- _Open-ended chat with the full transaction history in context._
  Rejected on both privacy grounds (raw ledger data would be sent to a
  third-party API whose free-tier data-use terms may permit training on
  it — see `docs/security.md` §5) and reliability grounds (LLMs
  hallucinating numbers is a real, documented failure mode, and a finance
  app confidently stating a wrong dollar figure is a much worse failure
  than one correctly saying "I don't understand that question").

**Consequences:** The assistant only supports a bounded set of question
types at MVP (see `docs/PRD.md`, "narrow scope") — an honest, disclosed
trade-off rather than a silent limitation discovered by users.

---

## ADR-012: Per-user classical statistics for anomaly detection and forecasting, not a shared pretrained model

**Status:** Accepted

**Context:** Need duplicate/price-creep detection and cash-flow
forecasting.

**Decision:** Rule-based checks (duplicates, price creep) plus
per-user `IsolationForest`/statistical baselines (unusual transactions,
forecasting) — see `docs/ml-strategy.md` §3–4. No shared/pretrained model,
no external dataset.

**Alternatives considered:**

- _A population-level pretrained anomaly/forecasting model._ Rejected on
  conceptual grounds, not just budget: "anomalous" and "predictable
  spending pattern" are inherently relative to an individual's own
  baseline, not the population average — a shared model would answer the
  wrong question even if it were free and available.

**Consequences:** These features have a cold-start period — a brand-new
account has no history to detect anomalies against or forecast from.
Document this honestly in the demo/report rather than trying to paper
over it with synthetic history that overstates real-world performance.

---

## ADR-013: Adopt PowerSync for offline sync, replacing the hand-rolled Dexie.js/Expo SQLite approach

**Status:** Accepted

**Context:** `docs/architecture.md` originally described a DIY sync layer:
IndexedDB via Dexie.js on web, Expo SQLite on mobile, with a hand-written
idempotent-upsert protocol keyed on a client-generated UUID. This works,
but means writing and debugging the same conflict-handling logic twice —
once now for web, again at Phase 10 for mobile — and getting it subtly
wrong in only one of the two places is a realistic risk for a small team.

**Decision:** Use PowerSync's client SDKs (web and, later, React Native)
against Supabase Postgres via logical replication, with sync rules
scoping data per user. See `docs/architecture.md` §5 for the resulting
design and `docs/database.md` §6 for the Postgres-side requirements.

**Alternatives considered:**

- _Keep the original hand-rolled approach._ Simpler to reason about with
  zero new services, but means writing the sync/conflict logic twice
  across platforms and re-verifying it works identically both times —
  rejected because retrofitting sync architecture after the ledger is
  built is more expensive than adopting the right tool from Phase 2.
- _PowerSync's self-hosted "Open Edition"_ (source-available, Docker-
  deployable, avoids PowerSync's own 7-day inactivity pause entirely).
  Genuinely appealing, but adds a third piece of infrastructure the team
  operates itself, on top of the web app, the ai-engine, and Supabase.
  Not chosen given the team's limited operational bandwidth — worth
  revisiting if the hosted free tier's pause behavior becomes a real
  problem in practice.

**Consequences:** PowerSync's hosted free tier has the _same_ 7-day
inactivity pause as Supabase's — adopting it does not reduce
operational complexity, it adds a second service needing the same
keep-alive treatment. Mitigated by extending the single keep-alive
workflow (ADR-017) to cover both rather than building two. Also
introduces a new concept the team needs to learn (sync rules) and a
one-time Supabase-side setup step (enabling logical replication) that
should happen deliberately, not be discovered mid-Phase-2.

---

## ADR-014: Defer the ai-engine hosting decision between Render and Google Cloud Run

**Status:** Deferred — Render remains the working default; this is not a
closed decision, and shouldn't be read as one by a future session.

**Context:** Cloud Run was proposed specifically to avoid Render's
~15-minute inactivity spin-down and ~30–60s cold starts. Research
surfaced a correction worth preserving so it isn't re-litigated from
scratch later: **Cloud Run's free tier does not eliminate cold starts.**
It scales to zero by default exactly like Render; the difference is cold
start _duration_, typically shorter for a lean container (roughly
500ms–2s for a "typical" Node/Python app) — but the ai-engine loads
PaddleOCR and `sentence-transformers` at startup, which will likely erode
a meaningful part of that advantage relative to a "typical" app. Actually
eliminating cold starts requires `min-instances ≥ 1`, which bills for
idle time continuously — a real ongoing cost, not a free way to stay
warm.

**Decision:** Keep Render as the default for now (unblocks Phase 1
immediately, no billing account required), but do not close out Cloud
Run as an option. Containerizing the ai-engine (ADR-018) is deliberately
done either way, so this decision can be revisited later without any
rework — only a redeploy target changes.

**Alternatives considered:**

- _Commit to Cloud Run now_ on the stated cold-start rationale. Not
  chosen because the rationale as stated ("avoid cold starts") isn't
  fully accurate — see Context above — and Cloud Run's billing-account
  requirement (a card on file, even if unused) is a real friction Render
  doesn't have, worth deciding deliberately rather than by default.
- _Commit to Render permanently._ Not chosen either — Cloud Run's
  request-based billing and shorter typical cold starts are genuine
  advantages worth a real evaluation once there's an actual container to
  benchmark, not a decision made on priors alone.

**Consequences:** `docs/deployment.md` documents both options with equal
detail rather than presenting Render as final. Revisit once the
ai-engine's actual cold-start time (with its real dependencies) can be
measured on both platforms — decide on data, not on the general
reputation of either platform.

---

## ADR-015: Keep PaddleOCR over Gemini Flash multimodal for receipt OCR, for now

**Status:** Accepted (for now — see Consequences)

**Context:** Gemini's multimodal API can take a receipt photo directly
and return structured fields in one call, potentially removing
PaddleOCR's hosting footprint (a real concern on a RAM-constrained free
host) and, plausibly, improving accuracy on messy receipts.

**Decision:** Keep PaddleOCR as the primary OCR engine. Full reasoning
is in `docs/ml-strategy.md` §2.

**Alternatives considered:**

- _Switch to Gemini Flash multimodal._ Rejected for now, for two
  reasons, in order of importance:
  1. **Privacy conflict:** confirmed via multiple independent sources
     that Gemini's free tier permits Google to use submitted content —
     images included — to improve its products. A receipt photo can
     carry more identifying/sensitive detail (addresses, specific
     health or lifestyle-revealing purchases) than the question-text-only
     pattern the NLP assistant uses (ADR-011), and sending it to a
     free-tier multimodal endpoint directly conflicts with the "the LLM
     never receives raw ledger data" principle in `docs/security.md` §5.
  2. **Unverified accuracy claim:** no rigorous benchmark was found
     supporting "much more accurate" as a settled fact for this specific
     comparison. Plausible, given how strong multimodal models generally
     are at document understanding — but an assertion, not yet evidence.
     A secondary factor: current Gemini free-tier rate limits are reported
     inconsistently across sources and appear to have tightened; routing
     OCR traffic through the same quota as the NLP assistant risks
     contention if usage grows even modestly.

**Consequences:** PaddleOCR's RAM footprint on a free-tier host remains a
real constraint to test for early (Phase 4), not assume away. If revisited,
Gemini multimodal should be evaluated against the same SROIE benchmark
already planned for PaddleOCR (`docs/ml-strategy.md` §2) before any
switch, and the privacy trade-off disclosed to users regardless of which
engine wins on accuracy.

---

## ADR-016: Ruff exclusively for Python linting and formatting, dropping Black

**Status:** Accepted

**Context:** The original plan used Black for formatting and Ruff for
linting — two tools, two config surfaces, and Black's pure-Python
implementation is dramatically slower than Ruff's Rust-based one.

**Decision:** Use Ruff for both (`ruff check` + `ruff format`).

**Alternatives considered:**

- _Keep Black + Ruff._ Rejected — confirmed Ruff's formatter has been
  stable (not beta) since v0.3.0 and is explicitly designed as a
  Black-compatible drop-in, verified at 99.9%+ identical output on large
  real codebases (Django, Zulip were the test cases). No real downside
  at this project's size in consolidating to one tool.

**Consequences:** `CONTRIBUTING.md` and the illustrative CI pipeline in
`docs/deployment.md` §5 both updated to run `ruff format --check` instead
of a separate Black step. No `pyproject.toml`/`black` config to maintain.

---

## ADR-017: The Supabase/PowerSync keep-alive must perform an actual write, and covers both services from one workflow

**Status:** Accepted (refines the mechanism already implied by ADR-003;
this ADR makes it explicit rather than changing it)

**Context:** The original keep-alive design (a GitHub Actions workflow
pinging `GET /v1/health` or "a trivial Supabase query") was already the
right _mechanism_ — external, scheduled, not `pg_cron` — but was
under-specified in a way that research showed matters: community reports
from Supabase's own GitHub discussions describe inactivity detection as
keying off actual **write** activity, not a read or an open connection,
and separately describe cases where a project paused _despite_ a daily
job running against it. `pg_cron` specifically cannot serve this purpose
at all, regardless of how it's used: if a project has already paused,
nothing runs inside it, including its own scheduled jobs — the mechanism
preventing a pause has to originate outside the thing being kept alive.

**Decision:** The keep-alive workflow performs an `INSERT ... ON
CONFLICT DO UPDATE` against a dedicated single-row heartbeat table (see
`docs/database.md` §6) — a genuine write — on both Supabase and
PowerSync (ADR-013 introduced the second target), every 3–4 days.

**Alternatives considered:**

- _A read-only health check as the keep-alive signal._ This was the
  original, vaguer phrasing ("or a trivial Supabase query") — tightened
  to an explicit write given the reliability reports above. The
  `GET /v1/health` endpoint still exists and is still useful (uptime
  monitoring, pre-warming before a demo), but is no longer described as
  the keep-alive mechanism itself — see the correction in
  `docs/deployment.md` §6.
- _Vercel Cron instead of GitHub Actions._ Considered and not chosen —
  Vercel Hobby's cron is constrained enough (effectively one meaningful
  job) that it doesn't offer anything GitHub Actions doesn't already do
  more flexibly, including covering two services from one workflow.

**Consequences:** Even a correctly-implemented write-based ping is
evidenced to not be a 100% guarantee (see the anecdotal-but-credible
reports above) — `docs/troubleshooting.md` was updated to recommend
checking the workflow's own run history (a silently-failing keep-alive is
worse than none) and to have the web app degrade gracefully if a paused
project is ever hit anyway, rather than treating prevention as absolute.

---

## ADR-018: Adopt Docker narrowly — `services/ai-engine` only, not `apps/web`

**Status:** Accepted

**Context:** Considered whether to containerize anything in this
monorepo at all, and if so, how much of it.

**Decision:** A single Dockerfile for `services/ai-engine`. `apps/web`
is not containerized.

**Alternatives considered:**

- _No Docker anywhere, rely on pinned `requirements.txt`/Python version
  alone._ Rejected — PaddleOCR carries system-level dependencies (OpenCV
  bindings, shared libraries) that a requirements file alone doesn't
  pin, a realistic source of "works on my machine" failures across a
  multi-person team. Also would have foreclosed Google Cloud Run
  (ADR-014) as an option entirely, since Cloud Run requires a container
  image — not a decision to make by omission.
- _Containerize the web app too, or the whole monorepo via Docker
  Compose._ Rejected for `apps/web` specifically — Vercel's native Next.js
  build pipeline already provides the reproducibility a container would,
  and wrapping it in one adds a layer with no corresponding benefit.
  Docker Compose for local dev convenience (spinning up the ai-engine
  alongside a local Postgres) remains a reasonable future addition, not
  a requirement now.

**Consequences:** The team needs baseline Docker familiarity, which
several members may not have yet — treated as an acceptable, resume-
relevant cost given it's scoped to one service. Also worth noting: the
Supabase CLI's local dev stack (`supabase start`) already runs Postgres/
Auth/Storage as Docker containers, so Docker was an implicit local-dev
dependency regardless of this decision — this ADR just makes the
dependency deliberate and extends it to deployment.

---

## ADR-019: Investment/market-data features explain and contextualize; they never recommend a specific action

**Status:** Accepted — this is the governing constraint for every ADR
that follows it in this section, and for any future feature touching
investments.

**Context:** A team review surfaced two related but distinct proposals
for the investment/wealth-building side of Fincue. The first (an
external technical review) correctly identified that this domain hadn't
received the same engineering discipline already applied everywhere else
in the project — market data treated as a fallible dependency, financial
calculations kept deterministic, AI scoped narrowly — and proposed
fixing that. The second (a feature idea layered on top) described the
app telling users "you can invest here" or "this investment might
degrade." Those are not the same proposal, and only the first was
adopted as written.

**Decision:** Fincue explains and contextualizes a user's own portfolio
— performance drivers, concentration, volatility relative to a stated
risk profile — and never issues or implies a buy/sell/hold directive
about a specific security, and never characterizes its own output as
predicting future prices. Every AI-generated sentence about investments
is checked against this rule mechanically, not just by prompt wording —
see ADR-022 and `docs/testing-strategy.md` §8-9.

**Alternatives considered:**

- _Build the "tells you where to invest" feature as originally
  described._ Rejected. Apps that give personalized investment
  recommendations — confirmed by looking at how comparable real
  products are actually structured — are, without exception among the
  examples reviewed, disclosed as registered investment advisers (SEC,
  FCA, SEBI, or the local equivalent — Pakistan's SECP included). That
  registration is a genuine regulatory undertaking with no path to
  acquisition within an FYP's scope, resources, or timeline. Building the
  unlicensed version of a regulated activity is not a smaller version of
  the same feature; it's a different, inappropriate thing to build.
- _Add a disclaimer and build it anyway._ Rejected — a disclaimer does
  not change what the feature functionally does, and reviewed evidence
  (Schwab's own AI portfolio feature, from a fully-licensed broker)
  shows that even licensed institutions keep this kind of AI output on
  the "explains, doesn't recommend" side of the line rather than
  treating a disclaimer as sufficient cover for a directive.
- _Drop the feature area entirely to avoid the question._ Rejected — the
  underlying user need (understanding your investments, noticing risk,
  seeing how you're tracking against goals) is legitimate and achievable
  without crossing into advice. Schwab's "Portfolio Insights" and the
  app "Tali" were both reviewed as public examples of products doing
  exactly this: explanatory AI commentary on a user's own holdings, nothing
  more.

**Consequences:** Every subsequent ADR in this section, `docs/security.md`
§1.1, `docs/PRD.md` §4.4, and `docs/ml-strategy.md` §6 all implement this
constraint concretely rather than referencing it abstractly. If a future
feature idea seems to need a directive ("you should..."), the correct
response is to reshape it into an explanation of the user's own
situation, not to implement it as first described.

---

## ADR-020: Alpha Vantage (free tier) for market data, with a scheduled-cache architecture rather than live lookups

**Status:** Accepted

**Context:** Portfolio valuation, and any historical/volatility analytics,
need a source of stock/ETF prices. The $0 budget constraint applies here
exactly as it does everywhere else in this project.

**Decision:** Alpha Vantage's free tier (no card, ~25 requests/day) as
the primary data provider, with CoinGecko's free tier as a separate
source for crypto holdings if supported. Prices are fetched on a
schedule (daily, via the existing GitHub Actions scheduled-job mechanism
— `docs/deployment.md` §4) into `market_data_cache` and
`historical_prices` (`docs/database.md` §7), never fetched live
per-request.

**Alternatives considered:**

- _A live-lookup-per-request design._ Rejected on two independent
  grounds that happen to agree: it's not affordable (25 requests/day
  cannot serve live per-viewer lookups at any real usage), and it's not
  the accurate framing regardless of budget — see ADR-019's sibling
  concern about not overclaiming "real-time" data. The cache-based design
  is the honest one, not just the cheap one.
- _A paid market-data API._ Explicitly excluded by the project's budget
  constraint; not evaluated further, consistent with how every other
  paid-API alternative has been treated throughout this project.
- _Specific alternative free providers (Marketstack, Twelve Data, IEX
  Cloud, etc.)._ Not committed to at this time — Alpha Vantage was
  chosen as the best-documented, most consistently available free option
  at time of review, but the client code should treat the provider as
  swappable (mirroring the pattern already used for Gemini/Groq in
  `docs/ml-strategy.md` §5), not hard-coded, since this space is exactly
  as volatile as the LLM-provider space already documented in
  `docs/deployment.md`.

**Consequences:** One fetch per symbol per day must serve every user
holding that symbol — the ingestion job fetches the _union_ of symbols
across all users' holdings, not one fetch per user. A symbol with no
active holders doesn't get fetched. This is a real architectural
constraint on the ingestion job's design, not an implementation detail
to figure out later.

---

## ADR-021: Deterministic financial calculation layer for portfolio math

**Status:** Accepted (extends ADR-006, ADR-011, ADR-012 into a new domain
rather than establishing a new principle)

**Context:** Portfolio value, unrealized gain/loss, returns, volatility,
maximum drawdown, and a goal's investment-derived contribution are all
genuine calculations with a single correct answer given the same inputs.

**Decision:** All of the above are implemented as plain, unit-tested
functions in `services/ai-engine` (or database-side SQL where
appropriate), fully evaluable without any AI involvement. AI never
computes a financial figure — only explains one already computed. See
`docs/ml-strategy.md` §6 steps 5–6 and `docs/testing-strategy.md` §8.

**Alternatives considered:**

- _Let the LLM compute portfolio metrics from a text description of
  holdings._ Rejected outright — this project has already rejected
  LLM-computed figures everywhere else (the NLP assistant explicitly
  never calculates a spending total itself, ADR-011) specifically
  because LLMs are a documented source of confidently-wrong arithmetic.
  There's no reason investment math would be an exception, and every
  reason (the compliance stakes from ADR-019) it should be held to a
  _higher_ bar than transaction totals, not a lower one.

**Consequences:** None of this is new engineering philosophy for the
project — it's the existing philosophy, applied somewhere it hadn't
explicitly been stated yet. Recorded as its own ADR specifically so
`docs/testing-strategy.md` §8's fixed-value testing requirement has a
clear decision to point back to.

---

## ADR-022: Risk tolerance as an explicit, stored system input — not a detail inside an LLM prompt

**Status:** Accepted

**Context:** Contextualizing a user's portfolio against "how much risk
they're comfortable with" requires knowing that risk tolerance somehow.
The easy version — asking the user in the assistant chat and letting the
LLM remember it within a conversation — was considered and rejected.

**Decision:** A short onboarding questionnaire captures stated risk
tolerance and investment horizon into a real table (`risk_profiles`,
`docs/database.md` §7). Concentration and volatility-mismatch flags are
computed by comparing stored holdings data against this stored profile
using plain rules — the AI interpretation step receives the _result_ of
that comparison, never the raw profile to reason about freshly each time.

**Alternatives considered:**

- _Pass risk tolerance as unstructured context in the assistant's
  prompt._ Rejected — this is exactly the pattern ADR-011 already
  rejected for the NLP assistant generally (an LLM computing something
  from context it was merely told, rather than a system computing it and
  the LLM explaining the result). It's also not auditable: a stored,
  versioned profile can be shown to the user ("your stated tolerance is
  X, set on this date") in a way that "the LLM inferred it from
  conversation" cannot.

**Consequences:** Requires an actual onboarding step before portfolio
insights are meaningful (a user with no stated profile gets valuation
and factual analytics, but not risk-mismatch commentary, until they set
one) — an acceptable, honestly-scoped limitation rather than guessing at
a default risk tolerance on someone's behalf.

---

## ADR-023: Investment/market-data capabilities are mostly Phase 2 or Stretch, not MVP

**Status:** Accepted

**Context:** The reviewed proposal, taken in full, adds real scope:
market data ingestion, historical storage, a risk-profiling
questionnaire, a new analytics layer, and new testing categories, on top
of a project that was already ambitious before this review. `docs/PRD.md`
already tags every feature area by phase specifically to protect the
project deadline (see that document's introduction); this domain gets
the same treatment rather than an exception.

**Decision:** Per the table in `docs/PRD.md` §4.4 and
`docs/ml-strategy.md` §6: manual holdings entry and net-worth rollup stay
MVP (already true before this review). Automatic revaluation, historical
analytics, risk profiling, and AI-generated portfolio summaries move to
Phase 2. Scenario/backtest-style projections and Plaid-based automatic
sync (already Stretch per ADR-009) stay Stretch.

**Alternatives considered:**

- _Treat all of it as MVP, given how thorough the reviewed proposal is._
  Rejected — thoroughness of a proposal isn't the same question as
  whether it fits the timeline. `docs/PRD.md`'s entire MVP/Phase 2/
  Stretch discipline exists to keep a good idea from silently becoming
  scope creep; this domain doesn't get to skip that discipline just
  because the proposal that introduced it was well-written.

**Consequences:** The FYP can demo a complete, honest, working ledger and
budgeting product even if Phase 2 wealth-building features run out of
time — which was already true before this review, and remains true after
it. `roadmap.md` places these items accordingly.
