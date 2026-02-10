# OpenClaw Daemon/Service Management

## Overview

OpenClaw implements **platform-specific daemon/service management** to enable always-on operation across macOS (launchd), Linux (systemd), and Windows (Task Scheduler).

**Location**: `/src/daemon/`

---

## Platform Support

| Platform | Mechanism | Service Type | Location |
|----------|-----------|--------------|----------|
| **macOS** | launchd | LaunchAgent | `~/.Library/LaunchAgents/` |
| **Linux** | systemd | User Service | `~/.config/systemd/user/` |
| **Windows** | Task Scheduler | Scheduled Task | Task Scheduler registry |

---

## macOS (launchd)

**File**: `/src/daemon/launchd.ts` (465 lines)

### Installation Flow

```
1. Create log directory: ~/.openclaw/logs/
2. Clean up legacy agents:
   - Find old agents (launchctl list)
   - Bootout domain registration
   - Unload plist files
   - Move to Trash
3. Build plist XML:
   - Program arguments
   - Working directory
   - Environment variables
   - Stdout/stderr paths
4. Write plist to ~/.Library/LaunchAgents/{label}.plist
5. Bootstrap service:
   - launchctl bootout (remove existing)
   - launchctl unload (legacy cleanup)
   - launchctl enable (clear disabled state)
   - launchctl bootstrap (load new version)
   - launchctl kickstart -k (start immediately)
```

### Plist Structure

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" ...>
<plist version="1.0">
  <dict>
    <key>Label</key>
    <string>ai.openclaw.gateway</string>

    <key>Comment</key>
    <string>OpenClaw Gateway Service</string>

    <key>RunAtLoad</key>
    <true/>  <!-- Start on login -->

    <key>KeepAlive</key>
    <true/>  <!-- Restart if crashes -->

    <key>ProgramArguments</key>
    <array>
      <string>/usr/local/bin/node</string>
      <string>/usr/local/bin/openclaw</string>
      <string>gateway</string>
      <string>--port</string>
      <string>18789</string>
    </array>

    <key>WorkingDirectory</key>
    <string>/Users/user/.openclaw</string>

    <key>StandardOutPath</key>
    <string>/Users/user/.openclaw/logs/gateway.log</string>

    <key>StandardErrorPath</key>
    <string>/Users/user/.openclaw/logs/gateway.err.log</string>

    <key>EnvironmentVariables</key>
    <dict>
      <key>PATH</key>
      <string>/usr/local/bin:/usr/bin:/bin</string>
    </dict>
  </dict>
</plist>
```

### Key Properties

- **RunAtLoad**: Start automatically on login
- **KeepAlive**: Restart if process dies
- **StandardOutPath/StandardErrorPath**: Separate log files

### Service Operations

**Start**:
```bash
launchctl kickstart -k gui/$(id -u)/ai.openclaw.gateway
```

**Stop**:
```bash
launchctl bootout gui/$(id -u)/ai.openclaw.gateway
```

**Status**:
```bash
launchctl print gui/$(id -u)/ai.openclaw.gateway
```

Parses:
- `state`: running/stopped/unknown
- `pid`: Process ID
- `last exit status`: Exit code
- `last exit reason`: Error details

### Log Locations

- Stdout: `~/.openclaw/logs/gateway.log`
- Stderr: `~/.openclaw/logs/gateway.err.log`

---

## Linux (systemd)

**File**: `/src/daemon/systemd.ts` (450 lines)

### Installation Flow

```
1. Check systemd availability: systemctl --user status
2. Create unit directory: ~/.config/systemd/user/
3. Build unit file:
   - Service description
   - ExecStart command
   - Restart policy
   - Environment variables
4. Write unit file: ~/.config/systemd/user/{name}.service
5. Reload daemon: systemctl --user daemon-reload
6. Enable service: systemctl --user enable {name}.service
7. Start service: systemctl --user restart {name}.service
```

### Unit File Structure

```ini
[Unit]
Description=OpenClaw Gateway Service
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/bin/node /usr/local/bin/openclaw gateway --port 18789
Restart=always
RestartSec=5
KillMode=process
WorkingDirectory=/home/user/.openclaw
Environment="PATH=/usr/local/bin:/usr/bin:/bin"
Environment="NODE_ENV=production"

[Install]
WantedBy=default.target
```

### Key Properties

- **Restart=always**: Restart on failure
- **RestartSec=5**: 5-second delay between restarts
- **KillMode=process**: Only kill main process (important for podman)
- **After=network-online.target**: Start after network is available
- **WantedBy=default.target**: Auto-start with user session

### Service Operations

**Start**:
```bash
systemctl --user restart openclaw-gateway.service
```

**Stop**:
```bash
systemctl --user stop openclaw-gateway.service
```

**Status**:
```bash
systemctl --user show openclaw-gateway.service \
  --no-page \
  --property ActiveState,SubState,MainPID,ExecMainStatus,ExecMainCode
