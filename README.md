# Coming Soon AI Security Testing Platform

**Human-supervised, AI-assisted security testing. Every action requires explicit operator approval.**

-----

## Folder Structure

```
ai-security-platform/
│
├── api/main.py                  ← FastAPI app (10 endpoints)
├── agents/
│   ├── recon_agent.py           ← Gate 2: tool request generator
│   ├── payload_agent.py         ← RAG-powered payload suggestions
│   └── analyzer_agent.py        ← ML + LLM findings analysis
├── core/
│   ├── orchestrator.py          ← State machine + approval gates
│   ├── pipeline.py              ← Background workflow engine
│   ├── planner.py               ← Gate 1: attack plan generator
│   ├── decision.py              ← Gate 3: next-step recommender
│   └── execution_runner.py      ← Concurrent tool dispatcher
├── config/
│   ├── llm_client.py            ← Ollama/Anthropic/Stub + cache
│   ├── prompts.py               ← All system prompts (centralised)
│   ├── output_validator.py      ← Per-agent Pydantic schemas
│   ├── schemas.py               ← Shared data models
│   ├── settings.py              ← All env-var config
│   └── logging_config.py        ← Structured JSON logging
├── ml/
│   ├── anomaly.py               ← IsolationForest
│   ├── classifier.py            ← RandomForest (9 labels)
│   ├── predictor.py             ← GradientBoosting (exploit prob)
│   └── behavior.py              ← LSTM sequence analysis
├── rag/
│   ├── indexer.py               ← FAISS / keyword KB
│   └── retriever.py             ← Ranked chunk selection
├── tools/
│   ├── _base.py                 ← Async executor + gate guard
│   ├── nmap.py / httpx.py / ffuf.py / nuclei.py
├── utils/
│   ├── feature_extractor.py     ← Tool output → ML vectors
│   ├── validator.py             ← Finding validation + FP scoring
│   └── feedback.py              ← Payload feedback storage
├── data/                        ← Runtime data (gitignore this)
├── logs/                        ← Rotating log files
├── scripts/
│   ├── start.sh                 ← Start server (gunicorn/uvicorn)
│   ├── test.sh                  ← Smoke tests
│   ├── poll.sh                  ← Watch scan progress
│   └── debug.sh                 ← Full system diagnostics
├── .env                         ← Configuration (copy from .env.example)
└── requirements.txt
```

-----

## Part 1 — Kali Linux Setup (Step by Step)

### Prerequisites

|Item      |Minimum                |
|----------|-----------------------|
|Kali Linux|2023.x or 2024.x       |
|RAM       |4 GB (8 GB recommended)|
|Disk      |10 GB free             |
|Python    |3.10+ (3.11 preferred) |

-----

### Step 1 — Get the project files

```bash
# Copy from USB / shared folder:
cp -r /media/sf_shared/security_platform ~/ai-security-platform

# OR extract from archive:
tar xzf security_platform.tar.gz -C ~/
mv security_platform ~/ai-security-platform

cd ~/ai-security-platform
```

-----

### Step 2 — System packages

```bash
sudo apt update
sudo apt install -y \
    python3 python3-pip python3-venv python3-dev \
    build-essential gcc cmake libssl-dev libffi-dev \
    nmap curl wget unzip git jq
```

-----

### Step 3 — Install security tools

```bash
# httpx (ProjectDiscovery version — NOT Kali's httpx)
wget https://github.com/projectdiscovery/httpx/releases/latest/download/httpx_linux_amd64.zip
unzip httpx_linux_amd64.zip && sudo mv httpx /usr/local/bin/httpx && rm *.zip

# ffuf
sudo apt install -y ffuf
# OR: wget ffuf release, tar xzf, sudo mv ffuf /usr/local/bin/

# nuclei
wget https://github.com/projectdiscovery/nuclei/releases/latest/download/nuclei_linux_amd64.zip
unzip nuclei_linux_amd64.zip && sudo mv nuclei /usr/local/bin/nuclei && rm *.zip
nuclei -update-templates

# Verify all tools
for t in nmap httpx ffuf nuclei; do command -v $t && echo "✓ $t" || echo "✗ $t"; done
```

