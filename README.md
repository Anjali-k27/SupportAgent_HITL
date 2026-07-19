# Enterprise AI Support Platform
### Session 11 of 12 — Interruptions & Breakpoints

A LangGraph-powered multi-agent customer support system with a **human-in-the-loop (HITL) approval gate**. Before any GitHub issue is created, a human reviewer can approve, deny, or edit the AI-generated draft. The graph pauses mid-execution and resumes only after an explicit decision.

---

## What's New in Session 11

- Graph execution **pauses** inside `github_tool_node` via `interrupt()` before any write occurs
- Human reviewer sees the AI-drafted issue title, body, and severity in the UI
- Three approval actions: **Approve** (create as-is), **Deny** (skip write, log decision), **Edit & Approve** (modify then create)
- New API endpoints: `GET /api/pending-approvals`, `POST /api/approve/{thread_id}`, `POST /api/deny/{thread_id}`, `POST /api/edit-approve/{thread_id}`
- Approval queue sidebar polls every 30 seconds and shows all threads awaiting review

---

## Setup

**1. Clone the repository**
```bash
git clone https://github.com/waseemkhan606/phase3-session11
cd phase3-session11
```

**2. Create and activate a virtual environment**
```bash
python -m venv .venv
source .venv/bin/activate        # macOS/Linux
# .venv\Scripts\activate         # Windows
```

**3. Install dependencies**
```bash
pip install -r requirements.txt
```

**4. Download the spaCy language model** (required by Presidio)
```bash
python -m spacy download en_core_web_lg
```

**5. Set environment variables**

Create a `.env` file in the project root:
```
GOOGLE_API_KEY=your-gemini-api-key-here
```

For real GitHub issue creation (optional — system works in mock mode without these):
```
GITHUB_TOKEN=ghp_your_personal_access_token
GITHUB_REPO=your-org/your-repo
```

Get a Gemini key at: https://aistudio.google.com/app/apikey  
Generate a GitHub token: Settings → Developer settings → Personal access tokens → scope: `repo`

---

## Running

**Start the web server**
```bash
python api.py
```
Open http://localhost:8000 in your browser.

**Run the CLI verification suite**
```bash
python support_agent.py
```
Expected: `SESSION 11 COMPLETE — 5/5 checks passed`

---

## How It Works — Step by Step

### Step 1: Ticket arrives → Security scan

Every ticket enters `ingress_node` first.

- **PII detection**: Presidio scans for credit cards, emails, phone numbers, SSNs, IBANs, IPs, and names. Matches are anonymized before any LLM sees the text.
- **Injection detection**: 14 regex patterns block prompt-injection attempts (e.g. "ignore all previous instructions").
- If either check fails, the ticket is routed to `blocked_response_node` and the user gets a refusal. Safe tickets continue.

### Step 2: Classification

`classify_node` sends the sanitized text to Gemini and categorizes it as one of: `technical`, `billing`, `fraud`, or `general`.

- **technical / fraud / multi-issue** → `dispatcher` (parallel path)
- **billing / general** → `supervisor` (hub-and-spoke orchestrator path)

### Step 3a: Parallel specialist analysis (technical path)

`dispatcher` uses LangGraph's **Send API** to fan out to up to three agents simultaneously in a single superstep:

| Agent | Scoped tool | What it looks for |
|-------|-------------|-------------------|
| `tech_analysis_agent` | `search_knowledge_base` | Outages, bugs, API errors — severity: none / low / medium / high / critical |
| `billing_analysis_agent` | `get_customer_details` | Payment failures, subscription issues |
| `fraud_analysis_agent` | `check_fraud_signals` | Unauthorized access, suspicious activity |

Each agent writes a structured finding dict to `internal_notes` (append-only via `operator.add` reducer). `tech_analysis_agent` also populates `github_draft` when severity is `high` or `critical`.

### Step 4: Human approval gate (Session 11)

When `tech_analysis_agent` sets `github_draft`, the graph reaches `github_tool_node`. Instead of writing immediately, the node calls:

```python
decision = interrupt({'draft': draft})
```

Execution **pauses here**. `graph.invoke()` returns to the caller with the graph in a suspended state. `snapshot.next` contains `'github_tool_node'`, which signals that a human decision is pending.

The UI detects `interrupted: true` in the `/api/run` response and shows the **Human Approval Gate** panel with the draft title, body, severity, and three buttons:

- **Approve** — calls `POST /api/approve/{thread_id}` → `graph.invoke(Command(resume={'approved': True}), config)` → `interrupt()` returns `{'approved': True}` → issue created
- **Deny** — calls `POST /api/deny/{thread_id}` → `graph.invoke(Command(resume={'approved': False}), config)` → `interrupt()` returns `{'approved': False}` → write skipped, denial logged in `internal_notes`
- **Edit & Approve** — calls `POST /api/edit-approve/{thread_id}` with edited title/body → `graph.invoke(Command(resume={'approved': True, 'edited_draft': {...}}), config)` → issue created with corrected content

