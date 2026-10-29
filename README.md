# Multi-Agent-System-using-LangGraph-MCP-Supervisor-Guardrails-HITL

A complete multi-agent system that uses **LangGraph** and the **Model Context Protocol (MCP)** to implement a production-ready travel-planning assistant with a **Supervisor**, input **Guardrails**, and **Human-In-The-Loop (HITL)** approval flows.

The project includes a FastAPI web frontend, custom MCP server adapters, client helpers, and PostgreSQL persistence to demonstrate how autonomous agents, supervisors, and guardrails can be composed into a safe, reviewable planning pipeline.

---

## Key Ideas

- **Multi-Agent Coordination using LangGraph and MCP**: Orchestrates specialized domain agents (Flights, Hotels, Weather, Budget, and Itinerary) using LangGraph's state machine and MCP adapters.
- **Supervisor Agent**: Decomposes user travel queries, extracts trip constraints (destination, duration, budget, travel style), and dynamically routes tasks to only the necessary specialist agents.
- **Input Guardrails**: Screens incoming requests to ensure they remain within the travel planning domain while rejecting unrelated or unsafe inputs.
- **Human-In-The-Loop (HITL) Approval**: Uses LangGraph's native `interrupt()` pattern to pause execution after generating a draft itinerary, allowing the user to review, approve, or provide revision feedback before the final response is produced.
- **Persistent State Management**: Checkpoints conversation and thread state in PostgreSQL (`PostgresSaver`) for resilience and seamless resumption across asynchronous approval cycles.

---

## Contents

- **`app.py`**: FastAPI web application and REST API endpoints (`/api/travel`, `/api/travel/approve`, `/health`).
- **`backend.py`**: Core agent orchestration, LangGraph `StateGraph` definition, supervisor logic, specialist agents, input guardrails, and HITL handlers.
- **`mcp_client.py`**: Client helpers to manage the `MultiServerMCPClient` connection across multiple transports.
- **`custom_weather_mcp_server.py`**: Example standalone FastMCP server providing live weather observations and 5-day forecasts via OpenWeatherMap.
- **`test.py`**: Standalone terminal-based CLI runner for headless multi-agent testing and terminal-based HITL reviews.
- **`test_db.py`**: Diagnostic utility to test and verify PostgreSQL database connectivity.
- **`templates/`, `static/`**: Web UI assets (HTML, CSS, JavaScript) providing an interactive user dashboard.
- **`requirements.txt`**: Python dependencies for the project.
- **`.env`**: Local configuration file for API credentials and database connection strings.

---

## Features

- **Interactive Web UI**: A clean, responsive chat dashboard for submitting natural-language travel requests.
- **Review & Approval Modal**: Built-in HITL interface displaying draft itineraries with one-click approval or custom feedback submission.
- **Real-Time Data Integration via MCP**:
  - **Tavily MCP** (`streamable_http`): Live web search for hotel accommodations, amenities, and location insights.
  - **AviationStack MCP** (`stdio` via `uvx`): Real-time flight status, routes, airport metadata, and airline information.
  - **Custom Weather MCP Server** (`stdio` via `FastMCP`): Real-time temperature, conditions, and 5-day forecast data.
- **Supervisor-Driven Specialist Delegation**: Automatically selects only the required agents (e.g., skips flight search if the user only asks for local weather or hotel recommendations).
- **Dual Execution Modes**: Run as a full-featured web app or as a terminal CLI application.

---

## Architecture Overview

```text
User Request
     │
     ▼
[Input Guardrail] ──(Rejected)──► [Guardrail Blocked Message] ──► End
     │
 (Allowed)
     ▼
[Supervisor Agent] (Extracts constraints & routes)
     │
     ├──────────────────┬─────────────────┬──────────────────┐
     ▼                  ▼                 ▼                  ▼
[Flight Agent]    [Hotel Agent]    [Weather Agent]    [Budget Agent]
 (AviationStack)     (Tavily)      (Custom FastMCP)     (LLM Logic)
     │                  │                 │                  │
     └──────────────────┴─────────────────┴──────────────────┘
                                │
                                ▼
                       [Itinerary Agent]
                    (Generates Draft Plan)
                                │
                                ▼
                     [Human Approval (HITL)]
                    (LangGraph interrupt())
                                │
                  ┌─────────────┴─────────────┐
             (Approved)                   (Feedback)
                  │                            │
                  ▼                            ▼
        [Final Polishing Agent] ◄──────────────┘
         (Produces Final Plan)
                  │
                  ▼
          [PostgreSQL State]
                  │
                  ▼
             User Output
```

---

## Prerequisites

