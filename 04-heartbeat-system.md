# OpenClaw Heartbeat & System Events

## Overview

The heartbeat system implements **always-on, event-driven polling** that keeps agents responsive to external stimuli without blocking on I/O. Combined with the system events queue, it enables proactive agent behavior.

**Location**: `/src/infra/`

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│              Event Sources (External Triggers)                │
│  - Cron jobs                                                  │
│  - Async exec completion                                      │
│  - Web messages                                               │
│  - Manual wake calls                                          │
└────────────────┬─────────────────────────────────────────────┘
                 ↓
┌──────────────────────────────────────────────────────────────┐
│         System Events Queue (In-Memory, Per-Session)          │
│  - Max 20 events per session                                  │
│  - Deduplicated on enqueue                                    │
│  - Drained on heartbeat execution                             │
└────────────────┬─────────────────────────────────────────────┘
                 ↓
┌──────────────────────────────────────────────────────────────┐
│    Heartbeat Wake Handler (Coalescing & Deduplication)        │
│  - 250ms coalesce window                                      │
│  - Retries on "requests-in-flight"                            │
│  - Single pending reason tracking                             │
└────────────────┬─────────────────────────────────────────────┘
                 ↓
┌──────────────────────────────────────────────────────────────┐
│   Heartbeat Runner (Interval-Based Scheduler)                 │
│  - Multi-agent heartbeat intervals                            │
│  - Active hours enforcement                                   │
│  - Per-session heartbeat state                                │
└────────────────┬─────────────────────────────────────────────┘
                 ↓
┌──────────────────────────────────────────────────────────────┐
│          Heartbeat Execution Engine                           │
│  - Empty content detection                                    │
│  - Visibility & channel routing                               │
│  - Duplicate suppression (24h window)                         │
│  - Model invocation with specialized prompts                  │
└──────────────────────────────────────────────────────────────┘
```

---

## System Events Queue

**File**: `/src/infra/system-events.ts` (110 lines)

### Queue Architecture

```typescript
type SystemEvent = {
  text: string,
  ts: number
};

type SessionQueue = {
  queue: SystemEvent[],      // Max 20 events
  lastText: string | null,   // For deduplication
  lastContextKey: string | null
};

const queues = new Map<string, SessionQueue>();  // Session-scoped
```

### Core Operations

**Enqueue**:
```typescript
function enqueueSystemEvent(text: string, options: SystemEventOptions) {
  const key = requireSessionKey(options?.sessionKey);
  const entry = queues.get(key) ?? {
    queue: [],
    lastText: null,
    lastContextKey: null
  };

  const cleaned = text.trim();
  if (!cleaned) return;

  // Skip consecutive duplicates
  if (entry.lastText === cleaned) return;

  entry.lastText = cleaned;
  entry.queue.push({ text: cleaned, ts: Date.now() });

  // FIFO rotation when exceeding max
  if (entry.queue.length > MAX_EVENTS) {
    entry.queue.shift();  // Remove oldest
  }

  queues.set(key, entry);
}
```

**Drain** (consume & empty):
```typescript
function drainSystemEventEntries(sessionKey: string): SystemEvent[] {
  const entry = queues.get(key);
  if (!entry || entry.queue.length === 0) return [];

  const out = entry.queue.slice();
  entry.queue.length = 0;
  entry.lastText = null;
  queues.delete(key);  // Clean up
  return out;
}
```

**Peek** (non-destructive):
```typescript
function peekSystemEvents(sessionKey: string): string[] {
  return queues.get(key)?.queue.map(e => e.text) ?? [];
}
```

### Integration with Prompts

Events are prepended to agent prompts:

```typescript
// From /src/auto-reply/reply/session-updates.ts
async function prependSystemEvents(params) {
  const queued = drainSystemEventEntries(params.sessionKey);  // Consume

  const systemLines = queued
    .map(event => {
      const compacted = compactSystemEvent(event.text);
      return `[${formatTimestamp(event.ts)}] ${compacted}`;
    })
    .filter(Boolean);

  if (systemLines.length === 0) return params.prefixedBodyBase;

  const block = systemLines.map(l => `System: ${l}`).join("\n");
  return `${block}\n\n${params.prefixedBodyBase}`;
}
```

**Output Format**:
```
System: [14:35:22] Exec finished: npm test passed
System: [14:36:10] Cron: Weekly report generated

