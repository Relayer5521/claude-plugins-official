# Auth for MCP Servers

Auth is the reason most people end up needing a **remote** server even when a local one would be simpler. OAuth redirects, token storage, and refresh all work cleanly when there's a real hosted endpoint to redirect back to.

---

## The three tiers

### Tier 1: No auth / static API key

Server reads a key from env. User provides it once at setup. Done.

```typescript
const apiKey = process.env.UPSTREAM_API_KEY;
if (!apiKey) throw new Error("UPSTREAM_API_KEY not set");
```

Works for local stdio, MCPB, and remote servers alike. If this is all you need, stop here.

### Tier 2: OAuth 2.0 via Dynamic Client Registration (DCR)

The MCP host (Claude desktop, Claude Code, etc.) discovers your server's OAuth metadata, **registers itself as a client dynamically**, runs the auth-code flow, and stores the token. Your server never sees credentials — it just receives bearer tokens on each request.

This is the **recommended path** for any remote server wrapping an OAuth-protected API.

**Server responsibilities:**

1. Serve OAuth Authorization Server Metadata (RFC 8414) at `/.well-known/oauth-authorization-server`
2. Serve an MCP-protected-resource metadata document pointing at (1)
3. Implement (or proxy to) a DCR endpoint that hands out client IDs
4. Validate bearer tokens on incoming `/mcp` requests

Most of this is boilerplate — the SDK has helpers. The real decision is whether you **proxy** to the upstream's OAuth (if they support DCR) or run your own **shim** authorization server that exchanges your tokens for upstream tokens.

```
┌─────────┐   DCR + auth code   ┌──────────────┐   upstream OAuth   ┌──────────┐
│ MCP host│ ──────────────────> │ Your MCP srv │ ─────────────────> │ Upstream │
└─────────┘ <── bearer token ── └──────────────┘ <── access token ──└──────────┘
```

### Tier 3: CIMD (Client ID Metadata Document)

An alternative to DCR for ecosystems that don't want dynamic registration. The host publishes its client metadata at a well-known URL; your server fetches it, validates it, and issues a client credential. Lower friction than DCR for the host, slightly more work for you.

Use CIMD when targeting hosts that advertise CIMD support in their client metadata. Otherwise default to DCR — it's more broadly implemented.

---

## Hosting providers with built-in DCR/CIMD support

Several MCP-focused hosting providers handle the OAuth plumbing for you — you implement tool logic, they run the authorization server. Check their docs for current capabilities. If the user doesn't have strong hosting preferences, this is usually the fastest path to a working OAuth-protected server.

---

## Local servers and OAuth

Local stdio servers **can** do OAuth (open a browser, catch the redirect on a localhost port, stash the token in the OS keychain). It's fragile:

- Breaks in headless/remote environments
- Every user re-does the dance
- No central token refresh or revocation

If OAuth is required, lean hard toward remote HTTP. If you *must* ship local + OAuth, the `@modelcontextprotocol/sdk` includes a localhost-redirect helper, and MCPB is the right packaging so at least the runtime is predictable.

---

## Token storage

| Deployment | Store tokens in |
|---|---|
| Remote, stateless | Nowhere — host sends bearer each request |
| Remote, stateful | Session store keyed by MCP session ID (Redis, etc.) |
| MCPB / local | OS keychain (`keytar` on Node, `keyring` on Python). **Never plaintext on disk.** |
