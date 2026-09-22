# ML Strategy & Feasibility

This is the research deliverable behind every AI/ML claim elsewhere in this
repository. For each feature: the recommended approach, why, the specific
free dataset(s) and/or model(s) to use, a realistic accuracy expectation,
and a phased rollout so the MVP doesn't depend on the hardest version of
the problem working first.

**Guiding principle:** use the simplest technique that meets the accuracy
bar, not the most impressive-sounding one. A well-evaluated classical ML
pipeline with a documented confusion matrix is worth more, both technically
and academically, than an unevaluated deep-learning pipeline that "seems to
work" on a handful of manual tests.

## 1. Merchant categorization

**Approach:** sentence embeddings + a classical classifier, not a
fine-tuned deep transformer.

1. Embed the (cleaned) merchant text using `sentence-transformers/all-MiniLM-L6-v2`
   (open source, Apache 2.0, ~80 MB, runs comfortably on CPU — no GPU
   needed for inference, which matters given the free-tier hosts in
   `docs/deployment.md` are CPU-only).
2. Train a classical classifier (Logistic Regression or XGBoost) on top of
   the embeddings to predict category.
3. Layer a **rule-based fallback** underneath: a keyword/MCC-style
   dictionary (e.g., strings containing "UBER"/"LYFT" → Transport,
   "NETFLIX"/"SPOTIFY" → Subscriptions) for merchants the model has never
   seen and is unconfident about — this also gives the categorizer
   sensible behavior on day one, before any training data exists.
4. Log every user correction to `ml_feedback` (see `docs/database.md`) and
   periodically retrain — an active-learning loop, not a train-once model.

**Why not a fine-tuned transformer (e.g., DistilBERT) end-to-end:** it's a
reasonable Phase 2 upgrade, but for short, templated merchant strings
("UBER *TRIP", "SQ *COFFEE SHOP"), embeddings + a classical head already
capture most of the useful signal, trains in seconds instead of GPU-hours,
and is trivial to host on a free CPU instance. Don't reach for a bigger
model than the problem needs.

**Evidence this approach works:** a 2026 study on automatic classification
of personal expenses from unstructured transaction text found that
semantic sentence-embedding representations clearly outperformed TF-IDF
bag-of-words features, and reported **93.6% accuracy** using embeddings
with a supervised XGBoost classifier on short financial transaction
text — a strong, recent, directly-relevant benchmark to cite as validation
of this approach and to compare your own results against.
(Source: *"A Web-Based Intelligent System for Automatic Classification of
Personal Expenses,"* International Journal of Engineering and Information
Technology, 2026 — search for the paper title if the direct link changes.)
A similarly-scoped open-source capstone project, `fin-classifier` on
Hugging Face, took a DistilBERT fine-tuning approach to the same problem
and is worth reading as a second reference point on dataset structure and
active-learning design, even though this project recommends the lighter
embeddings-based approach for a first version.

**Datasets (free, Kaggle):**

| Dataset | What it gives you | Link |
|---|---|---|
| "Customer Transaction Dataset" (bkcoban) | Merchant name + category pairs — the most directly useful for training merchant → category classification. | `kaggle.com/datasets/bkcoban/customer-transactions` |
| "My Expenses Data" (Tharun Prabu) | Realistic personal transaction notes + category/subcategory labels, income vs. expense flag. | Search "My Expenses Data Tharun Prabu" on Kaggle |
| "Personal Budget Transactions Dataset" (ismetsemedov) | Smaller, clean date/category/amount dataset — good for the forecasting/behavioral side, not just categorization. | Search on Kaggle by name |

**Note on Kaggle's terms:** don't commit these datasets to the repository
(see `services/ai-engine/datasets/README.md`) — fetch them at
training time via the Kaggle API using your own free Kaggle account token.

**Evaluation target:** report precision/recall/F1 per category (not just
overall accuracy — category imbalance is real, e.g. far more "Groceries"
examples than "Insurance") and a confusion matrix, on a held-out split.
Treat anything above ~80% overall accuracy on your own labeled/derived
data as a credible MVP result, and cite the 93.6% figure above as the
ceiling a more mature version of this pipeline has achieved elsewhere —
not as a number you should expect to match immediately with a smaller,
differently-sourced dataset.

## 2. Receipt OCR & key-information extraction

**Approach:** PaddleOCR (open source, Apache 2.0) for text
localization + recognition, running server-side in `services/ai-engine`,
followed by a small rules/regex layer to pull out company name, date, and
total from the raw OCR output (the same structured-extraction problem the
SROIE benchmark below evaluates).

**Why PaddleOCR over Tesseract.js:** general benchmarking of open-source
OCR engines consistently shows PaddleOCR handling varied real-world
layouts (including receipts, which have inconsistent fonts, thermal-print
noise, and non-uniform layouts) more robustly than Tesseract. Since OCR
happens server-side anyway (not constrained to what runs in a browser),
there's no deployment reason to prefer the JS port. Keep **Tesseract.js**
in mind as a documented fallback for a fully offline/client-side mode
(receipt never leaves the device) if that becomes a priority later — it's
a real privacy advantage, just a lower-accuracy one.

