# Saige — AI Agricultural Advisory Assistant

> A production-ready conversational AI system that guides farmers through personalized diagnostics across livestock, crops, weather, and market intelligence — powered by Google Gemini, LangGraph, and Retrieval-Augmented Generation.

![Python](https://img.shields.io/badge/Python-3.11+-3776AB?style=flat&logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-0.100+-009688?style=flat&logo=fastapi&logoColor=white)
![LangChain](https://img.shields.io/badge/LangChain-Enabled-1C3C3C?style=flat&logo=langchain&logoColor=white)
![Google Gemini](https://img.shields.io/badge/Gemini-2.5_Flash_Lite-4285F4?style=flat&logo=google&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-7+-DC382D?style=flat&logo=redis&logoColor=white)
![Firestore](https://img.shields.io/badge/Firestore-Vector_Search-FFCA28?style=flat&logo=firebase&logoColor=black)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?style=flat&logo=docker&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-green?style=flat)

---

## What is Saige?

Saige is an intelligent conversational AI assistant built for farmers. It conducts a structured diagnostic conversation, learns about the farmer's context — their location, crops, livestock, and problems — and then routes the query to a specialized advisory engine to deliver precise, actionable recommendations.

It is built on a **multi-node LangGraph agentic workflow**, a **RAG pipeline backed by Firestore vector search**, and a **production FastAPI backend** with Redis for session memory and rate limiting.

---

## Features

- **Multi-domain Advisory** — covers Livestock, Crops, Weather, Agricultural News, and Product Knowledge
- **Conversational Assessment** — up to 8 structured diagnostic questions to build full farm context before advising
- **Hybrid Query Routing** — combines keyword scoring with LLM classification to route queries to the right advisory node
- **RAG-Powered Knowledge** — retrieves domain-specific knowledge from Firestore vector collections using `text-embedding-004`
- **Real-Time Weather Integration** — fetches live conditions and forecasts via Open-Meteo as a LangChain tool
- **Short-Term Memory** — Redis message buffer keeps the last N conversation turns for fast in-context injection
- **Long-Term Persistence** — every conversation thread is stored in Firestore with full analytics support
- **JWT Authentication** — secure HS256 Bearer token verification on all protected endpoints
- **Rate Limiting** — Redis-backed per-thread rate limiting (fail-open by design)
- **Graceful Fallbacks** — Redis and RAG degrade gracefully if unavailable, keeping the system running

---

## Architecture

```
Frontend (Next.js / React 19)
          │
          ▼
  FastAPI REST API  ──── JWT Auth ──── Rate Limiter (Redis)
          │
          ├── Redis          → Short-term message buffer (last N messages)
          │                  → LangGraph RedisSaver checkpoints
          │
          ├── Firestore      → Long-term chat history & thread persistence
          │                  → RAG knowledge collections (vector search)
          │
          └── LangGraph StateGraph
                    │
                    ├── assessment_node      ← Builds farm context (up to 8 turns)
                    ├── routing_node         ← Hybrid keyword + LLM classifier
                    │
                    ├── weather_advisory_node
                    ├── livestock_advisory_node
                    ├── crop_advisory_node
                    ├── mixed_advisory_node
                    ├── bakasura_advisory_node
                    └── news_advisory_node
                              │
                              └── Google Gemini 2.5 Flash Lite
```

![Farm Advisory Graph](farm_advisory_graph.png)

---

## Tech Stack

| Layer | Technology |
|---|---|
| LLM | Google Gemini 2.5 Flash Lite (`langchain-google-genai` / Vertex AI) |
| AI Orchestration | LangGraph (StateGraph, conditional edges, interrupts, checkpointing) |
| RAG | Firestore Vector Search + `text-embedding-004` |
| API | FastAPI 0.100+ / Uvicorn |
| Short-Term Memory | Redis (message buffer + LangGraph RedisSaver) |
| Long-Term Storage | Google Cloud Firestore |
| Weather | Open-Meteo API (via LangChain tool wrapper) |
| Database | Google Cloud SQL |
| Authentication | JWT HS256 (`python-jose`) |
| Validation | Pydantic v2 |
| Frontend | Next.js, React 19, TypeScript, Tailwind CSS |
| Containerization | Docker, Docker Compose |
| Testing | pytest (unit + integration) |

---

## Project Structure

```
saige/
├── api.py                  # FastAPI app, endpoints, rate limiting, middleware
├── graph.py                # LangGraph StateGraph construction and compilation
├── nodes.py                # All node functions, routing logic, advisory engine
├── models.py               # FarmState TypedDict and Pydantic models
├── config.py               # Centralized env-var configuration and feature flags
├── llm.py                  # Google Gemini LLM initialization
├── rag.py                  # Firestore vector search (livestock, plant, bakasura, news)
├── chat_history.py         # Firestore-backed conversation persistence
├── message_buffer.py       # Redis short-term message buffer (last N messages)
├── jwt_auth.py             # JWT Bearer token verification (FastAPI dependency)
├── redis_client.py         # RedisClientManager (connection pooling, health checks)
├── weather.py              # Open-Meteo weather service and LangChain tool wrapper
├── database.py             # Google Cloud SQL query helpers
├── Data_Contract.py        # Pydantic data contracts for external integrations
├── main.py                 # Application entry point / server startup
├── sync_embeddings.py      # Script to sync embeddings into Firestore RAG collections
├── seed_firestore.py       # Script to seed initial knowledge data into Firestore
├── test_api_flow.py        # Integration tests for the full API flow
├── test_main.py            # Unit tests for core logic
└── test_redis.py           # Redis connectivity and buffer tests
```

---

## How It Works

Saige follows a three-phase conversation loop:

**1. Assessment** — Saige opens with an open-ended question and guides the farmer through up to 8 structured turns to gather location, farm size, crops/animals, and current problems. Each answer is classified and stored in a typed `FarmState`.

**2. Routing** — Once the assessment is complete, a hybrid classifier (keyword scoring + LLM fallback) reads the assessment summary and selects one of six advisory routes: `weather`, `livestock`, `crops`, `mixed`, `news`, or `bakasura`.

**3. Advisory** — The selected node fetches relevant RAG context from Firestore vector collections and/or live weather data, then generates a final advisory response via Gemini — structured with recommendations and an optional quiz UI.

---

## Getting Started

### Prerequisites

- Python 3.11+
- Redis 7+
- Google Cloud project (Firestore + Vertex AI enabled)
- Node.js 18+ (frontend)

### Backend Setup

```bash
# Clone the repo
git clone https://github.com/your-username/saige.git
cd saige

# Create virtual environment
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Configure environment
cp .env.example .env
# Fill in your keys in .env
```

### Run with Docker (Recommended)

```bash
docker compose up --build
```

This starts the FastAPI backend on `http://localhost:8000` and Redis on port `6379`.

### Run Manually

```bash
# Backend
uvicorn api:app --reload --port 8000

# Frontend
cd frontend
npm install && npm run dev
```

### Seed Knowledge Data

```bash
python seed_firestore.py       # Load initial RAG knowledge into Firestore
python sync_embeddings.py      # Sync/refresh vector embeddings
```

---

## API Overview

All endpoints except health checks require a JWT Bearer token.

```
POST   /chat                        → Main advisory endpoint
GET    /threads                     → List conversation threads (paginated)
GET    /threads/{id}/messages       → Fetch thread messages (paginated)
DELETE /threads/{id}                → Delete a thread
GET    /analytics                   → Conversation stats
GET    /health/firestore            → Firestore health check
GET    /health/redis                → Redis health check
GET    /ready                       → Full readiness probe
```

**Example Request:**

```bash
curl -X POST http://localhost:8000/chat \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"user_input": "my cattle have been losing weight", "thread_id": "thread_abc123"}'
```

**Example Response:**

```json
{
  "status": "requires_input",
  "ui": {
    "type": "quiz",
    "question": "What type of cattle are you raising?",
    "options": ["Beef cattle", "Dairy cattle", "Mixed herd", "Not sure"]
  }
}
```

---

## Environment Variables

Create a `.env` file in the project root:

```env
# Auth
SECRET_KEY=your_jwt_secret_key

# Google / Gemini
GOOGLE_API_KEY=your_gemini_api_key
GEMINI_MODEL=gemini-2.5-flash-lite

# Firestore
GOOGLE_CLOUD_PROJECT=your-gcp-project
FIRESTORE_DATABASE=charlie
CHAT_HISTORY_DATABASE=chat-history

# Redis
REDIS_ENABLED=true
REDIS_HOST=localhost
REDIS_PORT=6379

# Frontend
FRONTEND_URL=http://localhost:3000
```

See the full variable reference in the [documentation](README.md#configuration).

---

## Testing

```bash
# Run all tests
pytest

# Run specific test files
pytest test_api_flow.py
pytest test_redis.py
pytest test_main.py
```

---

## About the Developer

Built by **David** as part of the **Oatmeal AI** platform — a Generative AI system designed to make expert agricultural knowledge accessible to every farmer through conversational AI.

**Role:** Generative AI Engineer
**Stack:** LangGraph · LangChain · Google Gemini · FastAPI · RAG · Redis · Firestore · Docker

---

## License

This project is licensed under the MIT License.
