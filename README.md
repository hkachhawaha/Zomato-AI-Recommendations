# Zomato AI Restaurant Recommendation System

An AI-powered restaurant recommendation engine that combines deterministic data filtering with a Groq LLM to deliver personalised, concise recommendations — like having a knowledgeable food concierge on demand.

---

## What It Does

A user picks a locality, sets a budget and minimum rating, optionally names a cuisine, and describes their vibe ("romantic rooftop dinner", "quick lunch with colleagues"). The system:

1. **Hard-filters** the dataset using those exact parameters (locality, budget, rating, cuisine).
2. **Passes the shortlist** to an LLM (Llama 3.1 via Groq) which re-ranks and writes a personalised two-sentence explanation for each pick.
3. **Returns the top 3–5 restaurants** with name, cuisine, rating, cost, and AI insight.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend API | FastAPI + Uvicorn |
| Data | Pandas + Parquet (`data/processed/restaurants.parquet`) |
| LLM | Groq API (llama-3.1-8b-instant) via LangChain |
| Backend UI | Streamlit (deployed on Streamlit Community Cloud) |
| Frontend UI | Next.js 16 + TypeScript (deployed on Vercel) |
| Dataset | HuggingFace — `ManikaSaini/zomato-restaurant-recommendation` |

---

## Project Structure

```
Zomato Project 1/
│
├── src/
│   ├── phase1_setup/
│   │   ├── main.py          # FastAPI app entry point + CORS config
│   │   └── config.py        # Settings (reads from .env or Streamlit secrets)
│   │
│   ├── phase2_data_ingestion/
│   │   ├── data_loader.py   # Loads restaurants.parquet into a Pandas DataFrame
│   │   └── ingest.py        # One-time script: downloads HuggingFace dataset → parquet
│   │
│   ├── phase3_integration/
│   │   ├── schemas.py       # Pydantic models: UserPreferences, RestaurantRecommendation
│   │   ├── filtering.py     # Hard-filter logic (locality, budget, rating, cuisine)
│   │   └── endpoints.py     # FastAPI router: /health, /localities, /recommend
│   │
│   ├── phase4_recommendation/
│   │   └── llm_service.py   # Groq LLM call + structured output + fallback logic
│   │
│   ├── phase5_frontend/
│   │   └── app.py           # Streamlit UI (deployed on Streamlit Cloud)
│   │
│   └── phase6_testing/
│       └── test_api.py      # Pytest API tests
│
├── frontend/                # Next.js production UI (deployed on Vercel)
│   ├── app/                 # Next.js App Router pages + layouts
│   ├── components/          # Header, Sidebar, FeaturedCard, CompactCard, Map, etc.
│   ├── lib/api.ts           # Fetch helpers (fetchLocalities, fetchRecommendations)
│   └── types/index.ts       # TypeScript interfaces mirroring Pydantic schemas
│
├── data/
│   └── processed/
│       └── restaurants.parquet   # Pre-processed dataset (~960 KB, ~5000 restaurants)
│
├── Docs/
│   ├── PhaseWiseArchitecture.md  # Full technical architecture document
│   ├── ProblemStatementZomato.md # Original problem statement
│   └── Improvements.md           # Planned improvements
│
├── .streamlit/
│   └── secrets.toml.example      # Template for Streamlit Cloud secrets
│
├── requirements.txt         # Python dependencies
└── .env                     # Local secrets (not committed)
```

---

## How Data Flows

```
User fills form (locality, budget, rating, cuisine, vibe)
        │
        ▼
[ Phase 3 — Hard Filter ]
  filtering.py reads restaurants.parquet
  Applies: exact locality match → budget ≤ max → rating ≥ min → cuisine substring
  Returns top 15 candidates sorted by rating
        │
        ▼
[ Phase 4 — LLM Ranking ]
  llm_service.py sends candidates + user preferences to Groq
  LLM re-ranks by vibe fit, writes 2-sentence personalised explanations
  Falls back to top-3 by rating if Groq is unavailable
        │
        ▼
[ Response ]
  List of 3–5 RestaurantRecommendation objects
  { name, cuisine, rating, cost, explanation }
```

---

## Key Files to Read First

| File | Why |
|---|---|
| [src/phase1_setup/main.py](src/phase1_setup/main.py) | FastAPI app — CORS, router wiring |
| [src/phase3_integration/schemas.py](src/phase3_integration/schemas.py) | Data contracts used everywhere |
| [src/phase3_integration/filtering.py](src/phase3_integration/filtering.py) | Core filtering logic |
| [src/phase4_recommendation/llm_service.py](src/phase4_recommendation/llm_service.py) | LLM prompt + structured output + fallback |
| [src/phase5_frontend/app.py](src/phase5_frontend/app.py) | Streamlit UI — calls logic directly (no HTTP) |
| [frontend/lib/api.ts](frontend/lib/api.ts) | Next.js API client |
| [src/phase1_setup/config.py](src/phase1_setup/config.py) | How secrets are loaded (env / Streamlit secrets) |

---

## Running Locally

### Prerequisites
- Python 3.9+
- Node.js 18+
- A [Groq API key](https://console.groq.com)

### 1. Python setup

```bash
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

### 2. Environment variables

Create a `.env` file in the project root:

```
GROQ_API_KEY=your-groq-api-key-here
DATASET_NAME=ManikaSaini/zomato-restaurant-recommendation
MAX_CANDIDATES=15
```

### 3. Start the FastAPI backend

```bash
uvicorn src.phase1_setup.main:app --reload --port 8000
```

API is available at `http://localhost:8000`
Interactive docs at `http://localhost:8000/docs`

### 4. Start the Streamlit UI (optional)

```bash
streamlit run src/phase5_frontend/app.py
```

### 5. Start the Next.js frontend

```bash
cd frontend
npm install
npm run dev
```

Frontend at `http://localhost:3000`

---

## API Endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/` | Welcome message |
| `GET` | `/api/v1/health` | Health check |
| `GET` | `/api/v1/localities` | List of all unique localities |
| `POST` | `/api/v1/recommend` | Get AI restaurant recommendations |

### POST `/api/v1/recommend` — Request body

```json
{
  "locality": "Koramangala",
  "budget": 1500,
  "cuisine": "Italian",
  "min_rating": 4.0,
  "additional_preferences": "romantic dinner with rooftop seating"
}
```

---

## Deployment

| Service | Platform | Config |
|---|---|---|
| Streamlit backend UI | Streamlit Community Cloud | Main file: `src/phase5_frontend/app.py` |
| Next.js frontend | Vercel | Root directory: `frontend/` |

**Streamlit Cloud secrets** (set under App Settings → Secrets):
```toml
GROQ_API_KEY = "your-groq-api-key-here"
```

**Vercel environment variable:**
```
NEXT_PUBLIC_API_BASE = https://your-fastapi-host/api/v1
```

---

## Environment Variables Reference

| Variable | Required | Default | Description |
|---|---|---|---|
| `GROQ_API_KEY` | Yes | — | Groq API key for LLM calls |
| `DATASET_NAME` | No | `ManikaSaini/zomato-restaurant-recommendation` | HuggingFace dataset ID |
| `MAX_CANDIDATES` | No | `15` | Max restaurants sent to LLM |
