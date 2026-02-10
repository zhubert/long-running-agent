# OpenClaw Cron Scheduler System

## Overview

The OpenClaw cron scheduler is a **custom in-process job scheduler** (not system cron) that manages scheduled tasks with sophisticated job management, exponential backoff, and session isolation.

**Location**: `/src/cron/` (38 files)

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     CronService (Facade)                     │
├─────────────────────────────────────────────────────────────┤
│  service.ts - Thin API layer                                │
└────────────┬────────────────────────────────────────────────┘
             ↓
┌─────────────────────────────────────────────────────────────┐
│              CronServiceState (Core State)                   │
├─────────────────────────────────────────────────────────────┤
│  - jobs: Map<string, CronJob>                               │
│  - timer: NodeJS.Timeout                                     │
│  - running: boolean                                          │
│  - deps: CronServiceDeps (callbacks)                        │
└────────────┬────────────────────────────────────────────────┘
             ↓
┌─────────────────────────────────────────────────────────────┐
│                  Timer & Execution Engine                    │
├─────────────────────────────────────────────────────────────┤
│  timer.ts:                                                   │
│  - armTimer() - Schedule next wakeup (max 60s)             │
│  - onTimer() - Execute due jobs                             │
│  - executeJobCore() - Main/isolated execution               │
└────────────┬────────────────────────────────────────────────┘
             ↓
┌─────────────────────────────────────────────────────────────┐
│                 Persistence & Recovery                       │
├─────────────────────────────────────────────────────────────┤
│  store.ts:                                                   │
│  - loadCronStore() - Load from disk with migrations         │
│  - writeCronStore() - Atomic write with backup              │
│  - Location: ~/.openclaw/cron/jobs.json                     │
└─────────────────────────────────────────────────────────────┘
```

---

## Schedule Types

### 1. One-Shot Jobs (`kind: "at"`)

```typescript
{
  kind: "at",
  at: "2026-02-15T14:30:00Z"  // ISO string or unix timestamp
}
```

**Behavior**:
- Runs once at absolute time
- Auto-disables after execution (with `lastStatus="ok"`)
- Optional `deleteAfterRun` removes job from store
- If missed during downtime, runs on next startup

### 2. Recurring Interval (`kind: "every"`)

```typescript
{
  kind: "every",
  everyMs: 300000,        // 5 minutes
  anchorMs?: 1706123456   // Optional alignment point
}
```

**Behavior**:
- Repeats at fixed intervals
- Anchor ensures regular intervals (e.g., every 5min from :00)
- Without anchor: `nextRunAtMs = lastRunAtMs + everyMs`
- With anchor: `nextRunAtMs = anchor + ceil((now - anchor) / everyMs) * everyMs`

### 3. Cron Expressions (`kind: "cron"`)

```typescript
{
  kind: "cron",
  expr: "0 9 * * MON-FRI",  // Weekdays at 9am
  tz?: "America/New_York"   // Optional timezone
}
```

**Behavior**:
- Full cron expression support via `croner` library
- Timezone-aware scheduling
- Second-precision (floors to second boundary)

---

## Timer & Wakeup System

### Wakeup Mechanism (`timer.ts`)

```typescript
const MAX_TIMER_DELAY_MS = 60_000;  // 1-minute max delay

function armTimer(state: CronServiceState) {
  // Find earliest nextRunAtMs across all enabled jobs
  const nextWakeAtMs = Math.min(...enabledJobs.map(j => j.state.nextRunAtMs));

  if (nextWakeAtMs === Infinity) {
    state.timer = null;  // No jobs scheduled
    return;
  }

  const delay = Math.max(nextWakeAtMs - Date.now(), 0);
  const clampedDelay = Math.min(delay, MAX_TIMER_DELAY_MS);

  state.timer = setTimeout(() => onTimer(state), clampedDelay);
}
```

**Why 60-second clamping?**
- Prevents drift from system clock jumps
- Ensures scheduler wakes periodically
- Catches up missed jobs from process pauses

### Execution Flow

```
Timer fires → onTimer()
  ↓
Find all jobs where: enabled && now >= nextRunAtMs
  ↓
Mark jobs as runningAtMs = now
  ↓
Execute jobs in parallel (with timeout)
  ↓
For each job result:
  - Update state (lastRunAtMs, lastStatus, lastError)
  - Apply exponential backoff if error
  - Recompute nextRunAtMs
  - Delete if one-shot + deleteAfterRun
  ↓
Persist state to disk
  ↓
Sweep session reaper (cron session cleanup)
  ↓
