# EMR System - Phase 1

## Backend
```
cd backend
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
copy .env.example .env
python seed.py
uvicorn app.main:app --reload
```

## Frontend
```
cd frontend
npm install
npm run dev
```

## Tests
```
cd backend
pytest -q
```

PostgreSQL/Docker not installed on this host; SQLite used for dev. Install PostgreSQL or Docker for production parity.