**Why not Gemini Flash multimodal instead (evaluated, not adopted, for
now — see `decisions.md` ADR-015):** sending the receipt image directly
to a multimodal LLM and asking it to return structured fields is a real
alternative, and would remove PaddleOCR's hosting footprint entirely. It
was not adopted for two reasons, in order of importance: first, it
conflicts with the "the LLM never receives raw ledger data" boundary in
`docs/security.md` §5 — a receipt photo can carry more sensitive detail
(addresses, specific pharmacy or health-related purchases) than the
question-text-only pattern the NLP assistant uses, and Gemini's free
tier permits using submitted content, images included, to improve
Google's products. Second, no rigorous benchmark was found supporting
"much more accurate" as a settled fact for this specific comparison —
it's a plausible claim given how strong multimodal models generally are
at document understanding, but not yet an evidenced one. If revisited,
it should be evaluated against the same SROIE benchmark below before any
decision, and the privacy trade-off disclosed to users regardless of the
outcome.

**Dataset for evaluation (and optional fine-tuning of the extraction
layer):** SROIE — the ICDAR 2019 "Scanned Receipts OCR and Information
Extraction" dataset. 1,000 annotated scanned receipt images with text
bounding boxes, transcripts, and key-field metadata (company, address,
date, total) — exactly the four fields this feature needs, with an
established evaluation protocol (exact-match accuracy per field). Mirrored
on Hugging Face Datasets (`Voxel51/scanned_receipts`), which avoids the
original competition site's registration friction.

**Phased rollout:**
1. **MVP:** PaddleOCR raw text + a regex/heuristic layer (e.g., "the line
   matching `TOTAL|AMOUNT DUE` followed by a currency-formatted number" for
   totals; the first non-empty line for merchant name; a date-pattern
   regex for the date). Evaluate this heuristic layer against SROIE's
   ground truth to get a real accuracy number, not just "it looked right
   on my test receipts."
2. **Phase 2:** if the heuristic layer's SROIE accuracy is unsatisfying,
   train a lightweight key-information-extraction model (e.g., a small
   token-classification model) using SROIE's own training split — this is
   a well-trodden research problem (see the "FUNSD"/"Kleister" family of
   related benchmarks if you want to read further) and a good place to
   spend extra engineering time if the timeline allows, since it's more
   novel/impressive for a report than the regex layer.

## 3. Anomaly detection (duplicates, price creep)

**Approach:** classical, unsupervised, per-user statistical methods — no
external dataset needed, because this model trains on each user's own
transaction history, not a shared corpus.

- **Duplicate charges:** flag two transactions from the same
  normalized merchant, same amount, within a short time window (e.g., a
  few minutes to a few hours) — a rule, not a model, and a good one:
  duplicates are rare and structurally obvious, so a simple rule
  outperforms an ML model here on both accuracy and explainability.
- **Unusual transactions:** scikit-learn's `IsolationForest` (or, for an
  even simpler and more explainable MVP, a rolling z-score per merchant/
  category) flags transactions that are statistical outliers relative to
  a user's own history — e.g., a $340 restaurant charge when their
  typical dining transaction is $15–40.
- **Subscription price creep:** compare the latest entry in a
  subscription's `price_history` (see `docs/database.md`) against the
  previous one — again a rule, not a model, since "the price changed" is
  a deterministic fact, not something requiring statistical inference.

**Why not a shared/pretrained anomaly model:** anomaly, by definition, is
relative to an individual's own baseline spending — a model trained on
population-wide data would flag "unusual for the average person," not
"unusual for you," which is the wrong question. Per-user, mostly-rule-based
detection is not a simpler cop-out here; it is the *correct* approach.

## 4. Overspending prediction & cash-flow forecasting

**Approach:** simple statistical methods first — a rolling average of
daily spend by category compared against the remaining budget and days
left in the period is enough to power both "predictive overspending
alerts" and a first-pass cash-flow forecast. A basic linear regression on
cumulative spend-vs-day-of-month can generate the "safe-to-spend"
projection in `docs/api.md`.

**Phase 2 enhancement:** if time allows, compare this baseline against
Meta's open-source **Prophet** library (handles seasonality — e.g., higher
spend around specific weekdays or a monthly payday) purely as an
A/B comparison exercise; don't adopt it by default; the added complexity
should earn its place with a measured improvement over the simple
baseline, not be assumed.

**No external dataset needed** — like anomaly detection, this is inherently
per-user and trains/computes on the user's own accumulating transaction
history.

## 5. Natural-language assistant

**Approach: function-calling / intent-parsing, not open-ended chat, and
not text-to-SQL.** This is as much a security design as an ML one — see
`docs/security.md` §5 and `docs/architecture.md` §4.2.

1. The user's question text is sent to an LLM with a constrained
   function-calling schema (a fixed list of supported intents:
   `sum_by_category`, `compare_periods`, `goal_progress`,
   `budget_remaining`, etc., each with typed parameters like
   `category`, `date_range`).
2. The LLM's *only* job is mapping natural language to one of these
   structured intents — it never receives, and does not need, any actual
   financial figures.
