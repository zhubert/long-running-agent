# OpenClaw Command Lanes & Session Management

## Overview

OpenClaw uses two complementary systems to ensure **safe concurrent operation**:

1. **Command Lanes**: In-process task scheduler preventing race conditions
2. **Session Management**: Persistent conversation contexts with routing

**Locations**:
- Command Lanes: `/src/process/command-queue.ts`
- Session Management: `/src/infra/session*.ts`

---

# Part 1: Command Lanes System

## Architecture

```
┌────────────────────────────────────────────────────────────┐
│              Command Queue Manager                          │
├────────────────────────────────────────────────────────────┤
│                                                              │
│  lanes: Map<string, LaneState>                              │
│  ├─ "main"           → Main auto-reply workflow            │
│  ├─ "cron"           → Scheduled cron jobs                 │
│  ├─ "subagent"       → Subagent operations                 │
│  ├─ "nested"         → Nested context execution            │
│  └─ "session:{key}"  → Per-session isolated lanes         │
│                                                              │
│  Each LaneState:                                            │
│  ├─ queue: QueueEntry[]        (pending tasks)            │
│  ├─ active: number             (currently executing)       │
│  ├─ maxConcurrent: number      (concurrency limit)        │
│  └─ draining: boolean          (pump active)               │
│                                                              │
└────────────────────────────────────────────────────────────┘
```

## Lane Types

```typescript
export const enum CommandLane {
  Main = "main",        // Auto-reply processing
  Cron = "cron",        // Scheduled jobs
  Subagent = "subagent",// Subagent calls
  Nested = "nested"     // Nested operations
}
```

### Concurrency Configuration

```typescript
// From /src/gateway/server-lanes.ts
function applyGatewayLaneConcurrency(cfg: OpenClawConfig) {
  setCommandLaneConcurrency(
    CommandLane.Cron,
    cfg.cron?.maxConcurrentRuns ?? 1
  );

  setCommandLaneConcurrency(
    CommandLane.Main,
    resolveAgentMaxConcurrent(cfg)  // Default: 1
  );

  setCommandLaneConcurrency(
    CommandLane.Subagent,
    resolveSubagentMaxConcurrent(cfg)  // Default: 2
  );
}
```

## Race Condition Prevention

### Problem

Without serialization:
- Multiple auto-reply workflows interleave
- Stdin/stdout corruption
- Mixed log output
- Concurrent file writes

### Solution: Pump-and-Drain Pattern

```typescript
function drainLane(lane: string) {
  const state = getLaneState(lane);
  if (state.draining) return;  // Re-entrancy guard
  state.draining = true;

  const pump = () => {
    // Execute up to maxConcurrent tasks
    while (state.active < state.maxConcurrent && state.queue.length > 0) {
      const entry = state.queue.shift() as QueueEntry;
      state.active += 1;

      void (async () => {
        try {
          const result = await entry.task();
          state.active -= 1;
          pump();  // Continue draining
          entry.resolve(result);
        } catch (err) {
          state.active -= 1;
          pump();  // Continue even on error
          entry.reject(err);
        }
      })();
    }
    state.draining = false;
  };

  pump();
}
```

**Key safeguards**:
1. **Serialization**: Tasks execute one at a time (default `maxConcurrent=1`)
2. **Re-entrancy guard**: `draining` flag prevents concurrent pump calls
3. **Queue ordering**: FIFO ensures predictable execution
4. **Callback continuity**: Pump reschedules itself after each task

## Session Lanes Pattern

```typescript
// Per-session isolation via named lanes
function resolveSessionLane(key: string): string {
  const cleaned = key.trim() || CommandLane.Main;
  return cleaned.startsWith("session:")
    ? cleaned
    : `session:${cleaned}`;
}
```

**Usage**:
- Session `"agent:main:slack:direct:user123"` → Lane `"session:agent:main:slack:direct:user123"`
- Prevents concurrent executions for same user
- Different sessions execute in parallel

## API

### Enqueue to Named Lane

```typescript
function enqueueCommandInLane<T>(
  lane: string,
  task: () => Promise<T>,
  opts?: {
    warnAfterMs?: number,      // Warn if wait exceeds threshold
    onWait?: (waitMs: number, queuedAhead: number) => void
  }
): Promise<T>
```

### Enqueue to Main Lane

```typescript
function enqueueCommand<T>(
  task: () => Promise<T>,
  opts?
): Promise<T>
```

### Monitor Queue

```typescript
// Get queue depth
getQueueSize(lane?: string): number
getTotalQueueSize(): number

// Clear pending tasks
clearCommandLane(lane?: string): number
```

## Test Evidence