-----

### Step 4 — Install Ollama (local LLM)

```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Start the service
ollama serve &

# Pull a model (pick one):
ollama pull llama3        # best quality (~4.7 GB)
ollama pull mistral       # faster (~4.1 GB)
ollama pull phi3          # smallest (~2.3 GB) — use if RAM < 8 GB

# Test inference
ollama run llama3 "Say PONG"
```

-----

### Step 5 — Python virtual environment

```bash
cd ~/ai-security-platform

python3 -m venv venv
source venv/bin/activate

pip install --upgrade pip wheel
pip install -r requirements.txt

# PyTorch CPU-only (large download — skip if RAM < 6 GB)
pip install torch==2.4.1 --index-url https://download.pytorch.org/whl/cpu
```

-----

### Step 6 — Configure .env

```bash
cp .env.example .env
nano .env
```

**Key settings to verify:**

```bash
LLM_BACKEND=ollama           # or "stub" for offline testing
OLLAMA_MODEL=llama3          # match what you pulled above
API_HOST=0.0.0.0             # 0.0.0.0 = reachable from VM host
API_PORT=8000
HTTPX_PATH=/usr/local/bin/httpx    # important: ProjectDiscovery version
```

-----

### Step 7 — Run diagnostics

```bash
./scripts/debug.sh
```

All items should show ✓ before starting the server.

-----

### Step 8 — Start the server

```bash
# Production (gunicorn + 2 workers):
./scripts/start.sh

# Development (uvicorn --reload, shows live errors):
./scripts/start.sh --dev

# Offline test mode (no Ollama needed, stub responses):
./scripts/start.sh --stub
```

**Expected output:**

```
╔═══════════════════════════════════════════════╗
║   AI Security Testing Platform  v4.4          ║
╚═══════════════════════════════════════════════╝
  API:   http://0.0.0.0:8000
  Docs:  http://0.0.0.0:8000/docs
```

-----

## Part 2 — Test Workflow

### Quick smoke test

```bash
# In a second terminal:
./scripts/test.sh
```

### Full lifecycle test

```bash
./scripts/test.sh --full
```

-----

## Part 3 — Complete Scan Walkthrough (curl)

### Start a scan

```bash
RESP=$(curl -s -X POST http://localhost:8000/scan/start \
  -H "Content-Type: application/json" \
  -d '{
    "host": "127.0.0.1",
    "ports": [80, 443, 8080],
    "scope": ["/api", "/admin"],
    "authorization_confirmed": true,
    "notes": "Local test — authorised"
  }')

SCAN_ID=$(echo $RESP | python3 -c "import sys,json; print(json.load(sys.stdin)['scan_id'])")
echo "Scan ID: $SCAN_ID"
```

### Watch progress

```bash
./scripts/poll.sh $SCAN_ID
```

### Gate 1 — Review and approve plan

```bash
# Review the plan
curl -s http://localhost:8000/scan/$SCAN_ID/plan | jq .human_readable

# Approve
curl -s -X POST http://localhost:8000/scan/$SCAN_ID/approve-plan \
  -H "Content-Type: application/json" \
  -d '{"approved": true, "operator_notes": "Plan reviewed — approved"}'
```

### Gate 2 — Review and approve tool execution

```bash
# Review proposed tools (waits for recon to complete)
curl -s http://localhost:8000/scan/$SCAN_ID/tool-requests | jq .

# Approve
curl -s -X POST http://localhost:8000/scan/$SCAN_ID/execute \
  -H "Content-Type: application/json" \
  -d '{"approved": true, "operator_notes": "Tools reviewed — execute"}'
```

### Gate 3 — Review decision and complete

