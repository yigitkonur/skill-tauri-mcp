---
name: tauri-mcp-bridge
description: "Activate when a user wants an AI agent to directly interact with their running Tauri v2 app — take screenshots, execute JavaScript in the WebView, inspect the DOM, call Rust backend commands via IPC, monitor IPC events in real time, click UI elements, type keyboard input, or automate UI interactions. This skill enables the agent to SEE and DRIVE the app, not just read its source code. Use whenever the user says 'test my app', 'take a screenshot', 'click this button', 'call my Rust command', 'watch IPC events', 'debug my UI', or any variation of wanting the AI to interact with a live running Tauri application."
---

# Tauri MCP Bridge — AI Agent ↔ Tauri App

## Architecture: Two-Part System

The MCP bridge consists of two parts that MUST both be installed:

**Part 1: Node.js MCP Server** (runs on the AI client side)
- npm package: `@hypothesi/tauri-mcp-server` v0.9.0
- Exposes 20 tools + 3 slash commands to the AI client
- Communicates with the Tauri plugin via WebSocket

**Part 2: Rust Plugin** (runs inside the Tauri app)
- Crate: `tauri-plugin-mcp-bridge` v0.9.0
- Starts a WebSocket server on port 9223 (scans up to 9322 if busy)
- Handles commands: IPC monitoring, JS execution, screenshots, window management

```
┌─────────────────────┐         WebSocket          ┌─────────────────────┐
│  AI Client          │         (port 9223)         │  Tauri App          │
│  (Claude/Cursor/    │◄───────────────────────────►│  (cargo tauri dev)  │
│   Windsurf/etc.)    │                             │                     │
│                     │                             │  Rust Plugin:       │
│  MCP Server:        │   ┌─────────────────┐       │  tauri-plugin-      │
│  @hypothesi/        │   │ Commands:       │       │  mcp-bridge         │
│  tauri-mcp-server   │──►│ screenshot      │──────►│                     │
│                     │   │ execute_js      │       │  WebSocket Server   │
│  20 tools           │   │ ipc_monitor     │       │  ├─ dispatch_cmd    │
│  3 slash commands   │   │ find_element    │       │  ├─ IPC monitor     │
│                     │   │ ...             │       │  └─ script registry │
└─────────────────────┘   └─────────────────┘       └─────────────────────┘
```

Neither side works alone. The MCP server without the plugin has no app to connect to. The plugin without the MCP server has no AI client to serve.

## Tool Categories and Decision Guide

### WHEN AGENT NEEDS TO SEE THE APP

| Tool | What It Does | Key Parameters |
|---|---|---|
| `webview_screenshot` | Captures a screenshot of the current WebView state | `maxWidth` (optional, resize), `filePath` (optional, save to disk), `format` (png/jpeg) |
| `webview_dom_snapshot` | Returns accessibility tree or DOM structure as YAML | `type`: "accessibility" (roles, names, states, refs) or "structure" (DOM hierarchy) |
| `webview_get_styles` | Gets computed CSS styles for an element | `selector` (CSS selector), `properties` (array of CSS property names) |

**Use `webview_screenshot` first** to understand the current app state visually, then use `webview_dom_snapshot` for programmatic element discovery.

### WHEN AGENT NEEDS TO INTERACT WITH THE APP

| Tool | What It Does | Key Parameters |
|---|---|---|
| `webview_find_element` | Finds elements matching a CSS selector | `selector`, returns outerHTML + bounding rect |
| `webview_interact` | Clicks, types, or performs gestures on elements | `selector`, `action` (click/type/scroll/hover), `value` |
| `webview_keyboard` | Sends keyboard input to the focused element | `keys` (key sequence), `modifiers` (ctrl/shift/alt/meta) |
| `webview_wait_for` | Waits for an element to appear/disappear/change | `selector`, `condition`, `timeout` |
| `webview_select_element` | Visual element picker — user clicks, agent gets metadata | Returns tag, id, classes, attributes, text, bounding rect, CSS selector, XPath, computed styles, parent chain, element screenshot |
| `webview_execute_js` | Executes arbitrary JavaScript in the WebView | `script` (JS code string), must return a value |
| `webview_get_pointed_element` | Gets element metadata from Alt+Shift+Click | Returns same metadata as webview_select_element |

### WHEN AGENT NEEDS TO CALL RUST BACKEND

| Tool | What It Does | Key Parameters |
|---|---|---|
| `ipc_execute_command` | Invokes a Tauri command as if called from JS | `command` (command name), `args` (JSON object) |
| `ipc_emit_event` | Emits a Tauri event | `event` (event name), `payload` (any JSON) |
| `ipc_get_backend_state` | Gets application metadata and backend state | None — returns app config, version, plugins |

