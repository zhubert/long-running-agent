# OpenClaw Skills & Plugin System

## Overview

The OpenClaw skills platform is a **progressive disclosure system** that allows agents to discover and use modular packages without overwhelming the context window. Skills are self-contained directories with documentation, scripts, and resources.

**Locations**:
- Skills: `/skills/` (54+ bundled)
- Plugin SDK: `/src/plugin-sdk/`
- Plugin System: `/src/plugins/`

---

## Skill Anatomy

### Directory Structure

```
skills/skill-name/
├── SKILL.md (required)
│   ├── YAML frontmatter (name, description)
│   └── Markdown body (instructions, examples)
├── scripts/ (optional)
│   └── *.py, *.sh - Executable code
├── references/ (optional)
│   └── *.md - Documentation loaded on-demand
└── assets/ (optional)
    └── templates, boilerplate (not loaded to context)
```

### SKILL.md Format

```markdown
---
name: skill-id
description: |
  Brief explanation of what this skill does and when to use it.
  This is the ONLY part Codex reads initially to decide whether
  to use the skill. Keep it concise and action-oriented.
metadata:
  install: npm install -g some-tool
  icon: 🔧
  platforms: [macos, linux]
---

# Skill Name

## Quick Start

\`\`\`bash
command-to-run --with-args
\`\`\`

## Examples

### Example 1: Common use case
\`\`\`bash
concrete-command-here
\`\`\`

### Example 2: Advanced usage
\`\`\`bash
another-command
\`\`\`

## References

For detailed API documentation, see [API.md](references/API.md).
```

---

## Progressive Disclosure Model

### Level 1: Metadata (Always Loaded)

```typescript
// Extracted from YAML frontmatter
{
  name: "github",
  description: "Interact with GitHub using the gh CLI..."
}
```

**Context cost**: ~100 words per skill
**Total cost**: 5,400 words for 54 skills
**Purpose**: Agent scans all skills to find relevant ones

### Level 2: SKILL.md Body (Loaded on Trigger)

```typescript
// Loaded when agent decides to use the skill
const skillBody = await readFile(`skills/${skillName}/SKILL.md`);
```

**Context cost**: ~1,000-5,000 words
**Purpose**: Detailed instructions and examples

### Level 3: Resources (Loaded as Needed)

```typescript
// Agent explicitly reads references
const apiDocs = await readFile(`skills/${skillName}/references/API.md`);

// Or executes scripts directly
await exec(`python skills/${skillName}/scripts/tool.py --args`);
```

**Context cost**: Variable (0 if scripts execute without loading)
**Purpose**: Deep documentation and deterministic execution

---

## Built-in Skills Survey

### Development Tools (7 skills)

- **github**: `gh` CLI for PRs, issues, workflows
- **coding-agent**: AI pair programming with Codex
- **tmux**: Terminal multiplexer management
- **canvas**: Web-based creation workspace
- **clawhub**: Skill registry/marketplace
- **blucli**: BlueBubbles iMessage CLI
- **wacli**: WhatsApp CLI

### Document Processing (3 skills)

- **nano-pdf**: Edit PDFs with natural language
- **nano-banana-pro**: Advanced PDF operations
- **video-frames**: Extract frames from videos

### Communication (6 skills)

- **discord**: Send messages, manage servers, polls
- **slack**: Workspace interactions
- **imsg**: iMessage automation (macOS)
- **bluebubbles**: iMessage via BlueBubbles server
- **voice-call**: Phone calls via Vapi
- **openai-whisper**: Audio transcription

### AI/ML (4 skills)

- **gemini**: Google Gemini API
- **openai-image-gen**: DALL-E image generation
- **sherpa-onnx-tts**: Local text-to-speech
- **model-usage**: Track API usage/costs

### Data/Notes (7 skills)

- **notion**: Notion workspace integration
- **obsidian**: Obsidian vault management
- **apple-notes**: Apple Notes automation
- **bear-notes**: Bear notes app
- **trello**: Trello board management
- **session-logs**: Session history analysis
- **model-usage**: Per-model cost tracking

### Entertainment/Utilities (9 skills)

- **weather**: Weather forecasts
- **spotify-player**: Spotify control
- **sonoscli**: Sonos speaker control
- **eightctl**: 8sleep bed control
- **gifgrep**: GIF search
- **gog**: GOG game library
- **food-order**: Food delivery automation
- **songsee**: Song recognition
- **peekaboo**: Camera preview

