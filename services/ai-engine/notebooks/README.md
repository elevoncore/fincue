# `services/ai-engine/notebooks`

Jupyter notebooks for data exploration, model training, and evaluation.
None exist yet — this is the intended sequence once ML work (Phase 5 in
`roadmap.md`) begins:

| Notebook (to be created) | Purpose |
|---|---|
| `01_explore_datasets.ipynb` | Load and profile the Kaggle datasets listed in `docs/ml-strategy.md`; check category balance, merchant-text noise, missing values. |
| `02_train_categorizer.ipynb` | Generate sentence embeddings, train/evaluate the classical classifier, export `models/categorizer_embeddings.*` and the label map. |
| `03_train_anomaly_detector.ipynb` | Fit an Isolation Forest / statistical baseline on synthetic + sample transaction sequences; evaluate false-positive rate. |
| `04_evaluate_ocr.ipynb` | Run PaddleOCR (and, for comparison, Tesseract) against the SROIE dataset; report field-level accuracy for company/date/total extraction. |
| `05_forecasting_baselines.ipynb` | Compare a simple moving-average/linear-regression cash-flow forecast against a Prophet baseline; decide if the extra dependency is worth it. |

## Ground rules

- Notebooks train and evaluate; they do not get imported by `app/` at
  runtime — export a plain artifact (`.pkl`/`.onnx`/`.json`) that the
  FastAPI app loads instead.
- Clear notebook outputs before committing (`nbstripout` recommended) so
  diffs stay reviewable and no user data accidentally gets committed in a
  cell's printed output.
- Only use the datasets referenced in `docs/ml-strategy.md`, or synthetic
  data you generate yourself — never real personal financial exports, even
  your own, once the app is handling live-ish data.
