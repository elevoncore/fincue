# `services/ai-engine/datasets`

Raw datasets are **never committed** to this repository (see root
`.gitignore` — `datasets/raw/`, `*.csv`, `*.zip`). This folder instead holds
small scripts that fetch the datasets documented in
[`docs/ml-strategy.md`](../../../docs/ml-strategy.md) on demand.

## Why not commit raw data

- Kaggle's terms of use generally prohibit redistributing datasets outside
  Kaggle — committing a downloaded CSV to a public GitHub repo can violate
  that license even if the data itself is "free."
- Keeps repository size small and free-tier Git hosting happy.
- Forces the download step to be reproducible and documented (a script,
  not "I downloaded it once and it's just... there").

## Intended contents

| File (to be created) | Purpose |
|---|---|
| `download_categorization_data.py` | Pulls the Kaggle categorization datasets via the Kaggle API (requires the developer's own free Kaggle API token — never commit that token). |
| `download_sroie.py` | Fetches the SROIE receipt OCR dataset (mirrored on Hugging Face Datasets — no Kaggle auth needed). |
| `DATASETS.md` | A living record of exact dataset versions/commit hashes used, for reproducibility in the FYP report. |

See `docs/ml-strategy.md` for the specific dataset names, links, and
licensing notes.
