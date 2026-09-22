# API Design

Fincue has **two** API surfaces, and it matters which one a given
feature uses:

1. **Supabase's auto-generated PostgREST API** — used for standard CRUD
   (transactions, budgets, goals, debts, accounts, subscriptions), secured
   by the RLS policies in `docs/database.md`. The web/mobile clients call
   this directly via the `@supabase/supabase-js` SDK; there is deliberately
   **no custom CRUD API to build or maintain** for these resources.
2. **The `services/ai-engine` REST API** — a small, hand-written FastAPI
   surface for everything that needs server-side compute: OCR, ML
   categorization, anomaly detection, forecasting, and the NLP assistant.
   This is the API documented in detail below.

## 1. Conventions

- Base URL: `{AI_ENGINE_URL}/v1` (see `.env.example`).
- Auth: every request includes `Authorization: Bearer <supabase_jwt>` (the
  same JWT the client already holds from Supabase Auth) **and**
  `X-Internal-Key: <AI_ENGINE_INTERNAL_API_KEY>`. The ai-engine verifies the
  Supabase JWT to identify the user (so it can scope queries correctly even
  though it connects to Postgres with the service role) and the internal
  key to ensure only `apps/web`/`apps/mobile` — not the public internet —
  can call it. See `docs/security.md`.
- Content type: `application/json` for all requests/responses except the
  receipt-OCR endpoint, which accepts `multipart/form-data`.
- Errors follow a consistent envelope:

```json
{
  "error": {
    "code": "invalid_image",
    "message": "The uploaded file could not be read as an image.",
    "request_id": "b3f1..."
  }
}
```

- Standard HTTP status codes: `400` validation, `401` missing/invalid
  auth, `403` valid auth but not permitted, `404` not found, `422`
  well-formed but semantically invalid (e.g. a date range in the future
  for a historical query), `429` rate limited (see below), `500` unexpected.
- **Rate limiting:** since the underlying LLM/OCR calls have their own
  free-tier caps (see `docs/ml-strategy.md`), the ai-engine enforces a
  conservative per-user rate limit on `/assistant/query` and
  `/ocr/receipt` (suggested starting point: 30 requests/hour/user) and
  returns `429` with a `Retry-After` header before the upstream provider
  would reject the call — better to fail predictably in our own API than
  to surface an opaque upstream 429.

## 2. Endpoints

### `POST /v1/ocr/receipt`

Extracts merchant, date, total, and line items from a photographed
receipt.

**Request** (`multipart/form-data`):
| Field | Type | Notes |
|---|---|---|
| `image` | file | JPEG/PNG/HEIC, max 10 MB |
| `account_id` | string (uuid) | which account this will post to, for currency context |

**Response `200`:**
```json
{
  "merchant_raw_text": "SHNG STAR MART #4471",
  "merchant_normalized": "Shing Star Mart",
  "txn_date": "2026-09-10",
  "total": 34.52,
  "currency": "USD",
  "confidence": 0.88,
  "line_items": [
    { "description": "2% Milk 1gal", "amount": 4.29, "suggested_category": "Groceries", "category_confidence": 0.94 },
    { "description": "Paper Towels", "amount": 6.99, "suggested_category": "Household", "category_confidence": 0.81 }
  ],
  "requires_review": true
}
```
`requires_review` is `true` whenever any field's confidence is below a
configurable threshold — the frontend should always show the review screen
in that case rather than silently trusting a low-confidence extraction.

### `POST /v1/categorize`

Categorizes a single merchant string (used for manual entry, and
internally by the OCR endpoint above for each line item).

**Request:**
```json
{ "merchant_text": "UBER *TRIP HELP.UBER.COM", "amount": 12.40 }
```
**Response `200`:**
```json
{
  "category": "Transport",
  "subcategory": "Rideshare",
  "confidence": 0.92,
  "model_version": "categorizer-v0.1"
}
```

### `POST /v1/categorize/feedback`

Records a user's correction for retraining (writes to `ml_feedback`, see
`docs/database.md`).
```json
{ "transaction_id": "uuid", "predicted_category_id": "uuid", "corrected_category_id": "uuid" }
```
Response: `204 No Content`.

### `POST /v1/assistant/query`

The NLP assistant. See `docs/ml-strategy.md` and
`docs/architecture.md` §4.2 for the "LLM never sees raw ledger data"
design constraint this endpoint implements.

**Request:**
```json
{ "question": "How much did I spend on dining out last weekend?" }
```
**Response `200`:**
```json
{
  "answer": "You spent $86.40 on Dining Out last weekend (Sat–Sun, Sep 6–7).",
  "intent": {
    "type": "sum_by_category",
    "category": "Dining Out",
    "date_range": { "start": "2026-09-06", "end": "2026-09-07" }
  },
  "supporting_transaction_ids": ["uuid1", "uuid2", "uuid3"]
}
```
**Response `422`** (question doesn't match a supported intent — see
`docs/ml-strategy.md` for why the assistant intentionally supports a
bounded set of intents rather than open-ended chat):
```json
{ "error": { "code": "unsupported_intent", "message": "I can answer spending, budget, and goal-progress questions, but not that one yet." } }
```

### `POST /v1/anomalies/scan`

Triggers (or, via the scheduled job described in `docs/deployment.md`,
is triggered on a schedule for) an anomaly pass over a user's recent
transactions.

**Response `200`:**
```json
{
  "anomalies": [
    {
      "type": "duplicate_charge",
      "transaction_ids": ["uuid1", "uuid2"],
      "detail": "Two $45.00 charges from 'Netflix' within 3 minutes.",
      "severity": "medium"
    },
    {
      "type": "subscription_price_increase",
      "subscription_id": "uuid",
      "detail": "Spotify increased from $9.99 to $12.99 this cycle.",
      "severity": "low"
    }
  ]
}
```

### `GET /v1/forecast/cashflow?horizon_days=30`

**Response `200`:**
```json
{
  "safe_to_spend_today": 214.30,
  "projected_balance": [
    { "date": "2026-09-13", "balance": 1820.10 },
    { "date": "2026-09-14", "balance": 1795.44 }
  ],
  "upcoming_known_bills": [
    { "name": "Rent", "amount": 650.00, "due_date": "2026-10-01" }
  ]
}
```

### `GET /v1/health`

Unauthenticated liveness check for the scheduled keep-alive ping described
in `docs/deployment.md`.
```json
{ "status": "ok", "version": "0.1.0" }
```

## 3. Versioning

The `/v1` prefix is the whole strategy for now — breaking changes get a
`/v2`, additive changes (new optional fields) do not bump the version. This
is deliberately simple; a project this size does not need a formal
deprecation policy, but starting with a version prefix costs nothing and
avoids an awkward retrofit later.

## 4. Why not GraphQL

Considered and rejected: the ai-engine's endpoints are few, purpose-built,
and don't benefit much from GraphQL's client-specified-shape flexibility;
and the CRUD surface is already handled by Supabase's PostgREST API (which
does offer resource embedding/filtering via query parameters, covering most
of what would motivate GraphQL). Revisit only if the client ends up making
many round trips to assemble one screen — see `decisions.md`.
