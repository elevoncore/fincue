# `services/ai-engine/models`

Trained model artifacts live here at runtime — **this folder is intentionally
empty in version control** (see root `.gitignore`). Binary model files
bloat repository size and become stale/undocumented very quickly; the
notebooks in `../notebooks` are the source of truth for how to regenerate
them.

## Expected contents (generated, not committed)

| File | Produced by | Used by |
|---|---|---|
| `categorizer_embeddings.onnx` or `.pkl` | `../notebooks/02_train_categorizer.ipynb` | `app/services/categorizer.py` |
| `anomaly_isolation_forest.joblib` | `../notebooks/03_train_anomaly_detector.ipynb` | `app/services/anomaly_detector.py` |
| `category_label_map.json` | `../notebooks/02_train_categorizer.ipynb` | `app/services/categorizer.py` |

## Why not commit models to Git

1. Binary diffs bloat the repository and break `git diff`/review workflows.
2. A model file with no versioned link to the data/code that produced it is
   a liability — "which dataset trained this?" should always be answerable
   by re-running a notebook, not by tribal knowledge.
3. Free hosting (Render, Hugging Face Spaces) rebuilds from source on
   deploy anyway, so the model needs to be either (a) regenerated in the
   build step, or (b) fetched from a small artifact store (e.g., a private
   GitHub Release asset, or Hugging Face Hub's free model hosting) rather
   than living in the Git tree.

See `docs/ml-strategy.md` for the training pipeline and
`docs/deployment.md` for how model artifacts get into the deployed
`ai-engine` instance.
