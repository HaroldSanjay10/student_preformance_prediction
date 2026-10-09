# Student Performance Prediction System – CI/CD Deployment on AWS

Predicts whether a student will **Pass** or be **At Risk** from 12 features using a scikit-learn
RandomForest. A Jenkins pipeline tests, builds, scans and deploys it to AWS EC2.

## Architecture

```text
 Developer ──git push──▶ GitHub ──webhook──▶ Jenkins
                                             │ Checkout → Test(pytest) → Build → Trivy → Push → Deploy(ssh)
                                             ▼
                                     Container registry
                                             │ docker compose pull
 ┌──────────────────────────── AWS EC2 ─────▼────────────────────────────┐
 │  :80  frontend (Nginx + React build) ──/api──▶ backend (Gunicorn+Flask)│
 │                                                 │  model.joblib        │
 │                                                 ▼                      │
 │                                          db (PostgreSQL 15, volume)    │
 └────────────────────────────────────────────────────────────────────────┘
```

Layout:

```text
shared/features.json      single source of truth for the 12 features (API validation + UI form)
backend/app/              Flask app factory, models, schemas, routes, errors, services/
backend/ml/               generate_data.py (1,000 rows, seed 42), train.py (writes model.joblib + metrics.json)
backend/tests/            24 pytest cases (in-memory SQLite)
frontend/                 React 18 + Vite + Tailwind + Chart.js, nginx.conf
scripts/deploy.sh         pull → up → health check → .last_good_tag → rollback on failure
docker-compose.yml, Jenkinsfile, .env.example
```

## Model

- Dataset: synthetic, generated from a fixed formula plus noise with `numpy` seed 42 (it is **not** real student data).
- 80/20 stratified split, `RandomForestClassifier(n_estimators=200, max_depth=10, random_state=42)`.
- `metrics.json` stores the measured accuracy, precision, recall, F1, 5-fold CV mean/std, confusion matrix and feature importances. The dashboard reads them from `GET /api/model/info`; nothing is hard-coded.
- Risk: probability ≥ 0.70 → Low, 0.40–0.70 → Medium, < 0.40 → High. Recommendations are rule-based (attendance < 75%, study < 8 h, sleep < 6 h, etc.).

## Run locally

### Option A – Docker Compose (recommended)

```bash
cp .env.example .env          # then edit every secret value
docker compose up -d --build
docker compose ps             # wait until backend is "healthy"
curl http://localhost/api/health
```

Open http://localhost, register, then use Predict / Dashboard / History.

### Option B – Without Docker

```bash
# Backend (Python 3.11)
cd backend
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
python -m ml.train                         # writes ml/artifacts/model.joblib + metrics.json
python -m pytest                           # 24 passed
export SECRET_KEY=dev-secret JWT_SECRET_KEY=dev-jwt-secret-at-least-32-characters
export DATABASE_URL=sqlite:///dev.db       # or a postgresql+psycopg2:// URL
gunicorn --preload -b 0.0.0.0:5000 wsgi:app

# Frontend (Node 20), in another terminal
cd frontend
npm install
npm run dev                                # http://localhost:5173 (proxies /api to :5000)
```

## API (prefix `/api`)

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | /health | – | Status, DB and model availability |
| POST | /auth/register | – | `{username,email,password}` → 201 + JWT; 409 duplicate; 400 weak password |
| POST | /auth/login | – | `{username,password}` → JWT; 401 on bad credentials |
| GET | /auth/me | JWT | Current user |
| GET | /features | – | Feature input spec (from `shared/features.json`) |
| POST | /predict | JWT | `{student_name, reg_no?, features:{…12}}` → prediction + recommendations; unknown fields rejected |
| GET | /predictions | JWT | `?page=&per_page=&outcome=Pass|At Risk` |
| DELETE | /predictions/<id> | JWT | Own records only, else 404 |
| GET | /stats | JWT | Totals, pass rate, risk distribution, last 10 |
| GET | /model/info | – | metrics.json content |

Errors are JSON: 400 `{error:"validation_error", details:{…}}`, 401, 404, 409, 503 when model artifacts are missing.

## Tests

`cd backend && python -m pytest` runs 24 tests: health (1), auth (9), predict (6), history (5), stats (1), model info (1), ML probability ordering (1). A session fixture trains a model into a temp folder, so tests don't depend on a prebuilt artifact.

## One-time AWS EC2 setup

1. Launch Ubuntu 22.04/24.04, `t3.small` or larger, 20 GB disk.
2. Security group inbound: **22** from your IP / the Jenkins server IP only; **80** from 0.0.0.0/0 (443 if you add TLS). Never open 5000 or 5432.
3. Install Docker:
   ```bash
   curl -fsSL https://get.docker.com | sudo sh
   sudo usermod -aG docker ubuntu && newgrp docker
   sudo mkdir -p /opt/spps && sudo chown ubuntu:ubuntu /opt/spps
   ```
4. Create `/opt/spps/.env` from `.env.example` with real secrets and `REGISTRY` set to your registry namespace (`chmod 600 .env`).
5. If the registry is private: `docker login` once on the instance.

## One-time Jenkins setup

1. Jenkins host needs Docker; add the `jenkins` user to the `docker` group. Plugins: Pipeline, Git, GitHub, Docker Pipeline, SSH Agent, JUnit.
2. Credentials (Manage Jenkins → Credentials):
   - `registry-namespace` – Secret text, e.g. `docker.io/youruser`
   - `registry-creds` – Username/password (Docker Hub access token)
   - `ec2-host` – Secret text, e.g. `ubuntu@<ec2-public-ip>`
   - `ec2-ssh-key` – SSH Username with private key (the EC2 key pair)
3. Create a **Multibranch Pipeline** (or Pipeline from SCM) pointing at the repo; it uses `Jenkinsfile`. Deploy runs only on `main`.
4. GitHub webhook: repo → Settings → Webhooks → `http://<jenkins-host>:8080/github-webhook/`, content type `application/json`, "Just the push event". Enable "GitHub hook trigger for GITScm polling" on the job.

Pipeline stages: Checkout → Test (pytest, JUnit report) → Build images → Trivy scan (fails on unfixed HIGH/CRITICAL) → Push → Deploy to EC2 (`scripts/deploy.sh <tag>`).

## Deploy and rollback

`scripts/deploy.sh <tag>` pulls the tagged images, runs `docker compose up -d`, polls `/api/health` until `status` is `ok`, writes the tag to `.last_good_tag`, and on failure redeploys the previous good tag. Manual rollback: `./scripts/deploy.sh $(cat .last_good_tag)` or any earlier tag.

## Security notes

- No secrets in the repo: everything comes from `.env` (git-ignored) or Jenkins credentials; the app refuses to start without `SECRET_KEY`, `JWT_SECRET_KEY` and `DATABASE_URL`.
- Passwords hashed with bcrypt; minimum 8 chars with letters and digits.
- JWT in `localStorage` is simple but readable by injected scripts; keep dependencies patched and consider HttpOnly cookies for higher-risk deployments.
- Backend runs as non-root; only Nginx is published; Postgres stays on the internal network.
- Images are scanned with Trivy before push. Add TLS (e.g. an ALB with ACM, or certbot) before real use.
- There is no rate limiting on login; add it (e.g. Nginx `limit_req`) for public deployments.
