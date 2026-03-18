# Remote Streamable-HTTP MCP Server — Scaffold

Minimal working servers in both recommended frameworks. Start here, then add tools.

---

## TypeScript SDK (`@modelcontextprotocol/sdk`)

```bash
npm init -y
npm install @modelcontextprotocol/sdk zod express
npm install -D typescript @types/express @types/node tsx
```

**`src/server.ts`**

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import express from "express";
import { z } from "zod";

const server = new McpServer({
  name: "my-service",
  version: "0.1.0",
});

// Pattern A: one tool per action
server.tool(
  "search_items",
  "Search items by keyword. Returns up to `limit` matches ranked by relevance.",
  {
    query: z.string().describe("Search keywords"),
    limit: z.number().int().min(1).max(50).default(10),
  },
  async ({ query, limit }) => {
    const results = await upstreamApi.search(query, limit);
    return {
      content: [{ type: "text", text: JSON.stringify(results, null, 2) }],
    };
  },
);

server.tool(
  "get_item",
  "Fetch a single item by its ID.",
  { id: z.string() },
  async ({ id }) => {
    const item = await upstreamApi.get(id);
    return { content: [{ type: "text", text: JSON.stringify(item) }] };
  },
);

// Streamable HTTP transport (stateless mode — simplest)
const app = express();
app.use(express.json());

app.post("/mcp", async (req, res) => {
  const transport = new StreamableHTTPServerTransport({
    sessionIdGenerator: undefined, // stateless
  });
  res.on("close", () => transport.close());
  await server.connect(transport);
  await transport.handleRequest(req, res, req.body);
});

app.listen(process.env.PORT ?? 3000);
```

**Stateless vs stateful:** The snippet above creates a fresh transport per request (stateless). Fine for most API-wrapping servers. If tools need to share state across calls in a session (rare), use a session-keyed transport map — see the SDK's `examples/server/simpleStreamableHttp.ts`.

---

## FastMCP 2.0 (Python)

```bash
pip install fastmcp
```

**`server.py`**

```python
from fastmcp import FastMCP

mcp = FastMCP(name="my-service")

@mcp.tool
def search_items(query: str, limit: int = 10) -> list[dict]:
    """Search items by keyword. Returns up to `limit` matches ranked by relevance."""
    return upstream_api.search(query, limit)

@mcp.tool
def get_item(id: str) -> dict:
    """Fetch a single item by its ID."""
    return upstream_api.get(id)

if __name__ == "__main__":
    mcp.run(transport="http", host="0.0.0.0", port=3000)
```

FastMCP derives the JSON schema from type hints and the docstring becomes the tool description. Keep docstrings terse and action-oriented — they land in Claude's context window verbatim.

---

## Search + execute pattern (large API surface)

When wrapping 50+ endpoints, don't register them all. Two tools:

```typescript
const CATALOG = loadActionCatalog(); // { id, description, paramSchema }[]

server.tool(
  "search_actions",
  "Find available actions matching an intent. Call this first to discover what's possible. Returns action IDs, descriptions, and parameter schemas.",
  { intent: z.string().describe("What you want to do, in plain English") },
  async ({ intent }) => {
    const matches = rankActions(CATALOG, intent).slice(0, 10);
    return { content: [{ type: "text", text: JSON.stringify(matches, null, 2) }] };
  },
);

server.tool(
  "execute_action",
  "Execute an action by ID. Get the ID and params schema from search_actions first.",
  {
    action_id: z.string(),
    params: z.record(z.unknown()),
  },
  async ({ action_id, params }) => {
    const action = CATALOG.find(a => a.id === action_id);
    if (!action) throw new Error(`Unknown action: ${action_id}`);
    validate(params, action.paramSchema);
    const result = await dispatch(action, params);
    return { content: [{ type: "text", text: JSON.stringify(result) }] };
  },
);
```

`rankActions` can be simple keyword matching to start. Upgrade to embeddings if precision matters.

---

## Deployment checklist

- [ ] `POST /mcp` responds to `initialize` with server capabilities
- [ ] `tools/list` returns your tools with complete schemas
- [ ] Errors return structured MCP errors, not HTTP 500s with HTML bodies
- [ ] CORS headers set if browser clients will connect
- [ ] Health check endpoint separate from `/mcp` (hosts poll it)
- [ ] Secrets from env vars, never hardcoded
- [ ] If OAuth: DCR endpoint implemented — see `auth.md`
