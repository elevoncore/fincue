# Security

This document covers the platform's security **architecture and posture**.
To report an actual vulnerability, see the root [`SECURITY.md`](../SECURITY.md)
instead.

## 1. Compliance disclaimer (read this first)

Fincue is a **student Final Year Project**, not a regulated
financial institution. It does not hold funds, move money, or provide
financial advice in a regulated sense. It should not be described, in the
FYP report or otherwise, as PCI-DSS compliant, a licensed money
transmitter, or a registered investment adviser — none of that is in
scope, and claiming it would be inaccurate. What *is* in scope, and what
this document covers, is applying reasonable, industry-standard security
practices to a project that handles personal financial data, because
that's good engineering regardless of regulatory status — and because
demonstrating that judgment is itself part of the FYP evaluation.

## 2. Threat model (summary)

| Threat | Primary mitigation |
|---|---|
| Attacker gains a valid session and reads another user's data | Postgres Row-Level Security (RLS) — enforced at the database layer, not just in application code. See `docs/database.md` §4. |
| Stolen device with an unlocked browser/app session | Biometric re-auth (WebAuthn/passkeys on web, platform biometrics on mobile) + session auto-timeout. See §4. |
| Secrets leaked via committed `.env` files or client bundles | `.gitignore` excludes all `.env*` except `.env.example`; `SUPABASE_SERVICE_ROLE_KEY` and all LLM/Plaid keys are server-side only, never `NEXT_PUBLIC_*`. See §6. |
| Sensitive financial data sent to a third-party LLM and retained/used for training | Function-calling design constraint: the LLM only ever receives question text and returns structured intent — never balances, transaction amounts, or account identifiers. See §5. |
| SQL injection / arbitrary query execution | The NLP assistant never generates free-form SQL; it returns a constrained structured intent that the ai-engine maps to a fixed set of parameterized queries. See `docs/architecture.md` §4.2. |
| Malicious or malformed receipt image (e.g., a decompression bomb, or an attempt to exploit the OCR library) | File-type/size validation before processing (`docs/api.md` — 10 MB cap, allow-listed MIME types); OCR runs in the isolated `ai-engine` service, not in a process with direct database credentials. |
| Compromised dependency (supply-chain attack) | Dependabot/`npm audit`/`pip-audit` in CI (see `docs/deployment.md`); pinned lockfiles committed. |

## 3. Authentication & session management

- **Identity provider:** Supabase Auth (email/password + optional OAuth
  providers). JWTs are short-lived; Supabase's SDK handles silent refresh.
- **Session auto-timeout:** the web client enforces an inactivity timeout
  (suggested default: 15 minutes) that clears the in-memory session and
  requires re-authentication — implemented client-side via an idle-timer
  hook, since Supabase JWT expiry alone is usually longer than is
  appropriate for a finance app left open on a shared machine.
- **Biometric app-lock ("locked behind biometrics even if the phone is
  unlocked"):** on web, implemented via the **WebAuthn** API using
  platform authenticators (Face ID/Touch ID/Windows Hello) as a
  re-authentication gate layered on top of the Supabase session — the
  Supabase session proves *who you are*, WebAuthn proves *you're still
  physically present* before revealing the app. On mobile (Phase 10+),
  `expo-local-authentication` provides the equivalent gate against the
  device's biometric APIs. Neither implementation should treat biometric
  success as a substitute for the underlying Supabase session — it's an
  additional local gate, not a replacement identity check.

## 4. Authorization

Row-Level Security (see `docs/database.md` §4) is the primary
authorization boundary for anything the web/mobile client touches
directly. The `services/ai-engine` backend, which connects with the
Postgres **service role** (bypassing RLS, because it needs cross-user
access for scheduled jobs like anomaly scanning), must therefore enforce
`user_id` scoping in its own application code for every request path.
**This is the single most important code-review checkpoint in the whole
project** — a missing `WHERE user_id = :current_user` clause in the
ai-engine is a direct cross-user data leak that RLS does not catch,
because RLS is bypassed by design for that service. Every new ai-engine
endpoint's PR review should explicitly check this.

## 5. Third-party AI data-sharing risk (read carefully)

This is a genuinely easy mistake to make and worth stating plainly: **free
tiers of hosted LLM APIs often permit the provider to log and/or use
submitted prompts to improve their products**, in a way that a paid
enterprise tier typically does not. For a personal finance app, sending
raw transaction data, balances, or account numbers to such an endpoint
would be a real privacy problem, independent of whether any breach ever
occurs.

Design constraint (already reflected in `docs/architecture.md` and
`docs/api.md`): **the LLM used for the NLP assistant only ever sees the
user's question text**, and returns a structured intent (category,
date range, aggregation type). It never receives — and does not need —
account balances, merchant names tied to real transactions, or any PII.
The actual numeric answer is computed by a direct, parameterized database
query after the LLM's involvement ends. This is documented in detail in
`docs/ml-strategy.md`.

