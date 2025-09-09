# TaskPilot‑AI — Intelligent Todo Assistant (Backend + RAG Agent)

TaskPilot‑AI is an end-to-end demo that pairs a production-style Java Spring Boot backend (secure task CRUD, JWT auth, and OpenAPI) with a Python RAG-enabled agent service that can understand, reason about, and act on user requests.

The AI service uses Sentence‑Transformers + FAISS for retrieval and a local Ollama LLM (Llama 3.1 8B) to power conversational agents and automated task operations, while the backend exposes JWT-protected APIs for persistence and orchestrated tooling. This repository demonstrates safe integration patterns (idempotency checks, token forwarding, defensive parsing) and a reproducible architecture for adding agent-driven automation to existing services.

## Why this project exists (Motivation)

- Build a practical demo that combines standard CRUD APIs for tasks with an AI agent that can reason, act, and augment responses using Retrieval‑Augmented Generation (RAG).
- Show how to safely wire an LLM-based agent to real CRUD endpoints (idempotency, token handling, defensive parsing) and how to keep short, fast queries on a direct backend path.
- Provide a reproducible, modular reference architecture for teams who want to add agent-driven automation to existing APIs.

## One-line summary

TaskPilot‑AI is a todo application where a Spring Boot backend provides secure task CRUD and an AI service adds conversational, automated task management via RAG (FAISS embeddings) and a local LLM (Ollama, Llama 3.1 8B). The two parts are complementary and demonstrated in this repo.

## Quick links

- Backend (Java / Spring Boot): `backend/` — contains API, services, security, and OpenAPI (Swagger) config.
- AI Service (Python): `ai-service/` — contains FastAPI entrypoint, agent + RAG modules, memory, and tool wrappers for backend APIs.

## Highlights / Key features

- Secure JWT authentication and layered Spring Boot backend for tasks (register/login, task CRUD, task status enum).
- Fast direct retrieval path for simple queries (counts, lists, due-this-week) that calls the backend directly.
- LLM-driven agent for intentful actions: create/update/delete tasks and perform complex reasoning using RAG context.
- RAG index per user using Sentence‑Transformers + FAISS for similarity search over task text.
- Local LLM via Ollama (configured to use Llama 3.1 8B by default in examples) for privacy and offline capability.
- Conversation memory persisted in Postgres via SQLAlchemy (used by the agent to provide context across sessions).

## High-level architecture & flow

ASCII flow (simplified):

User -> AI Service `/chat` (FastAPI)
   ├─> Input handler (load memory)
   ├─> Decision node (LLM classifier or heuristic) -> branch
   │     ├─ If `backend` -> Backend fetch node -> format response -> output handler
   │     └─ If `agent`   -> Agent node (LLM + tools + RAG context) -> output handler
   └─> Output handler (persist conversation, return response)

If agent node runs, the following tool wrappers call the Spring Boot API: `create_task`, `get_tasks`, `update_task`, `delete_task`, `get_task_by_id` (see `ai-service/app/api/task_tools.py`).

## Tech stack (by component)

- Backend (Java)
  - Spring Boot (web, security, data-jpa)
  - JWT authentication, SpringDoc OpenAPI (Swagger)
  - PostgreSQL for persistence
  - Build: Maven (wrapper included: `mvnw` / `mvnw.cmd`)

- AI Service (Python)
  - FastAPI + Uvicorn
  - LangChain/agent orchestration (agent tools + zero-shot/react style)
  - Ollama LLM integration (local Llama 3.1 8B model example)
  - Sentence‑Transformers (`all‑MiniLM‑L6‑v2`) for embeddings
  - FAISS vector store for similarity search
  - SQLAlchemy + Postgres for conversation memory
  - httpx for HTTP calls to the backend

## Important implementation details

- Decision routing: a small LLM classifier (with a conservative heuristic fallback) decides whether a message needs CRUD/tooling (`agent`) or a fast retrieval (`backend`). This prevents running the agent for trivial retrievals and reduces latency.
- Tools: tool wrappers are defensive — they accept dicts or strings, try to parse natural inputs, perform idempotency checks (create), and normalize fields (camelCase vs snake_case). See `ai-service/app/api/task_tools.py`.
- RAG: indexes are built per user from the backend task list using Sentence‑Transformers embeddings and stored in an in-memory FAISS index (see `ai-service/app/rag/vector_store.py`). The agent queries the index for relevant tasks to seed prompts.
- Memory: conversation messages are saved to a `conversation_memory` table via SQLAlchemy and used to seed chat history for the agent (see `ai-service/app/memory/*`).
- Token handling: the incoming Authorization header is captured by the FastAPI `/chat` endpoint and injected into the tool wrappers at runtime, then cleared after the request to avoid token leakage.

## Module map & where to look (quick guide)

- `backend/` — full Spring Boot backend
  - `src/main/java/.../todo/controller/TaskController.java` — REST endpoints for tasks
  - `src/main/java/.../todo/service/TaskService.java` — business logic for CRUD
  - Security: `auth/*` (JWT filter, SecurityConfig)
  - OpenAPI: SpringDoc config exposes Swagger UI at `/swagger-ui.html` (when enabled)

- `ai-service/` — RAG + Agent
  - `main.py` — FastAPI app with `/chat` endpoint
  - `app/api/task_tools.py` — tool wrappers calling backend APIs (defensive parsing, idempotency)
  - `app/rag/graph.py` — orchestrates decision node, agent node, backend fetch node, and output handler (StateGraph)
  - `app/rag/vector_store.py` — builds and queries FAISS indexes per user
  - `app/memory/manager.py` & `models.py` — conversation memory persistence
  - `requirements.txt` — Python dependencies

## How they work together (data & control flows)

- Authentication: web/mobile clients authenticate with the Spring Boot backend and receive a JWT. When calling the AI service `/chat`, include the same `Authorization: Bearer <token>` header. The AI service uses that header to call backend APIs on behalf of the user.
- Read queries: for queries like "How many tasks do I have?" or "Which tasks are due this week?" the decision node will route to the backend fast path, fetch tasks, and return a compact human-friendly answer.
- Action queries: for intentful messages like "Create a task called Finish report due Friday", the agent node will run. The agent has tools which call the backend API to perform the operation and return confirmations.
- RAG + reasoning: when the agent needs context ("Which tasks reference the client meeting?"), it queries the FAISS index built from task text and includes the top results in the prompt.

## Run locally (recommended quick start)

1) Backend (Windows PowerShell example):

```powershell
cd .\backend
.\mvnw.cmd clean package
.\mvnw.cmd spring-boot:run
```

The backend exposes REST APIs at `http://localhost:8080` by default and provides OpenAPI docs when SpringDoc is enabled.

2) AI Service (Windows PowerShell example):

```powershell
cd .\ai-service
python -m venv .venv; .\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
uvicorn main:app --reload --port 8001
```

3) Example chat call (PowerShell):

```powershell
curl -X POST http://localhost:8001/chat -H "Content-Type: application/json" -H "Authorization: Bearer <token>" -d '{"user_id":1,"session_id":"session-123","message":"What tasks are due this week?"}'
```

For more environment config and advanced options see `backend/README.md` and `ai-service/README.md`.





