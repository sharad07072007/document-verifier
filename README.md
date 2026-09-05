# Scrutiny API

A FastAPI backend for document forgery detection: metadata forensics, Error
Level Analysis (ELA), OCR-based text consistency checks, an optional
pluggable deep-learning tampering model, and checksum validation (Luhn /
IBAN / MRZ). Every analysis is persisted to a database as a scan report.

This is the backend for the client-side `scrutiny.html` prototype — same
scoring logic, but with real OCR (Tesseract), a slot for a real pretrained
forgery-detection model, and a persistent history of scans.

## 1. Local setup

**Prerequisites:** Python 3.11+, and the Tesseract OCR engine installed on
your system (not just the Python wrapper):

- macOS: `brew install tesseract`
- Ubuntu/Debian: `sudo apt-get install tesseract-ocr`
- Windows: install from https://github.com/UB-Mannheim/tesseract/wiki, then
  point `SCRUTINY_TESSERACT_CMD` at `tesseract.exe` in your `.env`

```bash
cd document-forensics-api
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt

cp .env.example .env            # adjust as needed

uvicorn app.main:app --reload
```

The API is now running at `http://localhost:8000`. Interactive docs (Swagger
UI) are at `http://localhost:8000/docs`.

## 2. Docker setup (includes Tesseract, no local install needed)

```bash
cp .env.example .env
docker compose up --build
```

## 3. API overview

| Method | Path                       | Description                                  |
|--------|----------------------------|-----------------------------------------------|
| POST   | `/analyze`                 | Upload a JPG/PNG/PDF, run the full pipeline   |
| GET    | `/reports`                 | List past scan reports (paginated)            |
| GET    | `/reports/{id}`            | Full detail for one report                    |
| GET    | `/reports/{id}/heatmap`    | The ELA heatmap PNG for that report           |
| DELETE | `/reports/{id}`            | Purge a report and its stored heatmap         |
| POST   | `/checksum`                | Validate a Luhn / IBAN / MRZ checksum         |

### Example: analyze a document

```bash
curl -X POST http://localhost:8000/analyze \
  -F "file=@/path/to/document.jpg"
```

Response (abridged):

```json
{
  "id": 1,
  "filename": "document.jpg",
  "trust_score": 62,
  "risk_label": "inconsistencies_found",
  "flags": [
    {"level": "warn", "text": "Image editing software detected in metadata: Adobe Photoshop"},
    {"level": "fail", "text": "Elevated compression inconsistency across 3.2% of the image..."}
  ],
  "metadata_json": { "...": "..." },
  "ela_mean_diff": 4.81,
  "ela_hot_ratio": 0.032,
  "heatmap_url": "/reports/1/heatmap",
  "ocr_mean_confidence": 88.4,
  "deep_model_score": null,
  "created_at": "2026-09-05T10:03:11Z"
}
```

### Example: validate a checksum

```bash
curl -X POST http://localhost:8000/checksum \
  -H "Content-Type: application/json" \
  -d '{"type": "luhn", "value": "4111 1111 1111 1111"}'
```

## 4. Plugging in a real deep-learning model

Out of the box, `deep_model_score` is `null` — the shipped CNN in
`app/forensics/deep_model.py` is untrained and intentionally disabled until
you configure a real checkpoint, so it can never masquerade as a genuine
signal. To wire up a real one:

1. Download a pretrained checkpoint for a model such as **TruFor**,
   **Mantra-Net**, or **CAT-Net** from its official repository (not
   distributed by Anthropic), or train/fine-tune your own on a labeled
   tampering dataset.
2. Update `TamperingClassifier` in `app/forensics/deep_model.py` to match
   that model's actual architecture (the shipped class is a minimal
   placeholder), or write a small adapter that loads the checkpoint
   directly.
3. Set `SCRUTINY_FORGERY_MODEL_CHECKPOINT=/path/to/checkpoint.pt` and
   `SCRUTINY_ENABLE_DEEP_MODEL=true` in your `.env`.

## 5. Database

Defaults to a local SQLite file at `storage/scrutiny.db`. For production,
point `SCRUTINY_DATABASE_URL` at Postgres (add `psycopg2-binary` to
`requirements.txt`) — the SQLAlchemy models require no other changes.

## 6. Privacy & data handling

- `SCRUTINY_RETAIN_UPLOADED_FILES=false` (default) means original file bytes
  are **not** written to disk — only the derived report (score, flags,
  metadata, OCR excerpt, and the ELA heatmap) is persisted.
- Uploaded documents may contain PII (IDs, account numbers, signatures).
  Serve this API over HTTPS in production, restrict `SCRUTINY_ALLOWED_ORIGINS`
  to your actual frontend, and add authentication before exposing `/reports`
  publicly — as shipped, anyone who can reach the API can list all reports.
- Use `DELETE /reports/{id}` to purge a report and its heatmap on request.

## 7. Limitations — read before relying on this for real decisions

- ELA and the OCR-confidence heuristic are indicative signals, not proof.
  Legitimate scans, screenshots, and heavily recompressed originals can
  trigger false positives.
- The deep-learning slot is inert until you supply a real, properly
  evaluated checkpoint.
- Checksum validation confirms a number is *internally consistent*, not that
  it is real, unexpired, or authorized.
- This tool does not replace a human reviewer or an authoritative
  verification system (e.g. an issuing bank, government ID database, or
  document authentication service) for decisions with real consequences.
