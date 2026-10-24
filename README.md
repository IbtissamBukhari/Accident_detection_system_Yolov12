# AI Accident Detection & Emergency Response System

An end-to-end web application that analyses uploaded road-scene images and videos with a custom YOLO model, records incidents, and presents results through a secure monitoring dashboard.

This project is intended for academic demonstration and controlled local deployment. A model prediction supports human review; it is not a verified emergency-service dispatch or medical assessment.

<p align="center">
  <img src="ml_training/accident_model_bundle/training_run/val_batch0_pred.jpg" width="85%" alt="YOLOv12 predictions on validation images"/>
</p>

## Model results

Custom **YOLOv12n** model trained for 40 epochs at 640 px on a 3-class dataset (`fire`, `moderate`, `severe`).

Evaluated on the held-out **test set (1,800 images, 2,443 objects)**:

| Class | Precision | Recall | mAP@50 | mAP@50-95 |
|---|---|---|---|---|
| **All** | **92.5%** | **86.8%** | **89.9%** | **79.0%** |
| fire | 83.0% | 64.3% | 72.9% | 49.5% |
| moderate | 97.6% | 96.9% | 97.3% | 94.3% |
| severe | 96.9% | 99.0% | 99.4% | 93.2% |

`fire` is the weakest class (smoke and flames have no clear box edges and vary a lot); collecting more fire images is the next improvement.

Source: `ml_training/accident_model_bundle/metrics.json`. Training curves, PR curves and the confusion matrix are in `ml_training/accident_model_bundle/training_run/`.

## Tech stack

YOLOv12 (Ultralytics) · PyTorch · OpenCV · FastAPI · SQLAlchemy + Alembic · MySQL · React + Vite + Tailwind · Docker Compose · pytest · Gemini API (optional)

## What it does

- Authenticated users upload images or videos for accident detection.
- The model classifies detections as `fire`, `moderate`, or `severe`.
- Images return annotated results immediately; videos run as background jobs with progress polling.
- Events, confidence, boxes, media paths, statuses, and audit information are persisted.
- Dashboards show history, alerts, analytics, and generated emergency assessments.
- Optional SMTP email and Kapso/WhatsApp delivery can notify responders; mock mode is safe for demos.
- Gemini can generate the assessment; a deterministic local fallback works without an API key.

## Quick start with Docker

### Requirements

- Docker Engine with Compose
- Model file at `backend/ml_model/best.pt`

### Configure

Create a root `.env` for Compose and never commit it:

```env
SECRET_KEY=replace-with-a-long-random-value
MYSQL_ROOT_PASSWORD=replace-with-a-root-password
MYSQL_PASSWORD=replace-with-an-app-password
MYSQL_DATABASE=accident_db
MYSQL_USER=accident_app
ENABLE_DOCS=true
```

Compose also references `backend/.env` for optional integrations. Create it:

```bash
touch backend/.env
```

Add SMTP, Gemini, or Kapso values there only when intentionally enabling those integrations.

### Start

```bash
docker compose up --build
```

| Service | URL |
| --- | --- |
| Frontend | http://localhost |
| API | http://localhost:8000 |
| Health | http://localhost:8000/health |
| Swagger UI | http://localhost:8000/docs when `ENABLE_DOCS=true` |

Stop with `docker compose down`. Named volumes preserve MySQL and generated media. Use `docker compose down -v` only when intentionally removing local data.

## Manual development

### Backend

```bash
cd backend
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
alembic upgrade head
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

Set `DATABASE_URL` and `SECRET_KEY` in `backend/.env`, for example:

```env
DATABASE_URL=mysql+pymysql://user:password@127.0.0.1:3306/accident_db
```

### Frontend

```bash
cd frontend
npm install
echo "VITE_API_URL=http://localhost:8000" > .env
npm run dev
```

Open http://localhost:5173. Run `npm run lint` and `npm run build` before handing off frontend changes.

## Repository map

```text
backend/app/main.py          FastAPI app, middleware, startup, route registration
backend/app/api/routes/      HTTP endpoints and access checks
backend/app/core/            Inference, video, alerts, agent, reports, security
backend/app/db/              SQLAlchemy engine, models, CRUD helpers
backend/app/schemas/         Pydantic request/response contracts
backend/alembic/             Database migrations
backend/tests/               API, security, alert, and workflow tests
frontend/src/pages/          Route-level screens
frontend/src/components/     Shared dashboard/UI components
frontend/src/api/            Axios API modules and token handling
frontend/src/context/         Authentication state
database/                    Initial MySQL schema and seed script
ml_training/                 Training notebook, metrics, model artifacts
docker-compose.yml           MySQL, backend, and frontend services
docs/SYSTEM_DETAILS.md       Complete implementation walkthrough
```

## Model contract

The runtime class order is fixed:

```text
0 = fire
1 = moderate
2 = severe
```

If a new model changes labels or order, update the weights, `classes.yaml`, `DetectionEngine.CLASS_NAMES`, tests, and documentation together.

## Important safety notes

- Never commit `.env` files, API keys, passwords, phone numbers, or real recipient addresses.
- Original videos belong in private `data/uploads`; generated outputs are under `static/processed`, `static/snapshots`, and `static/reports`.
- Email and WhatsApp delivery are configuration-dependent and can fail; inspect persisted delivery status.
- Video jobs are currently in memory, suitable for a single-process demo but not multi-worker production.

Read [SYSTEM_DETAILS.md](docs/SYSTEM_DETAILS.md) for the full frontend, backend, API, model, database, agent, alert, storage, security, and development explanation.
