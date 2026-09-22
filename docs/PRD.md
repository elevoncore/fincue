# Product Requirements Document (PRD)

**Product:** Fincue
**Status:** Draft v0.1 — derived from the original FYP concept brief
**Owner:** [Team name]
**Last updated:** [update when you actually revise this]

> **A note on scope discipline.** The original concept brief describes an
> extremely ambitious feature set — genuinely "enterprise fintech" in scope.
> That's appropriate for a vision document, but a Final Year Project has a
> hard deadline and (per the constraints of this build) a $0 budget. This
> PRD keeps every original feature but tags each one **MVP**, **Phase 2**,
> or **Stretch**, so the team always knows what "done" means for a demo,
> and what's acceptable to describe as "designed for, not yet built" in the
> final report. Re-negotiate these tags with your supervisor, not silently.

## 1. Vision

Most personal finance apps are passive ledgers: they show you what you
already did. Fincue is designed to behave like a proactive
financial advisor — it notices patterns, warns before problems happen, and
nudges behavior change — while remaining a single-user, privacy-respecting,
$0-infrastructure-cost application suitable for an academic demo and a
realistic portfolio piece.

## 2. Target users / personas

| Persona | Description | Primary need |
|---|---|---|
| **Student saver** (primary demo persona) | Limited, irregular income; multiple small subscriptions; wants to avoid overdraft and build a small emergency fund. | Safe-to-Spend clarity, subscription creep alerts, gamified motivation. |
| **Young professional** | Regular salary, some debt (student loan/credit card), starting to invest. | Budgeting discipline, debt payoff simulation, net worth tracking. |
| **Traveler / multi-currency user** | Spends in 2+ currencies regularly. | Reliable multi-currency ledger and reporting. |

Assumption: the demo/evaluation will primarily showcase the **Student
saver** and **Young professional** personas, since they're achievable with
synthetic demo data and don't require real bank connections.

## 3. Goals and non-goals

**Goals**

- Demonstrate a coherent, working slice of every one of the 7 feature
  pillars from the original brief — even if a given pillar's Phase 2/Stretch
  items are only designed, not implemented, by the final deadline.
- Show genuine (not superficial) use of ML/AI: a trained classifier, not
  just "we called an LLM"; a real anomaly-detection method; a real
  NLP-to-structured-query pipeline.
- Ship something that runs entirely on free infrastructure, with no
  surprise bills, and is architecturally ready for a "real" deployment
  later (hence the decoupled backend, from day one, in a monorepo ready
  to add a mobile client).

**Non-goals (explicitly out of scope for this project)**

- Becoming a licensed financial institution, money transmitter, or
  investment advisor. No feature should require regulatory licensing
  (see `docs/security.md`, "Compliance disclaimer").
- Real production-scale multi-tenant hosting. The free-tier stack is
  chosen for a demo/portfolio scale (dozens to low hundreds of users), not
  for real-world production traffic.
- Building custom bank-scraping/screen-scraping for account aggregation —
  if investment/account sync is attempted at all, it goes through Plaid's
  Sandbox/Trial environment (see `docs/ml-strategy.md`), never a homegrown
  scraper (security and ToS risk).

## 4. Feature pillars, scoped

Each pillar from the original brief, broken into user stories with an MVP /
Phase 2 / Stretch tag. "MVP" = must exist, even in a simplified form, for
the project to be considered complete. "Phase 2" = designed and
architecturally supported, implemented if time allows. "Stretch" = nice
story for the final report, not expected to ship.

### 4.1 Core transaction & ledger management

| User story | Tag |
|---|---|
| As a user, I can manually add an income or expense transaction with amount, category, date, and note. | MVP |
| As a user, my transactions entered while offline are queued locally and sync automatically when I'm back online. | MVP |
| As a user, I can photograph a receipt and have merchant, date, and total extracted automatically for review before saving. | MVP (OCR MVP = PaddleOCR + regex/heuristic field extraction; see `docs/ml-strategy.md`) |
| As a user, I can split one receipt into multiple transactions across categories. | MVP |
| As a user, I can log a transaction in a foreign currency and have it converted at the live rate, or enter my own rate. | MVP |
| As a user, I can attach free-text tags to any transaction and filter by them. | MVP |

### 4.2 AI & machine learning layer

| User story | Tag |
|---|---|
| As a user, my transactions are automatically assigned a category from raw merchant text, with a visible confidence level and a one-tap correction if it's wrong. | MVP (classical ML — embeddings + classifier; see `docs/ml-strategy.md`) |
| As a user, I get warned before I'm on track to overspend a budget category, based on my spending pace this period. | MVP (simple velocity/statistical model, not deep learning) |
| As a user, I'm alerted to likely duplicate charges or a subscription's price quietly increasing. | MVP (Isolation Forest / statistical baseline, per-merchant) |
| As a user, I can ask a plain-language question about my spending and get an accurate, data-grounded answer. | MVP, narrow scope (a fixed set of supported query intents — see `docs/ml-strategy.md` — not open-ended chat) |

### 4.3 Budgeting & cash flow control