Practical checklist for anyone extending the assistant:

- If a new feature seems to require sending transaction-level data to an
  LLM, treat that as a design smell and look for a way to keep the LLM's
  role limited to intent parsing instead.
- Disclose this data flow to users in plain language (e.g., in
  `docs/user-guide.md` / the in-app privacy notice): "your questions may
  be processed by a third-party AI service; your financial data itself is
  not."
- Re-verify each provider's current data-use terms before relying on this
  characterization — terms change (see the citations in
  `docs/ml-strategy.md`) and free tiers are the most likely tier to change
  unfavorably.

## 6. Secrets management

- Real secrets live only in `.env.local` (web), `.env` (ai-engine), and
  each hosting provider's dashboard (Vercel/Render environment variables) —
  never in the repository. `.env.example` is the only committed env file,
  and contains no real values.
- `SUPABASE_SERVICE_ROLE_KEY`, `GEMINI_API_KEY`, `GROQ_API_KEY`,
  `PLAID_SECRET`, and `AI_ENGINE_INTERNAL_API_KEY` must never be prefixed
  `NEXT_PUBLIC_` or otherwise exposed to a client bundle — only
  `NEXT_PUBLIC_SUPABASE_URL` and `NEXT_PUBLIC_SUPABASE_ANON_KEY` are safe
  for the browser (the anon key is meant to be public; RLS is what makes
  that safe).
- Before every push, a pre-commit hook or CI secret-scanning step
  (e.g., `gitleaks`) should run — see `docs/deployment.md`'s CI section.

## 7. Encryption

- **In transit:** TLS is provided end-to-end by the hosting platforms
  (Vercel, Supabase, Render all terminate HTTPS by default) — no custom
  TLS handling is needed or should be built.
- **At rest:** Supabase's underlying Postgres storage is encrypted at
  rest by the platform. For an additional layer on the *most* sensitive
  fields (e.g., a linked bank account number if Plaid/Phase 2 is
  implemented), consider column-level encryption via Postgres's
  `pgcrypto` extension, decrypting only in the `ai-engine` service that
  holds the key — never decrypting such fields client-side.
- **Backups:** the free Supabase tier does not include automated daily
  backups (that's a paid-tier feature — see `docs/deployment.md`); the
  data-portability export feature (§8) doubles as a manual backup
  mechanism during the free-tier phase of the project.

## 8. Data portability & deletion

The brief specifies one-click export of a user's full financial history.
Implement this as a Supabase Edge Function or an `ai-engine` endpoint that
assembles a JSON (or CSV bundle) of every table scoped to `auth.uid()`,
gated behind re-authentication (exporting your entire financial history is
exactly the kind of action worth an extra confirmation step). Provide an
equivalent full-account-deletion path for the same reason — even though
this is a class project rather than a GDPR-regulated production service,
building both is good practice and a reasonable thing to highlight in the
FYP report.

## 9. Dependency & CI security

- Enable Dependabot (or `renovate`) for automated dependency update PRs.
- Run `pnpm audit` / `pip-audit` in CI (see `docs/deployment.md`) and treat
  new high/critical findings as blocking for `main`.
- Pin lockfiles (`pnpm-lock.yaml`, a `requirements.txt`/`poetry.lock` for
  the ai-engine) and commit them — reproducible installs are themselves a
  security property.

## 10. Incident response (lightweight, appropriate to project scale)

If a credential leak or data-exposure bug is discovered:

1. Rotate the affected credential immediately (Supabase service role key,
   any LLM/Plaid API key) via the provider's dashboard.
2. If user data (even synthetic demo data) may have been exposed, note
   the timeline and scope in `decisions.md` for transparency in the FYP
   report — treat "we found and fixed X" as a legitimate, even valuable,
   thing to document rather than something to hide.
3. Force-expire active sessions if the exposure could affect live
   sessions (Supabase Auth supports this from the dashboard).
