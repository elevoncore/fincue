# `services/ai-engine/app`

The FastAPI application code goes here — **not yet written**, per this
repository's "structure and docs only" build directive.

## Intended internal shape (for whoever implements this next)

```text
app/
├── main.py              # FastAPI app instance, router registration, CORS/auth middleware
├── routers/             # One module per resource: ocr.py, categorize.py, assistant.py, anomalies.py, forecast.py
├── services/            # Business logic: ocr_service.py, categorizer.py, anomaly_detector.py, forecaster.py, assistant_orchestrator.py
├── schemas/             # Pydantic request/response models — should mirror docs/api.md exactly
├── clients/             # Thin wrappers around external calls: gemini_client.py, groq_client.py, exchange_rate_client.py
└── core/                # config.py (env loading), security.py (internal API key check), logging.py
```

This is a suggested shape, not a mandate — adjust as the implementation
teaches you something the docs didn't anticipate, and update
`docs/architecture.md` when you do.