### WHEN AGENT NEEDS TO MONITOR WHAT'S HAPPENING

| Tool | What It Does | Key Parameters |
|---|---|---|
| `ipc_monitor` | Starts/stops IPC call interception | `action`: "start" or "stop" |
| `ipc_get_captured` | Returns captured IPC events since monitoring started | `clear` (optional, clear buffer after read) |
| `read_logs` | Reads application logs | `lines` (number of recent lines), `level` (filter) |

### WHEN AGENT NEEDS TO MANAGE WINDOWS

| Tool | What It Does | Key Parameters |
|---|---|---|
| `manage_window` | Lists, inspects, or resizes windows | `action`: "list", "info", or "resize"; `label` (window label) |

### WHEN AGENT IS SETTING UP

| Tool | What It Does | Key Parameters |
|---|---|---|
| `driver_session` | Starts, stops, or checks the bridge session | `action`: "start", "stop", or "status"; `host`, `port` |
| `get_setup_instructions` | Returns step-by-step setup instructions | None |
| `list_devices` | Lists connected Android/iOS devices | None |

> **CRITICAL:** Always call `driver_session({action: "start"})` before using any other tool. If the session isn't active, all tools will fail.

## Installation Phase 1: AI Client Side

### Automatic (recommended)

```bash
npx -y install-mcp @hypothesi/tauri-mcp-server --client claude-code
```

Replace `claude-code` with your client: `cursor`, `windsurf`, `vscode`, `cline`, `roo-cline`, `claude`, `zed`, `goose`, `warp`, `codex`.

### Manual Configuration

Add to your MCP configuration file:

```json
{
  "mcpServers": {
    "tauri": {
      "command": "npx",
      "args": ["-y", "@hypothesi/tauri-mcp-server"]
    }
  }
}
```

**Config file locations by client:**

