# OpenClaw Gateway WebSocket Server

## Overview

The Gateway WebSocket server is the **central control plane** for all OpenClaw operations. It exposes 40+ RPC methods via WebSocket and manages connections from browsers (Control UI), CLI tools, and remote nodes (macOS/iOS/Android apps).

**Location**: `/src/gateway/`
**Default Port**: 18789

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    WebSocket Server                           │
│                  ws://127.0.0.1:18789                         │
├──────────────────────────────────────────────────────────────┤
│  HTTP Server → WebSocket Server → Connection Handler         │
└────────────┬─────────────────────────────────────────────────┘
             │
             ├─ Control UI (browser)
             ├─ CLI (terminal)
             ├─ macOS app
             ├─ iOS app
             ├─ Android app
             └─ Custom SDK clients
```

### Protocol Stack

```
┌──────────────────────────────────────────────────────────────┐
│                  JSON-RPC-like Protocol                       │
├──────────────────────────────────────────────────────────────┤
│  Request Frame:  { type: "req", id, method, params }        │
│  Response Frame: { type: "res", id, ok, payload?, error? }  │
│  Event Frame:    { type: "event", event, payload, seq }     │
└──────────────────────────────────────────────────────────────┘
             ↓
┌──────────────────────────────────────────────────────────────┐
│                     WebSocket Layer                           │
├──────────────────────────────────────────────────────────────┤
│  - Binary/text frames                                        │
│  - Max payload: 25 MB                                        │
│  - Heartbeat: 30s tick events                                │
└──────────────────────────────────────────────────────────────┘
             ↓
┌──────────────────────────────────────────────────────────────┐
│                      TCP/TLS Layer                            │
└──────────────────────────────────────────────────────────────┘
```

---

## Protocol Messages

### Request Frame

```typescript
{
  type: "req",
  id: "uuid",              // Request ID (for matching response)
  method: "agent",         // Method name
  params: {                // Method-specific parameters
    sessionId: "abc-123",
    message: "Hello"
  }
}
```

### Response Frame

```typescript
// Success
{
  type: "res",
  id: "uuid",              // Matches request ID
  ok: true,
  payload: {               // Method result
    text: "Response text"
  }
}

// Error
{
  type: "res",
  id: "uuid",
  ok: false,
  error: {
    code: "INVALID_REQUEST",
    message: "Error details"
  }
}

