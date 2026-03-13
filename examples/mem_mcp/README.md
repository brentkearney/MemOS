# MemOS MCP Server

An MCP (Model Context Protocol) bridge that exposes MemOS memory operations to AI tools like Claude Code, Claude Desktop, and other MCP-compatible clients.

## Prerequisites

- MemOS services running (see [Docker setup](#1-start-memos-services) below)
- Python 3.10+ with a virtual environment
- An OpenAI-compatible API key for LLM and embedding (e.g., OpenAI, TARS, or any compatible gateway)

## Setup

### 1. Start MemOS Services

```bash
cd docker
docker compose up -d
```

This starts:
- **MemOS API** on `http://localhost:8000` (Swagger docs at `/docs`)
- **Neo4j** on `bolt://localhost:7687` (browser: `http://localhost:7474`)
- **Qdrant** on `http://localhost:6333`

### 2. Configure `.env`

Copy `docker/.env.example` to the project root as `.env` and configure these key settings:

```bash
cp docker/.env.example .env
```

**Required changes:**

| Variable | Description |
|----------|-------------|
| `OPENAI_API_KEY` | Your API key (OpenAI, TARS, or compatible gateway) |
| `OPENAI_API_BASE` | API base URL (e.g., `https://api.openai.com/v1`) |
| `MEMRADER_API_KEY` | Same key (or separate one) for the retrieval LLM |
| `MEMRADER_API_BASE` | Same base URL |
| `MOS_EMBEDDER_API_KEY` | Same key for embeddings |
| `MOS_EMBEDDER_API_BASE` | Same base URL |
| `MOS_EMBEDDER_MODEL` | Embedding model name (e.g., `text-embedding-3-small`) |
| `EMBEDDING_DIMENSION` | Must match your model (1536 for `text-embedding-3-small`) |

**Common gotchas:**

- `MOS_RERANKER_HEADERS_EXTRA` must be `{}` (not empty) — an empty value causes a JSON parse error on startup
- `MOS_RERANKER_BACKEND` — set to `cosine_local` if you don't have a reranker endpoint
- `QDRANT_URL` and `QDRANT_API_KEY` — comment these out for local Docker; they override `QDRANT_HOST`/`QDRANT_PORT` if set
- After changing `.env`, use `docker compose up -d --force-recreate` (not just `restart`) to reload

### 3. Install MCP Dependencies

```bash
python -m venv .mcp-venv
source .mcp-venv/bin/activate
pip install requests python-dotenv fastmcp
```

### 4. Test the API

```bash
# Health check
curl http://localhost:8000/docs

# Add a memory
curl -X POST http://localhost:8000/product/add \
  -H "Content-Type: application/json" \
  -d '{"user_id": "test", "messages": [{"role": "user", "content": "I like coffee"}]}'

# Search memories
curl -X POST http://localhost:8000/product/search \
  -H "Content-Type: application/json" \
  -d '{"query": "beverages", "user_id": "test", "relativity": 0}'
```

## Connecting to AI Tools

The MCP server (`simple_fastmcp_serve.py`) bridges the MemOS API to any MCP client. Each tool has its own config location.

### Claude Code (CLI / VSCode)

```bash
claude mcp add memos --scope user \
  -e MEMOS_API_BASE_URL=http://localhost:8000/product \
  -- /path/to/.mcp-venv/bin/python3 /path/to/MemOS/examples/mem_mcp/simple_fastmcp_serve.py
```

This writes to `~/.claude.json`. Verify with `claude mcp list`.

> **Note:** `~/.claude/settings.json` has an `mcpServers` field but it is **silently ignored** by Claude Code for MCP registration. Always use `claude mcp add`.

### Claude Desktop

Edit `~/Library/Application Support/Claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "memos": {
      "command": "/path/to/.mcp-venv/bin/python3",
      "args": ["/path/to/MemOS/examples/mem_mcp/simple_fastmcp_serve.py"],
      "env": {
        "MEMOS_API_BASE_URL": "http://localhost:8000/product"
      }
    }
  }
}
```

After saving, fully quit Claude Desktop (Cmd+Q) and relaunch. You should see a permission prompt for the MCP tools on first use.

### Claudian (Obsidian Plugin)

Create or edit `<vault>/.claude/mcp.json`:

```json
{
  "mcpServers": {
    "memos": {
      "command": "/path/to/.mcp-venv/bin/python3",
      "args": ["/path/to/MemOS/examples/mem_mcp/simple_fastmcp_serve.py"],
      "env": {
        "MEMOS_API_BASE_URL": "http://localhost:8000/product"
      }
    }
  }
}
```

## MCP Tools

| Tool | Description |
|------|-------------|
| `add_memory(memory_content, user_id)` | Store a memory. Content is wrapped in chat message format. |
| `search_memories(query, user_id)` | Semantic search across stored memories. |
| `chat(query, user_id)` | Conversational query with memory context. |

Use any consistent `user_id` string (e.g., `"brent"`) across tools to share the same memory space.

## Known Issues

- **RabbitMQ connection errors** in Docker logs are harmless — RabbitMQ is optional and not configured by default.
- **Search relativity threshold**: The default API threshold (0.45) can be too aggressive when you have few memories. The MCP script overrides this to `0` for broader results.
- **Messages format**: The `add` endpoint requires messages in chat format: `[{"role": "user", "content": "..."}]`. A plain string will be silently ignored.
