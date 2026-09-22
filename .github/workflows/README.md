# `.github/workflows`

Intentionally empty of actual workflow files for now. This repository's
build directive scoped this deliverable to structure, configuration, and
documentation — CI/CD pipelines are documented in detail (including an
illustrative example workflow) in
[`docs/deployment.md`](../../docs/deployment.md#cicd-pipeline) so that
whoever implements them has an exact spec to work from, rather than
committing untested YAML now.

## Planned workflows (see `docs/deployment.md` for full detail)

| File (to be added) | Trigger | Purpose |
|---|---|---|
| `ci.yml` | Pull request to `main` | Install, lint, type-check, test `apps/web` and `services/ai-engine` (`ruff check` + `ruff format --check` + `pytest` for the latter — see `docs/deployment.md` §5). |
| `deploy-web.yml` | Push to `main` (or handled natively by Vercel's Git integration) | Deploy `apps/web` to Vercel. |
| `deploy-ai-engine.yml` | Push to `main` affecting `services/ai-engine/**` | Build the Dockerfile (see `docs/architecture.md` §7) and deploy the ai-engine container to Render (or the chosen host, once `decisions.md` ADR-014 is settled). |
| `keep-alive.yml` | Cron schedule, every 3–4 days | Upserts a heartbeat row (a real write, not a read) against **both** Supabase and PowerSync — see `docs/deployment.md` §4 and `decisions.md` ADR-017. Kept separate from `scheduled-jobs.yml` because it's load-bearing for the whole stack staying reachable, not just a feature. |
| `scheduled-jobs.yml` | Cron schedule | Runs product-logic jobs that need Python rather than pure SQL — e.g. subscription price-creep detection, anomaly scans. |

Once implemented, this README should be replaced by the workflows
themselves; GitHub does not require a README in this folder.
