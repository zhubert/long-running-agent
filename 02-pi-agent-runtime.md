# OpenClaw Pi Agent Runtime

## Overview

The Pi embedded agent runtime is OpenClaw's **execution engine** that wraps Anthropic's Pi agent framework with multi-model support, context management, failover, and streaming capabilities.

**Location**: `/src/agents/pi-embedded-runner/` (27 files)

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│              runEmbeddedPiAgent() - Entry Point              │
├─────────────────────────────────────────────────────────────┤
│  - Parameter validation                                      │
│  - Lane resolution (per-session + global)                   │
│  - Enqueue in command lane for serialization                │
└────────────┬────────────────────────────────────────────────┘
             ↓
┌─────────────────────────────────────────────────────────────┐
│           Model Resolution & Auth Setup                      │
├─────────────────────────────────────────────────────────────┤
│  model.ts:                                                   │
│  - resolveModel(provider, modelId)                          │
│  - Build ModelRegistry + AuthStorage                        │
│  - Forward-compat handling (gpt-5.3, opus-4.6)             │
└────────────┬────────────────────────────────────────────────┘
             ↓
┌─────────────────────────────────────────────────────────────┐
│              Session Creation & Lifecycle                    │
├─────────────────────────────────────────────────────────────┤
│  run/attempt.ts:                                             │
│  - createAgentSession() - Initialize Pi session             │
│  - Load conversation history from disk                       │
│  - Apply hooks (before_agent_start)                         │
└────────────┬────────────────────────────────────────────────┘
             ↓
┌─────────────────────────────────────────────────────────────┐
│             Execution Loop with Retry Logic                  │
├─────────────────────────────────────────────────────────────┤
│  - Max attempts: 5 (default)                                │
│  - Auth profile rotation on failure                         │
│  - Thinking level fallback                                  │
│  - Context overflow recovery                                │
└────────────┬────────────────────────────────────────────────┘
             ↓
┌─────────────────────────────────────────────────────────────┐
│            Streaming & Tool Execution                        │
├─────────────────────────────────────────────────────────────┤
│  pi-embedded-subscribe.ts:                                   │
│  - subscribeEmbeddedPiSession() - Real-time streaming       │
│  - Tool invocation handling                                 │
│  - Block reply chunking                                     │
│  - Reasoning stream capture                                 │
└────────────┬────────────────────────────────────────────────┘
             ↓
┌─────────────────────────────────────────────────────────────┐
│            Result Processing & Cleanup                       │
├─────────────────────────────────────────────────────────────┤
│  - Usage tracking (input/output/cache tokens)               │
│  - Payload building (text + tool results)                   │
│  - Apply hooks (agent_end)                                  │
│  - Session state snapshot                                   │
└─────────────────────────────────────────────────────────────┘
```

---

## Execution Modes

### 1. Interactive (Streaming) Mode

**Trigger**: User sends prompt to agent

```typescript
const result = await runEmbeddedPiAgent({
  sessionId: "abc-123",
  sessionFile: "~/.openclaw/sessions/abc-123.json",
  workspaceDir: "~/.openclaw/workspace",
  prompt: "List files in my home directory",

  // Streaming callbacks
  onPartialReply: async (payload) => {
    console.log(payload.text);  // Print incrementally
  },
  onToolResult: async (payload) => {
    console.log(`Tool: ${payload.name}`);
  }
});
```

**Flow**:
1. Session.prompt(text) invoked
2. Model streams response
3. Tools execute automatically
4. Results stream back to client
5. Final result returned

### 2. RPC Mode (Request-Response)

**Trigger**: Gateway method invocation

```typescript
// Gateway exposes RPC methods
await gatewayClient.request("agent", {
  sessionId: "abc-123",
  message: "What's the weather?"
});
```

**Flow**:
1. Gateway receives RPC request
2. Invokes runEmbeddedPiAgent internally
3. Waits for completion
4. Returns final result
5. No streaming to client

### 3. Compaction Mode

**Trigger**: Context window approaching limit

```typescript
const compactResult = await compactEmbeddedPiSessionDirect({
  sessionId,
  sessionKey,
  sessionFile,
  workspaceDir,
  model,
  thinkLevel: "off"
});
```

**Flow**:
1. Load session messages
2. Calculate token budget
3. Run summarization LLM call
4. Replace messages with summary
5. Persist compacted session

---

## Model Resolution & Auth

### Model Discovery

```typescript
// model.ts
function resolveModel(provider: string, modelId: string) {
  // 1. Check built-in model registry
  if (builtInModels.has(modelId)) {
    return builtInModels.get(modelId);
  }

  // 2. Check inline provider configs
  if (cfg.models?.providers?.[provider]) {
    return createCustomModel(cfg.models.providers[provider]);
  }

  // 3. Forward-compat handling
  if (provider === "openai" && modelId === "gpt-5.3") {
    // Clone gpt-5.2 config
    return cloneModel("gpt-5.2", { id: "gpt-5.3" });
  }

  if (provider === "anthropic" && modelId === "claude-opus-4-6") {
    // Clone claude-opus-4-5
    return cloneModel("claude-opus-4-5", { id: "claude-opus-4-6" });
  }

  // 4. Generic provider config with defaults
  return createGenericModel(provider, modelId);
}
```

### Auth Profile Resolution

**Configuration**:
```typescript
agents: {
  defaults: {
    model: {
      primary: "anthropic/claude-opus-4-6",
      fallbacks: ["anthropic/claude-sonnet-4-5", "openai/gpt-5.2"],
      profileOrder: ["profile-1", "profile-2", "profile-3"]
    }
  }
}

