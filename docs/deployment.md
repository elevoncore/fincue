# Deployment

This document is the most time-sensitive one in the repository — free-tier
terms change often. Every limit below is dated; **re-verify at the
official source before you rely on it**, especially if you're reading this
more than a few months after the date shown.

## 1. Chosen stack at a glance

| Component | Provider | Tier | Verified |
|---|---|---|---|
| Web frontend | [Vercel](https://vercel.com/pricing) | Hobby (free) | Sep 2026 |
| Static assets / bandwidth-heavy fallback | [Cloudflare Pages](https://developers.cloudflare.com/pages/platform/limits/) | Free | Sep 2026 |
| Database + Auth + Storage | [Supabase](https://supabase.com/pricing) | Free | Sep 2026 |
| Offline sync (web + mobile) | [PowerSync](https://powersync.com/pricing) | Free | Sep 2026 |
| Decoupled AI backend — **decision deferred, see ADR-014** | [Render](https://render.com/docs/free) | Free Web Service | Sep 2026 |
| AI backend — alternative under evaluation | [Google Cloud Run](https://cloud.google.com/run/pricing) | "Always Free" | Sep 2026 |
| AI backend — alternative | [Hugging Face Spaces](https://huggingface.co/docs/hub/spaces-overview) | Free CPU Basic | Sep 2026 — see caveat below |
| NLP assistant LLM | [Google AI Studio (Gemini)](https://ai.google.dev/pricing) | Free tier | Sep 2026 — volatile, verify before relying |
| NLP assistant LLM — fallback | [Groq](https://console.groq.com/docs/rate-limits) | Free tier | Sep 2026 |
| Currency rates | [`@fawazahmed0/currency-api`](https://github.com/fawazahmed0/currency-api) | Free, no key | Sep 2026 |
| Error monitoring | [Sentry](https://sentry.io/pricing/) | Developer (free) | Sep 2026 |
| CI/CD | [GitHub Actions](https://docs.github.com/en/billing/managing-billing-for-your-products/managing-billing-for-github-actions/about-billing-for-github-actions) | Free minutes | Sep 2026 |
| Python lint + format | [Ruff](https://docs.astral.sh/ruff/) | Open source, local | Sep 2026 |

## 2. Free-tier limits, in detail

### Vercel Hobby (web frontend)

- 100 GB "Fast Data Transfer" per month, 1M Edge Requests/month, 1M
  Function invocations/month.
- Cron jobs are capped at **2 invocations/day** on Hobby — this is why
  scheduled jobs (subscription price-creep checks, net-worth snapshots,
  anomaly scans) are documented below as GitHub Actions / Supabase
  `pg_cron` jobs instead of Vercel Cron.
- Hobby deployments are officially scoped to **personal, non-commercial**
  use per Vercel's terms — fine for an FYP demo; if this ever becomes a
  real product, budget for the Pro plan ($20/month) at that point.
- Hobby projects now default to **30-day deployment retention** (older
  preview deployments are pruned; your 10 most recent production
  deployments are always kept).
- There is no hard spend cap mechanism on paid usage beyond Hobby — this
  doesn't matter while you stay on Hobby (there's no overage billing on
  Hobby itself), but don't casually upgrade to Pro without reading
  Vercel's current bandwidth-overage pricing first.

### Cloudflare Pages + Workers (fallback / bandwidth-heavy scale-up)

- Pages: **effectively unlimited bandwidth** for standard web assets, 500
  builds/month, only 1 concurrent build, 20,000 files/site.
- Workers (if used for lightweight edge logic): 100,000 requests/day free,
  a strict **30ms CPU time per request** — far too little for OCR/ML
  inference, which is exactly why those live in `services/ai-engine`
  instead of a Worker.
- Cloudflare R2 (object storage, useful for receipt images at scale): 10
  GB storage free, 1M Class A + 10M Class B operations/month, **zero
  egress fees** — the natural scale-up path once Supabase Storage's 1 GB
  free cap is outgrown.

### Supabase (database, auth, storage)

- 2 active free projects per organization, 500 MB database, 1 GB file
  storage, ~5 GB bandwidth/month, 50,000 monthly active users (Auth),
  500,000 Edge Function invocations/month.
- **Free projects pause after 7 days of API inactivity** and must be
  manually restored from the dashboard. Community reports (Supabase's own
  GitHub discussions) describe inactivity detection as based on actual
  database **write** activity, not merely an open connection or a read —
  and also describe cases where a project paused despite a daily job
  running against it, so treat this as a reliable *reduction* in risk,
  not a guarantee. Mitigation: the scheduled GitHub Actions workflow in
  §4 upserts a heartbeat row (a genuine write) every 3–4 days — see
  `decisions.md` ADR-017 — and the web app should show a friendly
  "warming up" state rather than a raw error if a request ever does hit a
  paused project.
- No automated daily backups on the free tier — use the data-export
  feature from `docs/security.md` §8 as a manual substitute during
  development.

### PowerSync (offline sync, web + mobile)

- Free tier: $0/month, no card required. Up to 2 GB data synced/month, 500
  MB hosted on the PowerSync Service, 50 peak concurrent connections, 2
  PowerSync Service instances, 1 source database connection — comfortably
  enough for FYP demo traffic.
- **Free projects are deactivated after 7 days of inactivity** — the
  identical failure mode as Supabase's, and not a coincidence to treat as
  two separate problems: the same GitHub Actions workflow in §4 pings
  both.
- A genuinely free, self-hosted alternative exists (PowerSync "Open
  Edition," source-available, Docker-deployable) that removes the
  inactivity risk entirely at the cost of one more service the team
  operates itself — not adopted for now given the team's limited bandwidth
  for extra operational surface area, but worth revisiting if the
  hosted free tier's limits or pause behavior become a real problem.
- Requires enabling Postgres logical replication on the Supabase project
  (a one-time setup step — see `docs/database.md`) and defining sync
  rules scoping data per user.

### Render (ai-engine hosting — default while this decision stays open)

**This choice is intentionally not finalized** — see `decisions.md`
ADR-014. Render is documented as the default so Phase 1 isn't blocked,
not as a closed decision.

- 750 free instance-hours/month (pooled across your free services), 100
  GB outbound bandwidth, 500 build-pipeline minutes/month.
- Free web services **spin down after ~15 minutes of inactivity**; the
  next request pays a cold-start cost (roughly 30–60 seconds). Acceptable
  for a demo; mention it explicitly if timing a live demo (or hit the
  health endpoint a minute before you present).
- **Do not use Render's free Postgres** for this project's database — it
  is explicitly time-boxed (deleted ~30 days after creation, with a short
  grace period), unlike Supabase/Neon's indefinite free tiers. Render is
  recommended here for **compute only** (the FastAPI service).
- Requires no billing account or card on file at all — the main practical
  advantage over Cloud Run below.

### Hugging Face Spaces (ai-engine hosting — alternative)

- Historically, "CPU Basic" free Spaces offer 2 vCPU / 16 GB RAM / 50 GB
  disk, sleeping after inactivity — a good fit for a Dockerized FastAPI
  app.
- **Caveat as of mid-to-late 2026:** reporting on Hugging Face's free
  compute policy has been inconsistent — some sources describe persistent
  Docker/Gradio Spaces as increasingly gated behind paid plans, while
  HF's own current documentation still shows a concurrent-runtime
  allowance for free CPU-based Spaces. **Verify at
  `huggingface.co/docs/hub/spaces-overview` at build time** before
  committing to this option; Render is the safer default specifically
  because its free-tier terms were unambiguous at time of writing.

### Google Cloud Run (ai-engine hosting — alternative under evaluation)

- The "Always Free" tier (2M requests/month, plus either ~180,000
  vCPU-seconds + ~360,000 GiB-seconds under request-based billing, or
  ~240,000 vCPU-seconds + ~450,000 GiB-seconds under instance-based
  billing) does not expire the way a trial credit does — but **requires a
  billing account with a card on file**, even though you won't be charged
  within the free allowance. Render requires no card at all.
- **Important correction to a common assumption:** Cloud Run's free tier
  does **not**, by itself, eliminate cold starts. It scales to zero by
  default, same as Render, and pays a cold-start cost on the next request
  — typically faster for a lean container (roughly 500ms–2s for a
  "typical" Node/Python app) than Render's ~30–60s, but a slower Python
  service loading PaddleOCR and `sentence-transformers` at startup will
  likely erode a meaningful part of that advantage. Actually eliminating
  cold starts requires setting `min-instances ≥ 1`, which bills for idle
  time continuously — a real, ongoing cost outside the free tier's scope,
  not a free way to get always-warm compute.
- Requires a Dockerized service — see `docs/architecture.md` §7. This is
  already satisfied regardless of which way ADR-014 is ultimately
  decided, since the ai-engine is containerized either way.

### LLM APIs (NLP assistant)

- **Gemini API (Google AI Studio), free tier:** no credit card required.
  **Published rate limits for this tier have been genuinely volatile** —
  different current sources report figures ranging from roughly 500–1,500
  requests/day down to as low as double-digit daily requests for
  comparable Flash-tier models, depending on exactly which model version
  and when it was checked. Treat any specific number (including earlier
  figures in this repository's own history) as unreliable without
  checking `ai.google.dev/gemini-api/docs/rate-limits` immediately before
  you build against it. **Confirmed and stable across sources, and more
  important than the exact numbers:** free-tier prompts and images may be
  used by Google to improve its products — see `docs/security.md` §5.
- **Groq, free tier (fallback/low-latency option):** no credit card
  required; rate-limited (on the order of 30 requests/minute and a daily
  token cap, varying by model). Very fast inference (custom LPU
  hardware), useful if the assistant's perceived responsiveness matters
  for the demo.
- Both are subject to change — recheck `console.groq.com` and
  `ai.google.dev/pricing` before the final submission.

### Error monitoring — Sentry

- Free "Developer" tier: a capped monthly error volume (order of a few
  thousand events), 1 team member, short data retention. Enough for a
  solo/small-team FYP; verify the current cap at `sentry.io/pricing`.

### CI — GitHub Actions

- Public repositories: unlimited minutes. Private repositories on a free
  GitHub account: a monthly free-minutes allowance (order of 2,000
  minutes on Linux runners; Windows/macOS runners consume the allowance
  faster via a multiplier). Keeping the repository public (reasonable for
  an FYP, subject to your university's policy) avoids this limit
  entirely.

## 3. Step-by-step: first deploy

1. **Supabase:** create a project at `supabase.com`, note the project URL
   and keys (Project Settings → API), run the initial schema migration
   (see `docs/database.md` §5), and enable the RLS policies before any
   real/demo data goes in.
2. **Vercel:** import the GitHub repository, set the **root directory**
   to `apps/web`, add the `NEXT_PUBLIC_SUPABASE_URL` /
   `NEXT_PUBLIC_SUPABASE_ANON_KEY` / `AI_ENGINE_URL` environment
   variables in the Vercel dashboard, deploy.
3. **Render (ai-engine):** create a new Web Service pointing at the same
   repository with **root directory** `services/ai-engine`, build from
   the Dockerfile (see `docs/architecture.md` §7) rather than
   auto-detected buildpacks — this keeps the environment identical to
   local dev and to Cloud Run if ADR-014 is later decided the other way
   — and add the ai-engine's environment variables (`SUPABASE_DB_URL`,
   `GEMINI_API_KEY`, `GROQ_API_KEY`, `AI_ENGINE_INTERNAL_API_KEY`, etc.)
   in Render's dashboard.
4. **Wire them together:** set `AI_ENGINE_URL` in Vercel to the Render
   service's public URL; set `SUPABASE_*` values in Render to match the
   Supabase project from step 1.
5. **PowerSync:** create a PowerSync Cloud project, enable logical
   replication on the Supabase Postgres instance, point PowerSync at it,
   and define sync rules scoping each table to `auth.uid()` (see
   `docs/database.md`). Add the PowerSync connection details to
   `apps/web`'s environment variables.
6. **Smoke test:** hit `GET /v1/health` on the deployed ai-engine and load
   the deployed web app before doing anything else.

Full local-machine setup (before any of the above) is in
`docs/developer-guide.md`.

## 4. Scheduled jobs (working around Vercel's 2/day cron cap)

Two categories, kept deliberately separate:

- **Derived-data jobs** (pure SQL, e.g., the daily `net_worth_snapshots`
  insert): **Supabase `pg_cron`**, a Postgres extension Supabase
  supports. Schedule SQL directly in the database — simple, no extra
  service, and this is *not* the mechanism relied on for keep-alive (see
  below).
- **Everything needing Python/ML logic**, plus the **keep-alive
  workflow**: a scheduled **GitHub Actions** workflow (`schedule:`
  trigger, cron syntax).

**The keep-alive workflow specifically** (see `decisions.md` ADR-017):
runs every 3–4 days and performs an actual **write** — an
`INSERT ... ON CONFLICT DO UPDATE` upserting a timestamp into a small
heartbeat table — against **both** Supabase and PowerSync, not a
read-only health check. This matters because community reports indicate
Supabase's inactivity detection keys off write activity specifically, and
because a `pg_cron` job cannot serve this purpose at all: if a project
has already paused, nothing runs inside it, including its own scheduled
jobs — the mechanism that prevents pausing has to come from outside the
service being kept alive.

## 5. CI/CD pipeline (illustrative reference — not yet implemented)

The actual workflow files belong in `.github/workflows/` (currently just a
placeholder README, per this repository's build directive). The following
is a reference shape for whoever implements `ci.yml`:

```yaml
# .github/workflows/ci.yml (illustrative — implement when app code exists)
name: CI
on:
  pull_request:
    branches: [main]
jobs:
  web:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm test
      - run: pnpm build

  ai-engine:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: services/ai-engine
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - run: pip install -r requirements.txt --break-system-packages
      - run: ruff check .
      - run: ruff format --check .
      - run: pytest
```

Deployment itself is handled natively by Vercel's and Render's GitHub
integrations (auto-deploy on push to `main`) rather than a custom deploy
step in Actions — simpler, and keeps deploy credentials out of GitHub
Secrets entirely.

## 6. Monitoring & observability

- Wire `SENTRY_DSN` into both `apps/web` and `services/ai-engine` for
  error tracking.
- Log structured JSON from the ai-engine (request id, user id — never
  full transaction payloads — endpoint, latency) to make Render's log
  viewer usable for debugging.
- The `GET /v1/health` endpoint (see `docs/api.md`) is useful as a basic
  uptime check and as a way to pre-warm a cold-started instance before a
  demo, but it is a read-only check and **does not** substitute for the
  write-based keep-alive workflow in §4 — the two serve different
  purposes and both are worth having.

## 7. Rollback strategy

Vercel and Render both keep prior deployments and support one-click
rollback from their dashboards — for a project this size, that is a
sufficient rollback strategy; no custom blue/green or canary tooling is
warranted.
