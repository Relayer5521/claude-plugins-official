# MCPB Manifest Schema

Every `.mcpb` bundle has a `manifest.json` at its root. The host validates it before install.

---

## Top-level fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `name` | string | ✅ | Unique identifier. Lowercase, hyphens only. Shown in settings. |
| `version` | string | ✅ | Semver. Host compares for update prompts. |
| `description` | string | ✅ | One line. Shown in the install prompt. |
| `entry` | object | ✅ | How to launch the server — see below. |
| `permissions` | object | ✅ | What the server needs — see below. |
| `config` | object | — | User-settable values surfaced in settings UI. |
| `icon` | string | — | Path to PNG inside the bundle. 256×256 recommended. |
| `homepage` | string | — | URL shown in settings. |
| `minHostVersion` | string | — | Refuse install on older hosts. |

---

## `entry`

```json
{ "type": "node", "main": "server/index.js" }
```

| `type` | `main` points at | Runtime resolution |
|---|---|---|
| `node` | `.js` or `.mjs` file | `runtime/node` if present, else host-bundled Node |
| `python` | `.py` file | `runtime/python` if present, else host-bundled Python |
| `binary` | executable | Run directly. Must be built per-platform. |

**`args`** (optional array) — extra argv passed to the entry. Rarely needed.

**`env`** (optional object) — static env vars set at launch. For user-configurable values use `config` instead.

---

## `permissions`

```json
{
  "filesystem": { "read": true, "write": false },
  "network": { "allow": ["localhost:*", "127.0.0.1:*"] },
  "process": { "spawn": false }
}
```

### `filesystem`

| Value | Meaning |
|---|---|
| `false` or omitted | No filesystem access beyond the bundle itself |
| `{ "read": true }` | Read anywhere the OS user can |
| `{ "read": true, "write": true }` | Read and write |

There's no path scoping at the manifest level — scope in your code. The manifest permission is a coarse consent gate, not a sandbox.

### `network`

| Value | Meaning |
|---|---|
| `false` or omitted | No network (most local-first servers) |
| `{ "allow": ["host:port", ...] }` | Allowlisted destinations. `*` wildcards ports. |
| `true` | Unrestricted. Heavy scrutiny — explain why in `description`. |

### `process`

| Value | Meaning |
|---|---|
| `false` or omitted | Can't spawn child processes |
| `{ "spawn": true }` | Can spawn. Needed for wrapping CLIs. |
| `{ "spawn": true, "allow": ["git", "ffmpeg"] }` | Spawn only allowlisted binaries |

---

## `config`

User-editable settings, surfaced in the host UI. Each key becomes an env var: `MCPB_CONFIG_<UPPERCASE_KEY>`.

```json
{
  "config": {
    "rootDir": {
      "type": "string",
      "description": "Directory to expose",
      "default": "~/Documents"
    },
    "maxFileSize": {
      "type": "number",
      "description": "Skip files larger than this (MB)",
      "default": 10,
      "min": 1,
      "max": 500
    },
    "includeHidden": {
      "type": "boolean",
      "description": "Include dotfiles in listings",
      "default": false
    },
    "apiKey": {
      "type": "string",
      "description": "Optional API key for the sync feature",
      "secret": true
    }
  }
}
```

**`type`** — `string`, `number`, `boolean`. Enums: use `string` with `"enum": ["a", "b", "c"]`.

**`secret: true`** — host masks the value in UI and stores it in the OS keychain instead of a plain config file.

**`required: true`** — host blocks server launch until the user sets it. Use sparingly — a server that won't start until configured is a bad first-run experience.

---

## Minimal valid manifest

```json
{
  "name": "hello",
  "version": "0.1.0",
  "description": "Minimal MCPB server.",
  "entry": { "type": "node", "main": "server/index.js" },
  "permissions": {}
}
```

Empty `permissions` means no filesystem, no network, no spawn — pure computation only. Valid, if unusual.