armTimer() - Schedule next wakeup
```

---

## Job Types & Execution

### Main Session Jobs

```typescript
{
  sessionTarget: "main",
  wakeMode: "now" | "next-heartbeat",
  payload: {
    kind: "systemEvent",
    text: "Weekly report reminder"
  }
}
```

**Execution**:
1. Enqueue system event to main session
2. Wake mode determines trigger:
   - `"now"`: Run heartbeat immediately (wait up to 2min for queue)
   - `"next-heartbeat"`: Schedule for next heartbeat cycle
3. Agent processes event on next heartbeat

### Isolated Session Jobs

```typescript
{
  sessionTarget: "isolated",
  wakeMode: "now" | "next-heartbeat",
  payload: {
    kind: "agentTurn",
    message: "Generate weekly summary",
    model?: "anthropic/claude-opus-4-6",
    thinking?: "high",
    timeoutSeconds?: 600
  },
  delivery: {
    mode: "announce",
    channel: "telegram",
    to: "+1234567890",
    bestEffort?: true
  }
}
```

**Execution**:
1. Create ephemeral session: `cron:{jobId}:run:{uuid}`
2. Run agent with specialized prompt
3. Extract summary from output
4. Deliver result:
   - Via outbound payloads (structured content)
   - Or via subagent announce flow (text)
5. Prune old run sessions (24h retention default)

---

## Error Handling & Backoff

### Exponential Backoff Schedule

```typescript
const ERROR_BACKOFF_SCHEDULE_MS = [
  30_000,      // 1st error  → 30 seconds
  60_000,      // 2nd error  → 1 minute
  5 * 60_000,  // 3rd error  → 5 minutes
  15 * 60_000, // 4th error  → 15 minutes
  60 * 60_000, // 5th+ error → 60 minutes
];
```

### Backoff Logic

```typescript
if (result.status === "error") {
  job.state.consecutiveErrors++;

  const backoffMs = errorBackoffMs(job.state.consecutiveErrors);
  const normalNext = computeJobNextRunAtMs(job, endedAt);
  const backoffNext = endedAt + backoffMs;

  // Use whichever is later
  job.state.nextRunAtMs = Math.max(normalNext, backoffNext);

} else {  // Success or skip
  job.state.consecutiveErrors = 0;  // Reset
  job.state.nextRunAtMs = computeJobNextRunAtMs(job, endedAt);
}
```

**Example**: 10-second interval job fails 3 times
- Natural next: +10 sec
- Backoff next: +5 min
- Actual next: +5 min (backoff wins)

### Timeout Protection

```typescript
const DEFAULT_JOB_TIMEOUT_MS = 10 * 60_000;  // 10 minutes

Promise.race([
  executeJobCore(job),
  timeout(jobTimeoutMs)
])
```

---

## Persistence & Recovery

### Storage Format

```typescript
// ~/.openclaw/cron/jobs.json
{
  "version": 1,
  "jobs": [
    {
      "id": "job-uuid",
      "name": "Weekly Report",
      "enabled": true,
      "schedule": { "kind": "every", "everyMs": 604800000 },
      "sessionTarget": "main",
      "wakeMode": "now",
      "payload": { "kind": "systemEvent", "text": "..." },
      "state": {
        "nextRunAtMs": 1706789000000,
        "lastRunAtMs": 1706184200000,
        "lastStatus": "ok",
        "consecutiveErrors": 0
      }
    }
  ]
}
```

### Atomic Write Pattern

```typescript
// Write to temp file
await fs.writeFile(`${storePath}.tmp`, json);

// Atomic rename
await fs.rename(`${storePath}.tmp`, storePath);

// Create backup (best-effort)
await fs.copyFile(storePath, `${storePath}.bak`);
```

### Startup Recovery

```
1. Load store from disk
2. Clear stale runningAtMs (crash recovery)
3. runMissedJobs():
   - Find jobs that should have run during downtime
   - Execute synchronously in order
4. recomputeNextRuns()
5. Persist updated state
6. armTimer()
```

### Stuck Job Detection

```typescript
const STUCK_RUN_MS = 2 * 60 * 60 * 1000;  // 2 hours

if (job.state.runningAtMs && now - job.state.runningAtMs > STUCK_RUN_MS) {
  log.warn("clearing stuck running marker");
  job.state.runningAtMs = undefined;
}
```

---

## Session Management

### Session Isolation

**Main Session**: Single shared session
- Key: `agent:{agentId}:main`
- System events enqueued to main agent
- All jobs share conversation context

**Isolated Sessions**: Ephemeral per-run sessions
- Base key: `cron:{jobId}`
- Run key: `cron:{jobId}:run:{uuid}`
- Each execution gets fresh sessionId
- Old sessions pruned after 24h (configurable)

### Session Reaper

```typescript
// Runs every 5 minutes (throttled)
sweepCronRunSessions():
  - Find sessions matching pattern: ...cron:{id}:run:...
  - Delete if updatedAt < (now - retentionMs)
  - Default retention: 24 hours
  - Configurable via cron.sessionRetention
```

---

## Delivery System

### Delivery Configuration

```typescript
delivery: {
  mode: "announce" | "none",
  channel: "last" | "telegram" | "slack" | ...,
  to?: string,              // Recipient phone/email/ID
  bestEffort?: boolean      // Continue on delivery failure
}
```

### Delivery Flow

```
Agent completes isolated job
  ↓