| User story | Tag |
|---|---|
| As a user, I can set up a zero-based budget with an envelope per category. | MVP |
| As a user, unspent envelope funds roll over to next month if I choose. | MVP |
| As a user, I see a "Safe-to-Spend" number that accounts for upcoming known bills. | MVP |
| As a user, I have a dedicated view of my recurring subscriptions and their next charge dates. | MVP |
| As a user, the app forecasts my cash flow over the next 30/60/90 days. | Phase 2 (simple statistical forecast in MVP; more refined model is Phase 2) |
| As a user, the app notices a recurring bill has crept up in price and prompts me to negotiate/cancel. | Phase 2 (depends on subscription price-history tracking accumulating enough data) |

### 4.4 Goal setting & wealth building

| User story | Tag |
|---|---|
| As a user, I can create a short-term savings goal with a target amount and date. | MVP |
| As a user, I can view a long-term retirement projection factoring in compound interest and inflation. | MVP (deterministic formula, not a full Monte Carlo simulation) |
| As a user, I can enable round-up savings that sweep spare change into a goal. | MVP (computed at transaction-save time; MVP does **not** require a real linked bank sweep — it's an in-app virtual envelope unless Plaid/Phase 2 account linking is implemented) |
| As a user, I can compare Avalanche vs. Snowball debt payoff strategies side by side. | MVP |
| As a user, when my cash flow is tight, the app suggests which goals to pause ("Milestone Pausing"). | Phase 2 |
| As a user, I can see my net worth over time and (optionally) sync investment holdings. | Net worth: MVP (computed from manually entered accounts). Investment **sync** via Plaid: Phase 2/Stretch — see `docs/ml-strategy.md`. |

### 4.5 Gamification & behavioral finance

| User story | Tag |
|---|---|
| As a user, I see my current "no discretionary spending" streak. | MVP |
| As a user, I have a Financial Health Score (0–100) reflecting my habits. | MVP — formula in `docs/gamification-strategy.md` |
| As a user, budget progress bars shift color (green→yellow→red) as I approach a limit. | MVP |
| As a user, I unlock badges/animations for milestones (goal hit, debt cleared). | Phase 2 (a small starter badge set is MVP; the full catalog is Phase 2) |

### 4.6 Reporting & visual analytics

| User story | Tag |
|---|---|
| As a user, I see my cash flow as an interactive Sankey diagram (income → categories). | MVP |
| As a user, I can build a customizable, drag-and-drop dashboard. | Phase 2 (a fixed, well-designed default dashboard is MVP; drag-and-drop customization is Phase 2) |
| As a user, I can export tax-deductible transactions for my accountant. | MVP (CSV export with a tax-relevant filter) |
| As a user, I get a shareable "Financial Wrapped" summary monthly/yearly. | Phase 2 |
| As a user, I can see a burn-rate visualizer. | MVP |

### 4.7 Security & infrastructure

| User story | Tag |
|---|---|
| As a user, the app is locked behind biometric authentication even if my phone is unlocked. | MVP on web (WebAuthn/passkey re-auth); mobile biometric lock is part of the Phase 10 mobile build. |
| As a user, my sensitive data is encrypted at rest and in transit. | MVP (TLS everywhere + Postgres RLS + field-level encryption for the most sensitive fields) |
| As a user, my session times out automatically after inactivity. | MVP |
| As a user, I can export my entire financial history in one click. | MVP |

## 5. Success metrics (for the FYP evaluation, not a real business)

- **Functional completeness:** all MVP-tagged stories demoable end-to-end
  with synthetic data.
- **ML credibility:** the categorization model achieves a reported
  precision/recall on a held-out split of the training dataset (see
  `docs/ml-strategy.md` for target benchmarks and citations), and this is
  documented with an actual confusion matrix in the final report — not
  just asserted.
- **Zero infrastructure cost** maintained through the demo period (tracked
  informally against the limits in `docs/deployment.md`).
- **Security baseline:** no plaintext secrets in the repository (verified
  by `git log -p` review before submission), RLS policies verified with a
  negative test (user A cannot read user B's data).

## 6. Key assumptions

State these explicitly to your supervisor rather than letting them stay
implicit:

- Team size and timeline: this PRD and the accompanying `roadmap.md`
  assume a small team (2–4 students) over a typical 12–16 week FYP
  timeline. Re-scope the phase boundaries in `roadmap.md` if your actual
  timeline differs.
- The "bank sync" and "investment portfolio sync" stories use Plaid's free
  Sandbox (fully mocked data) for the primary demo, since real production
  bank-linking requires an approved, typically paid, Plaid plan beyond
  small free allowances — see `docs/ml-strategy.md`.
- Demo data is synthetic/generated, not real personal financial data,
  both for privacy and to keep the free-tier database well under its
  storage cap.

## 7. Risks

| Risk | Mitigation |
|---|---|
| Free-tier service is discontinued or changes terms mid-project (this space moves fast — see the dated citations in `docs/deployment.md`). | Every hosting/API choice in `docs/deployment.md` and `docs/ml-strategy.md` lists at least one fallback option. |
| ML scope creep (deep learning where classical ML would do). | `docs/ml-strategy.md` deliberately recommends the simplest approach that meets the accuracy bar, with citations. |
| Feature list is too large for the timeline. | The MVP/Phase 2/Stretch tagging above exists specifically to protect the deadline — treat it as a living negotiation, updated in `decisions.md` when priorities change. |
