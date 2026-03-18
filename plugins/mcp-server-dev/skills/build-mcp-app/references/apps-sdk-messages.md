# Apps SDK — Widget ↔ Host Message Protocol

Widgets communicate with the MCP host through `window.parent.postMessage`. The apps SDK wraps this in helpers so you rarely touch the raw envelope, but knowing the shape helps when debugging.

---

## Widget → host

### `submit(result)`

Ends the interaction. `result` is returned to Claude as the tool's output (serialized to JSON). The iframe is torn down after this fires.

```js
import { submit } from "@modelcontextprotocol/apps-sdk";
submit({ id: "usr_abc123", action: "selected" });
```

Raw envelope:
```json
{ "type": "mcp:result", "result": { "id": "usr_abc123", "action": "selected" } }
```

### `callTool(name, args)`

Ask the host to invoke **another tool on the same server** and return the result to the widget. Use for widgets that need to fetch more data after initial render (pagination, drill-down).

```js
import { callTool } from "@modelcontextprotocol/apps-sdk";
const page2 = await callTool("list_items", { offset: 20, limit: 20 });
```

Round-trips through the host, so it's slower than embedding all data upfront. Only use when the full dataset is too large to ship in the initial payload.

### `resize(height)`

Tell the host the widget's content height so the iframe can be sized. The SDK auto-calls this on load via `ResizeObserver`; call manually only if your content height changes after an async operation.

---

## Host → widget

### Initial data

The widget's initial payload is **not** a message — it's baked into the HTML by the server (the `__DATA__` substitution pattern). This avoids a round-trip and works even if the message channel is slow to establish.

### `onMessage(handler)`

Subscribe to pushes from the server. Used by progress widgets and anything live-updating.

```js
import { onMessage } from "@modelcontextprotocol/apps-sdk";
onMessage((msg) => {
  if (msg.type === "progress") updateBar(msg.percent);
});
```

Server side (TypeScript SDK), push via the notification stream keyed to the tool call's request context. The SDK exposes this as a `notify` callback on the tool handler:

```typescript
server.tool("long_job", "...", schema, async (args, { notify }) => {
  for (let i = 0; i <= 100; i += 10) {
    await step();
    notify({ type: "progress", percent: i, label: `Step ${i / 10}/10` });
  }
  return { content: [...] };
});
```

---

## Lifecycle

```
1. Claude calls tool
2. Server returns content with embedded resource (mimeType: text/html+skybridge)
3. Host renders resource text in sandboxed iframe
4. Widget hydrates from inline __DATA__
5. (optional) Widget ↔ host messages: callTool, progress pushes
6. Widget calls submit(result)
7. Host tears down iframe, injects result into conversation
8. Claude continues with the result
```

If step 6 never happens (user closes the widget, host times out), the tool call resolves with a cancellation result. Your tool description should account for this — "Returns the selected ID, or null if the user cancels."

---

## CSP gotchas

The iframe sandbox enforces a strict Content Security Policy. Common failures:

| Symptom | Cause | Fix |
|---|---|---|
| Widget renders but JS doesn't run | Inline script blocked | Use `<script type="module">` with SDK import; avoid inline event handlers in HTML attributes |
| `fetch()` fails silently | Cross-origin blocked | Route through `callTool()` instead |
| External CSS doesn't load | `style-src` restriction | Inline your styles in a `<style>` tag |
| Fonts don't load | `font-src` restriction | Use system fonts (`font: 14px system-ui`) |

When in doubt, open the iframe's devtools console — CSP violations log there.