Check: Should deliver?
  - delivery.mode === "announce"?
  - OR legacy payload.deliver === true?
  ↓
Skip if:
  - Heartbeat-only response
  - Message already sent via messaging tool
  ↓
Resolve target:
  - channel: "last" → use last delivery channel
  - to: undefined → use last recipient
  ↓
Deliver via:
  - Outbound payloads (structured content)
  - OR subagent announce flow (text)
  ↓
If bestEffort: true → ignore delivery errors
```

---

## Configuration

### Global Config

```typescript
cron: {
  enabled?: boolean,              // Default: true
  sessionRetention?: string | false,  // "24h", "7d", false=no pruning
}
```

### Job Configuration

```typescript
type CronJob = {
  id: string,
  name: string,
  description?: string,
  enabled: boolean,
  deleteAfterRun?: boolean,       // Auto-delete after "at" job

  schedule: CronSchedule,
  sessionTarget: "main" | "isolated",
  wakeMode: "now" | "next-heartbeat",

  payload: CronPayload,
  delivery?: CronDelivery,        // Isolated jobs only

  state: {
    nextRunAtMs?: number,
    runningAtMs?: number,
    lastRunAtMs?: number,
    lastStatus?: "ok" | "error" | "skipped",
    lastError?: string,
    lastDurationMs?: number,
    consecutiveErrors?: number
  }
}
```

---

## Integration with Gateway

### Dependency Injection

```typescript
const cron = new CronService({
  storePath: "~/.openclaw/cron/jobs.json",
  cronEnabled: true,

  // Callbacks to gateway
  enqueueSystemEvent: (text, opts) => { /* ... */ },
  requestHeartbeatNow: (opts) => { /* ... */ },
  runHeartbeatOnce: async (opts) => { /* ... */ },
  runIsolatedAgentJob: async (params) => { /* ... */ },

  // Session management
  resolveSessionStorePath: (agentId) => { /* ... */ },

  // Monitoring
  onEvent: (evt) => { /* ... */ },

  log: logger
});
```

### Event Notifications

```typescript
type CronEvent = {
  jobId: string,
  action: "added" | "updated" | "removed" | "started" | "finished",
  runAtMs?: number,
  durationMs?: number,
  status?: "ok" | "error" | "skipped",
  error?: string,
  summary?: string,
  sessionId?: string,
  nextRunAtMs?: number
}
```

---

## CLI Operations

### Add Job

```bash
openclaw cron add \
  --name "Daily Backup" \
  --schedule "0 2 * * *" \
  --message "Run daily backup script"
```

### List Jobs

```bash
openclaw cron list
```

### Update Job

```bash
openclaw cron update job-id \
  --enabled false
```

### Remove Job

```bash
openclaw cron remove job-id
```

### Run Job Immediately

```bash
openclaw cron run job-id
```

---

## Key Implementation Details

### Cron Expression Precision

```typescript
// Croner operates at second granularity
// Floor nowMs to second boundary to prevent lookback issues
const nowSecondMs = Math.floor(nowMs / 1000) * 1000;
const next = cron.nextRun(new Date(nowSecondMs - 1));
```

**Why?** If `nowMs = 12:00:00.500` and pattern matches second `:00`, lookback without flooring could land inside the matching second, causing croner to skip ahead.

### One-Shot Job Lifecycle

```
"at" job created
  ↓
Timer fires at scheduled time
  ↓
Execute job
  ↓
If status === "ok":
  - deleteAfterRun=true → DELETE job
  - deleteAfterRun=false → DISABLE job
Else:
  - DISABLE job (prevent retry loop)
```

### Delivery Reconciliation

Legacy format:
```typescript
payload: {
  deliver: true,
  channel: "telegram",
  to: "+1234567890"
}
```

Modern format:
```typescript
delivery: {
  mode: "announce",
  channel: "telegram",
  to: "+1234567890"
}
```

Migration automatically converts legacy → modern on load.

---

## Performance Characteristics

- **Memory**: O(n) where n = number of jobs
- **Timer overhead**: Single setTimeout per wakeup
- **Disk I/O**: Writes only on job changes
- **Concurrency**: Parallel job execution (Promise.all)
- **Lock-free**: No locking needed (single-threaded timer)

---

## Summary

The OpenClaw cron scheduler provides:

✓ Three schedule types (at, every, cron)
✓ Exponential backoff (30s → 60min)
✓ Session isolation (main vs isolated)
✓ Persistent storage with crash recovery
✓ Automatic session pruning
✓ Sophisticated delivery system
✓ Concurrency safety via sequential promise chains
✓ Rich metadata tracking
✓ Comprehensive event notifications

All built with careful attention to robustness, observability, and integration with the agent execution framework.
