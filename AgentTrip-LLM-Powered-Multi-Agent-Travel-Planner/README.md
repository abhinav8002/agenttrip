# ✈️ AgentTrip: LLM-Powered Multi-Agent Travel Planner

A production-ready **multi-agent AI travel planning system** built with **LangGraph**, **LangChain**, and **MCP (Model Context Protocol)**. AgentTrip uses a supervisor-orchestrated architecture where specialized AI agents collaborate to create comprehensive travel plans — complete with flights, hotels, weather forecasts, budgets, and day-by-day itineraries.

---

## 🏗️ Architecture

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

---

## ✨ Features

| Feature | Description |
|---|---|
| **Multi-Agent Architecture** | Supervisor agent dynamically routes tasks to specialist agents via LangGraph |
| **MCP Integration** | AviationStack (flights), OpenWeatherMap (weather), Tavily (hotel search) via Model Context Protocol |
| **Input Guardrails** | LLM-powered validation rejects non-travel queries before processing |
| **Human-in-the-Loop** | Draft itinerary is presented for human approval/revision before finalizing |
| **PostgreSQL Memory** | Persistent conversation state with LangGraph checkpoint support |
| **Streamlit Web App** | Interactive UI with session management, thread support, and live agent output |

---

## 📋 Prerequisites

### API Keys (all free tier available)

| Service | Purpose | Get Key |
|---|---|---|
| **Groq** | LLM inference (Llama 3.3 70B) | [console.groq.com](https://console.groq.com) |
| **Tavily** | Hotel & accommodation search | [tavily.com](https://www.tavily.com/) |
| **AviationStack** | Airport & airline data | [aviationstack.com](https://aviationstack.com/) |
| **OpenWeatherMap** | Current weather & forecasts | [openweathermap.org](https://openweathermap.org/) |

### Software

- **Python 3.10+**
- **PostgreSQL** — [Download](https://www.postgresql.org/download/)
- **UV** (for AviationStack MCP server) — `pip install uv`

---

## 🚀 Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/<your-username>/AgentTrip-LLM-Powered-Multi-Agent-Travel-Planner.git
cd AgentTrip-LLM-Powered-Multi-Agent-Travel-Planner
```

### 2. Create & Activate Virtual Environment

```bash
python -m venv venv
```

**Windows:**
```bash
venv\Scripts\activate
```

**macOS/Linux:**
```bash
source venv/bin/activate
```

### 3. Install Dependencies

```bash
pip install -r requirements.txt
```

### 4. Configure Environment Variables

```bash
cp .env.example .env
```

Edit `.env` and fill in your API keys:

```env
GROQ_API_KEY=your_groq_api_key_here
TAVILY_API_KEY=your_tavily_api_key_here
AVIATION_STACK_API_KEY=your_aviationstack_api_key_here
OPENWEATHER_API_KEY=your_openweathermap_api_key_here
DATABASE_URL=postgresql://postgres:your_password@localhost:5432/langgraph_memory_demo
```

### 5. Setup PostgreSQL Database

Open your PostgreSQL client (pgAdmin or psql) and run:

```sql
CREATE DATABASE langgraph_memory_demo;
```

### 6. Setup AviationStack MCP Server

```bash
git clone https://github.com/Pradumnasaraf/aviationstack-mcp.git
cd aviationstack-mcp
```

Create a `.env` file inside this directory:

```env
AVIATION_STACK_API_KEY=your_api_key_here
```

Install and activate:

```bash
uv sync
.venv\Scripts\activate        # Windows
# source .venv/bin/activate   # macOS/Linux
```

Start the MCP server (keep this running):

```bash
uv run -m aviationstack_mcp mcp run
```

### 7. Run the Application

```bash
streamlit run frontend.py
```

The app will open at `http://localhost:8501`.

---

## 💡 Example Prompts

```
Plan a complete 7-day Japan trip including flights, hotels and sightseeing under 2 lakhs.
```

```
I want a 5-day budget trip to Bali from Delhi. Prefer beach resorts and no red-eye flights.
```

```
Plan a weekend getaway to Goa for 2 people with a budget of ₹30,000.
```

---

## 📁 Project Structure

```
AgentTrip-LLM-Powered-Multi-Agent-Travel-Planner/
├── agents.py              # All specialist agents (supervisor, flight, hotel, weather, budget, itinerary, approval, final)
├── config.py              # Environment config & LLM initialization
├── frontend.py            # Streamlit web application
├── graph.py               # LangGraph state machine & routing logic
├── mcp_client.py          # MCP client for external tool integrations
├── state.py               # TravelState TypedDict definition
├── weather_mcp_server.py  # Custom Weather MCP server (OpenWeatherMap)
├── requirements.txt       # Python dependencies
├── .env.example           # Template for environment variables
├── .gitignore             # Git ignore rules
└── README.md              # This file
```

---

## ⚙️ Configuration Notes

### MCP Client Paths

The `mcp_client.py` file contains paths for local MCP server executables. **You must update these paths** to match your local setup:

- `aviationstack` → path to aviationstack-mcp `.venv/Scripts/python.exe`
- `weather` → path to your Python executable and `weather_mcp_server.py`

### Groq Model

The default model is `llama-3.3-70b-versatile`. You can change it by setting `GROQ_MODEL` in your `.env` file:

```env
GROQ_MODEL=llama-3.3-70b-versatile
```

---

## 🛡️ Guardrails

AgentTrip includes an **input guardrail** that validates every user request before processing. Non-travel-related queries are rejected early, saving API calls and preventing misuse.

--