<Agent's normal prompt>
```

---

## Heartbeat Wake Handler

**File**: `/src/infra/heartbeat-wake.ts` (77 lines)

### Coalescing Algorithm

```typescript
let handler: HeartbeatWakeHandler | null = null;
let pendingReason: string | null = null;
let scheduled = false;
let running = false;
let timer: NodeJS.Timeout | null = null;

const DEFAULT_COALESCE_MS = 250;  // Wait 250ms
const DEFAULT_RETRY_MS = 1_000;   // Retry if busy

function requestHeartbeatNow(opts?: { reason?: string, coalesceMs?: number }) {
  pendingReason = opts?.reason ?? pendingReason ?? "requested";
  schedule(opts?.coalesceMs ?? DEFAULT_COALESCE_MS);
}

function schedule(coalesceMs: number) {
  if (timer) return;  // Already scheduled

  timer = setTimeout(async () => {
    timer = null;
    scheduled = false;

    if (!handler) return;

    if (running) {
      // Still running, reschedule
      scheduled = true;
      schedule(coalesceMs);
      return;
    }

    const reason = pendingReason;
    pendingReason = null;
    running = true;

    try {
      const res = await handler({ reason });

      // If busy (requests-in-flight), retry
      if (res.status === "skipped" && res.reason === "requests-in-flight") {
        pendingReason = reason ?? "retry";
        schedule(DEFAULT_RETRY_MS);
      }
    } catch {
      pendingReason = reason ?? "retry";
      schedule(DEFAULT_RETRY_MS);
    } finally {
      running = false;
      if (pendingReason || scheduled) {
        schedule(coalesceMs);
      }
    }
  }, coalesceMs);

  timer.unref?.();  // Don't keep process alive
}
```

### Wake Reasons

| Reason | Source | Description |
|--------|--------|-------------|
| `"interval"` | Heartbeat scheduler | Regular interval tick |
| `"exec-event"` | Async command | Background command completed |
| `"cron:{jobId}"` | Cron job | Scheduled job triggered |
| `"wake"` | Manual call | Explicit wake request |
| `"requested"` | Direct request | Generic wake |
| `"retry"` | Backoff | Retry after busy |

---

## Heartbeat Runner

**File**: `/src/infra/heartbeat-runner.ts` (1,031 lines)

### Interval Scheduling

```typescript
type HeartbeatAgentState = {
  agentId: string,
  heartbeat?: HeartbeatConfig,
  intervalMs: number,      // How often to check
  lastRunMs?: number,      // When it last ran
  nextDueMs: number        // When to run next
};

function scheduleNext() {
  if (state.agents.size === 0) return;

  const now = Date.now();
  let nextDue = Number.POSITIVE_INFINITY;

  for (const agent of state.agents.values()) {
    if (agent.nextDueMs < nextDue) {
      nextDue = agent.nextDueMs;
    }
  }

  const delay = Math.max(0, nextDue - now);
  state.timer = setTimeout(() => {
    requestHeartbeatNow({ reason: "interval", coalesceMs: 0 });
  }, delay);
}
```

### Configuration

```typescript
type HeartbeatConfig = {
  enabled: boolean,
  every: string,              // "5m", "1h", etc.
  prompt?: string,            // Custom prompt
  target?: string,            // Delivery target ("last", "telegram")
  model?: string,             // Model override
  ackMaxChars?: number,       // Max chars for "OK" response
  includeReasoning?: boolean, // Include thinking
  activeHours?: {
    start: string,            // "09:00"
    end: string,              // "18:00"
    timezone: string          // "America/New_York"
  }
};
```

**Defaults**:
- `every`: "5m" (5 minutes)
- `ackMaxChars`: 50
- `target`: "last"

### Active Hours Enforcement

```typescript
function isWithinActiveHours(cfg, heartbeat, nowMs) {
  const active = heartbeat?.activeHours;
  if (!active) return true;

  // Parse times
  const startMin = parseTime(active.start);  // HH:MM → minutes
  const endMin = parseTime(active.end);

  // Get current time in agent's timezone
  const timeZone = resolveActiveHoursTimezone(cfg, active.timezone);
  const currentMin = resolveMinutesInTimeZone(nowMs, timeZone);

  // Check if within range
  if (endMin > startMin) {
    return currentMin >= startMin && currentMin < endMin;
  }

  // Handle wrap-around (e.g., 22:00 to 08:00)
  return currentMin >= startMin || currentMin < endMin;
}
```

---

## Execution Flow

### Gate Checklist

```
1. Enable Checks
   - heartbeatsEnabled?
   - isHeartbeatEnabledForAgent()?
   - has valid interval?