```typescript
// From /src/process/command-queue.test.ts
it("runs tasks one at a time in order", async () => {
  let active = 0;
  let maxActive = 0;
  const calls: number[] = [];

  const makeTask = (id: number) => async () => {
    active += 1;
    maxActive = Math.max(maxActive, active);  // Should never exceed 1
    calls.push(id);
    await delay(15);
    active -= 1;
  };

  const results = await Promise.all([
    enqueueCommand(makeTask(1)),
    enqueueCommand(makeTask(2)),
    enqueueCommand(makeTask(3))
  ]);

  expect(results).toEqual([1, 2, 3]);
  expect(calls).toEqual([1, 2, 3]);   // FIFO execution
  expect(maxActive).toBe(1);           // Never concurrent!
});
```

---

# Part 2: Session Management System

## Overview

**Sessions** are persistent conversation contexts identified by **session keys** that group messages, maintain context, and route replies across multiple channels.

## Session Key Hierarchy

```
┌──────────────────────────────────────────────────────────┐
│                Session Key Hierarchy                      │
├──────────────────────────────────────────────────────────┤
│                                                            │
│ Global (rare):                                            │
│ └─ "global"                                               │
│                                                            │
│ Agent Main Session:                                      │
│ └─ "agent:{agentId}:main"                                │
│    Example: "agent:main:main"                            │
│                                                            │
│ Channel-Peer Session:                                    │
│ └─ "agent:{agentId}:{channel}:{chatType}:{peerId}"      │
│    Examples:                                              │
│    ├─ "agent:main:slack:direct:user123"                 │
│    ├─ "agent:main:discord:group:channel_id"             │
│    └─ "agent:main:telegram:group:chat_id:topic123"      │
│                                                            │
│ Thread/Topic Session:                                    │
│ └─ "agent:{agentId}:{channel}:...:{peerId}:thread:{id}" │
│    Examples:                                              │
│    ├─ "agent:main:slack:direct:user:thread:1706123456"  │
│    └─ "agent:main:discord:group:ch:thread:msg_id"       │
│                                                            │
└──────────────────────────────────────────────────────────┘
```

## Session Key Building

### Main Agent Session

```typescript
buildAgentMainSessionKey({
  agentId: "main",
  mainKey: "main"
}): "agent:main:main"
```

### Peer-Aware Session

```typescript
buildAgentPeerSessionKey({
  agentId: "main",
  channel: "slack",
  peerKind: "direct",
  peerId: "user123",
  dmScope: "main" | "per-peer" | "per-channel-peer" | "per-account-channel-peer"
}):

dmScope="main"
  → "agent:main:main"  (collapse to main)

dmScope="per-peer"
  → "agent:main:direct:user123"

dmScope="per-channel-peer"
  → "agent:main:slack:direct:user123"

dmScope="per-account-channel-peer"
  → "agent:main:slack:default:direct:user123"
```

### Thread-Derived Session

```typescript
resolveThreadSessionKeys({
  baseSessionKey: "agent:main:slack:direct:user123",
  threadId: "1706123456",
  useSuffix: true  // Slack/Mattermost append ":thread:"
}): {
  sessionKey: "agent:main:slack:direct:user123:thread:1706123456",
  parentSessionKey: "agent:main:slack:direct:user123"
}
```

## Session Store Architecture

### Storage Format

```typescript
// ~/.openclaw/sessions.json (per agent)
{
  "agent:main:main": {
    sessionId: "abc-123-uuid",
    updatedAt: 1706123456789,
    chatType: "direct",
    channel: "slack",
    // ... metadata
  },
  "agent:main:slack:direct:user456": {
    sessionId: "def-456-uuid",
    updatedAt: 1706123500000,
    // ... metadata
  }
}
```

### Persistence Lifecycle

```
┌────────────────────────────────────────────────────────┐
│              Session Store Lifecycle                    │
├────────────────────────────────────────────────────────┤
│                                                          │
│  Load Phase:                                            │
│  ┌───────────────────────────────────────┐            │
│  │ loadSessionStore(storePath)           │            │
│  ├───────────────────────────────────────┤            │
│  │ 1. Check cache (TTL=45s)             │            │
│  │ 2. If hit & valid & mtime → return   │            │
│  │ 3. Cache miss: read from disk        │            │
│  │ 4. Parse JSON5, validate schema      │            │
│  │ 5. Best-effort migrations            │            │
│  │ 6. Cache with structuredClone()      │            │
│  │ 7. Return structuredClone() to caller│            │
│  └───────────────────────────────────────┘            │
│                                                          │
│  Update Phase (with Lock):                             │
│  ┌───────────────────────────────────────┐            │
│  │ updateSessionStore(storePath, mutator)│            │
│  ├───────────────────────────────────────┤            │
│  │ 1. Acquire lock (.lock file)         │            │
│  │ 2. Load store (skipCache: true)      │            │
│  │ 3. mutator(store) — user code        │            │
│  │ 4. Validate & normalize              │            │
│  │ 5. Maintenance:                      │            │
│  │    - Prune stale (30d)               │            │
│  │    - Cap entries (500 max)           │            │
│  │    - Rotate file (10 MB)             │            │
│  │ 6. Atomic write (tmp→rename)         │            │
│  │ 7. Invalidate cache                  │            │
│  │ 8. Release lock                      │            │
│  └───────────────────────────────────────┘            │
│                                                          │
└────────────────────────────────────────────────────────┘
```

