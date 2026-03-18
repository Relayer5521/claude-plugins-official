---
name: build-mcpb
description: This skill should be used when the user wants to "package an MCP server", "bundle an MCP", "make an MCPB", "ship a local MCP server", "distribute a local MCP", discusses ".mcpb files", mentions bundling a Node or Python runtime with their MCP server, or needs an MCP server that interacts with the local filesystem, desktop apps, or OS and must be installable without the user having Node/Python set up.
version: 0.1.0
---

# Build an MCPB (Bundled Local MCP Server)

MCPB is a local MCP server **packaged with its runtime**. The user installs one file; it runs without needing Node, Python, or any toolchain on their machine. It's the sanctioned way to distribute local MCP servers.

**Use MCPB when the server must run on the user's machine** — reading local files, driving a desktop app, talking to localhost services, OS-level APIs. If your server only hits cloud APIs, you almost certainly want a remote HTTP server instead (see `build-mcp-server`). Don't pay the MCPB packaging tax for something that could be a URL.

---

## What an MCPB bundle contains

```
my-server.mcpb              (zip archive)
├── manifest.json           ← identity, entry point, permissions, config schema
├── server/                 ← your MCP server code
│   ├── index.js
│   └── node_modules/       ← bundled dependencies (or vendored)
├── runtime/                ← optional: pinned Node/Python if not using host's
└── icon.png
```

The host reads `manifest.json`, launches the entry point as a **stdio** MCP server, and pipes messages. From your code's perspective it's identical to a local stdio server — the only difference is packaging.

---

## Manifest

```json
{
  "name": "local-files",
  "version": "0.1.0",
  "description": "Read, search, and watch files on the local filesystem.",
  "entry": {
    "type": "node",
    "main": "server/index.js"
  },
  "permissions": {
    "filesystem": { "read": true, "write": false },
    "network": false
  },
  "config": {
    "rootDir": {
      "type": "string",
      "description": "Directory to expose. Defaults to ~/Documents.",
      "default": "~/Documents"
    }
  }
}
```

**`entry.type`** — `node`, `python`, or `binary`. Determines which bundled/host runtime launches `main`.

**`permissions`** — declared upfront and shown to the user at install. Request the minimum. Broad permissions (`filesystem.write: true`, `network: true`) trigger scarier consent UI and more scrutiny.

**`config`** — user-settable values surfaced in the host's settings UI. Your server reads them from env vars (`MCPB_CONFIG_<KEY>`).

---

## Server code: same as local stdio

The server itself is a standard stdio MCP server. Nothing MCPB-specific in the tool logic.

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";
import { readFile, readdir } from "node:fs/promises";
import { join } from "node:path";
import { homedir } from "node:os";

const ROOT = (process.env.MCPB_CONFIG_ROOTDIR ?? "~/Documents")
  .replace(/^~/, homedir());

const server = new McpServer({ name: "local-files", version: "0.1.0" });

server.tool(
  "list_files",
  "List files in a directory under the configured root.",
  { path: z.string().default(".") },
  async ({ path }) => {
    const entries = await readdir(join(ROOT, path), { withFileTypes: true });
    const list = entries.map(e => ({ name: e.name, dir: e.isDirectory() }));
    return { content: [{ type: "text", text: JSON.stringify(list, null, 2) }] };
  },
);

server.tool(
  "read_file",
  "Read a file's contents. Path is relative to the configured root.",
  { path: z.string() },
  async ({ path }) => {
    const text = await readFile(join(ROOT, path), "utf8");
    return { content: [{ type: "text", text }] };
  },
);

const transport = new StdioServerTransport();
await server.connect(transport);
```

**Sandboxing is your job.** The manifest permissions gate what the *host* allows the process to do, but don't rely on that alone — validate paths, refuse to escape `ROOT`, etc. See `references/local-security.md`.

---

## Build pipeline

### Node

```bash
npm install
npx esbuild src/index.ts --bundle --platform=node --outfile=server/index.js
# or: copy node_modules wholesale if native deps resist bundling
npx @modelcontextprotocol/mcpb pack . -o my-server.mcpb
```

`mcpb pack` zips the directory, validates `manifest.json`, and optionally pulls a pinned Node runtime into `runtime/`.

### Python

```bash
pip install -t server/vendor -r requirements.txt
npx @modelcontextprotocol/mcpb pack . -o my-server.mcpb --runtime python3.12
```

Vendor dependencies into a subdirectory and prepend it to `sys.path` in your entry script. Native extensions (numpy, etc.) must be built for each target platform — `mcpb pack --multiarch` cross-builds, but it's slow; avoid native deps if you can.

---

## Permissions: request the minimum

The install prompt shows what you ask for. Every extra permission is friction.

| Need | Request |
|---|---|
| Read files in one directory | `filesystem.read: true` + enforce root in code |
| Write files | `filesystem.write: true` — justify in description |
| Call a local HTTP service | `network: { "allow": ["localhost:*"] }` |
| Call the internet | `network: true` — but ask yourself why this isn't a remote server |
| Spawn processes | `process.spawn: true` — highest scrutiny |

If you find yourself requesting `network: true` to hit a cloud API, stop — that's a remote server wearing an MCPB costume. The user gains nothing from running it locally.

---

## MCPB + UI widgets

MCPB servers can serve UI resources exactly like remote MCP apps — the widget mechanism is transport-agnostic. A local file picker that browses the actual disk, a dialog that controls a native app, etc.

Widget authoring is covered in the **`build-mcp-app`** skill; it works the same here. The only difference is where the server runs.

---

## Testing

```bash
# Run the server directly over stdio, poke it with the inspector
npx @modelcontextprotocol/inspector node server/index.js

# Pack and validate
npx @modelcontextprotocol/mcpb pack . -o test.mcpb
npx @modelcontextprotocol/mcpb validate test.mcpb

# Install into Claude desktop for end-to-end
npx @modelcontextprotocol/mcpb install test.mcpb
```

Test on a machine **without** your dev toolchain before shipping. "Works on my machine" failures in MCPB almost always trace to a dependency that wasn't actually bundled.

---

## Reference files

- `references/manifest-schema.md` — full `manifest.json` field reference
- `references/local-security.md` — path traversal, sandboxing, least privilege
