# Creator Audit System

**A FastAPI backend that sells YouTube channel audits and delivers them end to end, from Stripe checkout to an emailed PDF, with no manual step. Claude writes the narrative; code checks every number it writes.**

## The problem

Small YouTube creators want to know why their channel stalls, but a human audit is slow and expensive, and an LLM left alone will invent statistics. This project automates the whole service while keeping the numbers trustworthy: every metric, score and ranked problem is computed in code, and the AI is only allowed to explain them.

## What it does, end to end

1. **Checkout**: the buyer picks a tier (unlock / standard / premium) and pays through Stripe Checkout. A verified Stripe webhook (or the success redirect) marks the order paid and creates a job.
2. **Data ingest**: the buyer uploads their YouTube Studio CSV export (`csv_parser.py`). The system also pulls public channel and video data from the YouTube Data API v3 (`youtube.py`) and merges the two sources (`metrics.py`).
3. **Metrics and scoring**: cadence, momentum, click-through, retention, conversion, format mix and outliers are computed in pandas. `scoring.py` produces a 0-100 score, subscores, and a ranked list of rule-based problems, each carrying its evidence values.
4. **AI narrative**: Claude receives the metrics and problem list as JSON and returns the written audit as **structured JSON** that matches a per-tier JSON schema (`narrative.py`).
5. **Numeric validation**: `numcheck.py` extracts every number from the generated text and checks it against the source data. Failing sections are regenerated once with feedback; a second failure flags the job for human review instead of sending it.
6. **Report**: an fpdf2 PDF plus a web report page (`report.py`).
7. **Delivery**: the PDF is emailed over SMTP, with a link to the web report. The uploaded CSV is deleted once the report is built; only derived metrics are kept.

A background worker claims queued jobs from the database and retries transient errors (up to 3 attempts) before marking a job failed and alerting the admin.

### Outreach and a second pipeline

- **Creator outreach** (`outreach.py`): scan YouTube for channels in target niches, build a free public-data teaser audit, put the prospect in an admin review queue, send the approved teaser email inside send hours with daily caps, follow up, and handle unsubscribes and open/click tracking. Contacts are added by a person or imported; nothing is sent without approval unless configured.
- **Local business website scanner** (`local.py`, `places.py`): finds businesses without a website through the Google Places API, scores their online presence, and generates a private mock homepage preview that the owner can buy.

## How AI is used

- **Claude writes words, never numbers.** The score, subscores and problem ranking are deterministic code. The model's job is to explain them in plain language and propose concrete fixes.
- **Structured output.** Each request passes a JSON schema (`output_config.format` with `json_schema`) whose shape depends on the tier: summary, problems (`key`, `title`, `evidence`, `fix`), best videos, format breakdown, and for premium, title rewrites and a 30-day schedule. The response is parsed with `json.loads`, so downstream code never scrapes free text. Problems the model returns are kept only if their `key` matches a real problem candidate, and `fix_type` is set by code.
- **Number check against source data.** `numcheck.py` builds the set of allowed values from the metrics, subscores, score and problem evidence. A number in the text passes only if it equals a source value at the precision written (e.g. `4.3` matches `4.31`), its absolute value, a number inside a quoted video title, or a small set of structural constants ("out of 100", "last 10 uploads"). Anything else fails, and the failing numbers are fed back to the model in the retry prompt.
- **Template fallback.** With no `ANTHROPIC_API_KEY` set, a deterministic template writer produces the same JSON shape from the evidence values. Its output goes through the same number check, so the pipeline runs (and is tested) fully offline. When Claude is configured but refuses (`stop_reason == "refusal"`) or is cut off (`max_tokens`), the job raises an error and is retried by the worker, then flagged for the admin; it is never delivered unchecked.
- **Prompt design.** The system prompt sets hard rules (use only given numbers, no new calculations, no abbreviations, no invented benchmarks), and each tier has its own brief. Model, effort, token usage, prompt version and number-check failures are logged per audit.

## Architecture