## Session Locking

### Lock Mechanism

```typescript
async function withSessionStoreLock<T>(
  storePath: string,
  fn: () => Promise<T>,
  opts = {}
): Promise<T> {
  const lockPath = `${storePath}.lock`;
  const timeoutMs = 10_000;
  const staleMs = 30_000;  // Evict stale locks

  // Acquire lock (loop with timeout)
  while (true) {
    try {
      // Exclusive open (fails if exists)
      const handle = await fs.promises.open(lockPath, "wx");
      await handle.writeFile(JSON.stringify({ pid, startedAt }));
      await handle.close();
      break;  // Lock acquired!
    } catch (err) {
      if (err.code === "EEXIST") {
        // Lock held by another process
        // Check if stale (> 30s old) → evict & retry
        // Otherwise wait 25ms and retry
      }
    }
  }

  try {
    return await fn();
  } finally {
    await fs.promises.unlink(lockPath);
  }
}
```

**Why Locking Matters**:
- Multiple processes may write simultaneously
- Lock serializes all writes
- Stale lock eviction handles crashed processes

## Session Creation & Routing

### Inbound Message Flow

```
┌────────────────────────────────────────────────────────┐
│         Inbound Message → Session Routing Flow         │
├────────────────────────────────────────────────────────┤
│                                                          │
│ 1. Message arrives (e.g., Slack DM from user123)      │
│                                                          │
│ 2. resolveAgentRoute({                                 │
│      cfg, channel="slack",                             │
│      peer={kind:"direct", id:"user123"}                │
│    })                                                   │
│    → Check bindings for peer match                    │
│    → Fall back to channel/account/default             │
│    → Returns: { agentId, sessionKey, mainSessionKey } │
│                                                          │
│ 3. buildAgentSessionKey({                             │
│      agentId: "main",                                 │
│      channel: "slack",                                │
│      peer: {kind:"direct", id:"user123"},            │
│      dmScope: "main"                                  │
│    })                                                  │
│    → sessionKey: "agent:main:main"                   │
│                                                          │
│ 4. Load/Create session entry:                        │
│    updateSessionStore(storePath, (store) => {        │
│      const existing = store[sessionKey];             │
│      const patch = deriveSessionMetaPatch({...});    │
│      const merged = mergeSessionEntry(existing,      │
│                                       patch);         │
│      store[sessionKey] = merged;                     │
│    })                                                  │
│                                                          │
│ 5. Session persistent with:                          │
│    - sessionId (UUID, permanent)                     │
│    - updatedAt (now)                                 │
│    - channel, chatType, deliveryContext              │
│    - metadata (model, settings)                      │
│                                                          │
└────────────────────────────────────────────────────────┘
```

## Session Entry Structure

```typescript
type SessionEntry = {
  sessionId: string,                    // UUID (permanent)
  updatedAt: number,                    // Last update timestamp
  sessionFile?: string,                 // Transcript path

  // Chat context
  chatType?: "direct" | "group" | "channel",
  channel?: string,                     // "slack", "discord"
  groupId?: string,                     // Group/channel ID
  subject?: string,                     // Group subject

  // Delivery routing
  lastChannel?: string,                 // Last delivery channel
  lastTo?: string,                      // Last recipient
  lastAccountId?: string,               // Last account
  lastThreadId?: string | number,       // Last thread ID
  deliveryContext?: DeliveryContext,    // Where to send replies

  // AI Configuration
  modelOverride?: string,               // Session-specific model
  providerOverride?: string,            // Session-specific provider
  thinkingLevel?: string,               // "off" | "short" | "long"
  verboseLevel?: string,

  // Execution
  execHost?: string,                    // Host override
  execSecurity?: string,                // Sandbox mode

  // Queue settings
  queueMode?: string,                   // Message queue behavior
  queueDebounceMs?: number,
  queueCap?: number,

  // Metadata
  label?: string,                       // User-friendly name
  displayName?: string,                 // Generated display
  origin?: SessionOrigin,               // Creation context

  // Stats
  inputTokens?: number,
  outputTokens?: number,
  totalTokens?: number,
  compactionCount?: number
};
```

