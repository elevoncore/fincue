# Troubleshooting

Common problems, most of them specific to the free-tier stack chosen in
`docs/deployment.md` — easy to mistake for application bugs if you don't
know the underlying cause.

## "My deployed app suddenly can't reach the database"

**Likely cause:** the Supabase free-tier project auto-paused after 7 days
of inactivity (see `docs/deployment.md`). **PowerSync has the identical
behavior** — if sync specifically stops working while everything else is
fine, check PowerSync's dashboard too, not just Supabase's.

**Fix:** log into the relevant dashboard — a paused project shows a
"Restore" button. Restoring takes a minute or two. **Mitigation:** the
scheduled GitHub Actions keep-alive workflow (`docs/deployment.md` §4)
should already cover both services — confirm it's actually performing a
**write**, not just a read (community reports indicate read-only pings
don't reliably reset the inactivity clock), and check the workflow's own
run history in the Actions tab — a keep-alive that's been silently
failing for weeks is worse than no keep-alive, since it creates false
confidence right up until a demo.

## "The first request to the AI engine after a while takes 30–60 seconds"

**Likely cause:** this is expected behavior, not a bug — Render's (or
Hugging Face Spaces') free tier spins the service down after ~15 minutes
of inactivity and pays a cold-start cost on the next request.

**Fix (for demos):** hit `GET /v1/health` a minute or two before you need
the app to be responsive. **Fix (longer-term, still free):** a scheduled
GitHub Actions workflow pinging the health endpoint every 10–14 minutes
keeps it warm during active development/demo windows — don't run this
continuously and pointlessly 24/7, since it burns your 750 free
instance-hours faster for no benefit outside demo windows.

## "CORS errors calling the ai-engine from the web app"

**Likely cause:** FastAPI's CORS middleware isn't configured to allow the
deployed Vercel URL (which differs from `localhost:3000` and also differs
per-preview-deployment on Vercel).

**Fix:** configure `CORSMiddleware` in `app/main.py` with an explicit
allow-list (production URL + a wildcard pattern for Vercel preview URLs,
e.g. `https://fincue-*.vercel.app`), not `allow_origins=["*"]` —
a finance app's API should not accept cross-origin requests from arbitrary
sites even in development convenience mode.

## "Exchange rates look stale / the currency API call failed"

**Likely cause:** `@fawazahmed0/currency-api`'s CDN endpoint (jsDelivr) is
briefly unreachable, or a currency code isn't in that day's dataset yet.

**Fix:** the app should already be using `exchange_rates_cache` (see
`docs/database.md`) as a local cache with the previous successful rate as
a fallback, and `open.er-api.com` as a secondary provider (see
`.env.example`'s `EXCHANGE_RATE_FALLBACK_BASE_URL`) — if you're seeing
this in production, check that the fallback logic is actually wired up,
not just documented.

## "The categorizer is confidently wrong on a merchant I just added"

**Likely cause:** either the merchant genuinely wasn't in the training
data (expected — that's what the rule-based fallback and user-correction
loop in `docs/ml-strategy.md` are for), or `merchant_normalized` isn't
actually normalizing store-number suffixes ("WALMART #4471" vs. "WALMART
#2210") into the same string before embedding, fragmenting what should be
one merchant into many.

**Fix:** correct it once in the UI (this logs to `ml_feedback` for the
next retraining pass) and check the merchant-normalization regex if the
same *chain* is being miscategorized inconsistently across locations.

## "Receipt OCR extracted garbage from a real receipt"

**Likely cause:** low photo quality (blur, glare, a crumpled thermal
receipt with faded print) — OCR accuracy is fundamentally
input-quality-dependent, and this is a known, documented limitation (see
`docs/ml-strategy.md` §2), not something to "just fix" without bound.

**Fix:** the UI should always show the review/edit screen for OCR results
(never auto-save without confirmation — see `docs/api.md`'s
`requires_review` flag), and consider adding basic client-side image
quality checks (blur detection, adequate resolution) before sending to the
OCR endpoint at all, to fail fast with a "please retake this photo"
message instead of a slow round-trip to a bad result.

## "pnpm install is resolving workspace packages incorrectly"

**Likely cause:** a stale `node_modules` or lockfile after adding a new
workspace package.

**Fix:**
```bash
pnpm store prune
rm -rf node_modules apps/*/node_modules packages/*/node_modules services/*/node_modules
pnpm install
```

## "Turborepo says it has a cache hit but the output is stale/wrong"

**Likely cause:** an environment variable that affects the build output
(especially a `NEXT_PUBLIC_*` variable, which gets inlined into the
client bundle at build time) isn't declared in `turbo.json`'s `env` list
for that task, so Turborepo doesn't know to invalidate the cache when it
changes.

**Fix:** add the variable to the relevant task's `env` array in
`turbo.json` (or `globalEnv` if it affects everything), then
`turbo run build --force` once to clear the stale cache.

## "Auth works locally but not on the deployed Vercel preview URL"

**Likely cause:** Supabase Auth's allowed redirect URLs are configured for
`localhost` and your production domain, but not for Vercel's per-PR
preview URL pattern.

**Fix:** add a wildcard redirect URL pattern for your Vercel project
domain in Supabase Auth settings (Authentication → URL Configuration).

## Still stuck?

Check the specific provider's status page (`vercel-status.com`,
`status.supabase.com`, `groqstatus.com`, Render's status page) before
assuming it's your code — free-tier incidents do happen and are outside
this project's control.