// Streaming (intermediate response)
{
  type: "res",
  id: "uuid",
  ok: true,
  payload: {
    status: "accepted",    // More data coming
    runId: "xyz"
  }
}
```

### Event Frame

```typescript
{
  type: "event",
  event: "chat.delta",     // Event name
  payload: {               // Event data
    runId: "xyz",
    delta: "partial text"
  },
  seq: 42                  // Sequence number
}
```

---

## Exposed Methods (40+)

### Agent Management

| Method | Scope | Description |
|--------|-------|-------------|
| `agent` | WRITE | Run agent |
| `agent.wait` | READ | Wait for completion |
| `agent.identity.get` | READ | Get agent identity |
| `agents.list` | READ | List all agents |
| `agents.create` | WRITE | Create agent |
| `agents.update` | WRITE | Update agent |
| `agents.delete` | WRITE | Delete agent |
| `agents.files.list` | READ | List agent files |
| `agents.files.get` | READ | Get file content |
| `agents.files.set` | WRITE | Write file |

### Chat Operations

| Method | Scope | Description |
|--------|-------|-------------|
| `chat.send` | WRITE | Send chat message |
| `chat.abort` | WRITE | Abort chat run |
| `chat.history` | READ | Get history |

### Node Management

| Method | Scope | Description |
|--------|-------|-------------|
| `node.list` | READ | List remote nodes |
| `node.describe` | READ | Get node details |
| `node.invoke` | WRITE | Execute on node |
| `node.pair.request` | PAIRING | Request pairing |
| `node.pair.approve` | PAIRING | Approve pairing |
| `node.pair.reject` | PAIRING | Reject pairing |
| `node.pair.verify` | PAIRING | Verify pairing |

### Session Management

| Method | Scope | Description |
|--------|-------|-------------|
| `sessions.list` | READ | List sessions |
| `sessions.preview` | READ | Preview session |
| `sessions.patch` | WRITE | Modify session |
| `sessions.reset` | WRITE | Reset session |
| `sessions.delete` | WRITE | Delete session |
| `sessions.compact` | WRITE | Compact sessions |

### Configuration

| Method | Scope | Description |
|--------|-------|-------------|
| `config.get` | READ | Get config |
| `config.set` | ADMIN | Set value |
| `config.apply` | ADMIN | Apply full config |
| `config.patch` | ADMIN | Patch config |
| `config.schema` | READ | Get schema |

### Execution Approvals

| Method | Scope | Description |
|--------|-------|-------------|
| `exec.approval.request` | APPROVALS | Request approval |
| `exec.approval.resolve` | APPROVALS | Resolve approval |
| `exec.approvals.get` | APPROVALS | Get policies |
| `exec.approvals.set` | APPROVALS | Set policies |

### Other Operations

| Method | Scope | Description |
|--------|-------|-------------|
| `send` | WRITE | Send message |
| `wake` | WRITE | Wake agent |
| `health` | READ | Health status |
| `logs.tail` | READ | Stream logs |
| `models.list` | READ | List models |
| `skills.status` | READ | Skill status |
| `skills.install` | ADMIN | Install skill |
| `cron.list` | READ | List cron jobs |
| `cron.add` | ADMIN | Add job |
| `cron.update` | ADMIN | Update job |
| `cron.remove` | ADMIN | Remove job |
| `cron.run` | ADMIN | Run job now |
| `channels.status` | READ | Channel status |
| `channels.logout` | ADMIN | Logout channel |
| `browser.request` | WRITE | Browser automation |
| `wizard.start` | ADMIN | Start wizard |
| `wizard.next` | ADMIN | Next wizard step |

---

## Authorization System

### Scopes

```typescript
ADMIN_SCOPE = "operator.admin"          // Full access
READ_SCOPE = "operator.read"            // View-only
WRITE_SCOPE = "operator.write"          // Modify operations
APPROVALS_SCOPE = "operator.approvals"  // Exec approval management
PAIRING_SCOPE = "operator.pairing"      // Device/node pairing
```

### Method Authorization

```typescript
function authorizeGatewayMethod(method: string, client: GatewayWsClient) {
  const role = client?.connect.role ?? "operator";
  const scopes = client?.connect.scopes ?? [];

  // Node role can only send invoke results and events
  if (NODE_ROLE_METHODS.has(method)) {
    if (role === "node") return null;  // Allowed
    return error("unauthorized role");
  }

  // Operator role check
  if (role !== "operator") {
    return error("unauthorized role");
  }

  // Admin scope grants all access
  if (scopes.includes(ADMIN_SCOPE)) {
    return null;
  }

  // Check specific scopes
  if (READ_METHODS.has(method) && !hasReadOrWrite(scopes)) {
    return error("missing scope: operator.read");
  }

  if (WRITE_METHODS.has(method) && !scopes.includes(WRITE_SCOPE)) {
    return error("missing scope: operator.write");
  }

  if (method.startsWith("config.") || method.startsWith("wizard.")) {
    return error("missing scope: operator.admin");
  }

  return null;  // Authorized
}
```

---

## Authentication Mechanisms

### 1. Token-Based Auth

```typescript
// Config
gateway: {
  auth: {
    mode: "token",
    token: "your-secret-token"
  }
}

// Client request
{
  auth: {
    token: "your-secret-token"
  }
}
```

### 2. Password-Based Auth

```typescript
// Config
gateway: {
  auth: {
    mode: "password",
    password: "your-password"
  }
}

// Client request
{
  auth: {
    password: "your-password"
  }
}
```

### 3. Tailscale Integration

```typescript
// Detected via proxy headers
isTailscaleProxyRequest(req)
getTailscaleUser(req)  // From x-tailscale-user-login header
readTailscaleWhoisIdentity(clientIp)  // Verify identity
```

### 4. Device Identity (Public Key Cryptography)

```typescript
// Client generates signature
const payload = buildDeviceAuthPayload({
  deviceId,
  clientId,
  role: "operator",
  scopes: ["operator.admin"],
  signedAtMs: Date.now(),
  token: challengeNonce
});