## Main vs. Isolated Sessions

### Main Session (`dmScope="main"`)

```typescript
// All DMs collapse to single session
"agent:main:main"
```

**Behavior**:
- Unified conversation history
- Single per-agent session
- Use case: Single-user CLI bots

### Per-Peer Sessions (`dmScope="per-peer"`)

```typescript
// Each peer gets isolated session
"agent:main:direct:{peerId}"
```

**Behavior**:
- Separate history per user
- Use case: Multi-user agents

### Per-Channel-Peer (`dmScope="per-channel-peer"`)

```typescript
// Channel + peer combined
"agent:main:{channel}:direct:{peerId}"
```

**Behavior**:
- Channel-scoped isolation
- Account-specific sessions

## Session Key Parsing

```typescript
function parseAgentSessionKey(
  sessionKey: string
): ParsedAgentSessionKey | null {
  const parts = raw.split(":").filter(Boolean);
  if (parts.length < 3 || parts[0] !== "agent") return null;

  return {
    agentId: parts[1],
    rest: parts.slice(2).join(":")
  };
}

// Examples:
parseAgentSessionKey("agent:main:main")
  → { agentId: "main", rest: "main" }

parseAgentSessionKey("agent:main:slack:direct:user123")
  → { agentId: "main", rest: "slack:direct:user123" }
```

---

# Integration: Lanes × Sessions

## Combined Message Processing

```
┌─────────────────────────────────────────────────────────┐
│         Message Processing (Integrated)                  │
├─────────────────────────────────────────────────────────┤
│                                                           │
│ Inbound Message (e.g., Slack)                           │
│ ├─ Resolve agent & session key                         │
│ ├─ Lane = resolveSessionLane(sessionKey)               │
│ │  → "session:agent:main:slack:direct:user123"        │
│ │                                                       │
│ └─ enqueueCommandInLane(lane, async () => {           │
│    ├─ Load session from store                         │
│    ├─ Get reply from AI agent                        │
│    ├─ Update session (tokens, updatedAt)             │
│    ├─ Save session store (with lock)                 │
│    └─ Send reply                                     │
│ })                                                     │
│                                                           │
│ Effect:                                                 │
│ - Messages for same session execute serially          │
│ - Different sessions execute in parallel              │
│ - Session data never corrupted                        │
│ - Each session isolation prevents cross-talk          │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

## Multi-User Example

```
Slack Messages (concurrent):
├─ @user1: "hello"  → resolveSessionLane("...user1")
│                    → Lane #1 (serialized)
│
├─ @user2: "help"   → resolveSessionLane("...user2")
│                    → Lane #2 (parallel with #1)
│
└─ #general: "ping" → resolveSessionLane("...C1234")
                     → Lane #3 (parallel with #1, #2)

Result: 3 messages handled concurrently without races ✓
```

---

# Design Patterns

## 1. Lane-per-Session Pattern

Named lanes keyed by session ensure serialization while allowing parallelism across sessions.

## 2. TTL-based Cache

45s cache with mtime validation reduces disk I/O without stale data.

## 3. Lock-based Concurrency

File-based locks serialize writes across processes with stale lock eviction.

## 4. Stale Lock Eviction

30s timeout prevents deadlocks from crashed processes.

## 5. Atomic Writes

Rename-based atomicity on POSIX; direct write on Windows.

## 6. Session Lifecycle

Sessions auto-create, auto-persist, auto-prune (30d TTL).

## 7. Identity Linking

`identityLinks` config maps multi-channel usernames to single session.

---

# Performance Characteristics

**Command Lanes**:
- Memory: O(lanes) + O(queue_size)
- CPU: Minimal (event-driven pump)
- Latency: <1ms enqueue overhead
- Throughput: Unlimited (limited by maxConcurrent)

**Session Management**:
- Memory: O(sessions) with 45s cache
- Disk I/O: Atomic write on update
- Lock contention: Minimal (25ms retry)
- Storage: ~1KB per session

---

# Summary

The command lanes & session management systems provide:

✓ Race condition prevention via serialization
✓ Concurrent execution across sessions
✓ Persistent conversation contexts
✓ Multi-channel session routing
✓ File-based locking for concurrency
✓ TTL-based caching for performance
✓ Atomic writes for data integrity
✓ Stale lock eviction for reliability
✓ Automatic session pruning (30d)
✓ Identity linking across channels

This architecture ensures **consistency, concurrency, and reliability** in multi-user, multi-channel environments without race conditions or data corruption.