```

Parses:
- `ActiveState`: active/inactive/failed
- `SubState`: Detailed state
- `MainPID`: Process ID
- `ExecMainStatus`: Exit status
- `ExecMainCode`: Reason code

### User Linger

**File**: `/src/daemon/systemd-linger.ts` (74 lines)

**Purpose**: Enable user services to run even when user is not logged in

**Check status**:
```bash
loginctl show-user $USER --property=Linger
```

**Enable**:
```bash
sudo loginctl enable-linger $USER
```

**Without linger**:
- Service stops when user logs out
- Only runs during user session

**With linger**:
- Service persists across logouts
- Starts at boot (before login)
- Survives reboots

### Log Locations

- Journal: `journalctl --user --unit openclaw-gateway.service`
- Storage: `~/.local/share/systemd/journal/` or `/run/log/journal/`

---

## Windows (Task Scheduler)

**File**: `/src/daemon/schtasks.ts` (408 lines)

### Installation Flow

```
1. Create script directory: {state-dir}
2. Build batch script (.cmd):
   - Description comment
   - Change directory (cd /d)
   - Environment variables (set)
   - Command line
3. Write script: {state-dir}/gateway.cmd
4. Register scheduled task:
   - schtasks /Create /F
   - /SC ONLOGON (trigger)
   - /RL LIMITED (privileges)
   - /TN {task-name}
   - /TR {script-path}
5. Start task: schtasks /Run /TN {task-name}
```

### Batch Script Structure

```batch
@echo off
rem OpenClaw Gateway Service
cd /d C:\Users\user\.openclaw
set PATH=C:\Program Files\nodejs;%PATH%
set NODE_ENV=production
node "C:\Program Files\nodejs\openclaw" gateway --port 18789
```

### Task Registration Command

```bash
schtasks /Create /F \
  /SC ONLOGON \
  /RL LIMITED \
  /TN "OpenClaw Gateway" \
  /TR "C:\Users\user\.openclaw\gateway.cmd" \
  /RU "DOMAIN\user" \
  /NP /IT
```

**Flags**:
- `/F`: Force overwrite
- `/SC ONLOGON`: Run at login
- `/RL LIMITED`: Limited privileges
- `/TN`: Task name
- `/TR`: Task run (script path)
- `/RU`: Run as user
- `/NP`: No password
- `/IT`: Interactive token

### Service Operations

**Start**:
```bash
schtasks /Run /TN "OpenClaw Gateway"
```

**Stop**:
```bash
schtasks /End /TN "OpenClaw Gateway"
```

**Status**:
```bash
schtasks /Query /TN "OpenClaw Gateway" /V /FO LIST
```

Parses:
- `Status`: running/stopped/unknown
- `Last Run Time`: Timestamp
- `Last Run Result`: Result code

### Log Locations

- Task script: `{state-dir}/gateway.cmd`
- Windows Event Log: Application and Services Logs → Microsoft → Windows → TaskScheduler
- Task History: Via Task Scheduler GUI

---

## Auto-Start / Boot Behavior

### macOS (launchd)

**Mechanism**: `RunAtLoad=true` in plist

- Starts when user logs in
- Persistent across reboots for that user
- Controlled by launchd daemon
- Configuration survives reboots (plist stored permanently)

### Linux (systemd)

**Mechanism**: `WantedBy=default.target` + **User Linger**

**Without linger**:
- Service stops when user logs out
- Only runs during active session

**With linger** (recommended):
```bash
sudo loginctl enable-linger $USER
```
- Service runs even when user is not logged in
- Starts at boot (before login)
- Survives reboots
- Persists across logout/login cycles

### Windows (Task Scheduler)

**Mechanism**: `/SC ONLOGON` trigger

- Runs when user logs in
- Task runs with `LIMITED` privileges by default
- No persistence across reboots for other users
- Stored in Task Scheduler database
- Survives reboots for the same user account

---

## Profile Support

All platforms support multiple profiles with naming:

**Default profile**:
- macOS: `ai.openclaw.gateway`
- Linux: `openclaw-gateway`
- Windows: `OpenClaw Gateway`

**Custom profile** (e.g., `dev`):
- macOS: `ai.openclaw.dev`
- Linux: `openclaw-gateway-dev`
- Windows: `OpenClaw Gateway (dev)`

**Environment variable overrides**:
- `OPENCLAW_LAUNCHD_LABEL`
- `OPENCLAW_SYSTEMD_UNIT`
- `OPENCLAW_WINDOWS_TASK_NAME`

---

## Configuration Paths

### State Directory

```typescript
function resolveGatewayStateDir(env):
  // OPENCLAW_STATE_DIR override
  if (env.OPENCLAW_STATE_DIR) return env.OPENCLAW_STATE_DIR;

  // Default: ~/.openclaw{-profile}
  const home = env.HOME || env.USERPROFILE;
  const profile = env.OPENCLAW_PROFILE;
  return profile
    ? `${home}/.openclaw-${profile}`
    : `${home}/.openclaw`;