### Local/Places (4 skills)

- **local-places**: Location-based search
- **things-mac**: Things task manager
- **goplaces**: Place recommendations
- **healthcheck**: System health monitoring

### Authentication (2 skills)

- **1password**: 1Password vault access
- **apple-reminders**: Apple Reminders automation

### Meta Skills (2 skills)

- **skill-creator**: Create new skills
- **coding-agent**: Run other AI agents

---

## Plugin Discovery & Loading

### Discovery Phase

```
Search locations (priority order):
1. Config-specified paths (plugins.load.paths)
2. Workspace-local (.openclaw/extensions)
3. Global (~/.config/openclaw/extensions)
4. Bundled (pkg/bundled/plugins)

For each directory:
├─ Scan for .ts/.js files (direct plugin)
├─ Scan for package.json with openclaw.extensions field
└─ Scan for index.ts/js as entry point

Extract plugin ID from:
├─ Package name (preferred, unscoped)
├─ Directory name
└─ File name

Load openclaw.plugin.json manifest
```

### Loader Phase

```typescript
// /src/plugins/loader.ts
function loadOpenClawPlugins(options) {
  // 1. Discover candidates
  const candidates = discoverOpenClawPlugins();

  // 2. Load manifests
  const manifests = loadPluginManifestRegistry();

  // 3. Create Jiti loader (dynamic import with aliases)
  const jiti = createJiti();

  // 4. For each candidate:
  for (const plugin of candidates) {
    // Check enable state
    if (!isPluginEnabled(plugin.id)) continue;

    // Validate config
    validatePluginConfig(plugin.config);

    // Dynamic import
    const module = jiti(plugin.source);

    // Extract register/activate function
    const registerFn = resolvePluginModuleExport(module);

    // Create API
    const api = createOpenClawPluginApi({ id: plugin.id, ... });

    // Call plugin's register function
    await registerFn(api);

    // Track registrations
    registry.tools.push(...api.tools);
    registry.hooks.push(...api.hooks);
    registry.channels.push(...api.channels);
  }

  // 5. Initialize hook system
  initializeGlobalHookRunner();

  // 6. Cache registry
  return registry;
}
```

---

## Plugin SDK API

### OpenClawPluginApi

```typescript
type OpenClawPluginApi = {
  // Metadata
  id: string,
  name: string,
  version?: string,
  source: string,
  config: OpenClawConfig,
  pluginConfig?: Record<string, unknown>,

  // Registration methods
  registerTool(tool, opts?): void,
  registerHook(events, handler, opts): void,
  registerHttpHandler(handler): void,
  registerHttpRoute({path, handler}): void,
  registerChannel(plugin): void,
  registerGatewayMethod(method, handler): void,
  registerCli(registrar, opts): void,
  registerService(service): void,
  registerProvider(provider): void,
  registerCommand(command): void,

  // Utilities
  runtime: PluginRuntime,
  logger: PluginLogger,
  resolvePath(input: string): string,
  on<K extends PluginHookName>(...): void
}
```

### PluginRuntime (100+ Methods)

Organized by domain:

**Config**:
- `loadConfig()`: Get current config
- `writeConfigFile(path, config)`: Update config

**System**:
- `enqueueSystemEvent(event)`: Trigger agent
- `runCommandWithTimeout(cmd, opts)`: Execute command
- `formatNativeDependencyHint(hint)`: Format install instructions

**Media**:
- `loadWebMedia(url)`: Fetch remote media
- `detectMime(buffer)`: Detect MIME type
- `isVoiceCompatibleAudio(buffer)`: Check audio format
- `resizeToJpeg(buffer, width, height)`: Image processing

**Channel** (massive surface area):
- Text: `chunkByNewline`, `chunkMarkdownText`
- Reply: `dispatchReplyWithBufferedBlockDispatcher`
- Routing: `resolveAgentRoute`
- Pairing: `buildPairingReply`
- Media: `fetchRemoteMedia`, `saveMediaBuffer`
- Activity: `record`, `get`
- Session: `resolveStorePath`, `recordSessionMeta`
- Mentions: `buildMentionRegexes`, `matchesMentionPatterns`
- Commands: `resolveCommandAuthorizedFromAuthorizers`

**Channel-specific helpers** (Discord, Slack, Telegram, Signal, etc.):
- `discord.messageActions`, `discord.auditChannelPermissions`
- `slack.probeSlack`, `slack.resolveChannelAllowlist`
- (and many more...)

