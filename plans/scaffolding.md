# Scaffold plan: Flask (backend) + React (frontend) + PostgreSQL (infra)

This document describes a detailed implementation plan for creating a containerized web application scaffold using:

- Backend: Flask (Python 3.11+), SQLAlchemy, Flask-Migrate, Gunicorn
- Frontend: React (Vite), Node 18+
- Database: PostgreSQL (official image)
- Orchestration: Docker + docker-compose (infra/docker-compose.yml)

## Purpose

- Provide a precise, reviewable blueprint before implementing files.
- Include example code snippets, file layout, API endpoints, Dockerfiles, and run steps.

## Contract (short)

- Inputs: environment variables (DATABASE_URL / POSTGRES_*), HTTP requests from browser
- Outputs: HTTP JSON API from backend, static assets served by frontend
- Success: `docker-compose up --build` starts Postgres, backend, and frontend; backend exposes `/health` and `/api/hello`, frontend fetches and displays the `/api/hello` message.

## Assumptions

- Python 3.11, Node 18+, Docker & docker-compose installed.
- Development uses `.env` files; secrets are not committed.

## Project layout (top-level)

```text
ai-app-experiment/
├── backend/
│   ├── app/
│   │   ├── __init__.py        # create_app factory, config
│   │   ├── main.py            # routes / blueprints
│   │   ├── models.py          # SQLAlchemy models
│   │   ├── extensions.py      # db, migrate instances
│   │   └── api/
│   │       └── v1/
│   │           └── hello.py   # example endpoint
│   ├── migrations/            # flask-migrate generated
│   ├── requirements.txt
│   ├── Dockerfile
│   ├── start.sh                # entrypoint to wait for db & run migrations + gunicorn
│   └── .env.example

├── frontend/
│   ├── package.json
│   ├── vite.config.js
│   ├── src/
│   │   ├── main.jsx
│   │   └── App.jsx
│   └── Dockerfile

├── infra/
│   ├── docker-compose.yml
│   └── .env.example

├── plans/
│   └── scaffolding.md         # this file
└── README.md
```

## Detailed backend plan

### Dependencies

Minimal `backend/requirements.txt`:

```text
Flask>=2.3
gunicorn
SQLAlchemy>=2.0
psycopg[binary]>=3.0
Flask-Migrate
alembic
python-dotenv
flask-cors
wait-for-it  # optional: simple wait wrapper (or implement wait in start.sh)
```

### Extensions and app factory

`backend/app/__init__.py` (example):

```python
from flask import Flask
from .extensions import db, migrate
from .api.v1.hello import bp as hello_bp
from dotenv import load_dotenv
import os

def create_app(config_object=None):
  load_dotenv()
  app = Flask(__name__, static_folder=None)

  # config from env
  app.config['SQLALCHEMY_DATABASE_URI'] = os.environ.get('DATABASE_URL')
  app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
  app.config['SECRET_KEY'] = os.environ.get('SECRET_KEY', 'dev-secret')

  db.init_app(app)
  migrate.init_app(app, db)

  app.register_blueprint(hello_bp, url_prefix='/api')

  @app.route('/health')
  def health():
    return {'status': 'ok'}

  return app
```

`backend/app/extensions.py`:

```python
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate

db = SQLAlchemy()
migrate = Migrate()
```

### Simple model example

`backend/app/models.py`:

```python
from .extensions import db

class User(db.Model):
  id = db.Column(db.Integer, primary_key=True)
  username = db.Column(db.String(80), unique=True, nullable=False)

  def to_dict(self):
    return {'id': self.id, 'username': self.username}
```

### API blueprint

`backend/app/api/v1/hello.py`:

```python
from flask import Blueprint, jsonify

bp = Blueprint('hello', __name__)

@bp.route('/hello')
def hello():
  return jsonify({'message': 'Hello from backend'})
```

### WSGI entrypoint (for gunicorn)

`backend/main.py` (module root for gunicorn to import):

```python
from app import create_app

app = create_app()

if __name__ == '__main__':
  app.run(host='0.0.0.0', port=8000)
```

### Dockerfile (example)

```dockerfile
FROM python:3.11-slim

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends build-essential libpq-dev && rm -rf /var/lib/apt/lists/*

COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY . /app
RUN addgroup --system app && adduser --system --ingroup app app
RUN chown -R app:app /app

USER app

EXPOSE 8000

CMD ["/app/start.sh"]
```

### `start.sh` (entrypoint) — simple pattern

```bash
#!/usr/bin/env bash
set -e

# Wait for DB to be ready (simple loop)
until python - <<PYTHON
import sys
import sqlalchemy
from sqlalchemy import create_engine
import os
try:
  engine = create_engine(os.environ['DATABASE_URL'])
  conn = engine.connect()
  conn.close()
except Exception as e:
  print('waiting for db', e)
  sys.exit(1)
PYTHON

# Run migrations (flask-migrate expects FLASK_APP env var or manage it programmatically)
flask db upgrade || true

# Launch gunicorn
exec gunicorn --bind 0.0.0.0:8000 main:app
```

Note: the above inline Python wait is illustrative; consider using `wait-for-it.sh` or `dockerize`.

### `.env.example` (backend)

```text
DATABASE_URL=postgresql+psycopg://postgres:postgres@db:5432/app_db
FLASK_ENV=development
SECRET_KEY=replace-me
```

## Detailed frontend plan