const signature = signDevicePayload(privateKeyPem, payload);

// Server verifies
verifyDeviceSignature(publicKey, payload, signature);
```

### 5. Local Connection Bypass

```typescript
function isLocalDirectRequest(req, trustedProxies) {
  const clientIp = resolveRequestClientIp(req, trustedProxies);
  if (!isLoopbackAddress(clientIp)) return false;

  const host = getHostName(req.headers?.host);
  const hostIsLocal = host === "localhost" || host === "127.0.0.1";

  const hasForwarded = Boolean(
    req.headers?.["x-forwarded-for"] ||
    req.headers?.["x-real-ip"]
  );

  const remoteIsTrustedProxy = isTrustedProxyAddress(
    req.socket?.remoteAddress,
    trustedProxies
  );

  return hostIsLocal && (!hasForwarded || remoteIsTrustedProxy);
}
```

**Local requests skip authentication** but still require valid scopes.

---

## Connection Lifecycle

### 1. Handshake Phase

```
Client connects → Server sends challenge nonce
  ↓
Client sends connect request:
{
  minProtocol: 1,
  maxProtocol: 1,
  client: {
    id: "control-ui",
    displayName: "Control UI",
    version: "1.0.0",
    platform: "web",
    mode: "frontend"
  },
  auth: { token: "..." },
  device: { /* optional device identity */ }
}
  ↓
Server validates:
- Protocol version compatibility
- Authentication credentials
- Authorization scopes
  ↓
Server responds:
{
  type: "res",
  ok: true,
  payload: {
    event: "hello-ok",
    protocolVersion: 1,
    serverVersion: "2026.2.9",
    capabilities: [...]
  }
}
```

### 2. Active Phase

```
Client sends requests → Server processes → Client receives responses
Server sends events → Client handles events

Request flow:
  Client: { type: "req", id: "1", method: "agent", params: {...} }
  Server: { type: "res", id: "1", ok: true, payload: {...} }

Event flow:
  Server: { type: "event", event: "tick", payload: {...} }
  Client: handles event
```

### 3. Disconnect Phase

```
Client/Server closes connection
  ↓
Server cleanup:
- Remove from client registry
- Unsubscribe from events
- Clean up pending requests
```

---

## Node Communication

### Node Registration

```typescript
class NodeRegistry {
  register(client: GatewayWsClient, opts: { remoteIp?: string }) {
    const nodeId = client.connect.device?.id ?? client.connect.client.id;
    const session: NodeSession = {
      nodeId,
      connId: client.connId,
      client,
      displayName: client.connect.client.displayName,
      platform: client.connect.client.platform,
      caps: client.connect.caps || [],
      commands: client.connect.commands || [],
      permissions: client.connect.permissions,
      connectedAtMs: Date.now()
    };
    this.nodesById.set(nodeId, session);
    return session;
  }
}
```

### Node Invocation

```typescript
// Client requests execution on remote node
const result = await client.request("node.invoke", {
  nodeId: "my-macos-machine",
  command: "bash",
  params: { script: "ls -la ~" },
  timeoutMs: 10000
});

// Server flow:
1. Lookup node session
2. Send event to node: node.invoke.request
3. Wait for response: node.invoke.result (max 30s)
4. Return result to client
```

**Node-side handling**:
```typescript
// macOS app listens for invoke events
client.onEvent((evt) => {
  if (evt.event === "node.invoke.request") {
    const { id, command, paramsJSON } = evt.payload;
    const params = JSON.parse(paramsJSON);

    // Execute command
    execSync(params.script).then(output => {
      // Send result back
      client.request("node.invoke.result", {
        requestId: id,
        ok: true,
        payload: output
      });
    });
  }
});
```

---

## Streaming Architecture

### Chat Streaming Example

```typescript
// Client sends chat request
await client.request("chat.send", {
  sessionId: "abc-123",
  message: "List my files"
});

// Server processing:
1. Respond with accepted status:
   { ok: true, payload: { status: "accepted", runId: "xyz" } }

2. Start streaming via events:
   { type: "event", event: "chat.delta", payload: { runId, delta: "Here" } }
   { type: "event", event: "chat.delta", payload: { runId, delta: " are" } }
   { type: "event", event: "chat.delta", payload: { runId, delta: " your" } }