| Client | Config File Location |
|---|---|
| Claude Code | `~/.claude/claude_desktop_config.json` or project `.mcp.json` |
| Cursor | `.cursor/mcp.json` in project root |
| VS Code | `.vscode/mcp.json` in project root |
| Windsurf | `~/.windsurf/mcp.json` |
| Claude Desktop | `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS) |

> **CRITICAL:** After adding the MCP server config, you must **fully quit and relaunch** the AI client. A simple window reload is NOT enough — the MCP server process needs to be spawned fresh.

## Installation Phase 2: Rust Plugin

### Add the crate

```bash
cargo add tauri-plugin-mcp-bridge
```

Or manually in `src-tauri/Cargo.toml`:
```toml
[dependencies]
tauri-plugin-mcp-bridge = "0.9"
```

### Register in src-tauri/src/lib.rs (or main.rs)

```rust
fn main() {
    let mut builder = tauri::Builder::default();

    #[cfg(debug_assertions)]
    {
        builder = builder.plugin(tauri_plugin_mcp_bridge::init());
    }

    builder
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

### Builder Pattern (custom configuration)

```rust
use tauri_plugin_mcp_bridge::Builder;

fn main() {
    let mut builder = tauri::Builder::default();

    #[cfg(debug_assertions)]
    {
        // Restrict to localhost only (more secure)
        let mcp_plugin = Builder::new()
            .bind_address("127.0.0.1")
            .build();
        builder = builder.plugin(mcp_plugin);
    }

    builder
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

> **Security:** Default bind address is `0.0.0.0` (all interfaces) to support remote device testing. Use `127.0.0.1` for localhost-only access when not testing on mobile devices.

## Installation Phase 3: tauri.conf.json

Add `withGlobalTauri: true` inside the `"app"` section:

**Before:**
```json
{
  "app": {
    "windows": [
      {
        "label": "main",
        "title": "My App",
        "width": 800,
        "height": 600
      }
    ]
  }
}
```

**After:**
```json
{
  "app": {
    "withGlobalTauri": true,
    "windows": [
      {
        "label": "main",
        "title": "My App",
        "width": 800,
        "height": 600
      }
    ]
  }
}
```

**What it does:** Enables `window.__TAURI__` injection into the WebView, which the MCP bridge scripts use to call Tauri APIs from injected JavaScript.

> **Location anti-patterns:**
> - ❌ NOT in `"build"` section (that's for build commands)
> - ❌ NOT in `"tauri"` section (that's the Tauri v1 location — does not exist in v2)
> - ✅ ONLY in `"app"` section

## Installation Phase 4: Capabilities

Edit `src-tauri/capabilities/default.json`:

```json
{
  "$schema": "../gen/schemas/desktop-schema.json",
  "identifier": "default",
  "description": "Capability for the main window",
  "windows": ["main"],
  "permissions": [
    "core:default",
    "mcp-bridge:default"
  ]
}
```

### Granular Permissions (for security-conscious users)

Instead of `"mcp-bridge:default"`, you can specify individual permissions:

```json
"permissions": [
  "core:default",
  "mcp-bridge:allow-capture-native-screenshot",
  "mcp-bridge:allow-execute-js",
  "mcp-bridge:allow-get-window-info",
  "mcp-bridge:allow-list-windows",
  "mcp-bridge:allow-start-ipc-monitor",
  "mcp-bridge:allow-stop-ipc-monitor",
  "mcp-bridge:allow-get-ipc-events",
  "mcp-bridge:allow-emit-event",
  "mcp-bridge:allow-execute-command",
  "mcp-bridge:allow-get-backend-state",
  "mcp-bridge:allow-report-ipc-event",
  "mcp-bridge:allow-request-script-injection",
  "mcp-bridge:allow-script-result"
]
```

### Multi-Window Scoping

To restrict MCP bridge access to specific windows:

```json
{
  "identifier": "mcp-debug",
  "description": "MCP bridge for dev window only",
  "windows": ["dev-panel"],
  "permissions": ["mcp-bridge:default"]
}
```

> **Breaking change (v0.7.0):** The permission identifier was renamed from `"mcp-bridge:allow-all"` to `"mcp-bridge:default"`. If upgrading from v0.6.x, update your capabilities file.

## Verification Sequence

After completing all 4 installation phases:

### Step 1: Start the Tauri app
```bash
cargo tauri dev
```
Look for the WebSocket startup line in terminal output:
```
MCP Bridge WebSocket server listening on 0.0.0.0:9223
```

### Step 2: Verify tools are loaded in AI client
Ask the AI client: "What Tauri MCP tools do you have available?"
The client should list all 20 tools.

### Step 3: Use /setup slash command
Type `/setup` in the AI client. The setup tool will:
- Check if a Tauri app is running and reachable
- Verify the WebSocket connection
- Report any configuration issues
- Offer to fix common problems

### Step 4: Test with webview_screenshot
Ask the AI: "Take a screenshot of my Tauri app"
- **Success:** Returns a base64-encoded image of the app's current WebView state
- **Failure (blank/black):** macOS Screen Recording permission issue (see below)
- **Failure (connection error):** The app isn't running or the port is blocked

## Platform-Specific Gotchas

### macOS Screen Recording Permission

`webview_screenshot` uses native screenshot APIs on macOS that require Screen Recording permission.

**Which app needs the permission:** The **terminal application** running the AI client (Terminal.app, iTerm2, Warp, VS Code, etc.) — NOT the Tauri app itself.

**How to grant:**
1. Open **System Settings** → **Privacy & Security** → **Screen Recording**
2. Enable your terminal application
3. **Fully quit and relaunch the terminal** — the permission change does NOT take effect until restart
4. Restart the AI client inside the relaunched terminal

**Symptom if missing:** `webview_screenshot` returns a blank or black image with no error. The tool reports success but the image content is empty.

### Android Testing

```bash
adb forward tcp:9223 tcp:9223
```

Keep `bind_address` as `"0.0.0.0"` (required for the device to reach the WebSocket server on the host).

For devices connected via USB or WiFi, the MCP server connects through the forwarded port.

### Windows

No special setup required. Screenshots and all tools work out of the box.

### Linux

Native screenshots are NOT yet implemented due to webkit2gtk/glib version conflicts. Screenshots fall back to JavaScript-based capture (html2canvas-pro), which works but may not capture elements outside the WebView viewport.

## Multi-App Support

The MCP bridge supports connecting to multiple Tauri apps simultaneously.

- **Port range:** 9223–9322 (base port 9223, scans up to 100 ports)
- **Default app:** The most recently connected app is used when no `appIdentifier` is specified
- **Targeting specific app:** Pass `appIdentifier` parameter (port number or bundle ID) to any tool
- **Status:** `driver_session({action: "status"})` returns an array of all connected sessions
- **Stop all:** `driver_session({action: "stop"})` without identifier stops all sessions

For remote devices, configure the host:
```
driver_session({action: "start", host: "192.168.1.100", port: 9223})
```

## Breaking Change Registry

| Version | Change | Migration Required |
|---|---|---|
| v0.7.0 | Permission `"mcp-bridge:allow-all"` → `"mcp-bridge:default"` | Update `capabilities/default.json` |
| v0.8.0 | `tauri_` prefix dropped from all tool names (e.g., `tauri_driver_session` → `driver_session`) | Update any hardcoded tool name references |
| v0.8.1 | `html2canvas` replaced with `html2canvas-pro` | No action — fixes `oklch()` CSS screenshot failures |
| v0.8.2 | Window timing race condition fix (exponential backoff) | No action — automatic retry on connection |
| v0.9.0 | Added `webview_select_element` and `webview_get_pointed_element` | No action — new tools only |

## Security Considerations

1. **`#[cfg(debug_assertions)]` guard — NEVER remove.** The MCP bridge opens a WebSocket server that allows full app control. Shipping this in a release build exposes IPC execution, JS injection, and screenshot capabilities to anyone who can reach the port.

2. **`0.0.0.0` binding exposes to LAN.** Default bind address allows connections from any network interface. Use `Builder::new().bind_address("127.0.0.1").build()` when not testing on mobile devices.

3. **`mcp-bridge:default` in dev capabilities ONLY.** Create a separate capabilities file for development (`dev.json`) or use conditional capabilities. Never ship `mcp-bridge:default` permission in production capabilities.

4. **`ipc_execute_command` can call ANY registered Tauri command.** This tool bypasses all frontend validation and calls Rust commands directly with arbitrary arguments. Only use on trusted development machines.

5. **`webview_execute_js` runs arbitrary JavaScript.** Any code passed to this tool executes in the WebView context with full access to the DOM and `window.__TAURI__`. This is equivalent to an XSS vulnerability if exposed in production.

6. **`ipc_emit_event` can trigger any app event.** Events emitted through this tool are indistinguishable from real app events. In development this enables testing; in production it could trigger unintended state changes.

7. **`webview_screenshot` captures the entire WebView.** If the app displays sensitive user data, screenshots will contain that data. Be mindful when screenshots are sent to AI services.

8. **Port range is predictable (9223–9322).** Any process on the machine (or LAN, if bound to 0.0.0.0) can connect to the WebSocket server. There is no authentication on the WebSocket connection.

## Troubleshooting Matrix

| Symptom | Root Cause | Fix |
|---|---|---|
| "No tools available" in AI client | MCP server not configured | Run `npx -y install-mcp @hypothesi/tauri-mcp-server --client <your-client>` |
| Tools listed but "connection refused" | Tauri app not running | Start app with `cargo tauri dev` |
| "WebSocket connection failed" | Plugin not registered in lib.rs | Add `builder = builder.plugin(tauri_plugin_mcp_bridge::init())` inside `#[cfg(debug_assertions)]` block |
| "window.__TAURI__ is undefined" | `withGlobalTauri` not enabled | Add `"withGlobalTauri": true` in `"app"` section of `tauri.conf.json` |
| Screenshot returns blank/black (macOS) | Missing Screen Recording permission | Grant to terminal app in System Settings → Privacy → Screen Recording, then fully restart terminal |
| Screenshot returns blank (Linux) | Native screenshot not implemented | Expected — falls back to html2canvas-pro (JS-based) |
| "Permission denied" errors from IPC tools | Missing capabilities | Add `"mcp-bridge:default"` to `capabilities/default.json` permissions array |
| "mcp-bridge:allow-all" not recognized | Using outdated permission string | Replace with `"mcp-bridge:default"` (changed in v0.7.0) |
| Tool names with `tauri_` prefix not found | Using outdated tool names | Remove `tauri_` prefix from tool names (changed in v0.8.0) |
| Port 9223 already in use | Another Tauri app or process using the port | The plugin auto-scans ports 9223–9322; or use `Builder::new().base_port(9323).build()` |
| `oklch()` CSS causes screenshot failure | Old html2canvas version | Update to v0.8.1+ which uses html2canvas-pro |
| Window commands fail intermittently | Race condition on app startup | Update to v0.8.2+ which adds exponential backoff retry |
| Android device not reachable | Missing ADB port forward | Run `adb forward tcp:9223 tcp:9223` |
| "Module not found" when MCP server starts | Node.js < 20 | Upgrade to Node.js 20 or later |
| AI client shows stale tools after update | MCP server process cached | Fully quit and relaunch the AI client (not just window reload) |
| Multiple apps — wrong app targeted | Default app behavior | Pass `appIdentifier` parameter to specify which app to target |
