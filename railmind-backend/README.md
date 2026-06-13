# 🚆 RailMind AI — Backend

AI-powered backend for Indian Railways, built for **[Hackathon Name]** by **Team Rusty Coders**.

RailMind AI brings together delay prediction, station crowd forecasting, and AI-based track safety detection into a single FastAPI service — wrapped with a Groq-powered LLM agent that turns raw model outputs into human-readable advisories and alerts.

- 🌐 **Live Demo (Frontend):** [railmind-ai.netlify.app](https://railmind-ai.netlify.app/)
- 💻 **Frontend Repo:** [github.com/vybex-dev/railmind-ai](https://github.com/vybex-dev/railmind-ai)
- ⚙️ **Backend Repo:** [github.com/vybex-dev/rail.mind.ai](https://github.com/vybex-dev/rail.mind.ai)

---

## ✨ Features

### 🕐 Delay Prediction (`/delay`)
- ML-based train delay prediction (XGBoost) using train number, route, and time-of-day features.
- Falls back to a rule-based mock predictor when no trained model is present — the API always stays usable.
- AI agent (via Groq/Llama 3.1) generates a passenger-friendly advisory, reasoning, alternative train suggestions, and urgency level for every prediction.
- SSE endpoint to stream live agent reasoning token-by-token.

### 👥 Crowd Forecasting (`/crowd`)
- Station-wise crowd level forecasting with 1–24 hour lookahead.
- 24-hour crowd heatmap per station.
- Platform allocation recommendations based on predicted congestion.
- AI-generated operational advisory for station managers (Groq-powered, with mock fallback).
- Live NTES integration for real-time running status of major trains.

### 🛤️ Track Safety Detection (`/safety`)
- CLIP-based image classifier for detecting track defects from uploaded images.
- Lazily loads the ~600 MB CLIP model on the **first** `/safety/analyze` call (avoids OOM on free-tier deploys) — subsequent calls are fast.
- Falls back to weighted-random mock classification if CLIP fails to load.
- AI agent generates formal, operations-grade safety alerts based on defect severity.
- In-memory ring buffer of the last 10 analyses via `/safety/recent`.

### 🤖 RailAgent (AI Layer)
- Single agent class powering all three modules via Groq's free-tier API (`llama-3.1-8b-instant`).
- Every method degrades gracefully to curated, rule-based fallbacks if `GROQ_API_KEY` is missing or the API call fails — the platform **never breaks** due to LLM downtime.

---

## 🧰 Tech Stack

| Layer | Technology |
|---|---|
| API Framework | FastAPI + Uvicorn |
| Delay Model | XGBoost (scikit-learn pipeline) |
| Safety Model | CLIP (vision-language model) |
| AI Agent | Groq API (Llama 3.1 8B Instant) |
| Schemas/Validation | Pydantic |
| Data | Custom synthetic train delay dataset + sample station/train catalogues |

---

## 📁 Project Structure

```
railmind-backend/
├── app/
│   ├── main.py                  # FastAPI app entry point
│   ├── agent/
│   │   └── rail_agent.py        # Groq-powered advisory agent (+ fallbacks)
│   ├── models/
│   │   ├── delay_model.py        # DelayPredictor (XGBoost / mock)
│   │   ├── train_delay_model.py  # Training script for the delay model
│   │   ├── crowd_model.py        # CrowdForecaster
│   │   └── safety_model.py       # TrackSafetyDetector (CLIP / mock)
│   ├── routers/
│   │   ├── delay.py              # /delay endpoints
│   │   ├── crowd.py              # /crowd endpoints
│   │   └── safety.py             # /safety endpoints
│   └── schemas/
│       ├── delay.py
│       ├── crowd.py
│       └── safety.py
├── data/
│   ├── sample_trains.json
│   └── train_delays.csv
├── saved_models/                 # Generated after training
└── requirements.txt
```

---

## 🚀 Getting Started

### Prerequisites
- Python 3.10+
- pip

### 1. Clone the repo
```bash
git clone https://github.com/vybex-dev/rail.mind.ai.git
cd rail.mind.ai
```

### 2. Create a virtual environment & install dependencies
```bash
python -m venv venv
source venv/bin/activate        # On Windows: venv\Scripts\activate
pip install -r requirements.txt
```

### 3. (Optional) Configure environment variables
Create a `.env` file in the project root to enable the live AI agent:
```env
GROQ_API_KEY=your_groq_api_key_here
```
> Get a free key at [console.groq.com](https://console.groq.com/). Without it, the agent runs in **mock mode** with curated fallback responses — the API remains fully functional.

### 4. (Optional) Train the delay prediction model
```bash
python -m app.models.train_delay_model
```
This generates `saved_models/delay_xgb_model.joblib`, `delay_encoders.joblib`, and `delay_model_info.json`. Without this step, `/delay/predict` uses a rule-based mock predictor.

### 5. Run the server
```bash
uvicorn app.main:app --reload --port 8000
```

### 6. Explore the API
- Swagger UI: [http://localhost:8000/docs](http://localhost:8000/docs)
- ReDoc: [http://localhost:8000/redoc](http://localhost:8000/redoc)
- Health check: [http://localhost:8000/health](http://localhost:8000/health)

---

## 🔌 API Reference

### Meta
| Method | Endpoint | Description |
|---|---|---|
| GET | `/` | API welcome message + doc links |
| GET | `/health` | Module load status (delay, crowd, safety) |

### Delay Prediction — `/delay`
| Method | Endpoint | Description |
|---|---|---|
| GET | `/delay/trains` | List all trains in the catalogue |
| POST | `/delay/predict` | Predict delay for a train + AI advisory |
| GET | `/delay/stats` | Operational delay statistics |
| GET | `/delay/agent-stream` | SSE stream of live AI reasoning |

**Example — `POST /delay/predict`**
```json
{
  "train_number": "12301",
  "source": "HWH",
  "destination": "NDLS",
  "hour": 8,
  "day_of_week": 1,
  "month": 7
}
```

### Crowd Forecasting — `/crowd`
| Method | Endpoint | Description |
|---|---|---|
| GET | `/crowd/stations` | List supported stations |
| POST | `/crowd/predict` | Forecast crowd level + AI advisory |
| GET | `/crowd/heatmap/{station_code}` | 24-hour crowd heatmap |
| GET | `/crowd/ntes/delays` | Live running status for major trains (NTES) |

### Track Safety — `/safety`
| Method | Endpoint | Description |
|---|---|---|
| POST | `/safety/analyze` | Upload track image → defect classification + AI alert |
| GET | `/safety/recent` | Last ≤10 analyses (in-memory) |
| GET | `/safety/status` | CLIP model load status + supported defect classes |

> ⚠️ The first `/safety/analyze` call may take 30–90 seconds while CLIP loads (~600 MB). Subsequent calls are fast.

---

## 🧠 Design Notes

- **Graceful degradation everywhere**: every AI/ML component (delay model, CLIP, Groq agent) has a deterministic fallback so the API never returns a 500 due to a missing model or API key — ideal for demoing without setup.
- **Lazy CLIP loading**: prevents memory crashes on free-tier hosts (e.g., Railway) by deferring the ~600 MB model load until the first real request.
- **Single AI agent, multiple roles**: `RailAgent` is reused across delay, crowd, and safety modules — one Groq client, three specialized prompts.

---

## 👥 Team — Rusty Coders

Built with ☕ and 🚆 at **[Hackathon Name]**.

---

## 📄 License

[MIT](LICENSE) — feel free to fork and extend.
