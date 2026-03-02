---
name: tauri-mcp-bridge
description: "Activate when a user wants an AI agent to directly interact with their running Tauri v2 app — take screenshots, execute JavaScript in the WebView, inspect the DOM, call Rust backend commands via IPC, monitor IPC events in real time, click UI elements, type keyboard input, automate UI interactions, read device logs, or pick elements visually. This skill enables the agent to SEE and DRIVE the app, not just read its source code. Use whenever the user says 'test my app', 'take a screenshot', 'click this button', 'call my Rust command', 'watch IPC events', 'debug my UI', 'read logs', 'select that element', or any variation of wanting the AI to interact with a live running Tauri application."
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

## Slash Commands

The MCP server provides 3 slash commands for quick agent interaction:

| Command | Description |
|---|---|
| `/setup` | Runs setup diagnostics — checks if Tauri app is running, verifies WebSocket, reports config issues, offers to fix problems |
| `/fix-webview-errors` | Diagnoses and fixes common WebView errors — missing `withGlobalTauri`, permission failures, script injection issues |
| `/select` | Activates the visual element picker — user clicks an element, agent receives full metadata (tag, id, classes, attributes, bounding rect, CSS selector, XPath, computed styles, parent chain, screenshot) |

## Selector Strategy Parameter

Many webview tools accept a `strategy` parameter that controls how `selector` is interpreted:

| Strategy | Value | Example Selector | Description |
|---|---|---|---|
| CSS (default) | `"css"` | `"#login-btn"`, `".card > h2"` | Standard CSS selectors |
| XPath | `"xpath"` | `"//button[@type='submit']"` | XPath 1.0 expressions |
| Text content | `"text"` | `"Sign In"` | Matches elements containing exact text |
| ARIA label | `"aria-label"` | `"Close dialog"` | Matches `aria-label` attribute value |
| Ref ID | `"ref"` | `"ref-42"` | Uses ref IDs from `webview_dom_snapshot` accessibility output |

**Tools supporting `strategy`:** `webview_dom_snapshot`, `webview_get_styles`, `webview_find_element`, `webview_interact`, `webview_wait_for`

**The `ref` strategy** is especially useful: call `webview_dom_snapshot` with `type: "accessibility"`, get ref IDs from the YAML output, then use `strategy: "ref"` with those IDs to target specific elements without fragile CSS selectors.

## Tool Categories — Complete Schema Reference

### SETUP & CONFIG (3 tools)

#### `get_setup_instructions`
Returns setup instructions for the MCP bridge plugin.

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| *(none)* | — | — | — | No parameters |

**Returns:** Text with installation steps covering Cargo.toml, lib.rs, tauri.conf.json, and capabilities setup.

The returned text covers:
1. Adding the crate dependency
2. Registering the plugin in the Tauri builder
3. Enabling `withGlobalTauri` in tauri.conf.json
4. Adding `mcp-bridge:default` permission to capabilities
5. Running `cargo tauri dev` and verifying the WebSocket server starts

---

#### `driver_session`
Manages bridge session lifecycle. **MUST be called with `action: "start"` before using any other tool.**

| Parameter | Type | Required | Default | Constraints | Description |
|---|---|---|---|---|---|
| `action` | string | ✅ | — | `"start"` \| `"stop"` \| `"status"` | Session lifecycle action |
| `host` | string | ❌ | `"127.0.0.1"` | Valid hostname/IP | WebSocket host to connect to |
| `port` | number | ❌ | `9223` | 1–65535 | WebSocket port |
| `appIdentifier` | string \| number | ❌ | — | Port number or bundle ID | Target a specific running app |

---

#### `list_devices`
Lists connected Android/iOS devices and emulators.

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| *(none)* | — | — | — | No parameters |

---

### UI AUTOMATION & WEBVIEW (11 tools)

#### `webview_screenshot`
Captures a screenshot of the current WebView state.

