# AGENT SANDBOX API — Full Project Plan

## Overview

A REST API that exposes a persistent, isolated bash shell to an AI agent as a tool. Each caller gets their own stateful session (preserved `cwd`, env vars, installed packages). The machine comes pre-loaded with Python, data science libraries, and stock data tooling. Trained models are saved to disk with SQLite-tracked metadata.

---

## Stack

| Layer | Choice | Reason |
|---|---|---|
| Language | Go | Low overhead, excellent concurrency, fast process I/O |
| HTTP framework | Gin | Minimal, fast router; great middleware support |
| Session state | In-memory (`sync.Map`) | Simple, zero dependencies, resets cleanly on restart |
| Model metadata | SQLite (`modernc.org/sqlite`) | Embedded, no server needed, queryable |
| Python runtime | System Python 3.11 | Pre-installed via `setup.sh` |
| Config / secrets | `.env` via `godotenv` | Injected into every bash session at startup |

---

## Project Structure

```
agent-sandbox-api/
├── main.go              # Gin server, routes, startup
├── session.go           # Session manager & bash process lifecycle
├── models.go            # SQLite model registry (save, list, fetch)
├── middleware.go        # Auth (API key header), request logging
├── setup.sh             # One-shot host bootstrap script
├── requirements.txt     # Pinned Python packages
├── .env.example         # API key template
├── models/              # Saved .pkl / .pt model files (gitignored)
├── models.db            # SQLite model metadata (gitignored)
└── README.md
```

---

## API Endpoints

### `POST /cli`

Run a command inside a persistent bash session.

**Request**
```json
{
  "session_id": "agent-42",
  "command": "python3 analysis.py --ticker AAPL"
}
```

**Response**
```json
{
  "stdout": "...",
  "stderr": "",
  "exit_code": 0,
  "session_id": "agent-42"
}
```

- If `session_id` does not exist, a new bash session is created automatically.
- The session's `cwd`, environment variables, and any `pip install`ed packages persist across calls.
- Commands time out after a configurable deadline (default: 5 minutes).
- Output is returned in full when the command exits. For long-running jobs (model training), use the streaming variant below.

---

### `POST /cli/stream`

Same as `POST /cli` but streams output as Server-Sent Events (SSE). Useful for long-running training jobs.

**Response** (SSE stream)
```
data: {"type":"stdout","line":"Epoch 1/50 — loss: 0.4231"}
data: {"type":"stdout","line":"Epoch 2/50 — loss: 0.3891"}
data: {"type":"exit","exit_code":0}
```

---

### `DELETE /sessions/:id`

Kill and clean up a session. Terminates the bash process and frees memory.

**Response**
```json
{ "deleted": true, "session_id": "agent-42" }
```

---

### `GET /sessions`

List all active sessions with metadata.

**Response**
```json
[
  {
    "session_id": "agent-42",
    "created_at": "2025-03-30T10:00:00Z",
    "last_used": "2025-03-30T10:45:00Z",
    "command_count": 17
  }
]
```

---

### `GET /models`

List all saved models from the SQLite registry.

**Response**
```json
[
  {
    "id": 1,
    "name": "aapl_lstm_v2",
    "path": "./models/aapl_lstm_v2.pt",
    "type": "pytorch",
    "metrics": { "val_loss": 0.031, "val_mae": 0.018 },
    "created_at": "2025-03-30T11:20:00Z"
  }
]
```

---

### `POST /models/register`

Register a model that the agent saved to disk, so it appears in the registry.

**Request**
```json
{
  "name": "aapl_lstm_v2",
  "path": "./models/aapl_lstm_v2.pt",
  "type": "pytorch",
  "metrics": { "val_loss": 0.031 }
}
```

---

## Session Lifecycle

```
POST /cli { session_id: "x" }
    │
    ├─ session exists? ──YES──▶ pipe command to existing bash process
    │
    └─ NO ──▶ spawn bash -i
               └─ source .env
               └─ export API keys
               └─ set working dir
               └─ register in session map
               └─ pipe command
```