- **Python 3.10+** (Python 3.11 recommended)
- **PostgreSQL**: Local instance (pgAdmin / PostgreSQL server) or cloud-hosted database (e.g., Supabase, Neon, Render)
- **Git**: To clone the repository
- **uv / uvx**: Required for executing stdio MCP servers (`pip install uv`)
- **API Keys**:
  - [Groq](https://console.groq.com/) (LLM inference)
  - [Tavily](https://tavily.com/) (Web search MCP)
  - [OpenWeatherMap](https://openweathermap.org/api) (Weather MCP)
  - [AviationStack](https://aviationstack.com/) (Flight MCP)

---

## Quick Start (Windows)

### 1. Clone the repository
```bash
git clone https://github.com/your-username/Multi-Agent-System-using-LangGraph-MCP-Supervisor-Guardrails-HITL.git
cd Multi-Agent-System-using-LangGraph-MCP-Supervisor-Guardrails-HITL
```

### 2. Create and activate a virtual environment
```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
```
*(If using bash/cmd: `.venv\Scripts\activate`)*

### 3. Install dependencies
```powershell
pip install -r requirements.txt
pip install uv
```

### 4. Configure environment variables
Create a `.env` file in the root directory (see [Configuration & Environment](#configuration--environment) below).

### 5. Verify the database connection
```powershell
python test_db.py
```

### 6. Run the FastAPI application
```powershell
# Option A: Run directly
python app.py

# Option B: Run via uvicorn
uvicorn app:app --reload --host 127.0.0.1 --port 8000
```

### 7. Open the web UI
Visit **[http://127.0.0.1:8000](http://127.0.0.1:8000)** in your browser to start planning trips.

---

## Terminal CLI Runner

You can run the full multi-agent workflow directly in your terminal without starting the web server:

```powershell
# Run with a specific travel query
python test.py "Plan a 3-day trip to Paris with $1500"

# Or start interactive mode
python test.py
```

> **Windows Terminal Note**: Set the console encoding to UTF-8 to ensure emojis and special punctuation display properly:
> ```powershell
> chcp 65001
> $env:PYTHONIOENCODING="utf-8"
> ```

---

## Running the MCP Server (Example)

The project includes `custom_weather_mcp_server.py` as an example FastMCP server.

During normal web execution, `mcp_client.py` launches this server automatically via `stdio`. However, you can run and test it independently:

```powershell
python custom_weather_mcp_server.py
```

---

## API Endpoints

| Method | Endpoint | Description | Payload Example |
| :--- | :--- | :--- | :--- |
| **GET** | `/` | Web UI Dashboard | — |
| **GET** | `/health` | Health check and enabled feature list | — |
| **POST** | `/api/travel` | Submit a travel request or initiate a thread | `{"message": "Plan a 4-day trip to Tokyo", "thread_id": null}` |
| **POST** | `/api/travel/approve` | Approve draft itinerary or request revisions | `{"thread_id": "thread-uuid", "approved": true, "feedback": ""}` |

### Example cURL Requests

**Submit Travel Request:**
```bash
curl -X POST http://127.0.0.1:8000/api/travel \
  -H "Content-Type: application/json" \
  -d '{"message": "Plan a 3-day trip to Rome with a budget of $1200"}'
```

**Approve Draft Plan:**
```bash
curl -X POST http://127.0.0.1:8000/api/travel/approve \
  -H "Content-Type: application/json" \
  -d '{"thread_id": "YOUR_THREAD_ID", "approved": true, "feedback": ""}'
```

---

## Configuration & Environment

Create a `.env` file in the project root:

```ini
# =========================
# LLM Provider (Groq)
# =========================
GROQ_API_KEY=your_groq_api_key_here

# =========================
# Database (PostgreSQL Checkpointer)
# =========================
DATABASE_URL=postgresql://user:password@localhost:5432/travel_db?sslmode=require

# =========================
# Search & External APIs
# =========================
TAVILY_API_KEY=your_tavily_api_key_here
OPENWEATHER_API_KEY=your_openweather_api_key_here
AVIATIONSTACK_API_KEY=your_aviationstack_api_key_here
DEFAULT_ORIGIN_IATA=COK

# =========================
# Tracing & Observability (Optional)
# =========================
LANGCHAIN_TRACING_V2=true
LANGCHAIN_ENDPOINT=https://api.smith.langsmith.com
LANGCHAIN_API_KEY=your_langsmith_api_key_here
LANGCHAIN_PROJECT=TripMate-AI
```

---

## Development Notes

- **Asynchronous MCP & Sync Helpers**: The project keeps synchronous convenience wrappers in `backend.py` while running an asynchronous FastAPI server. `nest_asyncio.apply()` is executed in `app.py` to allow synchronous graph execution to call async MCP helpers cleanly.
- **Model Selection & Token Limits**: Groq's free tier enforces rate limits (Tokens Per Minute - TPM) depending on the model. Using `llama-3.3-70b-versatile` provides a generous allowance (12,000+ TPM) for multi-agent workloads. If using larger models such as `openai/gpt-oss-120b`, consider Groq's Developer tier or ensure inputs are sliced/summarized to avoid 413 token limit errors.
- **Database Fallback**: When using `test.py`, an in-memory checkpointer (`MemorySaver`) is activated as a fallback if the remote PostgreSQL database is temporarily offline.

---

## Contributing

Contributions are welcome! Please feel free to open issues or submit pull requests for:
- New specialist agents (e.g., currency converter, packing assistant, car rental)
- Additional MCP tools and server integrations
- Frontend enhancements and UI improvements
- Documentation and test coverage

---

## License

This project is licensed under the terms of the [MIT License](LICENSE).

---

## Acknowledgements

- Built as a demonstration of **LangGraph** multi-agent coordination patterns.
- Powered by the **Model Context Protocol (MCP)** specification.
- Fast LLM inference powered by **Groq**.
- Web search capabilities powered by **Tavily**.
- Flight and weather data provided by **AviationStack** and **OpenWeatherMap**.

---

## Contact

For questions, issues, or suggestions, please open an issue in the repository or contact the project maintainer.