| Parameter | Type | Required | Default | Constraints | Description |
|---|---|---|---|---|---|
| `maxWidth` | number | ❌ | — | Positive integer | Resize to max width (preserves aspect ratio) |
| `filePath` | string | ❌ | — | Valid file path | Save screenshot to disk |
| `format` | string | ❌ | `"png"` | `"png"` \| `"jpeg"` | Image format |
| `quality` | number | ❌ | — | 1–100 | JPEG quality (only for jpeg format) |
| `windowId` | string | ❌ | — | Window label | Target specific window |
| `appIdentifier` | string \| number | ❌ | — | Port or bundle ID | Target specific app |

**Annotations:** `readOnlyHint: true`, `destructiveHint: false`, `openWorldHint: false`

---

#### `webview_dom_snapshot`
Returns YAML accessibility tree or DOM structure snapshot.

| Parameter | Type | Required | Default | Constraints | Description |
|---|---|---|---|---|---|
| `type` | string | ❌ | `"accessibility"` | `"accessibility"` \| `"structure"` | Snapshot format |
| `selector` | string | ❌ | — | Depends on strategy | Root element to snapshot from |
| `strategy` | string | ❌ | `"css"` | `"css"` \| `"xpath"` \| `"text"` \| `"aria-label"` \| `"ref"` | Selector interpretation |
| `windowId` | string | ❌ | — | Window label | Target specific window |
| `appIdentifier` | string \| number | ❌ | — | Port or bundle ID | Target specific app |

**Accessibility output** includes: roles, names, states, and **ref IDs** usable with `strategy: "ref"` in other tools.

---

#### `webview_get_styles`
Gets computed CSS styles for a specific element.

| Parameter | Type | Required | Default | Constraints | Description |
|---|---|---|---|---|---|
| `selector` | string | ✅ | — | Depends on strategy | Target element |
| `properties` | string[] | ❌ | all | CSS property names | Specific properties to return |
| `strategy` | string | ❌ | `"css"` | `"css"` \| `"xpath"` \| `"text"` \| `"aria-label"` \| `"ref"` | Selector interpretation |
| `windowId` | string | ❌ | — | Window label | Target specific window |
| `appIdentifier` | string \| number | ❌ | — | Port or bundle ID | Target specific app |

---

#### `webview_find_element`
Finds elements matching a selector.

| Parameter | Type | Required | Default | Constraints | Description |
|---|---|---|---|---|---|
| `selector` | string | ✅ | — | Depends on strategy | Elements to find |
| `strategy` | string | ❌ | `"css"` | `"css"` \| `"xpath"` \| `"text"` \| `"aria-label"` \| `"ref"` | Selector interpretation |
| `windowId` | string | ❌ | — | Window label | Target specific window |
| `appIdentifier` | string \| number | ❌ | — | Port or bundle ID | Target specific app |

---

#### `webview_interact`
Performs UI interactions — click, type, scroll, hover, clear, select, focus.

| Parameter | Type | Required | Default | Constraints | Description |
|---|---|---|---|---|---|
| `selector` | string | ✅ | — | Depends on strategy | Target element |
| `action` | string | ✅ | — | `"click"` \| `"type"` \| `"scroll"` \| `"hover"` \| `"clear"` \| `"select"` \| `"focus"` | Interaction type |
| `value` | string | ❌ | — | — | Text for `"type"` action, option value for `"select"` action |
| `scrollDirection` | string | ❌ | — | `"up"` \| `"down"` | Direction for `"scroll"` action |
| `scrollAmount` | number | ❌ | — | Positive integer | Pixels to scroll |
| `strategy` | string | ❌ | `"css"` | `"css"` \| `"xpath"` \| `"text"` \| `"aria-label"` \| `"ref"` | Selector interpretation |
| `windowId` | string | ❌ | — | Window label | Target specific window |
| `appIdentifier` | string \| number | ❌ | — | Port or bundle ID | Target specific app |

