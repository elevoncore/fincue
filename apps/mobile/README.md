# `apps/mobile`

The secondary mobile client — **Phase 10+, not started**. See
[`roadmap.md`](../../roadmap.md) (local, git-ignored) for sequencing; mobile
intentionally comes after the web app and `ai-engine` are stable, so it can
reuse a proven API contract instead of co-evolving with it.

## What goes here (when it starts)

An [Expo](https://expo.dev/) (React Native) application that:

- Shares business-logic types with `apps/web` via `packages/shared-types`,
  and ideally some cross-platform UI primitives via `packages/ui` (using
  Tailwind-compatible tooling like NativeWind, evaluated at Phase 10).
- Talks to the **same** Supabase project and the **same** `services/ai-engine`
  API as the web app — this is the entire point of the decoupled-backend
  architecture described in `docs/architecture.md`.
- Uses `expo-local-authentication` for biometric app-lock (Face
  ID/fingerprint), mirroring the WebAuthn flow on web (`docs/security.md`).
- Uses Firebase Cloud Messaging for push notifications (budget alerts,
  subscription price-creep warnings, goal milestones).

## Why this isn't scaffolded yet

Scaffolding a mobile app before the API contract stabilizes usually means
re-doing the mobile data layer at least once. `docs/roadmap.md` sequences
mobile after `docs/api.md`'s contract has been implemented and used by the
web client for a full phase, to avoid that churn.
