# `scripts`

One-off setup and maintenance scripts that don't belong inside a specific
app/package/service. Empty for now. Likely future additions:

- `setup-supabase.sh` — walks through creating the Supabase project and
  running the initial migration (see `docs/database.md`).
- `seed-demo-data.ts` — populates a fresh database with realistic-looking
  **synthetic** transactions/budgets/goals for demos and screenshots —
  never real financial data (see `docs/security.md` and
  `docs/testing-strategy.md` on test data hygiene).
- `check-free-tier-usage.ts` — a small script (run manually or via a
  scheduled GitHub Action) that pings the Supabase and `ai-engine` health
  endpoints, partly to keep the Supabase free-tier project from pausing
  after 7 days of inactivity (see `docs/deployment.md`).