**Annotations:** `readOnlyHint: false`, `destructiveHint: false`, `openWorldHint: false`

---

#### `webview_keyboard`
Sends keyboard input — individual key presses with modifiers.

| Parameter | Type | Required | Default | Constraints | Description |
|---|---|---|---|---|---|
| `action` | string | ✅ | — | `"press"` \| `"down"` \| `"up"` \| `"type"` | Key event type |
| `key` | string | ✅ | — | Key name (e.g. `"Enter"`, `"a"`, `"Tab"`, `"Escape"`) | Key to send |
| `modifiers` | string[] | ❌ | `[]` | `"ctrl"` \| `"shift"` \| `"alt"` \| `"meta"` | Modifier keys held during action |
| `windowId` | string | ❌ | — | Window label | Target specific window |
| `appIdentifier` | string \| number | ❌ | — | Port or bundle ID | Target specific app |

---

#### `webview_wait_for`
Waits for an element to reach a specified condition.

| Parameter | Type | Required | Default | Constraints | Description |
|---|---|---|---|---|---|
| `selector` | string | ✅ | — | Depends on strategy | Element to wait for |
| `state` | string | ❌ | `"visible"` | `"visible"` \| `"hidden"` \| `"attached"` \| `"detached"` | Condition to wait for |
| `timeout` | number | ❌ | — | Positive integer (ms) | Maximum wait time in milliseconds |
| `strategy` | string | ❌ | `"css"` | `"css"` \| `"xpath"` \| `"text"` \| `"aria-label"` \| `"ref"` | Selector interpretation |
| `windowId` | string | ❌ | — | Window label | Target specific window |
| `appIdentifier` | string \| number | ❌ | — | Port or bundle ID | Target specific app |

---

#### `webview_execute_js`
Executes arbitrary JavaScript in the WebView context.

| Parameter | Type | Required | Default | Constraints | Description |
|---|---|---|---|---|---|
| `script` | string | ✅ | — | Valid JavaScript | JS code to execute |
| `windowId` | string | ❌ | — | Window label | Target specific window |
| `appIdentifier` | string \| number | ❌ | — | Port or bundle ID | Target specific app |

**Script must return a value.** Wrap in IIFE if needed: `(function() { ...; return result; })()`

---

#### `webview_select_element`
Visual element picker — shows an overlay, user clicks to select an element.

| Parameter | Type | Required | Default | Constraints | Description |
|---|---|---|---|---|---|
| `timeout` | number | ❌ | `60000` | 5000–120000 ms | How long the picker stays active |
| `windowId` | string | ❌ | — | Window label | Target specific window |
| `appIdentifier` | string \| number | ❌ | — | Port or bundle ID | Target specific app |

**Returns:** Rich metadata — tag, id, classes, attributes, text content, bounding rect, CSS selector, XPath, computed styles, parent chain, element screenshot.

---

#### `webview_get_pointed_element`
Gets element metadata from a previous Alt+Shift+Click by the user.

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `windowId` | string | ❌ | — | Target specific window |
| `appIdentifier` | string \| number | ❌ | — | Target specific app |

**Returns:** Same metadata structure as `webview_select_element`.

---

### WINDOW MANAGEMENT (1 tool)

#### `manage_window`
Lists windows, gets window info, or resizes windows.

| Parameter | Type | Required | Default | Constraints | Description |
|---|---|---|---|---|---|
| `action` | string | ✅ | — | `"list"` \| `"info"` \| `"resize"` | Window operation |
| `windowId` | string | ❌ | — | Window label | Target window (required for "info"/"resize") |
| `width` | number | ❌ | — | Positive integer | New width for "resize" |
| `height` | number | ❌ | — | Positive integer | New height for "resize" |
| `logical` | boolean | ❌ | `true` | — | Use logical (true) vs physical (false) pixels |
| `appIdentifier` | string \| number | ❌ | — | Port or bundle ID | Target specific app |