Sessions are automatically reaped after **30 minutes of inactivity** (configurable via `SESSION_TTL_MINUTES` in `.env`). A background goroutine sweeps expired sessions every 5 minutes.

---

## Pre-installed Environment (`setup.sh`)

The bootstrap script installs everything needed on a fresh Ubuntu/Debian machine.

### System packages
```
python3.11, python3-pip, python3-venv
build-essential, git, curl, wget
sqlite3
```

### Python packages (`requirements.txt`)
```
# Data
pandas
numpy
pyarrow

# Finance / stock data
pandas-datareader
ta              # technical analysis indicators
alpaca-py==0.43.2


# ML / modelling
scikit-learn
xgboost
lightgbm
torch
torchvision

# Model persistence
joblib

# Visualisation (agent can generate charts)
matplotlib
seaborn

# Utilities
python-dotenv
requests
tqdm
```

---

## Environment Variables (`.env.example`)

```bash
# Server
PORT=8080
API_KEY=your-server-api-key-here   # agents must send X-API-Key header
SESSION_TTL_MINUTES=30
COMMAND_TIMEOUT_SECONDS=300

# Stock data APIs (injected into every bash session)
# Apca Api Key Id (string)
APCA_API_KEY_ID=<string>
# APCA API SECRET key (string)
APCA_API_SECRET_KEY=<string>

# Model storage
MODELS_DIR=./models
MODELS_DB=./models.db
```

All variables defined in `.env` are automatically exported into every new bash session so Python scripts can access them via `os.environ`.

---

## Authentication

Every request must include the header:
```
X-API-Key: <your-server-api-key>
```

The key is set in `.env`. Requests without it return `401 Unauthorized`. This is intentionally simple — the API is designed to be called by a single trusted agent, not exposed publicly.

---

## Model Saving Convention

The agent saves models using standard Python conventions. The API then registers them via `POST /models/register`.

**scikit-learn**
```python
import joblib
joblib.dump(model, "./models/my_model.pkl")
```

**PyTorch**
```python
torch.save(model.state_dict(), "./models/my_model.pt")
```

The `models/` directory is on the host filesystem and persists across server restarts.

---

## Implementation Plan

### Phase 1 — Core server
- [ ] `main.go`: Gin setup, route registration, `.env` loading, graceful shutdown
- [ ] `session.go`: bash process spawning, stdin/stdout piping, session map, TTL reaper
- [ ] `POST /cli` endpoint: full-output mode
- [ ] `DELETE /sessions/:id` endpoint
- [ ] `GET /sessions` endpoint
- [ ] `middleware.go`: API key auth, request/response logging

### Phase 2 — Streaming + models
- [ ] `POST /cli/stream`: SSE streaming output
- [ ] `models.go`: SQLite schema, insert, list, fetch
- [ ] `POST /models/register` endpoint
- [ ] `GET /models` endpoint

### Phase 3 — Setup & packaging
- [ ] `setup.sh`: full bootstrap script (Go install, Python, pip packages, systemd service)
- [ ] `requirements.txt`: pinned versions
- [ ] `.env.example`: documented template
- [ ] `README.md`: install, run, usage examples

---

## Security Notes

- The API gives the caller **full bash access**. It must never be exposed to the public internet without strong auth.
- Run the server process as a **non-root user** with a restricted home directory.
- Consider wrapping each bash session in a **Docker container** or `systemd-nspawn` jail if the agent will run untrusted code.
- The API key in `.env` should be rotated regularly.

---

## Running Locally

```bash
# 1. Bootstrap the machine (one time)
chmod +x setup.sh && ./setup.sh

# 2. Copy and fill in your env vars
cp .env.example .env && nano .env

# 3. Start the server
go run .

# 4. Test it
curl -X POST http://localhost:8080/cli \
  -H "X-API-Key: your-key" \
  -H "Content-Type: application/json" \
  -d '{"session_id":"test","command":"python3 -c \"import yfinance; print(yfinance.Ticker(\\\"AAPL\\\").info[\\\"currentPrice\\\"])\"" }'
```