### Initialize with Vite

- Use `npm create vite@latest frontend -- --template react` (or the equivalent) and choose React + JS or TS.

### `package.json` scripts (examples)

```json
{
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "lint": "eslint src --ext .js,.jsx"
  }
}
```

### `src/App.jsx` (example that calls backend)

```jsx
import {useEffect, useState} from 'react'

export default function App(){
  const [msg, setMsg] = useState('loading...')

  useEffect(()=>{
    fetch('/api/hello')
      .then(r=>r.json())
      .then(d=>setMsg(d.message))
      .catch(e=>setMsg('error'))
  },[])

  return (
    <div>
      <h1>React Frontend</h1>
      <p>Backend says: {msg}</p>
    </div>
  )
}
```

### Development proxy (Vite)

`vite.config.js` snippet:

```js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    port: 3000,
    proxy: {
      '/api': {
        target: 'http://localhost:8000',
        changeOrigin: true,
      }
    }
  }
})
```

### Frontend Dockerfile (production build, serve with nginx)

```dockerfile
# build stage
FROM node:18-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

# production stage
FROM nginx:stable-alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY infra/nginx/frontend.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx","-g","daemon off;"]
```

### `infra/nginx/frontend.conf` (simple config)

```nginx
server {
  listen 80;
  server_name _;

  root /usr/share/nginx/html;
  index index.html;

  location /api/ {
    proxy_pass http://backend:8000/api/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
  }

  location / {
    try_files $uri $uri/ /index.html;
  }
}
```

Infra: docker-compose

`infra/docker-compose.yml` (example)

```yaml
version: '3.8'
services:
  db:
    image: postgres:15
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-postgres}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-postgres}
      POSTGRES_DB: ${POSTGRES_DB:-app_db}
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 5s
      timeout: 5s
      retries: 5

  backend:
    build: ../backend
    env_file:
      - ../backend/.env.example
      - .env
    environment:
      DATABASE_URL: ${DATABASE_URL}
    depends_on:
      db:
        condition: service_healthy
    ports:
      - "8000:8000"

  frontend:
    build: ../frontend
    ports:
      - "3000:80"
    depends_on:
      - backend

volumes:
  db-data:
```

### Notes

- `depends_on.condition` may be limited depending on compose version; the healthcheck for `db` helps ensure it's ready. Alternatively, rely on the backend `start.sh` to retry until DB is ready.

### DB migrations & init

- Use `Flask-Migrate` (Alembic) and commit migration scripts to `backend/migrations`.
- Typical workflow:
  - `flask db init` (one-time)
  - `flask db migrate -m "init"`
  - `flask db upgrade`

- Automate migration at container start via `start.sh` (shown above) or a separate `infra/init-db.sh` that runs migrations once and exits.

### Testing and linters

- Backend tests (pytest): `backend/tests/test_health.py`:

```python
from main import app

def test_health():
    client = app.test_client()
    rv = client.get('/health')
    assert rv.status_code == 200
    assert rv.json['status'] == 'ok'
```

- Frontend test: use Vitest or Jest and a small test that mocks fetch and renders `App`.
- Linting: add `.flake8`, `pylintrc` and `eslint` configs. Keep defaults minimal.

CI: GitHub Actions (example workflow)

```yaml
name: CI
on: [push, pull_request]
jobs:
  backend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      - name: Install deps
        run: |
          cd backend
          python -m pip install -r requirements.txt
      - name: Run tests
        run: |
          cd backend
          pytest -q

  frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Node
        uses: actions/setup-node@v4
        with:
          node-version: '18'
      - name: Install & build
        run: |
          cd frontend
          npm ci
          npm run build
```

### Run & dev workflows (developer notes)

- Development (using Vite proxy):

```bash
# start Postgres + backend + frontend (production-like)
docker-compose -f infra/docker-compose.yml up --build

# or run services individually for dev:
# backend dev
cd backend
export FLASK_APP=main.py
export FLASK_ENV=development
export DATABASE_URL=postgresql+psycopg://postgres:postgres@localhost:5432/app_db
flask run --port 8000

# frontend dev
cd frontend
npm install
npm run dev
```

## Verification checklist (success criteria)

- Backend responds:
  - GET /health -> 200 {"status":"ok"}
  - GET /api/hello -> 200 {"message":"Hello from backend"}
- Frontend loads and displays the message from `/api/hello`.
- `docker-compose up --build` starts all services and DB migrations applied.

## Edge cases and notes

- DB race conditions: ensure backend waits/retries for DB before running migrations. Use robust wait scripts.
- Secrets: do not check real secrets into repo. Use `.env` for local dev, and secret manager in prod.
- CORS vs proxy: dev uses Vite proxy. Production with nginx reverse-proxy is recommended so frontend can call `/api/*` without CORS.
- Ports: default mapping uses 8000 (backend) and 3000 (frontend). Document how to change.

## Optional extras (post-scaffold)

- Add Redis for caching or background job queue.
- Add Admin UI (Adminer/pgAdmin) as optional docker-compose service for convenience.
- Add HTTPS termination with Traefik or nginx + Let's Encrypt in infra.

## Next steps / request for review

- Please review this plan and tell me if you'd like any of the defaults changed (Python/Node versions, choice of static server, or DB image tag).
- After you approve, I can implement the scaffold in the repository following this plan and update the todo list as items are completed.

---
Document created for review.