---

### IPC & PLUGIN (5 tools)

#### `ipc_execute_command`
Invokes any registered Tauri command as if called from the frontend.

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `command` | string | ✅ | — | Tauri command name |
| `args` | unknown | ❌ | — | Command arguments as JSON |
| `appIdentifier` | string \| number | ❌ | — | Target specific app |

**Annotations:** `readOnlyHint: false`, `destructiveHint: false`, `openWorldHint: false`

---

#### `ipc_monitor`
Starts or stops IPC call interception.

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `action` | string | ✅ | — | `"start"` \| `"stop"` |
| `appIdentifier` | string \| number | ❌ | — | Target specific app |

---

#### `ipc_get_captured`
Returns captured IPC events since monitoring started.

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `filter` | string | ❌ | — | Filter captured events by command name substring |
| `appIdentifier` | string \| number | ❌ | — | Target specific app |

---

#### `ipc_emit_event`
Emits a Tauri event to the application.

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `eventName` | string | ✅ | — | Event name to emit |
| `payload` | unknown | ❌ | — | Event payload (any JSON-serializable value) |
| `appIdentifier` | string \| number | ❌ | — | Target specific app |

---

#### `ipc_get_backend_state`
Gets application metadata, version, environment, and registered plugins.

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `appIdentifier` | string \| number | ❌ | — | Target specific app |

---

### LOG READING (1 tool)

#### `read_logs`
Reads logs from multiple sources: WebView console, Android logcat, iOS device logs, or system logs.

| Parameter | Type | Required | Default | Constraints | Description |
|---|---|---|---|---|---|
| `source` | string | ✅ | — | `"console"` \| `"android"` \| `"ios"` \| `"system"` | Log source |
| `lines` | number | ❌ | `50` | Positive integer | Number of log lines to return |
| `filter` | string | ❌ | — | Substring match | Filter logs by content |
| `since` | string | ❌ | — | ISO 8601 timestamp | Only return logs after this time |
| `windowId` | string | ❌ | — | Window label | Target specific window |
| `appIdentifier` | string \| number | ❌ | — | Port or bundle ID | Target specific app |

**Source types:**
- `"console"` — WebView `console.log/warn/error` output
- `"android"` — Android logcat (requires connected device/emulator)
- `"ios"` — iOS device/simulator logs
- `"system"` — System-level application logs

---

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
| Claude Desktop | `%APPDATA%\Claude\claude_desktop_config.json` (Windows) |

> **CRITICAL:** After adding the MCP server config, you must **fully quit and relaunch** the AI client. A simple window reload is NOT enough — the MCP server process needs to be spawned fresh.

## Installation Phase 2: Rust Plugin

### Add the crate

```bash
cd src-tauri && cargo add tauri-plugin-mcp-bridge
```

Or manually in `src-tauri/Cargo.toml`:
```toml
[dependencies]
tauri-plugin-mcp-bridge = "0.9"
```

**If the dependency already exists:** Check the version. If it's older than 0.9, update it. If it's 0.9+, skip this step.

### Register in lib.rs or main.rs

**Standard pattern (lib.rs):**

```rust
pub fn run() {
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

**Main.rs pattern** (some projects use main.rs instead):

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

**Tauri v2 mobile entry point** (projects with `src-tauri/src/lib.rs` using `#[cfg_attr(mobile, ...)]`):

```rust
#[cfg_attr(mobile, tauri::mobile_entry_point)]
pub fn run() {
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

**Edge case — builder is chained, not reassigned:**
If the project uses a chained builder pattern:

```rust
// BEFORE (chained):
tauri::Builder::default()
    .plugin(some_other_plugin::init())
    .run(tauri::generate_context!())
    .expect("error");

// AFTER (must break the chain to insert conditional):
let mut builder = tauri::Builder::default()
    .plugin(some_other_plugin::init());