auth: {
  profiles: {
    "profile-1": { provider: "anthropic", apiKey: "..." },
    "profile-2": { provider: "openai", apiKey: "..." }
  }
}
```

**Resolution Flow**:
```typescript
// 1. Determine candidate profiles
const candidates = lockedProfileId
  ? [lockedProfileId]
  : profileOrder.length > 0
    ? profileOrder
    : [undefined];  // Default profile

// 2. Try each profile
for (let i = profileIndex; i < candidates.length; i++) {
  const profileId = candidates[i];

  // Skip profiles in cooldown
  if (isProfileInCooldown(authStore, profileId)) {
    continue;
  }

  // Apply API key
  await applyApiKeyInfo(profileId);

  // Attempt request
  const result = await session.prompt(text);

  if (success) {
    markAuthProfileGood(authStore, profileId);
    return result;
  }

  // Failure - mark and try next
  markAuthProfileFailure(authStore, profileId, reason);
}
```

---

## Context Management

### Session Compaction

**Trigger**: Context usage exceeds threshold

```typescript
// Automatic compaction on overflow
const MAX_OVERFLOW_COMPACTION_ATTEMPTS = 3;

for (let attempt = 0; attempt < MAX_OVERFLOW_COMPACTION_ATTEMPTS; attempt++) {
  try {
    return await session.prompt(text);
  } catch (err) {
    if (isContextOverflowError(err) && attempt < MAX_OVERFLOW_COMPACTION_ATTEMPTS - 1) {
      await compactEmbeddedPiSessionDirect({ sessionId, ... });
      continue;  // Retry
    }
    throw err;
  }
}
```

**Compaction Strategy**:
```typescript
// Calculate reserve tokens (for new prompt + tools)
const reserveTokens = Math.max(
  cfg.agents?.defaults?.compaction?.minReserveTokens ?? 0,
  32000  // Default minimum
);

// Available for history
const historyBudget = contextWindowTokens - reserveTokens;

// Run summarization
const summary = await summarizeSessionHistory({
  messages: session.messages,
  tokenBudget: historyBudget,
  strategy: "exact" | "aggressive" | "no-prune"
});

// Replace messages
session.replaceMessages([summary]);
```

### Tool Result Truncation

**Problem**: Single oversized tool result exceeds context limit

**Solution**:
```typescript
const MAX_TOOL_RESULT_CONTEXT_SHARE = 0.3;  // 30% of context
const HARD_MAX_TOOL_RESULT_CHARS = 400_000; // 100K tokens worst case

function calculateMaxToolResultChars(contextWindowTokens: number): number {
  const maxTokens = Math.floor(contextWindowTokens * 0.3);
  const maxChars = maxTokens * 4;  // ~4 chars per token
  return Math.min(maxChars, HARD_MAX_TOOL_RESULT_CHARS);
}

// Truncate oversized results
await truncateOversizedToolResultsInSession({
  sessionFile,
  contextWindowTokens,
  maxChars: calculateMaxToolResultChars(contextWindowTokens)
});
```

### History Limiting

**DM-specific limits**:
```typescript
// Config
channels: {
  telegram: {
    dmHistoryLimit: 20  // Keep only 20 turns per DM
  }
}

// Applied before prompt
const limit = getDmHistoryLimitFromSessionKey(sessionKey, config);
if (limit) {
  const limited = limitHistoryTurns(messages, limit);
  session.agent.replaceMessages(limited);
}
```

---

## Tool Execution

### Tool Setup

```typescript
// Create tools with context
const toolsRaw = createOpenClawCodingTools({
  exec: { elevated: bashElevated },
  sandbox,
  messageProvider,
  sessionKey,
  modelHasVision: model.input?.includes("image"),
  requireExplicitMessageTarget,
  disableMessageTool
});

