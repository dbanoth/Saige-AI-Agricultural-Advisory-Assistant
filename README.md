# Saige — AI Agricultural Advisory Assistant

> A production conversational AI system built at **Oatmeal AI** that helps farmers get farm-specific advice in chat — across crops, livestock, weather, monitoring, and market news — and never changes farm data unless a person approves it.

![Python](https://img.shields.io/badge/Python-3.11+-3776AB?style=flat&logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-0.100+-009688?style=flat&logo=fastapi&logoColor=white)
![LangChain](https://img.shields.io/badge/LangChain-Enabled-1C3C3C?style=flat&logo=langchain&logoColor=white)
![Google Gemini](https://img.shields.io/badge/Gemini-2.5_Flash_Lite-4285F4?style=flat&logo=google&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-7+-DC382D?style=flat&logo=redis&logoColor=white)
![Firestore](https://img.shields.io/badge/Firestore-Vector_Search-FFCA28?style=flat&logo=firebase&logoColor=black)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?style=flat&logo=docker&logoColor=white)

> **Note:** This project was developed as part of my work at [**Oatmeal AI**](https://oatmeal-ai.com/). The source code is proprietary and not publicly available. This README documents the architecture, design decisions, and engineering work I contributed.

---

## Overview

Saige is the farm assistant inside Oatmeal AI. A farmer asks in plain language — “my tomato leaves are yellow,” “should I irrigate the north field,” “update my business phone” — and Saige answers from the farmer’s account, live weather, and a retrieval-backed knowledge base.

Reads happen in the same turn. Writes do not. Specialist agents can look things up, but they cannot mutate farm data. A write is proposed, checked, and shown to the farmer as approve, edit, or reject. Only an allowlisted tool runs after that approval.

---

## My Role

**Generative AI Engineer** at Oatmeal AI

I designed and built the Saige backend: the agent graph, the retrieval pipeline, the API, and the write-approval boundary. Contributions include:

- Replaced a single assessment-then-route chain with a supervisor graph that can call one or more specialists in the same turn
- Kept specialists read-only and moved every mutation behind a human-in-the-loop gate
- Allowlisted writes at the tool level, with a separate executor per tool so extra model arguments are dropped
- Added a non-LLM policy check before approval (credentials, restricted inputs, risk class)
- Built the RAG pipeline on Firestore vector search and `text-embedding-004`
- Integrated Google Gemini 2.5 Flash Lite for routing, specialist answers, and the final reply
- Shipped the FastAPI API with JWT auth, per-thread rate limits, SSE streaming, and health checks
- Used Redis for short-term memory and LangGraph checkpoints, Firestore for chat history, and Cloud SQL for the proposal record

---

## Architecture

```
Frontend (React)
        │
        ▼
FastAPI  ──── JWT ──── Rate limit (Redis)
        │
        ├── Redis        short-term memory + graph checkpoints
        ├── Firestore    chat history + RAG collections
        ├── Cloud SQL    proposal records (approve / edit / reject)
        │
        └── LangGraph
              User Agent
                  │
              Supervisor ── joke ──► end
                  │
              Specialists (read-only, one or more)
                  │
              Synthesizer
                  │
              Policy gate
                  │
            ┌─────┴─────┐
            │           │
         no write     proposal
            │           │
          reply      HITL interrupt
                        │
                   approve / edit / reject
                        │
                  Execute (allowlisted tools only)
```

---

## How It Works

**1. User Agent**

Loads who is asking: person, business, active field, preferences, and recent memory. Password fields are never placed in state.

**2. Supervisor**

A hybrid router picks one or more specialists. Keyword matches run first. If the question is ambiguous, Gemini returns a structured route list. A joke is a short exit and does not enter the specialist loop. When the question belongs to another product assistant, the supervisor records a handoff instead of answering outside Saige’s scope.

**3. Specialists**

Each specialist is a read-only ReAct loop. Write-shaped tools are stripped before the model is bound, so a specialist can retrieve weather, field status, knowledge-base passages, or account facts, and cannot save, update, or delete. Results come back as packets.

**4. Synthesizer**

Merges the packets into one farmer-facing answer, with citations when retrieval was used. If the turn includes a change, it emits a proposal (tool name, arguments, risk, short summary) instead of performing the change.

**5. Policy, then a person**

A deterministic check runs before the model’s proposal is shown. Blocked tools and blocked argument content never reach the farmer as an approvable action. If nothing is proposed, the reply is returned immediately. If a proposal exists, the graph interrupts. The farmer approves, edits the arguments, or rejects. Rejected proposals are not executed.

**6. Execute**

After approval, a registry runs the tool by name. A name that is not registered never runs. Each executor copies only the fields that tool understands.

---

## Write approval

This is the boundary that keeps the assistant useful without letting it mutate the farm on its own.

| Step | What it decides |
|---|---|
| Specialist bind | Write tools are not available. The model can only read. |
| Proposal | The model names a tool and arguments. It does not call the database. |
| Policy gate | Hard blocks (credentials, restricted content) and a risk tag. Not an argument schema. |
| Human | Approve, edit, or reject. An edit merges into the arguments that will run. |
| Execute registry | Tool-name allowlist. Unknown tools never run. |

Allowlisting is at the **tool**, not at every argument combination. A per-argument schema would be a second API to maintain. Narrow executors drop unused keys, and the human edit step is where argument changes happen.

Examples of proposed writes: update a business profile, create or edit a field, toggle monitoring, log irrigation, or approve a spray review. Publishing a listing, event, or post stays a draft until the farmer approves it.

---

## Specialists

| Route | What it covers |
|---|---|
| Crop | Disease, soil, agronomy, planting |
| Livestock | Breeds, health, husbandry |
| Weather | Current conditions and forecast (Open-Meteo) |
| Monitoring | Field alerts, scouting, activity context |
| Account | Profile and field facts for the signed-in business |
| Product knowledge | Oatmeal AI product and equipment docs |
| News | Agricultural news and market context |

One turn can hit more than one specialist. “My cattle are losing weight and I changed the corn” can gather livestock and crop context before the synthesizer answers.

---

## RAG

Retrieval uses Firestore vector search with `text-embedding-004` embeddings.

| Collection | Used for |
|---|---|
| `livestock_knowledge` | Breeds, health, husbandry |
| `plant_knowledge` | Crops, soil, disease, agronomy |
| `bakasura-docs` | Product and equipment knowledge |
| `news_articles` | Agricultural news and markets |

RAG is skipped cleanly when Firestore or the embedding stack is unavailable. The rest of the turn still runs.

---

## Memory

**Redis** keeps the last N messages per thread (rolling list, 24-hour TTL) and stores LangGraph checkpoints so an in-progress approval survives a restart. If Redis is down, the graph can fall back to in-memory checkpoints.

**Firestore** stores the thread and its messages for history, pagination, and analytics.

**Cloud SQL** stores the proposal lifecycle: proposed, approved, edited, rejected, executed, or failed, with an event trail.

---

## API

Authenticated routes expect a JWT bearer token. Health routes do not.

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/chat` | One turn. Returns an answer, or pauses on a proposal. |
| `POST` | `/chat/stream` | Same turn as server-sent events (stage, tokens, done). |
| `POST` | `/proposals/{id}/decide` | Approve, edit, or reject a pending proposal. |
| `GET` | `/proposals` | List proposals for the signed-in user. |
| `GET` | `/proposals/{id}/events` | Audit trail for one proposal. |
| `GET` | `/threads` | List conversation threads. |
| `GET` | `/threads/{id}/messages` | Messages in a thread. |
| `DELETE` | `/threads/{id}` | Delete a thread. |
| `GET` | `/health` | Liveness. |
| `GET` | `/ready` | Graph, Redis, and Firestore readiness. |

Rate limit: 20 requests per 60 seconds per thread, using Redis. If Redis is unavailable, the limiter fails open.

```bash
curl -X POST http://localhost:8000/chat \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"user_input": "my tomato leaves are yellow", "thread_id": "thread_abc123"}'
```

A write comes back as proposals to approve, not as a completed mutation. Approving resumes the same thread and runs only the allowlisted tool.

---

## Tech Stack

| Layer | Technology |
|---|---|
| LLM | Google Gemini 2.5 Flash Lite |
| Orchestration | LangGraph (StateGraph, conditional edges, interrupts, checkpoints) |
| RAG | Firestore vector search + `text-embedding-004` |
| API | FastAPI, Uvicorn, SSE |
| Short-term memory | Redis |
| Chat history | Firestore |
| Proposals | Cloud SQL |
| Weather | Open-Meteo |
| Auth | JWT |
| Validation | Pydantic v2 |
| Frontend | React, TypeScript |
| Runtime | Docker, GCP |

---

## Engineering decisions

**Read-only specialists.** Binding write tools inside the answer loop makes “the model decided to save” a normal bug. Stripping them up front means a write can only exist as a proposal.

**Tool-level allowlist.** The registry is the list of mutations Saige is willing to perform. Arguments are not a second allowlist. Each executor accepts a fixed set of fields, and the farmer can edit those fields before anything runs.

**Interrupt instead of a side channel.** LangGraph pauses on the proposal and resumes on the farmer’s decision, so approval is the same conversation, not a separate job queue.

**Hybrid routing.** Keyword routes are fast. The LLM route runs when the question spans domains or the keywords are weak.

**Firestore for retrieval.** Native vector search avoided a separate vector database while still supporting top-K lookup across livestock, crop, product, and news collections.

---

## About

Built by **David** as a Generative AI Engineer at **Oatmeal AI**.

**Connect:** [LinkedIn](https://www.linkedin.com/in/devidbanoth) · [GitHub](https://github.com/dbanoth)
