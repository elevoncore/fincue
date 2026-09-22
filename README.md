# Fincue

> An AI-driven personal finance platform that acts as a proactive financial
> advisor, not just a passive expense tracker — built as a Final Year Project
> on a strict **$0 infrastructure budget**.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)
[![Node](https://img.shields.io/badge/node-%3E%3D22-339933?logo=node.js&logoColor=white)](./.nvmrc)
[![Status](https://img.shields.io/badge/status-pre--alpha-orange)](./roadmap.md)

> **This repository is a skeleton.** Per the project's build directive, no
> application code has been written yet — this commit contains only the
> folder structure, tooling configuration, and the documentation set below.
> See [`docs/developer-guide.md`](./docs/developer-guide.md) to start
> implementing, and [`roadmap.md`](./roadmap.md) (local, git-ignored) for the
> phase-by-phase build plan.

---

## What this is

Fincue is a full-stack, AI-assisted personal finance platform. Instead
of just recording transactions, it actively helps a user build wealth and
better financial habits through:

| Pillar | What it does |
|---|---|
| **1. Ledger & transactions** | Frictionless manual entry with offline queueing, receipt-photo OCR, split transactions, multi-currency support, free-text tagging. |
| **2. AI & ML layer** | Automatic merchant categorization, predictive overspending alerts, anomaly/duplicate-charge detection, and a natural-language assistant ("How much did I spend on dining out last weekend?"). |
| **3. Budgeting & cash flow** | Zero-based budgeting with an Envelope System UI, rollovers, a cash-flow forecaster, subscription manager, and a real-time "Safe-to-Spend" number. |
| **4. Goals & wealth building** | Short-term goals, long-term retirement projections, round-up auto-savings, an Avalanche vs. Snowball debt simulator, and AI-suggested "milestone pausing" when cash is tight. |
| **5. Gamification** | Spending streaks, a 0–100 Financial Health Score, color-coded budget nudges, and unlockable badges. |
| **6. Reporting & analytics** | Sankey cash-flow diagrams, drag-and-drop dashboards, a burn-rate visualizer, tax-deductible transaction export, and a Spotify-Wrapped-style yearly/monthly summary. |
| **7. Security** | Biometric app lock, end-to-end encryption for sensitive fields, session auto-timeout, and one-click data portability. |

The full feature spec lives in [`docs/PRD.md`](./docs/PRD.md).

## Tech stack (all free-tier)

Every choice below was made to fit a **$0 budget** while still being a
credible, defensible architecture for an enterprise-style FYP submission.
Full reasoning and rejected alternatives are in `decisions.md` (local,
git-ignored — see [Internal AI tracking files](#internal-ai-tracking-files)).

| Layer | Choice | Why (short version) |
|---|---|---|
| Web frontend | **Next.js** on **Vercel** (Hobby) | Native fit, generous free quota (100 GB transfer, 1M function invocations/mo), zero-config CI/CD. |
| Mobile (Phase 10+) | **Expo (React Native)** | Shares types/logic with web via `packages/`; free, open source. |
| Database + Auth + Storage | **Supabase** (free tier) | Postgres + Auth + Storage + Realtime + Row-Level Security bundled; biggest productivity win for a solo/small-team FYP. |
| Offline sync (web + mobile) | **PowerSync** (free tier) | One sync engine for both platforms instead of hand-rolled IndexedDB/SQLite logic; see `decisions.md` ADR-013. |
| Decoupled AI backend | **Python + FastAPI** (`services/ai-engine`), containerized | Hosts OCR, categorization, anomaly detection, forecasting, and NLP-assistant orchestration; called by both web and future mobile. **Hosting provider (Render vs. Cloud Run) is intentionally still open** — see `decisions.md` ADR-014. |
| NLP assistant | **Gemini API** (free tier) + **Groq** (free tier, fallback) | Function-calling only — the LLM parses intent, it never sees raw ledger data. |
| Categorization ML | Sentence embeddings + classical classifier (scikit-learn) | Trained on free Kaggle datasets; see `docs/ml-strategy.md`. |
| Receipt OCR | **PaddleOCR** (open source) | Better receipt accuracy than Tesseract in general benchmarks; evaluated against the SROIE dataset. A multimodal-LLM alternative was evaluated and deferred — see ADR-015. |
| Currency rates | **`@fawazahmed0/currency-api`** (free, unlimited, no key) | 200+ currencies, daily updates, CDN-hosted. |
| Monorepo tooling | **Turborepo** + **pnpm workspaces** | Caches builds across `apps/`, `packages/`, `services/`. |
| Python lint + format | **Ruff**, exclusively | One tool for both, not Ruff + Black — stable, Black-compatible formatter, dramatically faster. |

See [`docs/architecture.md`](./docs/architecture.md) for the full system
diagram and [`docs/deployment.md`](./docs/deployment.md) for exact free-tier
limits and deploy steps.

## Repository layout

```text
fincue/
├── apps/            # Deployable applications (web, future mobile)
├── services/        # Decoupled backend services (ai-engine)
├── packages/        # Shared code: types, UI kit, shared config
├── docs/            # Everything described below
└── scripts/         # One-off setup/maintenance scripts (added as needed)
```

The full annotated tree is documented in [`docs/architecture.md`](./docs/architecture.md#repository-layout).

## Getting started

See [`docs/developer-guide.md`](./docs/developer-guide.md) for full setup
instructions (accounts to create, environment variables, first run). The
short version, once this skeleton has real app code in it:

```bash
pnpm install
cp .env.example apps/web/.env.local
cp .env.example services/ai-engine/.env
pnpm dev
```

## Documentation index

| Doc | Purpose |
|---|---|
| [`docs/PRD.md`](./docs/PRD.md) | Product requirements: personas, user stories, MVP vs. later phases. |
| [`docs/architecture.md`](./docs/architecture.md) | System design, data flow, offline sync strategy. |
| [`docs/database.md`](./docs/database.md) | Schema, ER diagram, indexing, RLS strategy. |
| [`docs/api.md`](./docs/api.md) | REST endpoints and payload contracts. |
| [`docs/security.md`](./docs/security.md) | Auth, encryption, threat model, third-party data-sharing risk. |
| [`docs/deployment.md`](./docs/deployment.md) | How to deploy to the chosen free-tier stack, with current limits. |
| [`docs/ml-strategy.md`](./docs/ml-strategy.md) | Datasets, models, and feasibility research for every AI feature. |
| [`docs/developer-guide.md`](./docs/developer-guide.md) | Local setup from zero. |
| [`docs/troubleshooting.md`](./docs/troubleshooting.md) | Known free-tier gotchas and fixes. |
| [`docs/user-guide.md`](./docs/user-guide.md) | End-user-facing help content. |
| [`docs/gamification-strategy.md`](./docs/gamification-strategy.md) | Health Score formula, streaks, badges. |
| [`docs/testing-strategy.md`](./docs/testing-strategy.md) | Test pyramid, tools, coverage targets. |
| [`docs/glossary.md`](./docs/glossary.md) | Terminology (ZBB, RLS, MCC, KIE, etc.). |


## Contributing

See [`CONTRIBUTING.md`](./CONTRIBUTING.md). This project follows the
[`CODE_OF_CONDUCT.md`](./CODE_OF_CONDUCT.md) for all interactions.

## Security

Found a vulnerability? Please see [`SECURITY.md`](./SECURITY.md) rather than
opening a public issue.

## License

[MIT](./LICENSE) — see the license file for a note on checking your
university's FYP IP policy before publishing.