```bash
# Review ML-augmented decision report
curl -s http://localhost:8000/scan/$SCAN_ID/decision | jq .

# Complete scan
curl -s -X POST http://localhost:8000/scan/$SCAN_ID/next-step \
  -H "Content-Type: application/json" \
  -d '{"approved": false, "operator_notes": "complete"}'
```

### Get full results

```bash
curl -s http://localhost:8000/scan/$SCAN_ID/results | jq .
```

-----

## Part 4 — Test Scenarios

### Scenario 1 — Basic scan (127.0.0.1)

Verify the happy path works end-to-end with localhost as a safe target.

```bash
./scripts/test.sh --full
```

Manual variant:

```bash
curl -s -X POST http://localhost:8000/scan/start \
  -H "Content-Type: application/json" \
  -d '{"host":"127.0.0.1","authorization_confirmed":true,"notes":"scenario 1"}'
```

**Expected:** Scan created (HTTP 201), state=planning, plan generated within 30-60s.

-----

### Scenario 2 — Invalid input rejection

Verify the platform rejects unauthorized or malformed requests.

```bash
# Test 1: Missing authorization (must return HTTP 422)
curl -s -w "\n%{http_code}" -X POST http://localhost:8000/scan/start \
  -H "Content-Type: application/json" \
  -d '{"host":"127.0.0.1","authorization_confirmed":false}'

# Test 2: Empty host (must return HTTP 422)
curl -s -w "\n%{http_code}" -X POST http://localhost:8000/scan/start \
  -H "Content-Type: application/json" \
  -d '{"host":"","authorization_confirmed":true}'

# Test 3: Invalid JSON body (must return HTTP 422)
curl -s -w "\n%{http_code}" -X POST http://localhost:8000/scan/start \
  -H "Content-Type: application/json" \
  -d 'not-valid-json'
```

**Expected:** All return HTTP 422 with a validation error message.

-----

### Scenario 3 — LLM failure simulation (kill Ollama)

Verify the system degrades gracefully to the template stub.

```bash
# Step 1: Start a scan normally
SCAN_ID=$(curl -s -X POST http://localhost:8000/scan/start \
  -H "Content-Type: application/json" \
  -d '{"host":"127.0.0.1","authorization_confirmed":true,"notes":"ollama kill test"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['scan_id'])")
echo "Scan: $SCAN_ID"

# Step 2: Kill Ollama mid-scan
pkill -f "ollama" 2>/dev/null; sleep 2
echo "Ollama killed"

# Step 3: Watch what happens — should use stub fallback, NOT crash
./scripts/poll.sh $SCAN_ID --timeout 120

# Step 4: Check results — should see valid=False or degraded=True
curl -s http://localhost:8000/scan/$SCAN_ID/results | \
  python3 -c "import sys,json; d=json.load(sys.stdin); print('state:', d.get('state'))"

# Step 5: Check logs for stub activation
grep "LLM_STUB_ACTIVE\|STUB FALLBACK\|valid=False" logs/platform.log | tail -5

# Step 6: Restart Ollama
ollama serve &
```

**Expected:** Plan is generated with stub output (target_summary starts with “STUB”), scan continues. No crash. Logs show `LLM_STUB_ACTIVE | agent=planner`.

-----

## Part 5 — Troubleshooting

### Server won’t start

```bash
# Check what's on port 8000
sudo lsof -ti:8000 | xargs kill -9 2>/dev/null
# OR change port in .env: API_PORT=8080

# Check Python imports work
source venv/bin/activate
python3 -c "from api.main import app; print('OK')"

# Check for missing packages
pip install -r requirements.txt
```

-----

### LLM not responding

