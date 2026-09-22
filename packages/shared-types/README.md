# `packages/shared-types`

TypeScript types/interfaces shared between `apps/web` and (later)
`apps/mobile` — not yet created.

## What goes here

Types that mirror the Postgres schema in `docs/database.md` and the API
contracts in `docs/api.md`, so the frontend and the `ai-engine` API can
never silently drift apart:

```text
shared-types/
├── src/
│   ├── entities/        # Transaction, Account, Budget, Envelope, Goal, Debt, Subscription, ...
│   ├── api/              # Request/response shapes for each ai-engine endpoint
│   └── index.ts
└── package.json
```

## Recommended approach

Rather than hand-writing these twice (once in Postgres, once in
TypeScript), generate the base entity types from the Supabase schema using
`supabase gen types typescript`, then hand-author the `api/` request/response
types (since those describe HTTP contracts, not tables). Document the
generation command in `docs/developer-guide.md` once adopted.

## Consumed by

`apps/web`, `apps/mobile` (Phase 10+). Not consumed by `services/ai-engine`
directly, since that's a separate Python codebase — its request/response
shapes are defined independently as Pydantic models and should be kept in
sync manually with `docs/api.md` (a future improvement: generate an OpenAPI
schema from FastAPI and generate these TS types from it instead).
