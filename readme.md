# MCP Tools + Orchestrator Console

A local, enterprise-style stack:

- **Standalone MCP server** (`server.py`) exposing math + LLM + vector tools.
- **ChromaDB** vector store (two columns: testcaseId + embedding).
- **Chat orchestrator** (`chat_app.py`) — a dark "operations console" web UI that
  is the *master* to the tools: it uses the local LM Studio model to decide which
  tool to call, runs it over MCP, and streams the answer with live tool traces.

Target: **Python 3.12.x**

```
Browser ──HTTP──> chat_app.py (:8080) ──MCP──> server.py (:8000) ──> ChromaDB
                        │                            │
                        └──────► LM Studio (:1234) ◄─┘
                         (brain / routing)   (chat, vision, embeddings)
```

## ChromaDB storage (failure-log history)
`./chroma_db` next to `server.py`, collection `failure_logs`. This stores software
test **failure logs** for historical analysis. Only failures are stored (there is
no pass/fail field). Each failure is **one row**, so a test case's failures
accumulate over time instead of overwriting:

| Chroma field | Holds |
|--------------|-------|
| `ids` | a unique per-entry id (`{testcaseId}::{random}`) |
| `embeddings` | the vector of the failure log (for similarity search) |
| `documents` | the raw failure log **text** (so the model can read it back) |
| `metadatas` | `{ testcaseId, timestamp (UTC ISO), created_at (epoch) }` |

The user supplies the **testcaseId** and the **failure log**; the **timestamp is
added automatically by the backend**. Because the log text and timestamp are
stored, you can later ask for a test case's failure history or find common
patterns across cases within a time window.

## Tools
| Tool | Task |
|------|------|
| `add(a,b)` / `multiply(a,b)` | Arithmetic |
| `llm_generate(prompt, image_path?, image_base64?)` | Ask the chat model; text or text+image |
| `store_failure_log(testcase_id, log)` | Store a failure log (auto timestamp) |
| `get_failure_history(testcase_id, limit)` | One test case's failures, oldest→newest |
| `search_failures(query, n_results, since_days)` | Semantic search across failures (optional time window) |
| `list_recent_failures(limit, since_days)` | Recent failures across all cases, for pattern-finding |

> `since_days` limits results to the last N days (`0` = all time), e.g. `30` for
> the last month.

## Files
| File | Purpose |
|------|---------|
| `server.py` | Standalone MCP server (Streamable HTTP, :8000) |
| `llm_connect.py` | LM Studio functions + ChromaDB access |
| `chat_app.py` | FastAPI orchestrator + SSE + serves the UI (:8080) |
| `static/index.html` | Dark console UI (retractable panel, tool registry, chat history) |
| `chat_history.db` | SQLite chat history — created automatically next to `chat_app.py` |
| `client.py` | Minimal Python MCP client (no UI) |
| `mock_lmstudio.py` | Test-only fake LM Studio (chat tool-calls + embeddings) |
| `run_all.sh` | Dev launcher for server + console |
| `roo_code_mcp_settings.json` / `claude_mcp_config.json` | MCP host configs |
| `requirements.txt` | Pinned dependencies |

## Setup
```bash
python3.12 -m venv venv
source venv/bin/activate            # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

## Configure LM Studio (two machines)
This setup uses **two separate LM Studio hosts** — one runs the chat/vision model,
the other runs the embedding model. Point each at its own address:
```bash
# machine A: chat / vision model (also used by the orchestrator's router)
export LMSTUDIO_CHAT_BASE_URL="http://192.168.1.7:1234/v1"
export LMSTUDIO_CHAT_MODEL="your-chat-model-id"

# machine B: embedding model
export LMSTUDIO_EMBED_BASE_URL="http://192.168.1.5:1234/v1"
export LMSTUDIO_EMBED_MODEL="your-embed-model-id"
```
Notes:
- `chat_app.py` (the router) and `query_llm` use `LMSTUDIO_CHAT_BASE_URL`.
- `embed_and_store` / `search_similar_testcases` use `LMSTUDIO_EMBED_BASE_URL`.
- If you ever run both models on one host, set only `LMSTUDIO_BASE_URL` and both
  chat and embed fall back to it (embed still needs `LMSTUDIO_EMBED_BASE_URL` if
  the embed host differs).
- Each LM Studio must be started with network serving enabled (not localhost-only)
  so the other machines can reach it. Verify with `curl http://HOST:1234/v1/models`.
- `/api/health` reports `mcp`, `chat_llm`, and `embed_llm` separately, with the
  exact URL and error for whichever one is down.

> The orchestrator routes via OpenAI-style function calling. If the chat model
> doesn't support the `tools` parameter, the app falls back to plain chat.