```

### Log Directory

**macOS**:
```typescript
const logDir = `${stateDir}/logs`;
const stdoutPath = `${logDir}/gateway.log`;
const stderrPath = `${logDir}/gateway.err.log`;
```

**Linux**:
- Logs to systemd journal (no explicit files)

**Windows**:
- Script in `{stateDir}/gateway.cmd`
- Event log for task events

---

## Environment Variables

### macOS (EnvironmentVariables dict)

```xml
<key>EnvironmentVariables</key>
<dict>
  <key>PATH</key>
  <string>/usr/local/bin:/usr/bin:/bin</string>
  <key>NODE_ENV</key>
  <string>production</string>
</dict>
```

### Linux (Environment= lines)

```ini
Environment="PATH=/usr/local/bin:/usr/bin:/bin"
Environment="NODE_ENV=production"
```

### Windows (set statements)

```batch
set PATH=C:\Program Files\nodejs;%PATH%
set NODE_ENV=production
```

---

## Legacy Cleanup

### macOS

```typescript
async function findLegacyLaunchAgents() {
  // Find old agents from previous versions
  // Pattern: ai.openclaw.* excluding current label
}

async function uninstallLegacyLaunchAgents() {
  // For each legacy agent:
  // 1. launchctl bootout
  // 2. launchctl unload
  // 3. Move plist to ~/.Trash
}
```

### Linux

```typescript
const LEGACY_NAMES = [
  "openclaw-gateway-legacy",
  "openclaw-old"
];

async function uninstallLegacySystemdUnits() {
  // For each legacy unit:
  // 1. systemctl --user disable --now
  // 2. Remove unit file
}
```

---

## CLI Commands

### Install Daemon

```bash
openclaw gateway install
```

Creates platform-specific service configuration and starts it.

### Uninstall Daemon

```bash
openclaw gateway uninstall
```

Stops service and removes configuration.

### Start Service

```bash
openclaw gateway start
```

Starts the gateway service.

### Stop Service

```bash
openclaw gateway stop
```

Stops the gateway service.

### Restart Service

```bash
openclaw gateway restart
```

Restarts the gateway service.

### Status Check

```bash
openclaw gateway status
```

Shows service status:
- Running/stopped
- Process ID
- Last exit status
- Uptime

---

## Platform Comparison

| Feature | macOS (launchd) | Linux (systemd) | Windows (schtasks) |
|---------|-----------------|-----------------|-------------------|
| **Config Type** | XML Plist | INI Unit File | Batch Script |
| **Location** | `~/.Library/LaunchAgents/` | `~/.config/systemd/user/` | `{state-dir}/` |
| **Auto-start** | `RunAtLoad=true` | `WantedBy=default.target` | `/SC ONLOGON` |
| **Restart on Crash** | `KeepAlive=true` | `Restart=always` | No auto-restart |
| **Boot Persistence** | Yes (per user) | Requires linger | Yes (per user) |
| **Logging** | Separate files | systemd journal | Event Log + script |
| **Status Check** | `launchctl print` | `systemctl show` | `schtasks /Query` |
| **Start Command** | `launchctl kickstart` | `systemctl restart` | `schtasks /Run` |
| **Stop Command** | `launchctl bootout` | `systemctl stop` | `schtasks /End` |
| **Privilege Model** | User (GUI domain) | User with linger | Limited by default |

---

## Performance & Reliability

- **Startup time**: 1-3 seconds (depends on Node.js initialization)
- **Restart delay**:
  - macOS: Immediate (KeepAlive)
  - Linux: 5 seconds (RestartSec=5)
  - Windows: Manual restart only
- **Memory footprint**: ~50-100 MB (Node.js + OpenClaw)
- **CPU usage**: <1% idle, varies with activity
- **Crash recovery**:
  - macOS: Automatic (KeepAlive)
  - Linux: Automatic (Restart=always)
  - Windows: Manual restart required

---

## Summary

The daemon/service management system provides:

✓ Platform-native service integration (launchd, systemd, schtasks)
✓ Auto-start on boot/login
✓ Automatic restart on crash (macOS, Linux)
✓ Profile support for multiple instances
✓ Legacy cleanup for smooth upgrades
✓ Comprehensive logging
✓ Status monitoring
✓ CLI commands for lifecycle management

This architecture enables **truly always-on operation** with platform-appropriate reliability guarantees and zero-configuration user experience after initial setup.
