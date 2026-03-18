# Widget Templates

Minimal HTML scaffolds for the common widget shapes. Copy, fill in, ship.

All templates assume the apps-SDK helper is available at an ESM CDN. They're intentionally framework-free — widgets render in a fresh iframe each time, so React/Vue hydration cost usually isn't worth it for something this small.

---

## The render helper

Ten lines of string templating. Good enough for almost every case.

```typescript
import { readFileSync } from "node:fs";
import { join } from "node:path";

const TEMPLATE_DIR = join(import.meta.dirname, "../widgets");

export function renderWidget(name: string, data: unknown): string {
  const tpl = readFileSync(join(TEMPLATE_DIR, `${name}.html`), "utf8");
  return tpl.replace(
    "__DATA__",
    JSON.stringify(data).replace(/</g, "\\u003c"),
  );
}
```

Every template below hydrates from `<script id="data">__DATA__</script>`. The `<` escape prevents `</script>` injection.

---

## Picker (single-select list)

```html
<!doctype html>
<meta charset="utf-8" />
<script id="data" type="application/json">__DATA__</script>
<style>
  body { font: 14px system-ui; margin: 0; }
  ul { list-style: none; padding: 0; margin: 0; max-height: 280px; overflow-y: auto; }
  li { padding: 10px 14px; cursor: pointer; border-bottom: 1px solid #eee; }
  li:hover { background: #f5f5f5; }
  .sub { color: #666; font-size: 12px; }
</style>
<ul id="list"></ul>
<script type="module">
  import { submit } from "https://esm.sh/@modelcontextprotocol/apps-sdk";
  const { items } = JSON.parse(document.getElementById("data").textContent);
  const ul = document.getElementById("list");
  for (const it of items) {
    const li = document.createElement("li");
    li.innerHTML = `<div>${it.label}</div><div class="sub">${it.sub ?? ""}</div>`;
    li.onclick = () => submit({ id: it.id });
    ul.append(li);
  }
</script>
```

**Data shape:** `{ items: [{ id, label, sub? }] }`
**Result shape:** `{ id }`

---

## Confirm dialog

```html
<!doctype html>
<meta charset="utf-8" />
<script id="data" type="application/json">__DATA__</script>
<style>
  body { font: 14px system-ui; margin: 16px; }
  .actions { display: flex; gap: 8px; margin-top: 16px; }
  button { padding: 8px 16px; cursor: pointer; }
  .danger { background: #d33; color: white; border: none; }
</style>
<p id="msg"></p>
<div class="actions">
  <button id="cancel">Cancel</button>
  <button id="confirm" class="danger">Confirm</button>
</div>
<script type="module">
  import { submit } from "https://esm.sh/@modelcontextprotocol/apps-sdk";
  const { message, confirmLabel } = JSON.parse(document.getElementById("data").textContent);
  document.getElementById("msg").textContent = message;
  if (confirmLabel) document.getElementById("confirm").textContent = confirmLabel;
  document.getElementById("confirm").onclick = () => submit({ confirmed: true });
  document.getElementById("cancel").onclick = () => submit({ confirmed: false });
</script>
```

**Data shape:** `{ message, confirmLabel? }`
**Result shape:** `{ confirmed: boolean }`

---

## Progress (long-running)

```html
<!doctype html>
<meta charset="utf-8" />
<script id="data" type="application/json">__DATA__</script>
<style>
  body { font: 14px system-ui; margin: 16px; }
  .bar { height: 8px; background: #eee; border-radius: 4px; overflow: hidden; }
  .fill { height: 100%; background: #2a7; transition: width 200ms; }
</style>
<p id="label">Starting…</p>
<div class="bar"><div id="fill" class="fill" style="width:0%"></div></div>
<script type="module">
  import { submit, onMessage } from "https://esm.sh/@modelcontextprotocol/apps-sdk";
  const { jobId } = JSON.parse(document.getElementById("data").textContent);
  const label = document.getElementById("label");
  const fill = document.getElementById("fill");

  onMessage((msg) => {
    if (msg.type === "progress") {
      label.textContent = msg.label;
      fill.style.width = `${msg.percent}%`;
    }
    if (msg.type === "done") submit(msg.result);
  });
</script>
```

The server pushes updates via the transport's notification channel targeting this widget's session. See `apps-sdk-messages.md` for the server-side push.

---

## Display-only (chart / preview)

Display widgets don't need `submit()` — they render and sit there. Return a text summary **alongside** the widget so Claude can keep reasoning:

```typescript
return {
  content: [
    { type: "text", text: "Revenue is up 12% MoM. Chart rendered below." },
    { type: "resource", resource: { uri: "ui://widgets/chart", mimeType: "text/html+skybridge", text: renderWidget("chart", data) } },
  ],
};
```
