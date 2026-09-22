# `apps/web`

The primary web client — **not yet scaffolded**. Per this repository's build
directive, no frontend code has been generated; this folder is a placeholder
for wherever development starts next.

## What goes here

A Next.js 15+ (App Router) application that:

- Talks directly to Supabase (via `@supabase/supabase-js`) for standard
  CRUD on transactions, budgets, goals, and debts, secured by Row-Level
  Security (see `docs/database.md`).
- Calls the decoupled `services/ai-engine` HTTP API for anything
  AI/ML-related: receipt OCR, categorization, the NLP assistant, anomaly
  alerts, and forecasts (see `docs/api.md`).
- Consumes shared types from `packages/shared-types` and (once built)
  shared UI primitives from `packages/ui`.

## How to scaffold it

See [`docs/developer-guide.md`](../../docs/developer-guide.md) for the exact
command (`pnpm create next-app`) and the required config (Tailwind, the
Supabase client, environment variables). Once scaffolded, this README should
be replaced with the app-specific one Next.js generates, extended with any
project-specific setup notes.

## Deploys to

Vercel (Hobby/free tier) — see [`docs/deployment.md`](../../docs/deployment.md).