3. The `ai-engine` executes a **fixed, parameterized query** for that
   intent type against Postgres and computes the real answer.
4. A short natural-language response is generated by inserting the real
   number into a template (or, optionally, a second small LLM call that
   only sees the already-computed result, for a more natural sentence —
   still never touching raw ledger data beyond the single aggregated
   number being reported).

**Why not text-to-SQL:** letting an LLM generate arbitrary SQL against a
financial database is an injection and data-exfiltration risk that a
fixed, reviewable intent set avoids entirely, at the cost of only
supporting a bounded set of question types — an acceptable, honestly
disclosed trade-off for an MVP (see `docs/PRD.md`, "narrow scope").

**LLM provider choice (free tier):**

| Provider | Why | Rough free limits (verify before relying — see `docs/deployment.md`) |
|---|---|---|
| **Gemini API** (Google AI Studio) — primary | No card required; generous daily quota for a single-user-at-a-time demo workload. | ~1,500 req/day (Gemini 2.0 Flash) / ~500 req/day (2.5 Flash), ~15 RPM |
| **Groq** — fallback | No card required; extremely low latency (custom LPU hardware), good for a snappy live demo. | ~30 req/min, daily token cap, varies by model (Llama 3.3 70B and similar open models) |

Both are called with the *same* small, fixed function-calling prompt —
swapping providers should be a one-line config change in
`services/ai-engine/app/clients/`, not a rewrite.

## 6. Datasets & models — master reference table

| Purpose | Resource | Type | License/Access |
|---|---|---|---|
| Categorization training data | Kaggle: "Customer Transaction Dataset" (bkcoban) | Dataset | Kaggle terms — fetch via API, don't redistribute |
| Categorization training data | Kaggle: "My Expenses Data" (Tharun Prabu) | Dataset | Kaggle terms |
| Categorization training data | Kaggle: "Personal Budget Transactions Dataset" (ismetsemedov) | Dataset | Kaggle terms |
| Categorization embeddings | `sentence-transformers/all-MiniLM-L6-v2` | Model | Open source (Apache 2.0), via Hugging Face |
| Categorization validation benchmark | *"Web-Based Intelligent System for Automatic Classification of Personal Expenses"* (2026) | Academic paper | Public — cite, don't reproduce full text |
| Categorization prior-art reference | `CodeBlooded-capstone/fin-classifier` on Hugging Face | Model + writeup | Apache 2.0 |
| Receipt OCR engine | PaddleOCR | Open-source library | Apache 2.0 |
| Receipt OCR evaluation | SROIE (ICDAR 2019), mirrored as `Voxel51/scanned_receipts` | Dataset | Research/competition dataset — check current redistribution terms on the HF mirror before any commercial use |
| Anomaly detection | scikit-learn `IsolationForest` | Library | Open source (BSD) |
| Forecasting baseline | NumPy/statsmodels linear regression | Library | Open source |
| Forecasting Phase-2 option | Meta's Prophet | Library | Open source (MIT) |
| NLP assistant | Gemini API (Google AI Studio) | Hosted API, free tier | Google's terms — see privacy note below |
| NLP assistant fallback | Groq API | Hosted API, free tier | Groq's terms |
| Currency conversion | `@fawazahmed0/currency-api` | Hosted API, free, no key, unlimited | Open source, CDN-hosted (jsDelivr + Cloudflare Pages fallback) |
| Currency conversion fallback | `open.er-api.com` (ExchangeRate-API's Open Access endpoint) | Hosted API, free, no key | Requires attribution |
| Investment/account sync (Phase 2) | Plaid Sandbox (mock data) / Trial plan (up to 10 real Production Items, US/CA, since Apr 2026) | Hosted API | Plaid's terms — see `docs/PRD.md` non-goals |

## 7. Privacy considerations (see also `docs/security.md` §5)

- Only the NLP assistant's question text ever reaches a third-party LLM —
  no transaction amounts, merchant names tied to real activity, or account
  identifiers. This is a hard design constraint, not an optimization.
- Free-tier LLM terms sometimes permit the provider to use submitted
  content to improve their models — treat this as a live constraint on
  what's acceptable to send, not just a line item to disclose.
- Kaggle training datasets used for categorization are synthetic/aggregate
  or already-published public datasets, not this project's own users'
  data — never fold real user corrections (`ml_feedback`) into a dataset
  that leaves the project's own database, even for research purposes,
  without explicit consent flow.

## 8. Training/retraining pipeline (once implemented)

1. `services/ai-engine/datasets/download_*.py` fetches the source
   datasets.
2. `services/ai-engine/notebooks/02_train_categorizer.ipynb` trains and
   evaluates, exporting the model to `services/ai-engine/models/`
   (git-ignored — see that folder's README).
3. Periodically (suggested: monthly, or after N new `ml_feedback` rows
   accumulate), re-run the notebook incorporating corrections, and
   re-deploy the updated model artifact alongside the next `ai-engine`
   deploy.
4. Track the model's version string (e.g., `categorizer-v0.2`) in
   responses (`docs/api.md`) so accuracy regressions can be traced to a
   specific retraining event.