```
Stripe Checkout ──► /stripe/webhook ──► orders + jobs (Postgres / SQLite)
Upload page (Studio CSV) ─────────────────────┘
                                              │
                        worker: claim queued job
                                              │
     YouTube Data API v3 + CSV ─► metrics ─► scoring ─► narrative (Claude JSON | template)
                                                               │
                                              numcheck ── fail ─► regenerate once ─► flag
                                                               │ pass
                                              fpdf2 PDF + web report ─► SMTP email
GitHub Actions cron (every 3h) ─► `python -m creator_audit cron`: scans, outreach sends,
                                  follow-ups, outcome checks, then queued jobs
```

- `creator_audit/app.py`: FastAPI routes (checkout, webhook, upload, report, teaser, admin, unsubscribe, data deletion). `app_local.py` adds the local-business routes.
- `creator_audit/pipeline.py`: the worker (metrics, score, narrative, number check, report, delivery).
- `creator_audit/tasks.py`: scheduler (scans, sends, follow-ups, 30/60/90-day outcome checks, purges).
- `creator_audit/db.py`: one data layer for SQLite (local/tests) and Postgres (production, Neon), translating SQL where needed.
- `render.yaml`: Render web service blueprint. `.github/workflows/cron.yml`: scheduled job runner.

## Tech stack

Python 3.11, FastAPI, Uvicorn, Jinja2, pandas, Anthropic Python SDK (Claude), Stripe, YouTube Data API v3, Google Places API, fpdf2, psycopg 3 (Neon Postgres) / SQLite, SMTP (Resend in production), GitHub Actions, Render.

## Run it offline (demo mode)

Every external service has an offline stand-in, chosen by blank or `fake` config values (see `creator_audit/config.py`):

| Setting | Offline behaviour |
|---|---|
| `STRIPE_SECRET_KEY` blank | dev checkout that marks orders paid instantly |
| `YOUTUBE_API_KEY=fake` | synthetic channel and video data |
| `GOOGLE_PLACES_API_KEY=fake` | synthetic local businesses |
| `ANTHROPIC_API_KEY` blank | deterministic template writer (still number-checked) |
| `SMTP_HOST` blank | emails written as `.eml` files to `data/outbox/` |
| `DATABASE_URL` blank | SQLite in `data/` |

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env             # defaults are offline-safe
python -m creator_audit all      # web + worker + scheduler on http://localhost:8000
```

Open `http://localhost:8000/audit`, buy a tier with the dev checkout, upload `sample_studio_export.csv` (synthetic data; regenerate with `python -m creator_audit sample-csv out.csv`) and a made-up channel URL such as `https://www.youtube.com/@democreator`. The PDF appears in `data/reports/` and the delivery email in `data/outbox/`. The admin panel is at `/admin` (credentials from `ADMIN_USER` / `ADMIN_PASSWORD`).

## Tests

Both test files are offline end-to-end runs through the FastAPI `TestClient` (fake YouTube and Places data, dev checkout, template writer, outbox email), each in a temporary data directory.

- `tests/test_e2e.py`: standard and premium paid flows (checkout, upload validation, pipeline to PDF and email, CSV deletion), teaser outreach and review queue, click-redirect safety, unlock purchase, unsubscribe, video worker API auth, data deletion.
- `tests/test_local.py`: local business scan, preview and mock site, website purchase and owner setup form, opt-out, self-serve preview with rate limits.
- `tests/run_pg.py`: runs both files against a throwaway real Postgres 16 (needs `pip install pgserver`).

Observed results on Python 3.11.9 (Windows), fresh virtual environment, at the time of this commit:

```
python tests/test_local.py   -> LOCAL PASSED sqlite
python tests/test_e2e.py     -> ALL PASSED sqlite
python tests/run_pg.py       -> test_e2e.py: ok, test_local.py: ok (postgres)
```

## What I'd do next

- Add unit tests for `numcheck.py` edge cases (rounding, negatives, numbers in titles) and for the scoring rules.
- Fall back to the template writer automatically after repeated Claude refusals or outages, instead of only retrying and flagging.
- Add a golden-output evaluation set: real-shaped metrics with expected problem rankings, plus a check of narrative quality across prompt versions.
- Move from a polled database queue to a proper task queue once volume needs it, and add structured logging and metrics.
- Split the local-business scanner into its own service; it shares the database and admin but is a separate product.

## Notes

All data in this repository is synthetic. Domains and email addresses are placeholders (`example.com`). Configuration secrets are read from environment variables; `.env.example` contains no real values.

Built by Nikko Echevarria with AI-assisted development (Claude Code).
