# Saige — AI Agricultural Advisory Assistant

> A production-ready conversational AI system built at **Oatmeal AI** that guides farmers through personalized diagnostics across livestock, crops, weather, and market intelligence — powered by Google Gemini, LangGraph, and Retrieval-Augmented Generation.

![Python](https://img.shields.io/badge/Python-3.11+-3776AB?style=flat&logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-0.100+-009688?style=flat&logo=fastapi&logoColor=white)
![LangChain](https://img.shields.io/badge/LangChain-Enabled-1C3C3C?style=flat&logo=langchain&logoColor=white)
![Google Gemini](https://img.shields.io/badge/Gemini-2.5_Flash_Lite-4285F4?style=flat&logo=google&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-7+-DC382D?style=flat&logo=redis&logoColor=white)
![Firestore](https://img.shields.io/badge/Firestore-Vector_Search-FFCA28?style=flat&logo=firebase&logoColor=black)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?style=flat&logo=docker&logoColor=white)

> **Note:** This project was developed as part of my work at **Oatmeal AI**. The source code is proprietary and not publicly available. This README documents the architecture, design decisions, and engineering work I contributed to the project.

---

## Overview

Saige is a conversational AI assistant built for farmers. Instead of giving generic advice, it conducts a structured diagnostic conversation — asking smart, context-aware questions to understand the farmer's location, farm size, crops or livestock, and current problems. Once it has enough context, it automatically routes the query to the right specialist advisory engine and returns precise, actionable recommendations.

The system was designed from the ground up to be production-ready — with multi-agent orchestration, RAG-powered knowledge retrieval, real-time weather integration, secure authentication, Redis-backed session memory, and full cloud deployment on GCP.

---

## My Role

**Generative AI Engineer** at Oatmeal AI

I was responsible for the full design and implementation of the Saige backend — from the LangGraph agentic workflow and RAG pipeline, to the FastAPI REST API, cloud infrastructure, and test suite. Key contributions include:

- Architected the entire multi-node LangGraph StateGraph pipeline with six advisory domains
- Built the RAG pipeline using Firestore vector search and `text-embedding-004` embeddings
- Integrated Google Gemini 2.5 Flash Lite via LangChain for structured LLM output and query classification
- Designed a hybrid keyword + LLM classifier for intelligent query routing
- Built the FastAPI REST API with JWT authentication, rate limiting, and health monitoring
- Implemented Redis-backed short-term memory and LangGraph checkpointing for session continuity
- Designed Firestore schemas for long-term chat persistence and conversation analytics
- Integrated real-time weather data via Open-Meteo as a LangChain tool
- Containerized the full stack with Docker and deployed to GCP

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
          ├── Google Cloud SQL → Structured relational data
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

## How It Works

Saige operates in three phases:

**Phase 1 — Assessment**

Saige opens with an open-ended question and guides the farmer through up to 8 structured diagnostic turns. Each response is classified by an LLM with structured output (`AssessmentDecision`) and stored into a typed `FarmState` — capturing location, farm size, crops or animals, and current problems. The assessment ends when the LLM determines it has sufficient context.

**Phase 2 — Routing**

A hybrid classifier reads the completed `assessment_summary`. It first applies keyword scoring across domain-specific keyword sets (livestock, crops, weather, news), then falls back to an LLM classifier (`QueryClassification`) if scores are ambiguous. The result is one of six advisory routes: `weather`, `livestock`, `crops`, `mixed`, `news`, or `bakasura`.

**Phase 3 — Advisory**

The selected advisory node fetches relevant context from its Firestore RAG collection using `text-embedding-004` vector embeddings (top-K = 10). For weather queries, it calls the Open-Meteo API via a LangChain tool wrapper. The node then passes the retrieved context and farm state to Google Gemini to generate a structured advisory response with recommendations and an optional quiz-style UI.

---

## LangGraph State Design

The entire conversation is tracked in a typed `FarmState`:

| Field | Type | Purpose |
|---|---|---|
| `location` | `str` | Farmer's location for weather queries |
| `farm_size` | `str` | Farm area |
| `crops` | `List[str]` | Crops or animals being raised |
| `current_issues` | `List[str]` | Reported problems or goals |
| `history` | `List[str]` | Conversation turns |
| `assessment_summary` | `str` | Compact summary produced at assessment completion |
| `advisory_type` | `str` | Final routed type: weather / livestock / crops / mixed |
| `diagnosis` | `str` | Final advisory text |
| `recommendations` | `List[str]` | Structured recommendations |
| `weather_conditions` | `dict` | Fetched weather data |

---

## RAG Pipeline

All knowledge retrieval uses Firestore vector search with `text-embedding-004` embeddings (top-K = 10).

| Collection | Domain | Used By |
|---|---|---|
| `livestock_knowledge` | Animal husbandry, breeds, health | `livestock_advisory_node` |
| `plant_knowledge` | Crops, soil, disease, agronomy | `crop_advisory_node` |
| `bakasura-docs` | Oatmeal AI product knowledge | `bakasura_advisory_node` |
| `news_articles` | Agricultural news, market updates | `news_advisory_node` |

The `mixed_advisory_node` queries all three advisory collections simultaneously and merges the results before generating a response.

---

## Session Memory Design

### Short-Term Memory (Redis)

Redis stores the last `N` messages per thread for fast in-context injection at query time. Each message is serialized with role, content, timestamp, and metadata. The buffer uses a `LPUSH + LTRIM` pattern to maintain a rolling window with a 24-hour TTL.

Key format: `thread:{thread_id}:last_messages`

### Long-Term Persistence (Firestore)

Every conversation is persisted to Firestore under a `threads/{thread_id}` document with a `messages` subcollection. Thread metadata includes status, advisory type, message count, and farm context. The API supports paginated thread and message retrieval, and an analytics endpoint aggregates completion rates and advisory type distributions.

### LangGraph Checkpointing

The StateGraph is compiled with `RedisSaver` when Redis is available, enabling persistent graph checkpointing across restarts. If Redis is unavailable, the system falls back to `MemorySaver` automatically with no impact to the user.

---

## API Endpoints

All endpoints except health checks require a valid JWT Bearer token.

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/chat` | Main advisory endpoint |
| `GET` | `/threads` | List conversation threads (paginated) |
| `GET` | `/threads/{id}/messages` | Fetch thread messages (paginated) |
| `DELETE` | `/threads/{id}` | Delete a thread and all messages |
| `GET` | `/analytics` | Conversation stats for the authenticated user |
| `GET` | `/health` | API liveness probe |
| `GET` | `/health/firestore` | Firestore write/read/delete health check |
| `GET` | `/health/redis` | Redis ping health check |
| `GET` | `/ready` | Full readiness probe (graph + Redis + Firestore) |

**Rate Limiting:** 20 requests per 60-second window per `thread_id`, backed by Redis `INCR + EXPIRE`. Fails open if Redis is unavailable.

**Example Request:**

```bash
curl -X POST http://localhost:8000/chat \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"user_input": "my cattle have been losing weight", "thread_id": "thread_abc123"}'
```

**Assessment Response:**

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

**Advisory Response:**

```json
{
  "status": "complete",
  "advice": "Based on your farm context, your cattle may be experiencing nutritional deficiency...",
  "advisory_type": "livestock",
  "recommendations": ["Check feed quality", "Consult a vet for blood panel", "Review pasture rotation"]
}
```

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
| Relational Database | Google Cloud SQL |
| Weather | Open-Meteo API (LangChain tool wrapper) |
| Authentication | JWT HS256 (`python-jose`) |
| Validation | Pydantic v2 |
| Frontend | Next.js, React 19, TypeScript, Tailwind CSS |
| Containerization | Docker, Docker Compose |
| Cloud | GCP (Vertex AI, Firestore, Memorystore Redis, Cloud SQL) |
| Testing | pytest (unit + integration) |

---

## Project Structure

```
saige/
├── api.py                  # FastAPI app, endpoints, rate limiting, middleware
├── graph.py                # LangGraph StateGraph construction and compilation
├── nodes.py                # All node functions, routing logic, advisory engine
├── models.py               # FarmState TypedDict and Pydantic models
├── config.py               # Centralized configuration and feature flags
├── llm.py                  # Google Gemini LLM initialization
├── rag.py                  # Firestore vector search across all collections
├── chat_history.py         # Firestore-backed conversation persistence
├── message_buffer.py       # Redis short-term message buffer
├── jwt_auth.py             # JWT Bearer token verification
├── redis_client.py         # Redis connection pooling and health checks
├── weather.py              # Open-Meteo weather service + LangChain tool
├── database.py             # Google Cloud SQL query helpers
├── Data_Contract.py        # Pydantic data contracts for external integrations
├── main.py                 # Application entry point
├── sync_embeddings.py      # Sync embeddings into Firestore RAG collections
├── seed_firestore.py       # Seed initial knowledge data into Firestore
├── test_api_flow.py        # Integration tests for full API flow
├── test_main.py            # Unit tests for core logic
└── test_redis.py           # Redis connectivity and buffer tests
```

---

## Key Engineering Decisions

**Why LangGraph over a simple chain?** The diagnostic conversation requires stateful, multi-turn logic with conditional branching — LangGraph's StateGraph and interrupt mechanism made this clean and maintainable compared to a linear chain.

**Why Firestore for RAG?** Firestore's native vector search with `text-embedding-004` eliminated the need for a separate vector database, reducing infrastructure complexity while still supporting top-K similarity retrieval across multiple knowledge collections.

**Why Redis for short-term memory?** Injecting recent conversation history into each LLM call is critical for coherent multi-turn conversations. Redis provides sub-millisecond access and TTL-based automatic cleanup, making it ideal for session-scoped memory.

**Why a hybrid router?** Pure keyword routing is fast but breaks on ambiguous queries. Pure LLM routing is accurate but slow. The hybrid approach — keyword scoring first, LLM fallback only when needed — balances speed and accuracy.

---

## About

Built by **David** as a Generative AI Engineer at **Oatmeal AI** — a platform designed to make expert agricultural knowledge accessible to every farmer through conversational AI.

**Connect:** [LinkedIn](https://linkedin.com/in/your-profile) · [GitHub](https://github.com/your-username)
