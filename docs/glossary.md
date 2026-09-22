# Glossary

Terms used throughout this documentation set, in one place so nobody has
to guess or re-derive them from context.

## Finance / budgeting terms

| Term | Meaning |
|---|---|
| **Zero-Based Budgeting (ZBB)** | A budgeting method where every unit of income is assigned a specific job (an envelope) at the start of the period, so income minus all allocations equals zero — nothing is left "unassigned." |
| **Envelope System** | A budgeting UI/mental model (originally literal cash envelopes) where each spending category has its own pre-allocated pool of money; you can't overspend one envelope without consciously moving money from another. |
| **Safe-to-Spend** | A real-time number representing what a user can actually spend right now without jeopardizing upcoming known bills — distinct from raw account balance. |
| **Avalanche method** | A debt payoff strategy: always put extra payments toward the debt with the **highest interest rate** first. Minimizes total interest paid. |
| **Snowball method** | A debt payoff strategy: always put extra payments toward the debt with the **smallest balance** first. Often easier to stay motivated with, due to faster "wins." |
| **Milestone Pausing** | This project's term for an AI-suggested temporary pause of a long-term goal's contributions when short-term cash flow is tight, so a bill doesn't get missed. |
| **Net worth** | Total assets minus total liabilities at a point in time. |
| **Burn rate** | The rate at which money is being spent over a period, typically shown as a trend to project how long remaining funds will last. |
| **Round-up savings** | Rounding each purchase up to the nearest currency unit and diverting the difference ("spare change") into a savings goal. |
| **Financial Wrapped** | This project's term (inspired by Spotify Wrapped) for a shareable, visual monthly/yearly summary of a user's financial habits. |
| **MCC (Merchant Category Code)** | A standardized four-digit code (ISO 18245) used by card networks to classify the type of business a merchant operates — a useful reference point (though not directly used as this project's category taxonomy) when thinking about rule-based category fallbacks. |

## Technical / architecture terms

| Term | Meaning |
|---|---|
| **RLS (Row-Level Security)** | A Postgres feature (used heavily via Supabase in this project) that restricts which rows a given database query can see/modify, enforced at the database layer regardless of what the calling application code does. See `docs/database.md`. |
| **JWT (JSON Web Token)** | A signed token used to represent an authenticated session; Supabase Auth issues these, and both the web client and `ai-engine` verify them. |
| **MAU (Monthly Active User)** | A billing/limit metric (used by Supabase's Auth pricing) counting unique users who authenticated within a rolling 30-day window. |
| **Edge Function** | A small, serverless function running close to the user geographically (used by Supabase and Cloudflare); fast to invoke but with strict execution-time/CPU limits, unsuitable for heavy ML inference. |
| **Cold start** | The latency penalty paid on the first request to a service that has been scaled down/asleep due to inactivity — a defining characteristic of most free-tier compute hosting (Render, Hugging Face Spaces). See `docs/troubleshooting.md`. |
| **Idempotency / idempotent** | A property where performing the same operation multiple times has the same effect as performing it once — critical for safely retrying offline-sync uploads without creating duplicates. |
| **Function calling (LLM)** | A pattern where a language model is constrained to output a structured, schema-validated call to a predefined "function" (here: a financial query intent) rather than free-form text — the core design of this project's NLP assistant. See `docs/ml-strategy.md`. |
| **OCR (Optical Character Recognition)** | Technology that extracts machine-readable text from an image — used here to read receipts. |
| **KIE (Key Information Extraction)** | The task of pulling specific structured fields (e.g., company name, date, total) out of raw OCR text or a document image — the SROIE benchmark referenced in `docs/ml-strategy.md` is a KIE benchmark. |
| **Isolation Forest** | An unsupervised machine learning algorithm that detects anomalies/outliers by measuring how easily a data point can be "isolated" from the rest — used here for flagging unusual transactions. |
| **Sankey diagram** | A flow diagram where the width of each arrow/band is proportional to the quantity it represents — used here to visualize money flowing from income into spending categories. |
| **Monorepo** | A single repository containing multiple distinct applications/packages (here: the web app, the ai-engine, and shared packages), managed together with shared tooling (Turborepo, pnpm workspaces). |
| **ADR (Architecture Decision Record)** | A short document capturing a single architectural decision, its context, the alternatives considered, and the consequences — this project's are in `decisions.md`. |