```bash
# Is Ollama running?
curl http://localhost:11434/api/tags

# If not:
ollama serve &
sleep 3
curl http://localhost:11434/api/tags

# Is the model pulled?
ollama list

# If model missing:
ollama pull llama3

# Test inference directly:
curl -s -X POST http://localhost:11434/api/chat \
  -d '{"model":"llama3","messages":[{"role":"user","content":"Say PONG"}],"stream":false}'

# Use stub mode as a workaround:
export LLM_BACKEND=stub
./scripts/start.sh --stub
```

-----

### Tools not found

```bash
# Check which tools are missing
./scripts/debug.sh

# httpx conflict (wrong version):
which httpx                          # shows /usr/bin/httpx (wrong)
/usr/local/bin/httpx -version        # should show ProjectDiscovery version
# Fix in .env:
echo "HTTPX_PATH=/usr/local/bin/httpx" >> .env

# nmap permission denied (SYN scan):
sudo setcap cap_net_raw+eip $(which nmap)
# OR change scan_type to "T" in tool requests (TCP connect, no root needed)
```

-----

### Validation errors in API responses

```bash
# HTTP 422 — check the detail field
curl -s -X POST http://localhost:8000/scan/start \
  -H "Content-Type: application/json" \
  -d '{"host":"bad input"}' | python3 -m json.tool

# Check logs for validation failures
grep "VALIDATION_FAILED\|INVALID_OUTPUT" logs/platform.log | tail -10
```

-----

### Timeout issues

```bash
# Increase tool timeout in .env:
TOOL_TIMEOUT_SECONDS=300

# Increase Ollama timeout:
OLLAMA_TIMEOUT=300

# Check if Ollama is slow (model loading):
time curl -s -X POST http://localhost:11434/api/chat \
  -d '{"model":"llama3","messages":[{"role":"user","content":"hi"}],"stream":false}' | head -c 100

# Use a smaller model if slow:
ollama pull phi3
# Then in .env: OLLAMA_MODEL=phi3
```

-----

### Memory issues (Ollama OOM)

```bash
# Check available RAM
free -h

# Use a smaller model:
ollama pull phi3    # 2.3 GB — works on 4 GB RAM VMs
# In .env: OLLAMA_MODEL=phi3

# Or disable LSTM (reduces ML memory usage):
# In .env:
LSTM_HIDDEN_SIZE=32
LSTM_SEQUENCE_LEN=10
```

-----

## API Reference

|Method|Endpoint                  |Description                        |
|------|--------------------------|-----------------------------------|
|GET   |`/health`                 |Full system health (LLM, tools, ML)|
|GET   |`/docs`                   |Swagger UI                         |
|POST  |`/scan/start`             |Create scan + trigger planning     |
|GET   |`/scan/{id}/plan`         |View plan (Gate 1)                 |
|POST  |`/scan/{id}/approve-plan` |Gate 1 decision                    |
|GET   |`/scan/{id}/tool-requests`|View tool requests (Gate 2)        |
|POST  |`/scan/{id}/execute`      |Gate 2 decision                    |
|GET   |`/scan/{id}/decision`     |View ML+LLM decision (Gate 3)      |
|POST  |`/scan/{id}/next-step`    |Gate 3 decision                    |
|GET   |`/scan/{id}/results`      |Full results                       |
|GET   |`/scan/{id}/status`       |Lightweight status poll            |
|GET   |`/scans`                  |List all scans                     |
|GET   |`/feedback/stats`         |Payload feedback statistics        |
|GET   |`/feedback/recent`        |Recent feedback records            |

-----

## LLM Backend Modes

|Mode       |Config                 |Description                                    |
|-----------|-----------------------|-----------------------------------------------|
|`ollama`   |`LLM_BACKEND=ollama`   |Local Ollama (default, offline-capable)        |
|`anthropic`|`LLM_BACKEND=anthropic`|Anthropic API (needs internet + key)           |
|`stub`     |`LLM_BACKEND=stub`     |Template fallback, `valid=False`, no LLM needed|

The system automatically cascades: Ollama → Anthropic → Stub.
`valid=False` in any response means stub was activated.
