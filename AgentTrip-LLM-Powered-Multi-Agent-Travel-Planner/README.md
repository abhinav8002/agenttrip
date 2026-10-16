# ✈️ AgentTrip: LLM-Powered Multi-Agent Travel Planner

[![Python 3.10+](https://img.shields.io/badge/Python-3.10%2B-blue.svg)](https://www.python.org/downloads/)
[![LangGraph](https://img.shields.io/badge/LangGraph-1.2.0-orange.svg)](https://langchain-ai.github.io/langgraph/)
[![LangChain](https://img.shields.io/badge/LangChain-1.3.1-green.svg)](https://www.langchain.com/)
[![MCP](https://img.shields.io/badge/MCP-1.27.2-purple.svg)](https://modelcontextprotocol.io/)
[![Streamlit](https://img.shields.io/badge/Streamlit-1.57.0-FF4B4B.svg)](https://streamlit.io/)
[![Groq](https://img.shields.io/badge/LLM-Llama_3.3_70B_(Groq)-teal.svg)](https://groq.com/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-Checkpointer-336791.svg)](https://www.postgresql.org/)

**AgentTrip** is a production-ready **multi-agent AI travel planning system** built with **LangGraph**, **LangChain**, and the **Model Context Protocol (MCP)**. 

Using a supervisor-orchestrated state graph, AgentTrip dynamically routes user travel requests across specialized AI agents to research real-time flights, hotels, weather forecasts, and budgets—synthesizing everything into a comprehensive day-by-day itinerary with **Human-in-the-Loop (HITL)** approval and **PostgreSQL** persistent memory.

---

## 🌟 Key Features

* **🧠 Dynamic Supervisor Orchestration:** The `supervisor_agent` analyzes user prompts, extracts structured trip constraints (`destination`, `origin`, `duration`, `budget`, `travel_style`, `special_preferences`), and dynamically selects only the required specialist agents.
* **🛡️ LLM Input Guardrails:** Built-in input validation intercepts and blocks non-travel-related queries before invoking downstream agents or external APIs, saving tokens and preventing misuse.
* **🔌 Multi-Server MCP Integration:** Connects seamlessly to three external tool servers via `langchain-mcp-adapters`:
  * **AviationStack MCP (`stdio`):** Fetches real-time airport (`list_airports`) and airline (`list_airlines`) data.
  * **Custom Weather MCP Server (`stdio`):** Built with `FastMCP` (`weather_mcp_server.py`) wrapping the OpenWeatherMap API for current conditions (`get_current_weather`) and 5-step forecasts (`get_forecast`).
  * **Tavily Remote MCP (`streamable_http`):** Performs live web searches (`tavily_search`) for hotel recommendations and neighborhood guides.
* **👤 Human-in-the-Loop (HITL) Workflow:** Uses LangGraph's `interrupt()` and `Command(resume=...)` primitives to pause execution after generating a draft itinerary, allowing users to approve the plan or submit revision feedback before finalization.
* **💾 Persistent PostgreSQL State Memory:** Powered by `PostgresSaver` (`langgraph-checkpoint-postgres`), enabling multi-turn session persistence and isolated thread management per user.
* **🖥️ Interactive Streamlit UI:** Clean two-column dashboard displaying supervisor reasoning, live intermediate outputs from each specialist agent, interactive approval controls, and the polished final travel plan.

---

## 🏗️ System Architecture & Execution Flow

```
                    ┌──────────────┐
                    │   User UI    │
                    │  (Streamlit) │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │  Supervisor  │  ← Input Guardrail
                    │    Agent     │
                    └──────┬───────┘
                           │ routes to selected agents
          ┌────────┬───────┼───────┬────────┐
          ▼        ▼       ▼       ▼        ▼
     ┌─────────┐┌───────┐┌──────┐┌───────┐┌──────────┐
     │ Flight  ││ Hotel ││Weather││Budget ││Itinerary │
     │  Agent  ││ Agent ││Agent ││ Agent ││  Agent   │
     └────┬────┘└───┬───┘└──┬───┘└───┬───┘└────┬─────┘
          │         │       │        │          │
     MCP: Aviation  │  MCP: Weather  │     LLM Summary
     Stack API   MCP:Tavily  API   LLM Analysis
                    │                │
                    ▼                ▼
              ┌──────────────────────────┐
              │   Human-in-the-Loop      │
              │   Approval / Feedback    │
              └────────────┬─────────────┘
                           ▼
              ┌──────────────────────────┐
              │   Final Response Agent   │
              │   (Polished Travel Plan) │
              └──────────────────────────┘
```



### How Routing Works (`graph.py`)
1. **Guardrail & Planning:** `supervisor_agent` validates the query. If valid, it outputs a JSON list of `selected_agents` and `trip_constraints`.
2. **Sequential Filtered Execution:** `graph.py` enforces a strict logical execution order via `AGENT_ORDER`:
   `flight_agent` ➔ `hotel_agent` ➔ `weather_agent` ➔ `budget_agent` ➔ `itinerary_agent`
   Using `route_from_supervisor` and `route_after_agent`, the graph dynamically hops only through the agents chosen by the supervisor, ensuring downstream agents (like `budget_agent` and `itinerary_agent`) have full context from upstream tool calls.
3. **Interrupt & Resume:** `human_approval_agent` triggers a LangGraph `interrupt()`. Once the user submits approval or revision notes in the Streamlit UI, `Command(resume=...)` passes control to `final_response_agent`.

---

## 📂 Deep-Dive Module Breakdown

```text
AgentTrip-LLM-Powered-Multi-Agent-Travel-Planner/
├── agents.py              # 8 core node functions (Supervisor, Specialists, HITL, Final)
├── config.py              # Environment variable loader & ChatGroq LLM initialization
├── frontend.py            # Streamlit web UI with thread management & HITL controls
├── graph.py               # LangGraph StateGraph definition, conditional edges & PostgresSaver
├── mcp_client.py          # MultiServerMCPClient setup & async tool wrappers
├── state.py               # TravelState TypedDict schema shared across all graph nodes
├── weather_mcp_server.py  # Custom FastMCP server exposing OpenWeatherMap tools
├── requirements.txt       # Pinned Python package dependencies
├── .env.example           # Environment variable template
└── README.md              # Project documentation
```

### Shared State Schema (`state.py`)
All agents read from and write to `TravelState`:
| Key | Type | Description |
| :--- | :--- | :--- |
| `messages` | `Annotated[list[AnyMessage], operator.add]` | Append-only message history across graph execution |
| `user_id` / `user_query` | `str` | Active user identifier and raw natural language prompt |
| `trip_constraints` | `dict[str, Any]` | Extracted JSON (`destination`, `origin`, `duration`, `budget`, `travel_style`, `special_preferences`) |
| `selected_agents` | `list[str]` | Specialist agents activated by the supervisor |
| `supervisor_reasoning` | `str` | Explanation of why specific agents were chosen (or guardrail rejection reason) |
| `flight_results` | `str` | Synthesized airline, airport, fare, and routing advice |
| `hotel_results` | `str` | Raw/synthesized accommodation data from Tavily MCP |
| `weather_results` | `str` | Current weather + 5-entry forecast from Weather MCP |
| `budget_results` | `str` | Cost breakdown, risk areas, money-saving tips, and feasibility verdict |
| `itinerary` | `str` | Structured draft travel plan generated prior to human review |
| `approval_request` | `str` | Formatted prompt presented during HITL interrupt |
| `approved` / `human_feedback`| `bool` / `str` | User decision and optional revision instructions from the UI |
| `final_response` | `str` | Final polished travel plan Incorporating human feedback |
| `llm_calls` | `int` | Counter tracking total LLM invocations per run |

---

## 📋 Prerequisites

### 1. API Keys (Free Tiers Available)
| Service | Purpose | Get API Key |
| :--- | :--- | :--- |
| **Groq** | Fast LLM inference (`llama-3.3-70b-versatile`) | [console.groq.com](https://console.groq.com) |
| **Tavily** | Remote MCP search for hotels & stays | [tavily.com](https://www.tavily.com/) |
| **AviationStack** | Real-time airport & airline data | [aviationstack.com](https://aviationstack.com/) |
| **OpenWeatherMap** | Current weather & forecast data | [openweathermap.org](https://openweathermap.org/) |

### 2. Required Software
* **Python 3.10+**
* **PostgreSQL** (Optional but recommended for persistent checkpointing; falls back to in-memory compilation if `DATABASE_URL` is omitted) — [Download PostgreSQL](https://www.postgresql.org/download/)
* **uv** (Required to run the external `aviationstack-mcp` server) — `pip install uv`

---

## 🚀 Installation & Setup Guide

### Step 1: Clone the Repository
```bash
git clone [https://github.com/](https://github.com/)<your-username>/AgentTrip-LLM-Powered-Multi-Agent-Travel-Planner.git
cd AgentTrip-LLM-Powered-Multi-Agent-Travel-Planner
```

### Step 2: Create & Activate Virtual Environment
```bash
python -m venv venv

# Windows:
venv\Scripts\activate

# macOS / Linux:
source venv/bin/activate
```

### Step 3: Install Python Dependencies
```bash
pip install -r requirements.txt
```

### Step 4: Set Up the External `aviationstack-mcp` Server
By default, `mcp_client.py` expects `aviationstack-mcp` to be cloned alongside the project directory (or configured via environment variables):

```bash
cd ..
git clone [https://github.com/Pradumnasaraf/aviationstack-mcp.git](https://github.com/Pradumnasaraf/aviationstack-mcp.git)
cd aviationstack-mcp
uv sync
cd ../AgentTrip-LLM-Powered-Multi-Agent-Travel-Planner
```

### Step 5: Configure Environment Variables
Copy the example environment file and populate your keys:
```bash
cp .env.example .env
```

Update `.env` with your credentials and local executable paths:
```env
# ─── Core API Keys ───
GROQ_API_KEY=your_groq_api_key_here
TAVILY_API_KEY=your_tavily_api_key_here
AVIATION_STACK_API_KEY=your_aviationstack_api_key_here
OPENWEATHER_API_KEY=your_openweathermap_api_key_here

# ─── Optional LLM Override (Default: llama-3.3-70b-versatile) ───
GROQ_MODEL=llama-3.3-70b-versatile

# ─── PostgreSQL Checkpointer Connection ───
DATABASE_URL=postgresql://postgres:your_password@localhost:5432/langgraph_memory_demo

# ─── Optional Custom MCP Executable Paths (Overrides defaults in mcp_client.py) ───
# Windows defaults:
# AVIATIONSTACK_PYTHON_PATH=../aviationstack-mcp/.venv/Scripts/python.exe
# WEATHER_PYTHON_PATH=../venv/Scripts/python.exe
# WEATHER_SERVER_SCRIPT_PATH=./weather_mcp_server.py
```
> **💡 macOS/Linux Note:** On Unix systems, virtual environment Python binaries live inside `.venv/bin/python` rather than `.venv/Scripts/python.exe`. Set `AVIATIONSTACK_PYTHON_PATH` and `WEATHER_PYTHON_PATH` in your `.env` file accordingly.

### Step 6: Initialize PostgreSQL Database
Open `psql` or **pgAdmin** and create the checkpoint database:
```sql
CREATE DATABASE langgraph_memory_demo;
```
*(LangGraph's `checkpointer.setup()` in `graph.py` automatically creates the required checkpoint tables on startup.)*

### Step 7: Launch the Streamlit Application
```bash
streamlit run frontend.py
```
Open your browser at **`http://localhost:8501`**.

---

## 💬 Example Prompts to Try

**Full Multi-Agent Trip Plan:**
> *"Plan a complete 7-day Japan trip from Mumbai including flights, hotels, weather advice, and sightseeing under ₹2,00,000. Prefer budget hotels near transit and no overnight flights."*

**Beach Getaway with Specific Constraints:**
> *"I want a 5-day budget trip to Bali from Delhi. Prefer beach resorts in Seminyak or Uluwatu, pack for current weather, and keep total spend under $1,200."*

**Weekend Domestic Trip:**
> *"Plan a 3-day weekend getaway to Goa for 2 people with a budget of ₹30,000 focusing on South Goa stays and local food."*

**Guardrail Test (Non-Travel Query):**
> *"Write a Python script to train a convolutional neural network on MNIST."*
> *(Result: Blocked immediately by `supervisor_agent` input guardrail without calling specialist agents.)*

---

## 🔧 Troubleshooting

* **`Tool '...' not found` or Sub-Exceptions in `mcp_client.py`:**
  * Verify that `AVIATIONSTACK_PYTHON_PATH` and `WEATHER_PYTHON_PATH` point to valid Python executables containing the `aviationstack-mcp` and `mcp` packages respectively.
  * Ensure your `TAVILY_API_KEY`, `AVIATION_STACK_API_KEY`, and `OPENWEATHER_API_KEY` are active and free of trailing spaces in `.env`.
* **`psycopg.OperationalError` on Startup:**
  * Ensure PostgreSQL service is running on port `5432` and the database `langgraph_memory_demo` exists. If you want to test without PostgreSQL, leave `DATABASE_URL=` blank in `.env` to run with LangGraph's in-memory graph compiler.

---