### Step 5: GitHub issue creation (if approved)

`github_tool_node` resumes after `interrupt()` returns. With `approved=True`:

1. An idempotency key is generated: `SHA-256(thread_id + title + body)`. Same inputs always produce the same key, so retries never create duplicate issues.
2. The GitHub API is called (`POST /repos/{owner}/{repo}/issues`).
3. Without `GITHUB_TOKEN` / `GITHUB_REPO` set, the system runs in **mock mode** — all logic executes and a mock URL is returned, no real issue is created.
4. The resulting URL is stored in `github_issue_url` state.

### Step 6: Synthesis

`synthesizer` waits for all parallel agents (and `github_tool_node`) to complete, then:

- Filters `internal_notes` to relevant findings (severity ≠ none)
- Generates a single coherent customer-facing response via Gemini
- Includes the GitHub issue URL if one was created

### Step 7b: Supervisor path (billing / general)

For non-technical tickets, a hub-and-spoke orchestrator runs:

1. `supervisor` reads the ticket and decides which worker to call next
2. Workers: `tech_support_subgraph` (ReAct agent with tool loop), `fraud_handler`, `general_handler`
3. Supervisor keeps delegating (up to `MAX_DELEGATIONS=5`) until the ticket is resolved
4. Circuit breaker halts at `MAX_ITERATIONS=5` tool calls to prevent runaway loops

---

## Architecture

```
Ticket
  └── ingress_node          (PII masking · injection detection)
        └── classify_node   (technical / billing / fraud / general)
              │
              ├── dispatcher ─────────────────── [technical / fraud]
              │     ├── tech_analysis_agent
              │     │     └── github_tool_node  ← interrupt() HITL gate
              │     │           └── synthesizer (unified response)
              │     ├── billing_analysis_agent ──┘
              │     └── fraud_analysis_agent ────┘
              │
              └── supervisor ─────────────────── [billing / general]
                    ├── tech_support subgraph    (ReAct loop + tools)
                    ├── fraud_handler
                    └── general_handler
```

---

## Key Files

| File | Purpose |
|------|---------|
| `support_agent.py` | Graph definition, all nodes, tools, state schema, HITL approval functions |
| `api.py` | FastAPI backend — REST endpoints + SSE streaming |
| `index.html` | Single-page frontend — live execution inspector + approval gate UI |
| `requirements.txt` | Python dependencies |
| `support.db` | SQLite checkpoint store (git-ignored — created on first run) |

---

## API Reference

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/run` | Submit a ticket; returns full result including `interrupted` flag |
| `POST` | `/api/stream` | Submit a ticket; streams node-by-node execution events (SSE) |
| `GET`  | `/api/pending-approvals` | List all threads currently paused at the HITL gate |
| `POST` | `/api/approve/{thread_id}` | Resume with approval — creates the GitHub issue |
| `POST` | `/api/deny/{thread_id}` | Resume with denial — skips write, logs decision |
| `POST` | `/api/edit-approve/{thread_id}` | Resume with edited draft — creates issue with corrected content |
| `GET`  | `/api/threads` | List all known thread IDs |
| `GET`  | `/api/history/{thread_id}` | Full checkpoint history for a thread |
| `POST` | `/api/verify` | Run the Session 11 verification test suite |
| `GET`  | `/health` | System health + config |

---

## Verification

Click **Run Verification Test** in the UI or call the API directly:

```bash
curl -X POST http://localhost:8000/api/verify | python -m json.tool
```

| # | Check | Pass criteria |
|---|-------|---------------|
| 1 | Critical ticket → graph pauses | `snapshot.next` contains `github_tool_node` |
| 2 | `get_pending_approvals()` finds the thread | Thread ID appears in pending list |
| 3 | Approve → issue URL set | `github_issue_url` non-empty after resume |
| 4 | Deny → issue skipped, denial logged | URL empty + denial finding in `internal_notes` |
| 5 | Edit & Approve → issue created with corrected title | URL set + edited title used |

---

## Session Progress

| Session | Topic | Status |
|---------|-------|--------|
| 1 | The Blueprint | ✅ |
| 2 | Tool Binding | ✅ |
| 3 | The ReAct Architecture | ✅ |
| 4 | Persistence & Threading | ✅ |
| 5 | Context Management | ✅ |
| 6 | Guardrails & Bounding | ✅ |
| 7 | Multi-Agent Topologies | ✅ |
| 8 | The Supervisor Orchestrator | ✅ |
| 9 | Shared Scratchpads & Consensus | ✅ |
| 10 | System API Integration: Write Access | ✅ |
| 11 | Interruptions & Breakpoints | 🟢 **current** |
| 12 | Time Travel & State Forensics | 🔒 |
