# PeaKit — Python + Next.js

PeaKit is an AI store-manager assistant with exactly three capabilities:
current food stock remaining, restock timing ("when to buy"), and demand
forecasting (with runout ETA and expected footfall). This is the
**Python (FastAPI) backend + Next.js frontend** build of PeaKit — a
sibling to `../peakit_fullstack` (the Spring Boot/Java version of the same
product) and `../peakit_html_demo` (a zero-dependency single-file demo).
All three implement the same reasoning/tool-calling design and the same
Friday-7:04pm demo scenario; this one exists because Python was the
requested backend language for this build.

```
peakit_python_nextjs/
  api-contract/    the shared OpenAPI 3.0 contract both sides implement
  backend/         Python 3.11 + FastAPI + SQLAlchemy + Postgres — the real backend
  frontend/        Next.js app — the UI, plus a local mock backend for zero-setup demos
```

## The core idea: reasoning and computation are separate

Every number PeaKit gives a manager — a stock level, a restock
recommendation, a demand forecast, a runout ETA — comes from a small set
of deterministic tool functions (`backend/app/services/tools_service.py`).
The AI model is only ever asked one question: which of a few things is
the manager asking about? It never computes a forecast and never does
arithmetic — that split is enforced by `backend/app/services/reasoning/`
(`local_stub.py`, the default keyword router with no AWS dependency, or
`bedrock_adapter.py`, a real AWS Bedrock Converse API call — swap between
them with the `PEAKIT_AI_PROVIDER` env var; a Bedrock outage falls back to
the local stub automatically, never taking the endpoint down).

## Run it locally

```bash
# Backend
cd backend
docker compose up -d          # local Postgres
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000

# Frontend, in a second terminal
cd frontend
npm install
echo "NEXT_PUBLIC_API_BASE_URL=http://localhost:8000" > .env.local
npm run dev                   # http://localhost:3000
```

The frontend also runs with **zero backend at all** — leave
`NEXT_PUBLIC_API_BASE_URL` unset in `.env.local` and it talks to its own
in-memory mock API routes instead, using the same domain logic. See
`frontend/README.md`.

## What was and wasn't verified in this sandbox

This authoring sandbox has no real Postgres and a broken system SSL/boto3
install, so — same honesty pattern as `../peakit_fullstack`:

- **Verified:** the demand-forecast formula and the local intent
  classifier (unit tests, hand-checked numbers), 14 FastAPI `TestClient`
  tests against a throwaway SQLite database covering every endpoint and
  all three agent intents (20/20 tests passing), a live `uvicorn` boot +
  `curl` smoke test, and — usefully — a real (not simulated) exercise of
  the Bedrock-outage fallback path, since this sandbox's boto3/OpenSSL is
  actually broken here.
- **Not verified:** the real Postgres wiring end-to-end, and a real
  Bedrock call against an actual AWS account. See `backend/README.md` for
  the full detail.

## Relationship to the other PeaKit builds

| Build | Backend | Use it for |
|---|---|---|
| `peakit_python_nextjs` (this one) | Python / FastAPI | The requested Python + Next.js stack |
| `peakit_fullstack` | Java / Spring Boot | The original full-stack build, includes the AWS CDK infra + hackathon submission PPTX |
| `peakit_html_demo` | none — single static HTML file | Clicking through the concept with zero setup |

All three share the same seed scenario (store `IN-BLR-0142`, Friday
2026-09-18 7:04pm, chicken already below par, corn cups critically low)
so their numbers match exactly.