#[cfg(debug_assertions)]
{
    builder = builder.plugin(tauri_plugin_mcp_bridge::init());
}

builder
    .run(tauri::generate_context!())
    .expect("error");
```

### Builder Pattern (custom configuration)

```rust
use tauri_plugin_mcp_bridge::Builder;

#[cfg(debug_assertions)]
{
    let mcp_plugin = Builder::new()
        .bind_address("127.0.0.1")  // default: "0.0.0.0"
        .base_port(9323)            // default: 9223
        .build();
    builder = builder.plugin(mcp_plugin);
}
```

### Feature flag alternative to cfg(debug_assertions)

For more control over when the MCP bridge is active:

```toml
# src-tauri/Cargo.toml
[features]
mcp-debug = ["tauri-plugin-mcp-bridge"]

[dependencies]
tauri-plugin-mcp-bridge = { version = "0.9", optional = true }
```

```rust
// lib.rs
#[cfg(feature = "mcp-debug")]
{
    builder = builder.plugin(tauri_plugin_mcp_bridge::init());
}
```

Then run with: `cargo tauri dev --features mcp-debug`

> **Security:** Default bind address is `0.0.0.0` (all interfaces) to support remote device testing. Use `127.0.0.1` for localhost-only access when not testing on mobile devices.

## Installation Phase 3: tauri.conf.json

Add `withGlobalTauri: true` inside the `"app"` section:

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
> - ✅ ONLY in `"app"` section at the top level

**If `withGlobalTauri` is already `true`:** Skip this step — no action needed.

## Installation Phase 4: Capabilities

Find the active capabilities file in `src-tauri/capabilities/`. It's usually `default.json` but could be named differently (e.g., `main.json`, `desktop.json`). Look for the file containing `"core:default"` in its permissions array.

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

### If the capabilities file doesn't exist

Some projects use TOML capabilities or put permissions in `tauri.conf.json`. Create `src-tauri/capabilities/default.json`:

```json
{
  "identifier": "default",
  "description": "Default capabilities",
  "windows": ["main"],
  "permissions": [
    "core:default",
    "mcp-bridge:default"
  ]
}
```

### Custom window labels

If the app's main window has a label other than `"main"` (check `tauri.conf.json` → `app.windows[].label`), update the `"windows"` array to match:

```json
{
  "windows": ["app-window"],
  "permissions": ["core:default", "mcp-bridge:default"]
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
**Expected terminal output:**
```
   Compiling tauri-plugin-mcp-bridge v0.9.0
   ...
   Finished `dev` profile [unoptimized + debuginfo] target(s)
MCP Bridge WebSocket server listening on 0.0.0.0:9223
```

If you see a different port (e.g., 9224), another process was using 9223. The MCP server will auto-detect the correct port.

### Step 2: Verify tools are loaded in AI client
Ask the AI client: "What Tauri MCP tools do you have available?"

**Expected output:** 20 tools listed: `get_setup_instructions`, `driver_session`, `list_devices`, `webview_screenshot`, `webview_dom_snapshot`, `webview_get_styles`, `webview_find_element`, `webview_interact`, `webview_keyboard`, `webview_wait_for`, `webview_execute_js`, `webview_select_element`, `webview_get_pointed_element`, `manage_window`, `ipc_execute_command`, `ipc_monitor`, `ipc_get_captured`, `ipc_emit_event`, `ipc_get_backend_state`, `read_logs`.

### Step 3: Use /setup slash command
Type `/setup` in the AI client. The setup tool will:
- Check if a Tauri app is running and reachable
- Verify the WebSocket connection
- Report any configuration issues
- Offer to fix common problems

**Expected output on success:**
```
✓ Connected to Tauri app on port 9223
✓ WebSocket connection established
✓ App identifier: com.example.myapp
```

### Step 4: Test with webview_screenshot
Ask the AI: "Take a screenshot of my Tauri app"
- **Success:** Returns a base64-encoded image of the app's current WebView state
- **Failure (blank/black):** macOS Screen Recording permission issue (see Troubleshooting)
- **Failure (connection error):** The app isn't running or the port is blocked

### Step 5: Test with webview_dom_snapshot
Ask the AI: "Show me the accessibility tree of my app"
- **Success:** Returns YAML with elements, roles, names, states, and ref IDs
- **Failure:** Usually `withGlobalTauri` not set, or capabilities missing

## Agent Decision Tree

Use this flowchart to pick the right tool for any task:

```
What does the agent need to do?
│
├─► SEE the app's current state
│   ├─► Visual appearance? ──────────► webview_screenshot
│   ├─► Element structure? ──────────► webview_dom_snapshot (type: "structure")
│   ├─► Accessibility info? ─────────► webview_dom_snapshot (type: "accessibility")
│   ├─► Specific element's styles? ──► webview_get_styles
│   └─► Find a specific element? ────► webview_find_element
│
├─► INTERACT with the app
│   ├─► Click a button/link? ────────► webview_interact (action: "click")
│   ├─► Type into an input? ─────────► webview_interact (action: "type")
│   ├─► Scroll the page? ───────────► webview_interact (action: "scroll")
│   ├─► Hover over element? ─────────► webview_interact (action: "hover")
│   ├─► Clear an input field? ───────► webview_interact (action: "clear")
│   ├─► Select dropdown option? ─────► webview_interact (action: "select")
│   ├─► Focus an element? ──────────► webview_interact (action: "focus")
│   ├─► Press a key combo? ─────────► webview_keyboard
│   ├─► Wait for UI update? ────────► webview_wait_for
│   └─► Run custom JS? ─────────────► webview_execute_js
│
├─► IDENTIFY elements
│   ├─► User points at element? ─────► /select or webview_select_element
│   ├─► User already Alt+Shift+Clicked? ► webview_get_pointed_element
│   └─► Agent needs to find by selector? ► webview_find_element
│
├─► CALL backend / Rust commands
│   ├─► Invoke a Tauri command? ─────► ipc_execute_command
│   ├─► Emit an event? ─────────────► ipc_emit_event
│   └─► Get app metadata? ──────────► ipc_get_backend_state
│
├─► MONITOR what's happening
│   ├─► Start capturing IPC? ────────► ipc_monitor (action: "start")
│   ├─► Read captured IPC? ──────────► ipc_get_captured
│   ├─► Stop capturing? ────────────► ipc_monitor (action: "stop")
│   └─► Read logs? ─────────────────► read_logs
│
├─► MANAGE windows
│   ├─► List all windows? ──────────► manage_window (action: "list")
│   ├─► Get window details? ────────► manage_window (action: "info")
│   └─► Resize a window? ──────────► manage_window (action: "resize")
│
└─► SET UP the connection
    ├─► First time setup? ──────────► get_setup_instructions or /setup
    ├─► Connect to app? ────────────► driver_session (action: "start")
    ├─► Check connection? ──────────► driver_session (action: "status")
    ├─► List mobile devices? ───────► list_devices
    └─► Diagnose WebView issues? ──► /fix-webview-errors
```

## Common Agent Workflows

### 1. Initial App Exploration
```
driver_session({action: "start"})
→ webview_screenshot
→ webview_dom_snapshot({type: "accessibility"})
→ Now the agent knows what the app looks like AND its element structure
```

### 2. Click a Button and Verify Result
```
webview_find_element({selector: "#submit-btn"})
→ webview_interact({selector: "#submit-btn", action: "click"})
→ webview_wait_for({selector: ".success-message", state: "visible"})
→ webview_screenshot (confirm the result visually)
```

### 3. Fill Out a Form
```
webview_interact({selector: "#email", action: "clear"})
→ webview_interact({selector: "#email", action: "type", value: "test@example.com"})
→ webview_interact({selector: "#password", action: "type", value: "secret123"})
→ webview_interact({selector: "button[type='submit']", action: "click"})
→ webview_wait_for({selector: ".dashboard", state: "visible"})
```

### 4. Debug a UI Issue Using Accessibility Tree
```
webview_dom_snapshot({type: "accessibility"})
→ identify ref IDs from YAML output
→ webview_get_styles({selector: "ref-42", strategy: "ref"})
→ webview_find_element({selector: "ref-42", strategy: "ref"})
→ diagnose style/layout issues
```

### 5. Test a Rust Backend Command
```
ipc_get_backend_state
→ (learn available commands/plugins)
→ ipc_execute_command({command: "get_user", args: {id: 42}})
→ verify response matches expectations
```

### 6. Monitor IPC During a Workflow
```
ipc_monitor({action: "start"})
→ webview_interact({selector: "#save-btn", action: "click"})
→ webview_wait_for({selector: ".saved-indicator", state: "visible", timeout: 5000})
→ ipc_get_captured
→ ipc_monitor({action: "stop"})
→ analyze which commands were called and with what args
```

### 7. User Points at Element → Agent Fixes It
```
/select (user clicks an element in the app)
→ agent receives: tag, id, classes, attributes, CSS selector, XPath, computed styles
→ agent reads the source file containing that component
→ agent makes the fix
→ webview_screenshot (verify the fix)
```

### 8. Multi-Window Testing
```
manage_window({action: "list"})
→ manage_window({action: "info", windowId: "settings"})
→ webview_screenshot({windowId: "settings"})
→ webview_interact({selector: "#theme-toggle", action: "click", windowId: "settings"})
→ webview_screenshot({windowId: "main"}) (verify theme changed in main window too)
```

### 9. Mobile Log Debugging
```
list_devices
→ driver_session({action: "start", host: "192.168.1.100", port: 9223})
→ read_logs({source: "android", lines: 100, filter: "ERROR"})
→ diagnose the error
→ read_logs({source: "console", lines: 50}) (check WebView console too)
```

### 10. Responsive Design Testing
```
manage_window({action: "resize", windowId: "main", width: 375, height: 812})
→ webview_screenshot (mobile layout)
→ manage_window({action: "resize", windowId: "main", width: 1920, height: 1080})
→ webview_screenshot (desktop layout)
→ compare layouts, identify responsive issues
```

### 11. Event-Driven Feature Testing
```
ipc_monitor({action: "start"})
→ ipc_emit_event({eventName: "user:logout", payload: {reason: "session_expired"}})
→ webview_wait_for({selector: ".login-screen", state: "visible", timeout: 3000})
→ webview_screenshot
→ ipc_get_captured (verify the app handled the event correctly)
→ ipc_monitor({action: "stop"})
```

### 12. Custom JavaScript Diagnostic
```
webview_execute_js({script: "JSON.stringify(window.__TAURI__)"})
→ (verify Tauri APIs are available)
webview_execute_js({script: "document.querySelectorAll('[data-testid]').length"})
→ (count test IDs for element targeting)
webview_execute_js({script: "(function(){ return {url: location.href, title: document.title, readyState: document.readyState} })()"})
→ (get page state info)
```

## When NOT to Use This Skill

This skill is specifically for **interacting with a running Tauri v2 app via MCP**. Do NOT use it when:

| Scenario | Use Instead |
|---|---|
| Editing Tauri source code (Rust, TS, HTML, CSS) | Standard file editing tools |
| Building the Tauri app (`cargo tauri build`) | Shell commands |
| Reading Tauri config files statically | File reading tools |
| Tauri v1 apps | Not supported — v1 has different plugin architecture |
| Electron, React Native, or Flutter apps | Different MCP bridges exist for those |
| The app is not currently running | Start it first with `cargo tauri dev` |
| Production/release builds | MCP bridge should ONLY be in debug builds |
| Web-only apps (no Tauri) | Browser DevTools MCP or similar |
| Editing `tauri.conf.json` structure | File editing tools (but use this skill's knowledge for correct structure) |
| Managing Tauri CLI (`cargo tauri` subcommands) | Shell commands directly |

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

| # | Symptom | Root Cause | Fix |
|---|---|---|---|
| 1 | "No tools available" in AI client | MCP server not configured | Run `npx -y install-mcp @hypothesi/tauri-mcp-server --client <your-client>` |
| 2 | Tools listed but "connection refused" | Tauri app not running | Start app with `cargo tauri dev` |
| 3 | "WebSocket connection failed" | Plugin not registered in lib.rs | Add `builder = builder.plugin(tauri_plugin_mcp_bridge::init())` inside `#[cfg(debug_assertions)]` block |
| 4 | "window.__TAURI__ is undefined" | `withGlobalTauri` not enabled | Add `"withGlobalTauri": true` in `"app"` section of `tauri.conf.json` |
| 5 | Screenshot returns blank/black (macOS) | Missing Screen Recording permission | Grant to terminal app in System Settings → Privacy → Screen Recording, then fully restart terminal |
| 6 | Screenshot returns blank (Linux) | Native screenshot not implemented | Expected — falls back to html2canvas-pro (JS-based). May not capture off-viewport elements |
| 7 | "Permission denied" errors from IPC tools | Missing capabilities | Add `"mcp-bridge:default"` to `capabilities/default.json` permissions array |
| 8 | "mcp-bridge:allow-all" not recognized | Using outdated permission string (pre-v0.7.0) | Replace with `"mcp-bridge:default"` |
| 9 | Tool names with `tauri_` prefix not found | Using outdated tool names (pre-v0.8.0) | Remove `tauri_` prefix from tool names |
| 10 | Port 9223 already in use | Another Tauri app or process using the port | The plugin auto-scans ports 9223–9322; or use `Builder::new().base_port(9323).build()` |
| 11 | `oklch()` CSS causes screenshot failure | Old html2canvas version (pre-v0.8.1) | Update to v0.8.1+ which uses html2canvas-pro |
| 12 | Window commands fail intermittently | Race condition on app startup (pre-v0.8.2) | Update to v0.8.2+ which adds exponential backoff retry |
| 13 | Android device not reachable | Missing ADB port forward | Run `adb forward tcp:9223 tcp:9223` |
| 14 | "Module not found" when MCP server starts | Node.js < 20 | Upgrade to Node.js 20 or later |
| 15 | AI client shows stale tools after update | MCP server process cached | Fully quit and relaunch the AI client (not just window reload) |
| 16 | Multiple apps — wrong app targeted | Default app behavior uses most recent | Pass `appIdentifier` parameter to specify which app |
| 17 | `webview_select_element` times out | User didn't click within timeout | Increase `timeout` parameter (max: 120000ms) or retry with `/select` |
| 18 | "ref-XX" selector not found | Stale ref IDs from old snapshot | Re-run `webview_dom_snapshot` to get fresh ref IDs, then retry |
| 19 | `ipc_execute_command` returns "command not found" | Command name mismatch or not registered | Use `ipc_get_backend_state` to list registered commands; check for typos |
| 20 | `webview_execute_js` returns undefined | Script didn't return a value | Wrap in IIFE: `(function(){ ...; return result; })()` |
| 21 | Connection drops after app hot-reload | WebSocket reconnection needed | Call `driver_session({action: "start"})` again to reconnect |
| 22 | `read_logs` returns empty for Android/iOS | Device not connected or wrong source | Run `list_devices` to verify; use `source: "console"` for WebView logs |
| 23 | Monorepo — `cargo add` fails | Not in correct directory | `cd` into the `src-tauri/` directory before running `cargo add` |
| 24 | Capabilities file not found | Non-standard project structure | Create `src-tauri/capabilities/default.json` manually with correct schema |