---

## Hook System

### Available Hooks (14 lifecycle events)

```typescript
type PluginHookName =
  | "before_agent_start"      // Modify system prompt
  | "agent_end"               // Log completion
  | "before_compaction"       // React before compaction
  | "after_compaction"        // React after compaction
  | "message_received"        // React to inbound message
  | "message_sending"         // Modify/block outbound message
  | "message_sent"            // Log after send
  | "before_tool_call"        // Block/modify tool parameters
  | "after_tool_call"         // React to tool result
  | "tool_result_persist"     // Filter results before storage
  | "session_start"           // Initialize session
  | "session_end"             // Finalize session
  | "gateway_start"           // Gateway coming online
  | "gateway_stop"            // Gateway shutdown
```

### Hook Execution

**Void Hooks** (fire-and-forget, parallel):
```typescript
async function runVoidHook(hookName, event, ctx) {
  const handlers = registry.hooks.filter(h => h.event === hookName);
  await Promise.all(handlers.map(h => h.handler(event, ctx)));
}
```

**Modifying Hooks** (sequential, results merged):
```typescript
async function runModifyingHook(hookName, event, ctx, mergeResults) {
  const handlers = registry.hooks
    .filter(h => h.event === hookName)
    .sort((a, b) => b.priority - a.priority);  // Higher priority first

  let result = event;
  for (const handler of handlers) {
    const partialResult = await handler.handler(result, ctx);
    result = mergeResults(result, partialResult);
  }
  return result;
}
```

---

## Tool Registration

### Tool Factory Pattern

```typescript
type OpenClawPluginToolFactory = (
  ctx: OpenClawPluginToolContext
) => AnyAgentTool | AnyAgentTool[] | null;

type OpenClawPluginToolContext = {
  config?: OpenClawConfig,
  workspaceDir?: string,
  agentDir?: string,
  agentId?: string,
  sessionKey?: string,
  messageChannel?: string,
  agentAccountId?: string,
  sandboxed?: boolean
};
```

**Example**:
```typescript
api.registerTool((ctx) => {
  if (!ctx.config?.myTool?.enabled) {
    return null;  // Tool disabled
  }

  return {
    name: "my_command",
    description: "Execute my command",
    inputSchema: {
      type: "object",
      properties: {
        arg: { type: "string" }
      }
    },
    handler: async (params) => {
      // Implementation
      return { result: "success" };
    }
  };
}, { optional: true });  // Requires allowlist
```

### Tool Resolution

```typescript
// Resolved at runtime per context
function resolvePluginTools(params) {
  const registry = loadOpenClawPlugins(config);
  const tools = [];

  for (const entry of registry.tools) {
    // Check enable state
    if (blockedPlugins.has(entry.pluginId)) continue;

    // Call factory with context
    const resolved = entry.factory(pluginToolContext);
    if (!resolved) continue;

    // Check optional tool allowlist
    if (entry.optional && !isOptionalToolAllowed(toolName, allowlist)) {
      continue;
    }

    tools.push(...(Array.isArray(resolved) ? resolved : [resolved]));
  }

  return tools;
}
```

---

## Skill Creation Workflow

### Step 1: Initialize

```bash
cd skills/
python skill-creator/scripts/init_skill.py my-skill \
  --path ~/Code/my-skill \
  --resources scripts,references,assets \
  --examples
```

**Generated**:
```
my-skill/
├── SKILL.md (template with TODOs)
├── scripts/ (if requested)
│   └── example.py
├── references/ (if requested)
│   └── REFERENCE.md
└── assets/ (if requested)
    └── template.txt
```

### Step 2: Write SKILL.md

```markdown
---
name: my-skill
description: |
  Brief explanation of what this skill does.
  Use concrete examples: "Use this to X when you need Y."
---

# My Skill

## When to Use This

- Scenario 1: ...
- Scenario 2: ...

## Quick Start

\`\`\`bash
my-command --flag value
\`\`\`

## Examples

### Example 1
\`\`\`bash
my-command input.txt
\`\`\`

Output:
\`\`\`
Expected output here
\`\`\`
```

### Step 3: Add Resources