2. Active Hours
   - isWithinActiveHours()?

3. Queue Availability
   - getQueueSize(CommandLane.Main) === 0?

4. Content Check
   - HEARTBEAT.md has content?
   - OR exec event reason?
   - OR cron event reason?

5. Resolution
   - resolveHeartbeatSession()
   - resolveHeartbeatDeliveryTarget()
   - resolveHeartbeatVisibility()

6. Prompt Selection
   - hasExecCompletion? → EXEC_EVENT_PROMPT
   - hasCronEvents? → CRON_EVENT_PROMPT
   - else → resolveHeartbeatPrompt()

7. Visibility Check
   - showAlerts || showOk || useIndicator?

8. Model Invocation
   - getReplyFromConfig() with isHeartbeat=true

9. Result Processing
   - Extract main payload
   - Check for skip conditions
   - Route to delivery target
   - Record in session store
```

### Prompts

**Standard Heartbeat**:
```
Read HEARTBEAT.md if it exists. If there's actionable content,
briefly summarize.
```

**Exec Event**:
```
An async command you ran earlier has completed. The result is shown
in the system messages above. Please relay the command output to the
user in a helpful way.
```

**Cron Event**:
```
A scheduled reminder has been triggered. The reminder message is shown
in the system messages above. Please relay this reminder to the user
in a helpful and friendly way.
```

---

## Visibility Configuration

**File**: `/src/infra/heartbeat-visibility.ts` (74 lines)

### Visibility Levels

```typescript
type ResolvedHeartbeatVisibility = {
  showOk: boolean,       // Send "HEARTBEAT_OK" when quiet?
  showAlerts: boolean,   // Send content messages?
  useIndicator: boolean  // Show UI status indicator?
};

// Defaults
showOk: false          // Silent when nothing to say
showAlerts: true       // Show when there's content
useIndicator: true     // UI shows heartbeat status
```

### Precedence Order

```
Per-account config
  ↓ (if not set)
Per-channel config
  ↓ (if not set)
Channel defaults
  ↓ (if not set)
Global defaults
```

**Example Config**:
```yaml
channels:
  defaults:
    heartbeat:
      showOk: false
      showAlerts: true

  telegram:
    heartbeat:
      showOk: true        # Send OK on Telegram

    accounts:
      bot123:
        heartbeat:
          showAlerts: false  # Mute alerts for this bot
```

---

## Integration Examples

### Example 1: Cron → Heartbeat

```
Cron job executes
  ↓
enqueueSystemEvent("Weekly report reminder", { sessionKey: "main" })
  ↓
requestHeartbeatNow({ reason: "cron:job-id" })
  ↓
Wake handler coalesces (250ms)
  ↓
Heartbeat runner checks pending events
  ↓
hasCronEvents = true → CRON_EVENT_PROMPT
  ↓
Model invoked with:
  System: [14:30:00] Weekly report reminder

  A scheduled reminder has been triggered...
  ↓
Model output: "Your weekly report is due..."
  ↓
Deliver to target channel
```

### Example 2: Exec Command → Heartbeat

```
User runs: bash "npm test"
  ↓
Command executes in background
  ↓
On exit:
  enqueueSystemEvent("Exec finished: npm test passed")
  requestHeartbeatNow({ reason: "exec:session-id:exit" })
  ↓
Heartbeat runner:
  hasExecCompletion = true → EXEC_EVENT_PROMPT
  ↓
Model sees:
  System: [14:35:22] Exec finished: npm test passed

  An async command you ran earlier has completed...
  ↓