// Sanitize for Google models (remove unsupported fields)
const tools = sanitizeToolsForGoogle({ tools: toolsRaw, provider });

// Split into built-in vs custom
const { builtInTools, customTools } = splitSdkTools({ tools, sandboxEnabled });
```

### Tool Invocation

Tools execute automatically via Pi agent's internal loop:

```typescript
// Session automatically handles tool calls
await session.prompt(text);

// During execution:
// 1. Model returns tool_use content block
// 2. Pi agent invokes tool handler
// 3. Tool result appended to messages
// 4. Model continues with tool result
// 5. Repeat until final text response
```

### Client Tools (Hosted Tools)

```typescript
// Tools that execute in client (not agent)
const clientToolDefs = params.clientTools
  ? toClientToolDefinitions(
      params.clientTools,
      (toolName, toolParams) => {
        clientToolCallDetected = { name: toolName, params: toolParams };
      }
    )
  : [];

// If client tool detected, return early with pending tool call
if (clientToolCallDetected) {
  return {
    meta: {
      stopReason: "tool_calls",
      pendingToolCalls: [{
        id: `call_${Date.now()}`,
        name: clientToolCallDetected.name,
        arguments: JSON.stringify(clientToolCallDetected.params)
      }]
    }
  };
}
```

---

## Streaming Architecture

### Stream Function Setup

```typescript
// Base streaming function
session.agent.streamFn = streamSimple;

// Apply cache trace wrapper (debugging)
if (cacheTrace) {
  session.agent.streamFn = cacheTrace.wrapStreamFn(session.agent.streamFn);
}

// Apply Anthropic payload logging
if (anthropicPayloadLogger) {
  session.agent.streamFn = anthropicPayloadLogger.wrapStreamFn(
    session.agent.streamFn
  );
}

// Apply extra params (temperature, maxTokens, etc.)
applyExtraParamsToAgent(session.agent, cfg, provider, modelId);
```

### Session Subscription

```typescript
const subscription = subscribeEmbeddedPiSession({
  session,
  runId,
  verboseLevel,
  reasoningMode: "off" | "on",

  // Callbacks
  onPartialReply: async (payload) => {
    // Streaming text deltas
  },
  onToolResult: async (payload) => {
    // Tool execution results
  },
  onBlockReply: async (payload) => {
    // Structured content blocks
  },
  onReasoningStream: async (payload) => {
    // Thinking/reasoning text
  },
  onAgentEvent: (evt) => {
    // Debug events
  }
});

// Wait for completion
await session.prompt(text);

// Access results
const texts = subscription.assistantTexts;
const toolMetas = subscription.toolMetas;
const usage = subscription.getUsageTotals();
```

---

## Failover & Error Recovery

### FailoverError Classification

```typescript
class FailoverError extends Error {
  reason: "auth" | "rate_limit" | "billing" | "timeout" | "format"
  provider: string
  model: string
  profileId?: string
  status?: number  // HTTP status
}
```

### Recovery Strategies

**1. Auth Failure (401/403)**
```typescript
if (isAuthAssistantError(lastAssistant)) {
  markAuthProfileFailure(authStore, profileId, "auth");
  const rotated = await advanceAuthProfile();
  if (rotated) {
    continue;  // Retry with next profile
  }
  // Throw FailoverError to trigger model fallback
  throw new FailoverError("auth failure", { reason: "auth" });
}
```

**2. Rate Limit (429) / Timeout**
```typescript
if (isRateLimitAssistantError(lastAssistant) || timedOut) {
  markAuthProfileFailure(authStore, profileId, "rate_limit");
  const rotated = await advanceAuthProfile();
  if (rotated) {
    continue;  // Retry with next profile
  }
  throw new FailoverError("rate limit", { reason: "rate_limit", status: 429 });
}
```

**3. Billing Failure (402)**
```typescript
if (isBillingAssistantError(lastAssistant) && fallbackConfigured) {
  throw new FailoverError("billing issue", { reason: "billing", status: 402 });
}
```

**4. Thinking Level Fallback**
```typescript
const fallbackThinking = pickFallbackThinkingLevel({
  message: lastAssistant?.errorMessage,
  attempted: attemptedThinking
});
if (fallbackThinking) {
  thinkLevel = fallbackThinking;
  continue;  // Retry with lower thinking level
}
```

**5. Context Overflow**
```
Tier 1: Auto-compaction (max 3 attempts)
  ↓
Tier 2: Tool result truncation
  ↓
Tier 3: Graceful failure with user message
```

---

## Queue Management

### Active Run Tracking

```typescript
const ACTIVE_EMBEDDED_RUNS = new Map<string, EmbeddedPiQueueHandle>();

