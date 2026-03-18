# Local MCP Security

An MCPB server runs as the user, with whatever permissions the manifest was granted. Claude drives it. That combination means: **tool inputs are untrusted**, even though they come from an AI the user trusts. A prompt-injected web page can make Claude call your `delete_file` tool with a path you didn't intend.

Defense in depth. Manifest permissions are the outer wall; validation in your tool handlers is the inner wall.

---

## Path traversal

The #1 bug in local MCP servers. If you take a path parameter and join it to a root, **resolve and check containment**.

```typescript
import { resolve, relative, isAbsolute } from "node:path";

function safeJoin(root: string, userPath: string): string {
  const full = resolve(root, userPath);
  const rel = relative(root, full);
  if (rel.startsWith("..") || isAbsolute(rel)) {
    throw new Error(`Path escapes root: ${userPath}`);
  }
  return full;
}
```

`resolve` normalizes `..`, symlink segments, etc. `relative` tells you if the result left the root. Don't just `String.includes("..")` — that misses encoded and symlink-based escapes.

**Python equivalent:**

```python
from pathlib import Path

def safe_join(root: Path, user_path: str) -> Path:
    full = (root / user_path).resolve()
    if not full.is_relative_to(root.resolve()):
        raise ValueError(f"Path escapes root: {user_path}")
    return full
```

---

## Command injection

If you spawn processes, **never pass user input through a shell**.

```typescript
// ❌ catastrophic
exec(`git log ${branch}`);

// ✅ array-args, no shell
execFile("git", ["log", branch]);
```

If you're wrapping a CLI, build the full argv as an array. Validate each flag against an allowlist if the tool accepts flags at all.

---

## Read-only by default

Split read and write into separate tools. Most workflows only need read. A tool that's read-only can't be weaponized into data loss no matter what Claude is tricked into calling it with.

```
list_files   ← safe to call freely
read_file    ← safe to call freely
write_file   ← separate tool, separate permission, separate scrutiny
delete_file  ← consider not shipping this at all
```

If you ship write/delete, consider requiring a confirmation widget (see `build-mcp-app`) so the user explicitly approves each destructive call.

---

## Resource limits

Claude will happily ask to read a 4GB log file. Cap everything:

```typescript
const MAX_BYTES = 1_000_000;
const buf = await readFile(path);
if (buf.length > MAX_BYTES) {
  return {
    content: [{
      type: "text",
      text: `File is ${buf.length} bytes — too large. Showing first ${MAX_BYTES}:\n\n`
            + buf.subarray(0, MAX_BYTES).toString("utf8"),
    }],
  };
}
```

Same for directory listings (cap entry count), search results (cap matches), and anything else unbounded.

---

## Secrets

- **Config secrets** (`secret: true` in manifest): host stores in OS keychain, delivers via env var. Don't log them. Don't include them in tool results.
- **Never store secrets in plaintext files.** If the host's keychain integration isn't enough, use `keytar` (Node) / `keyring` (Python) yourself.
- **Tool results flow into the chat transcript.** Anything you return, the user (and any log export) can see. Redact before returning.

---

## Checklist before shipping

- [ ] Every path parameter goes through containment check
- [ ] No `exec()` / `shell=True` — `execFile` / array-argv only
- [ ] Write/delete split from read tools
- [ ] Size caps on file reads, listing lengths, search results
- [ ] Manifest permissions match actual code behavior (no over-requesting)
- [ ] Secrets never logged or returned in tool results
- [ ] Tested with adversarial inputs: `../../etc/passwd`, `; rm -rf ~`, 10GB file