## Run
```bash
# 1) MCP server
python server.py                    # http://127.0.0.1:8000/mcp
# 2) Chat console (new terminal)
python chat_app.py                  # http://127.0.0.1:8080
# or both at once:
./run_all.sh
```
Open **http://127.0.0.1:8080**. The left rail lists every tool; the header shows
MCP/LLM connection status; each turn shows the tools it called and their output.

## Try it without LM Studio
```bash
python mock_lmstudio.py &           # fake models on :1234
./run_all.sh                        # server + console
```
The mock returns canned chat replies, simple tool routing, and deterministic fake
embeddings so you can see the full flow before wiring real models.

## The UI features
1. **Code formatting** — Markdown with fenced code blocks, syntax highlighting,
   per-block language label and a copy button.
2. **Tool transparency** — the Tool Registry rail describes every tool, and each
   assistant turn renders inline traces: `▸ tool(args)` → `✓ result`.
3. **Master orchestration** — you just type; the model picks the right tool at the
   right time and the console shows the routing as it happens.

### Notes for enterprise / air-gapped deployment
- The UI loads `marked`, `highlight.js`, `DOMPurify`, and fonts from a CDN. For
  air-gapped installs, vendor these into `static/` and update the `<link>`/`<script>`
  tags to local paths.
- `127.0.0.1` is local-only. To expose the console or MCP server on a network, bind
  `0.0.0.0` and put authentication / a reverse proxy in front — nothing here is
  authenticated by default.

## Chat history (SQLite)
Conversations are saved to `chat_history.db` (SQLite3), created automatically next
to `chat_app.py`. Override the location with `CHAT_DB=/path/to/file.db`.

Schema:
- `conversations(id, title, created_at, updated_at)` — title is the first user message
- `messages(id, conversation_id, role, content, created_at)`

Endpoints:
- `GET /api/conversations` — saved chats, most recent first
- `GET /api/conversations/{id}` — all messages of one chat
- `DELETE /api/conversations/{id}` — delete a chat

In the UI, the left panel has two sections — **Tool Registry** and **Chat History** —
each collapsible by clicking its header. The whole panel retracts via the chevron in
its top-right corner; a thin tab on the left edge brings it back. "New chat" starts a
fresh conversation; clicking a saved chat reloads it. Note that tool traces are shown
live but not replayed from history (only the messages are stored).

## Swapping in your own on-prem LLM / embedding servers
Three connection swap points, each marked in the code with a
`██ CONNECTION SWAP POINT ██` banner. Nothing outside those blocks changes.

| Backend | File | Wire in | Activate |
|---------|------|---------|----------|
| Chat / vision | `llm_connect.py` | `custom_chat_connect()` → your `connectLLM()` | `CHAT_BACKEND=custom` |
| Embeddings | `llm_connect.py` | `custom_embed_connect()` → your `connectEmbedLLM()` | `EMBED_BACKEND=custom` |
| Router (tool picker) | `chat_app.py` | see note below | `ROUTER_BACKEND=textjson` |

### Wiring your functions
Both placeholders already contain the exact two lines, commented out — just
uncomment and fix the import path:

```python
def custom_chat_connect(messages):
    from your_module import connectLLM
    return connectLLM(_messages_to_prompt(messages))   # connectLLM returns a str

def custom_embed_connect(text):
    from your_module import connectEmbedLLM
    return _normalize_vector(connectEmbedLLM(text))    # returns list of vectors
```

Two adapter helpers bridge the shapes:
- `_messages_to_prompt(messages)` — flattens the OpenAI-style message list into a
  single prompt string, since `connectLLM()` takes a prompt. (If your function
  accepts a messages list, pass `messages` straight through instead.)
- `_normalize_vector(v)` — `connectEmbedLLM()` returns a **list of vectors**;
  ChromaDB needs one flat vector, so this unwraps `[[...]]` → `[...]`. It also
  passes an already-flat list through unchanged.

### Routing with a string-only LLM
`connectLLM()` returns plain text with no `tool_calls` field, so native
OpenAI-style routing can't work through it. Use the **textjson** router:

```bash
export ROUTER_BACKEND=textjson
```

It appends the tool catalog to the system prompt, asks the model to reply with
`{"tool": "...", "arguments": {...}}`, and parses that JSON back into a tool
call — so routing works over a plain-string connection. If the reply isn't JSON,
it's treated as the final answer. (`ROUTER_BACKEND=custom` remains available if
your server *does* expose native `tool_calls`.)

### Important: set the env vars for BOTH processes
Tools execute inside `server.py`, and routing happens in `chat_app.py`. Export the
variables before starting **each**, or the MCP server will still try LM Studio:

```bash
export CHAT_BACKEND=custom EMBED_BACKEND=custom ROUTER_BACKEND=textjson
python server.py &
python chat_app.py &
```

`llm_connect.backend_info()` and `GET /api/health` report which backends are live.
Keep the embedding dimension consistent — ChromaDB rejects mixed-size vectors in
one collection, so changing embed models means re-indexing `chroma_db/`.