// Track active runs
setActiveEmbeddedRun(sessionId, handle);

// Check if session is busy
function isSessionBusy(sessionId: string): boolean {
  return ACTIVE_EMBEDDED_RUNS.has(sessionId);
}

// Queue follow-up message
function queueEmbeddedPiMessage(sessionId: string, text: string): boolean {
  const handle = ACTIVE_EMBEDDED_RUNS.get(sessionId);
  if (!handle?.isStreaming()) return false;
  if (handle.isCompacting()) return false;

  void handle.queueMessage(text);  // Enqueue via session.steer()
  return true;
}
```

### Wait for Completion

```typescript
// Wait for session to finish
const completed = await waitForEmbeddedPiRunEnd(sessionId, timeoutMs);
if (!completed) {
  // Timeout - session still running
}
```

---

## Configuration

### RunEmbeddedPiAgentParams

**Core execution**:
```typescript
{
  sessionId: string,
  sessionKey?: string,
  sessionFile: string,
  workspaceDir: string,
  prompt: string,
  timeoutMs: number,
  runId: string
}
```

**Model selection**:
```typescript
{
  provider?: string,
  model?: string,
  authProfileId?: string,      // Lock to specific profile
  authProfileIdSource?: "auto" | "user"
}
```

**Streaming callbacks**:
```typescript
{
  onPartialReply?: (payload) => void | Promise<void>,
  onAssistantMessageStart?: () => void | Promise<void>,
  onBlockReply?: (payload) => void | Promise<void>,
  onBlockReplyFlush?: () => void | Promise<void>,
  blockReplyBreak?: "text_end" | "message_end",
  onReasoningStream?: (payload) => void | Promise<void>,
  onToolResult?: (payload) => void | Promise<void>,
  onAgentEvent?: (evt) => void
}
```

**AI configuration**:
```typescript
{
  thinkLevel?: "off" | "on" | "extended",
  verboseLevel?: "quiet" | "normal" | "verbose",
  reasoningLevel?: "off" | "on",
  toolResultFormat?: "markdown" | "plain",
  extraSystemPrompt?: string
}
```

**Execution context**:
```typescript
{
  messageChannel?: string,
  agentAccountId?: string,
  groupId?: string,
  spawnedBy?: string,
  senderIsOwner?: boolean,
  execOverrides?: { host?, security?, ask?, node? },
  bashElevated?: boolean
}
```

---

## Performance Optimizations

### Caching

- **Session file caching**: Reads cached for 100ms
- **Model registry caching**: Built once per config
- **Auth profile caching**: Validated profiles cached
- **Tool factory caching**: Tools created per context, cached

### Streaming

- **Chunk aggregation**: Buffer partial replies for efficient transmission
- **Parallel tool execution**: Multiple tools can run concurrently
- **Early termination**: Client can abort mid-stream

### Memory Management

- **Lazy loading**: Session files loaded on-demand
- **Compaction**: Old messages summarized
- **Truncation**: Oversized tool results capped
- **Pruning**: Old sessions deleted automatically

---

## Integration Points

### Gateway Integration

```typescript
// Gateway calls Pi agent for each message
const result = await runEmbeddedPiAgent({
  sessionId,
  sessionFile,
  workspaceDir,
  prompt: userMessage,

  // Gateway provides streaming handlers
  onPartialReply: (payload) => {
    gateway.broadcast("chat.delta", payload);
  },
  onToolResult: (payload) => {
    gateway.broadcast("chat.tool", payload);
  }
});
```

### Cron Integration

```typescript
// Cron jobs invoke isolated agent turns
const result = await runIsolatedAgentJob({
  job,
  message: cronJob.payload.message,
  model: cronJob.payload.model,
  thinking: cronJob.payload.thinking
});
```

### Hooks Integration

```typescript
// Hooks can intercept before/after agent execution
await runModifyingHook("before_agent_start", {
  sessionId,
  prompt,
  systemPrompt
});

const result = await runEmbeddedPiAgent({ ... });

await runVoidHook("agent_end", {
  sessionId,
  result,
  usage
});
```

---

## Summary

The Pi embedded agent runtime provides:

✓ Multi-model support (Anthropic, OpenAI, Google, local)
✓ Auth profile rotation with cooldown
✓ Multi-tier context recovery (compaction → truncation → graceful failure)
✓ Real-time streaming with interrupts
✓ Comprehensive error classification and failover
✓ Tool execution with sandbox support
✓ Session persistence and branching
✓ Hook system for extensibility
✓ Queue management for concurrent requests
✓ Performance optimizations (caching, lazy loading)

This architecture enables production-grade LLM integration with robustness, flexibility, and excellent developer experience.