**scripts/**: Executable code
```python
#!/usr/bin/env python3
import sys

def main():
    # Implementation
    pass

if __name__ == "__main__":
    main()
```

**references/**: Documentation
```markdown
# API Reference

## Function: foo()

Description...
```

**assets/**: Templates
```json
{
  "template": "value"
}
```

### Step 4: Package

```bash
python skill-creator/scripts/package_skill.py ~/Code/my-skill
```

**Output**: `my-skill.skill` (zip file)

### Step 5: Install

```bash
openclaw skills install my-skill.skill
```

---

## Configuration

### Enable/Disable Plugins

```yaml
plugins:
  enabled: true              # Global toggle
  allow: []                  # Whitelist (empty = allow all)
  deny: ["bad-plugin"]       # Blacklist

  entries:
    my-plugin:
      enabled: true          # Per-plugin override
      config:
        apiKey: "..."        # Plugin-specific config
```

### Memory Slot System

Only ONE memory plugin can be active:

```yaml
plugins:
  slots:
    memory: "my-memory-plugin"  # Exclusive slot
```

### Optional Tools

```yaml
agents:
  defaults:
    allowOptionalTools:
      - "group:plugins"      # Allow all plugin tools
      - "my_specific_tool"   # Allow specific tool
```

---

## Example Implementations

### Example 1: Simple Documentation Skill

```typescript
// skills/github/index.ts
export default {
  id: "github",
  name: "GitHub Skill",
  description: "Interact with GitHub using the `gh` CLI",

  register(api: OpenClawPluginApi) {
    // No tools/hooks needed - SKILL.md provides all guidance
  }
};
```

### Example 2: Skill with Scripts

```typescript
// skills/model-usage/index.ts
export default {
  id: "model-usage",
  name: "Model Usage Tracker",

  register(api) {
    // SKILL.md points to scripts/model_usage.py
    // Agent executes: python {baseDir}/scripts/model_usage.py --args
  }
};
```

### Example 3: Tool Registration

```typescript
export default {
  id: "my-tool-skill",

  register(api) {
    api.registerTool((ctx) => ({
      name: "my_command",
      description: "Execute my command",
      inputSchema: { /* ... */ },
      handler: async (params) => {
        // Implementation
      }
    }));
  }
};
```

### Example 4: Hook Registration

```typescript
export default {
  id: "logger-plugin",

  register(api) {
    // Modern typed hooks
    api.on("message_received", async (event, ctx) => {
      api.logger.info(`Message from ${event.from}: ${event.content}`);
    });

    // Legacy hook system
    api.registerHook("message_sending", async (event, ctx) => {
      // Modify message before send
      return {
        content: event.content.toUpperCase()
      };
    }, { priority: 100 });
  }
};
```

### Example 5: Plugin Commands

```typescript
export default {
  id: "status-plugin",

  register(api) {
    api.registerCommand({
      name: "mystatus",
      description: "Show status",
      acceptsArgs: false,
      handler: async (ctx) => {
        if (!ctx.isAuthorizedSender) {
          return { text: "⚠️ Unauthorized" };
        }
        return { text: "✅ Everything is running" };
      }
    });
  }
};
```

---

## Best Practices

### Context Efficiency

1. **Metadata-first**: Keep descriptions concise (1-2 sentences)
2. **Progressive disclosure**: Split large content into references
3. **Script execution**: Execute code without loading to context
4. **Concrete examples**: Show don't tell

### Skill Design

1. **Single responsibility**: One skill = one purpose
2. **Clear triggers**: Description should mention when to use
3. **Minimal instructions**: Agent is already smart
4. **Concrete commands**: Show exact commands with examples

### Resource Organization

```
SKILL.md:
- When to use (2-3 scenarios)
- Quick start (1-2 commands)
- Common examples (3-5)
- Link to references

references/:
- API documentation
- Complex workflows
- Edge cases

scripts/:
- Deterministic execution
- Data processing
- API calls

assets/:
- Output templates
- Boilerplate code
- Configuration examples
```

---

## Summary

The OpenClaw skills platform provides:

✓ Progressive disclosure (metadata → SKILL.md → resources)
✓ 54+ built-in skills across 10 categories
✓ Plugin SDK with 100+ runtime methods
✓ 14 lifecycle hooks for extensibility
✓ Tool factory pattern for context-aware tools
✓ Enable/disable controls with allowlists
✓ Memory slot system for exclusive plugins
✓ Comprehensive discovery and loading system
✓ CLI for skill installation and management

This architecture enables agents to access vast functionality without context window bloat, while maintaining excellent developer experience for skill authors.
