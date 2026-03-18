---
name: build-mcp-app
description: This skill should be used when the user wants to build an "MCP app", add "interactive UI" or "widgets" to an MCP server, "render components in chat", build "MCP UI resources", make a tool that shows a "form", "picker", "dashboard" or "confirmation dialog" inline in the conversation, or mentions "apps SDK" in the context of MCP. Use AFTER the build-mcp-server skill has settled the deployment model, or when the user already knows they want UI widgets.
version: 0.1.0
---

# Build an MCP App (Interactive UI Widgets)

An MCP app is a standard MCP server that **also serves UI resources** — interactive components rendered inline in the chat surface. Build once, runs in Claude *and* ChatGPT and any other host that implements the apps surface.

The UI layer is **additive**. Under the hood it's still tools, resources, and the same wire protocol. If you haven't built a plain MCP server before, the `build-mcp-server` skill covers the base layer. This skill adds widgets on top.

---

## When a widget beats plain text

Don't add UI for its own sake — most tools are fine returning text or JSON. Add a widget when one of these is true:

| Signal | Widget type |
|---|---|
| Tool needs structured input Claude can't reliably infer | Form |
| User must pick from a list Claude can't rank (files, contacts, records) | Picker / table |
| Destructive or billable action needs explicit confirmation | Confirm dialog |
| Output is spatial or visual (charts, maps, diffs, previews) | Display widget |
| Long-running job the user wants to watch | Progress / live status |

If none apply, skip the widget. Text is faster to build and faster for the user.

---

## Architecture: two deployment shapes

### Remote MCP app (most common)

Hosted streamable-HTTP server. Widget templates are served as **resources**; tool results reference them. The host fetches the resource, renders it in an iframe sandbox, and brokers messages between the widget and Claude.

```
┌──────────┐  tools/call   ┌────────────┐
│  Claude  │─────────────> │ MCP server │
│   host   │<── result ────│  (remote)  │
│          │  + widget ref │            │
│          │               │            │
│          │ resources/read│            │
│          │─────────────> │  widget    │
│ ┌──────┐ │<── template ──│  HTML/JS   │
│ │iframe│ │               └────────────┘
│ │widget│ │
│ └──────┘ │
└──────────┘
```

### MCPB-packaged MCP app (local + UI)

Same widget mechanism, but the server runs locally inside an MCPB bundle. Use this when the widget needs to drive a **local** application — e.g., a file picker that browses the actual local disk, a dialog that controls a desktop app.

For MCPB packaging mechanics, defer to the **`build-mcpb`** skill. Everything below applies to both shapes.

---

## How widgets attach to tools

A tool declares a widget by returning an **embedded resource** in its result alongside (or instead of) text content. The resource's `mimeType` tells the host to render it, and the `text` field carries the widget's HTML.

```typescript
server.tool(
  "pick_contact",
  "Open an interactive contact picker. The user selects one contact; its ID is returned.",
  {
    filter: z.string().optional().describe("Optional name/email prefix filter"),
  },
  async ({ filter }) => {
    const contacts = await listContacts(filter);
    return {
      content: [
        {
          type: "resource",
          resource: {
            uri: "ui://widgets/contact-picker",
            mimeType: "text/html+skybridge",
            text: renderContactPicker(contacts),
          },
        },
      ],
    };
  },
);
```

The host renders the resource in a sandboxed iframe. The widget posts a message back when the user picks something; the host injects that result into the conversation so Claude can continue.

---

## Widget runtime contract

Widgets run in a sandboxed iframe. They talk to the host via `window.parent.postMessage` with a small set of message types. The exact envelope is host-defined — the MCP apps SDK wraps it so you don't hand-roll `postMessage`.

**What widgets can do:**
- Render arbitrary HTML/CSS/JS (sandboxed — no same-origin access to the host page)
- Receive an initial `data` payload from the tool result
- Post a **result** back (ends the interaction, value flows to Claude)
- Post **progress** updates (for long-running widgets)
- Request the host **call another tool** on the same server

**What widgets cannot do:**
- Access the host page's DOM, cookies, or storage
- Make network calls to origins other than your MCP server (CSP-restricted)
- Persist state across renders (each tool call is a fresh iframe)

Keep widgets **small and single-purpose**. A picker picks. A form submits. Don't build a whole sub-app inside the iframe — split it into multiple tools with focused widgets.

---

## Scaffold: minimal form widget

**Tool (TypeScript SDK):**

```typescript
import { renderWidget } from "./widgets";

server.tool(
  "create_ticket",
  "Open a form to create a support ticket. User fills in title, priority, and description.",
  {},
  async () => ({
    content: [
      {
        type: "resource",
        resource: {
          uri: "ui://widgets/create-ticket",
          mimeType: "text/html+skybridge",
          text: renderWidget("create-ticket", {
            priorities: ["low", "medium", "high", "urgent"],
          }),
        },
      },
    ],
  }),
);
```

**Widget template (`widgets/create-ticket.html`):**

```html
<!doctype html>
<meta charset="utf-8" />
<style>
  body { font: 14px system-ui; margin: 12px; }
  label { display: block; margin-top: 8px; font-weight: 500; }
  input, select, textarea { width: 100%; padding: 6px; margin-top: 2px; }
  button { margin-top: 12px; padding: 8px 16px; }
</style>
<form id="f">
  <label>Title <input name="title" required /></label>
  <label>Priority
    <select name="priority">
      {{#each priorities}}<option>{{this}}</option>{{/each}}
    </select>
  </label>
  <label>Description <textarea name="description" rows="4"></textarea></label>
  <button type="submit">Create</button>
</form>
<script type="module">
  import { submit } from "https://esm.sh/@modelcontextprotocol/apps-sdk";
  document.getElementById("f").addEventListener("submit", (e) => {
    e.preventDefault();
    const data = Object.fromEntries(new FormData(e.target));
    submit(data);  // → flows back to Claude as the tool's result
  });
</script>
```

`renderWidget` is a ~10-line template function — see `references/widget-templates.md`.

---

## Design notes that save you a rewrite

**One widget per tool.** Resist the urge to build one mega-widget that does everything. One tool → one focused widget → one clear result shape. Claude reasons about these far better.

**Tool description must mention the widget.** Claude only sees the tool description when deciding what to call. "Opens an interactive picker" in the description is what makes Claude reach for it instead of guessing an ID.

**Widgets are optional at runtime.** Hosts that don't support the apps surface fall back to showing the resource as a link or raw text. Your tool should still return something sensible in `content[].text` alongside the widget for that case.

**Don't block on widget results for read-only tools.** A widget that just *displays* data (chart, preview) shouldn't require a user action to complete. Return the display widget *and* a text summary in the same result so Claude can continue reasoning without waiting.

---

## Testing

- **Local:** point Claude desktop's MCP config at `http://localhost:3000/mcp`, trigger the tool, check the widget renders and submits.
- **Host fallback:** disable the apps surface (or use a host without it) and confirm the tool degrades gracefully.
- **CSP:** open browser devtools on the iframe — CSP violations are the #1 reason widgets silently fail.

---

## Reference files

- `references/widget-templates.md` — reusable HTML scaffolds for form / picker / confirm / progress
- `references/apps-sdk-messages.md` — the `postMessage` protocol between widget and host
