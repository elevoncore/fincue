# `services/ai-engine`

The **decoupled backend** referenced throughout `docs/`: a Python + FastAPI
service that owns every AI/ML feature and is called over HTTPS by both
`apps/web` today and `apps/mobile` later. Not yet implemented — this folder
documents its intended shape so implementation has a clear target.

## Why a separate service (not Next.js API routes)

1. **Language fit** — the ML stack (scikit-learn, sentence-transformers,
   PaddleOCR) is Python-first; forcing it into Vercel's Node/Edge functions
   would mean fighting the platform.
2. **Independent scaling & hosting** — OCR and embedding inference are
   heavier and slower than typical CRUD requests. Isolating them means a
   slow ML call can't starve the web app's serverless function pool, and
   it can be hosted on infrastructure suited to persistent compute (Render)
   instead of short-lived edge functions.
3. **True platform decoupling** — both the web app and the future mobile
   app call the same `ai-engine` API, so AI logic is implemented exactly
   once. See `docs/architecture.md`.

Full reasoning, including rejected alternatives, is in `decisions.md`
(ADR-004, local/git-ignored).

## Subfolders

| Folder | Purpose |
|---|---|
| `app/` | The FastAPI application itself (routers, services, schemas) — not yet created. |
| `models/` | Trained model artifacts (`.pkl`/`.onnx`). Git-ignored — see that folder's README for how to regenerate them. |
| `notebooks/` | Jupyter notebooks for data exploration, training, and evaluation. |
| `datasets/` | Scripts/instructions to fetch the free datasets used for training — never the raw data itself. |

A `Dockerfile` lives at this folder's root (not yet created, alongside
`app/`) — this service is containerized, unlike `apps/web`. See
`docs/architecture.md` §7 for why, and `decisions.md` ADR-018.

## Endpoints (planned — see `docs/api.md` for full contracts)

- `POST /v1/ocr/receipt` — image in, structured merchant/date/total/line-items out.
- `POST /v1/categorize` — merchant text in, predicted category + confidence out.
- `POST /v1/assistant/query` — natural-language question in, structured query intent out (never raw ledger data — see `docs/ml-strategy.md`).
- `POST /v1/anomalies/scan` — run anomaly detection over a user's recent transactions.
- `GET  /v1/forecast/cashflow` — cash-flow / safe-to-spend projection.

## Hosting

**Not yet finalized** — see `decisions.md` ADR-014. Render (free Web
Service) is the default for now, with Google Cloud Run's Always Free tier
under active evaluation and Hugging Face Spaces documented as a further
alternative. The Dockerfile above works for any of the three, which is
deliberate — see `docs/deployment.md` for the full comparison and current
caveats.