3. Send final response:
   { type: "res", id, ok: true, payload: { type: "message", content: "..." } }
```

### Intermediate Responses

```typescript
// Method handler can send multiple responses
respond(true, { status: "accepted" });  // First response (more coming)
// ... do work ...
respond(true, { finalData: "..." });    // Final response
```

---

## Request Context

Every handler receives rich context:

```typescript
type GatewayRequestContext = {
  // Dependencies
  deps: { /* runtime utilities */ },
  cron: CronService,

  // State queries
  loadGatewayModelCatalog(): Promise<ModelCatalogEntry[]>,
  getHealthCache(): HealthSummary | null,
  refreshHealthSnapshot(): Promise<HealthSummary>,

  // Broadcasting
  broadcast(event: string, payload: unknown, opts?): void,
  broadcastToConnIds(event, payload, connIds, opts?): void,

  // Node communication
  nodeSendToSession(sessionKey, event, payload): void,
  nodeSendToAllSubscribed(event, payload): void,
  nodeRegistry: NodeRegistry,

  // Chat state
  chatRunBuffers: Map<string, string>,
  chatAbortControllers: Map<string, AbortController>,

  // Wizard sessions
  wizardSessions: Map<string, WizardSession>,

  // Channel management
  getRuntimeSnapshot(): ChannelRuntimeSnapshot,
  startChannel(channelId, accountId?): Promise<void>,
  stopChannel(channelId, accountId?): Promise<void>
};
```

---

## Control UI Integration

### Connection Flow

```typescript
const params: ConnectParams = {
  minProtocol: PROTOCOL_VERSION,
  maxProtocol: PROTOCOL_VERSION,
  client: {
    id: "control-ui",
    displayName: "Control UI",
    version: "...",
    platform: "web",
    mode: "frontend"
  },
  auth: {
    token: userToken  // From settings or env
  }
};

const client = new GatewayClient();
await client.connect("ws://127.0.0.1:18789", params);
```

### Origin Validation

```typescript
checkBrowserOrigin({
  requestHost,
  origin: requestOrigin,
  allowedOrigins: config.gateway?.controlUi?.allowedOrigins
});
```

Ensures browser requests come from trusted origins to prevent CSRF.

---

## Configuration

### Server Config

```yaml
gateway:
  port: 18789
  bind: "loopback"         # "loopback" | "all"

  auth:
    mode: "token"          # "token" | "password" | "none"
    token: "secret"
    allowTailscale: true   # Allow Tailscale users

  controlUi:
    enabled: true
    allowedOrigins: []     # Empty = allow all

  tailscale:
    mode: "off"            # "off" | "serve" | "funnel"
    resetOnExit: false

  trustedProxies: []       # IPs allowed to set proxy headers
```

### Client Config

```typescript
const client = new GatewayClient();
await client.connect("ws://127.0.0.1:18789", {
  client: { id: "cli", displayName: "CLI" },
  auth: { token: "..." },
  role: "operator",
  scopes: ["operator.admin"]
});
```

---

## Security Headers

```typescript
res.setHeader("X-Frame-Options", "DENY");
res.setHeader("X-Content-Type-Options", "nosniff");
res.setHeader("Content-Security-Policy", "default-src 'self'");
```

---

## Performance Characteristics

- **Max payload**: 25 MB per message
- **Heartbeat interval**: 30s tick events
- **Connection timeout**: Configurable
- **Concurrent connections**: Unlimited
- **Message queue**: Per-connection buffering

---

## Summary

The Gateway WebSocket server provides:

✓ JSON-RPC-like protocol (req/res/event)
✓ 40+ RPC methods across 10 categories
✓ Role-based + scope-based authorization
✓ Multiple auth mechanisms (token, password, Tailscale, device identity)
✓ Local connection bypass (localhost skip auth)
✓ Node communication (macOS/iOS/Android)
✓ Streaming support (intermediate responses)
✓ Control UI integration with CSRF protection
✓ Rich request context for handlers
✓ Device authentication via public key cryptography

This architecture enables **secure, scalable, real-time communication** between clients and the OpenClaw agent system with comprehensive authorization and flexible authentication.