Model output: "Your tests passed! All 42 tests successful."
  ↓
Send reply
```

### Example 3: Interval Polling

```
T = 0:00 - startHeartbeatRunner()
  ↓
agents["main"].nextDueMs = now + 5min
agents["ops"].nextDueMs = now + 10min
  ↓
scheduleNext() → setTimeout(5min)
  ↓
T = 5:00 - Timer fires
  ↓
requestHeartbeatNow({ reason: "interval" })
  ↓
Heartbeat runner executes:
  - Agent "main": within active hours? Yes
  - Queue empty? Yes
  - HEARTBEAT.md has content? Yes
  → Invoke model
  ↓
Model checks HEARTBEAT.md:
  "# TODO: Buy groceries"
  ↓
Model output: "Reminder: You wanted to buy groceries"
  ↓
Send to last delivery channel
  ↓
agents["main"].lastRunMs = now
agents["main"].nextDueMs = now + 5min
  ↓
scheduleNext() → setTimeout(5min)
```

---

## Skip Conditions

| Condition | Reason | Code Location |
|-----------|--------|---------------|
| **disabled** | Heartbeats globally off | Line 500-504 |
| **quiet-hours** | Outside active hours | Line 511-513 |
| **requests-in-flight** | Queue has pending work | Line 515-518 |
| **empty-heartbeat-file** | No actionable content | Line 530-540 |
| **alerts-disabled** | All visibility off | Line 598-607 |
| **no-target** | No delivery channel | Line 746-756 |
| **ok-empty** | Model returned nothing | Line 647-667 |
| **ok-token** | Model only returned token | Line 683-701 |
| **duplicate** | Same message (24h window) | Line 720-736 |

---

## Event Notifications

**File**: `/src/infra/heartbeat-events.ts`

```typescript
type HeartbeatEventPayload = {
  ts: number,
  status: "sent" | "ok-empty" | "ok-token" | "skipped" | "failed",
  to?: string,
  accountId?: string,
  preview?: string,
  durationMs?: number,
  hasMedia?: boolean,
  reason?: string,
  channel?: string,
  silent?: boolean,
  indicatorType?: "ok" | "alert" | "error"
};

// Indicator mapping
"ok-empty" / "ok-token" → "ok"      // ✓ Status OK
"sent" → "alert"                    // ! Message sent
"failed" → "error"                  // ✗ Error
"skipped" → undefined               // (no indicator)
```

---

## Configuration Examples

### Multi-Agent Heartbeats

```yaml
agents:
  defaults:
    heartbeat:
      every: "5m"
      target: "last"

  list:
    - id: main
      heartbeat:
        every: "3m"
        activeHours:
          start: "09:00"
          end: "18:00"
          timezone: "America/New_York"

    - id: monitoring
      heartbeat:
        every: "1m"
        target: "telegram"
        ackMaxChars: 0       # Never send OK
        includeReasoning: true

    - id: quiet
      heartbeat: null        # Disabled
```

### Channel-Specific Visibility

```yaml
channels:
  defaults:
    heartbeat:
      showOk: false
      showAlerts: true
      useIndicator: true

  telegram:
    heartbeat:
      showOk: true

    accounts:
      bot123:
        heartbeat:
          showAlerts: false
          useIndicator: false
```

---

## Performance Characteristics

- **Memory**: O(agents) for state tracking
- **CPU**: Minimal (single timer + event queue)
- **Latency**: 250ms coalesce window
- **Throughput**: Unlimited events (max 20 per session)
- **Concurrency**: Single-threaded (no locks)

---

## Summary

The heartbeat & system events system provides:

✓ Always-on polling with configurable intervals
✓ Event-driven triggering (cron, exec, manual)
✓ Intelligent coalescing (250ms window)
✓ Active hours enforcement (timezone-aware)
✓ Per-session event queues (max 20, deduplicated)
✓ Multi-agent support (independent intervals)
✓ Visibility controls (per-account/channel)
✓ Duplicate suppression (24h window)
✓ Comprehensive skip conditions
✓ Rich event notifications

This architecture enables **low-latency, event-driven agent responsiveness** while maintaining **strict session isolation** and **backpressure handling**.
