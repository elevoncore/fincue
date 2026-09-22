# Developer Guide

Getting from a fresh clone to a running local environment. This assumes
the app code described in `docs/architecture.md` has been scaffolded —
right now this repository is the skeleton described in the root `README.md`.

## 1. Prerequisites

| Tool | Version | Why |
|---|---|---|
| [Node.js](https://nodejs.org/) | ≥ 22 (see `.nvmrc`) | Web app + tooling. Node 22 is Active/Maintenance LTS as of 2026 — check `nodejs.org` if you're reading this much later. |
| [pnpm](https://pnpm.io/) | ≥ 9 | Monorepo package manager (`corepack enable` is the easiest way to get the pinned version from `package.json`). |
| [Python](https://www.python.org/) | ≥ 3.11 | `services/ai-engine`. |
| [Git](https://git-scm.com/) | any recent | |
| A [Supabase](https://supabase.com/) account | free | Database, Auth, Storage. |
| A [Kaggle](https://www.kaggle.com/) account | free | Only needed if you're working on `services/ai-engine` ML training. |
| A [Google AI Studio](https://ai.google.dev/) account | free | Gemini API key for the NLP assistant. |
| A [Groq](https://console.groq.com/) account | free | Fallback LLM for the NLP assistant. |

## 2. Clone and install

```bash
git clone <your-fork-or-repo-url> fincue
cd fincue
corepack enable
pnpm install
```

## 3. Set up Supabase

1. Create a new project at `supabase.com` (free tier — see
   `docs/deployment.md` for its limits, especially the 7-day inactivity
   pause).
2. From **Project Settings → API**, copy the project URL, anon key, and
   service role key.
3. Apply the schema in `docs/database.md` — once migration files exist
   under `supabase/migrations/`, run:
   ```bash
   npx supabase link --project-ref <your-project-ref>
   npx supabase db push
   ```
4. Confirm Row-Level Security is enabled on every user-owned table (see
   `docs/database.md` §4) before putting any data — even test data — in.

## 4. Configure environment variables

```bash
cp .env.example apps/web/.env.local
cp .env.example services/ai-engine/.env
```

Fill in, at minimum, for local development:

- `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY` (web)
- `SUPABASE_SERVICE_ROLE_KEY`, `SUPABASE_DB_URL` (ai-engine)
- `GEMINI_API_KEY` (from Google AI Studio — free, no card)
- `GROQ_API_KEY` (from console.groq.com — free, no card)
- `AI_ENGINE_INTERNAL_API_KEY` — invent any long random string; it just
  needs to match between `apps/web`'s `.env.local` and
  `services/ai-engine`'s `.env`.

Leave Plaid, Sentry, and R2 variables blank until you actually implement
those Phase 2 features.

## 5. Run everything locally

```bash
pnpm dev
```

This runs `turbo run dev` across all workspaces (see `turbo.json`) — once
`apps/web` exists, this starts Next.js on `localhost:3000`. The Python
`ai-engine` is not a pnpm workspace member (it's a separate language
runtime), so run it separately:

```bash
cd services/ai-engine
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt --break-system-packages   # once requirements.txt exists
uvicorn app.main:app --reload --port 8000
```

Verify both are up:
- Web: open `http://localhost:3000`
- AI engine: `curl http://localhost:8000/v1/health` should return
  `{"status": "ok", ...}` (see `docs/api.md`)

## 6. Common commands

| Command | What it does |
|---|---|
| `pnpm dev` | Run all apps/packages in dev mode (Turborepo-orchestrated). |
| `pnpm build` | Build all apps/packages. |
| `pnpm lint` | Lint the whole monorepo. |
| `pnpm test` | Run tests across the monorepo (see `docs/testing-strategy.md`). |
| `pnpm format` | Format with Prettier. |
| `turbo run dev --filter=web` | Run just the web app. |

## 7. Working on the ML pipeline specifically

See `docs/ml-strategy.md` for what to build and why. In short:

```bash
cd services/ai-engine
python datasets/download_categorization_data.py   # once this script exists — needs your own Kaggle API token in ~/.kaggle/kaggle.json, never committed
jupyter lab notebooks/
```

## 8. Folder structure recap

See `docs/architecture.md` §2 for the annotated full tree. The short
version: `apps/` is what gets deployed to users, `services/` is the
decoupled AI backend, `packages/` is code shared between apps, `docs/` is
this documentation set.

## 9. If something doesn't work

Check `docs/troubleshooting.md` first — it covers the specific free-tier
gotchas (Supabase pause, cold starts, CORS) that are easy to mistake for
bugs in your own code.
