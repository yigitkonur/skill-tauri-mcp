# MCP Bridge Tool Reference

All 20 tools exposed by `@hypothesi/tauri-mcp-server` v0.9.0. Tool names have NO `tauri_` prefix (removed in v0.8.0).

---

## Setup & Configuration

### get_setup_instructions
Returns step-by-step setup instructions for the MCP bridge plugin.

| Parameter | Type | Required | Description |
|---|---|---|---|
| (none) | — | — | No parameters needed |

**Returns:** String with installation and configuration steps.

**When to use:** First-time setup, or when a user needs help configuring the bridge.

---

### driver_session
Manages the bridge session lifecycle. MUST be called with `action: "start"` before any other tool.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `action` | string | ✅ | `"start"`, `"stop"`, or `"status"` |
| `host` | string | ❌ | WebSocket host (default: `"127.0.0.1"`) |
| `port` | number | ❌ | WebSocket port (default: `9223`) |

**Returns (start):**
```json
{ "status": "connected", "port": 9223, "appIdentifier": "com.example.myapp" }
```

**Returns (status):**
```json
[{ "port": 9223, "appIdentifier": "com.example.myapp", "connected": true }]
```

**When to use:** Always call `start` before any other tool. Call `status` to verify connection. Call `stop` when done.

**Gotchas:** If the Tauri app hasn't started yet, `start` will fail. Wait for `cargo tauri dev` to fully compile and launch.

---

### list_devices
Lists connected Android emulators/devices and iOS simulators.

| Parameter | Type | Required | Description |
|---|---|---|---|
| (none) | — | — | No parameters needed |

**Returns:**
```json
{ "android": [{"id": "emulator-5554", "name": "Pixel_6"}], "ios": [{"id": "UUID", "name": "iPhone 15"}] }
```

**When to use:** Before mobile testing, to discover available devices.

---

## Visual Inspection

### webview_screenshot
Captures a screenshot of the current WebView state.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `maxWidth` | number | ❌ | Maximum width in pixels (auto-resize) |
| `filePath` | string | ❌ | Save screenshot to disk path |
| `format` | string | ❌ | `"png"` (default) or `"jpeg"` |
| `appIdentifier` | string | ❌ | Target specific app (port or bundle ID) |

**Returns:**
```json
{ "type": "image", "data": "base64...", "mimeType": "image/png" }
```

**When to use:** To see the current app state. Use BEFORE interacting with elements.

**Gotchas:**
- macOS: Requires Screen Recording permission for the terminal app
- Linux: Falls back to JS-based capture (html2canvas-pro)
- Blank/black image = missing Screen Recording permission on macOS

---

### webview_dom_snapshot
Returns accessibility tree or DOM structure as YAML.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `type` | string | ❌ | `"accessibility"` (default) or `"structure"` |
| `appIdentifier` | string | ❌ | Target specific app |

**Returns:** YAML-formatted string with element hierarchy, roles, names, states, and ref IDs.

**When to use:** For programmatic element discovery. Ref IDs from snapshots can be used with other webview tools.

---

### webview_get_styles
Gets computed CSS styles for an element.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `selector` | string | ✅ | CSS selector for target element |
| `properties` | string[] | ❌ | Specific CSS properties to return |
| `appIdentifier` | string | ❌ | Target specific app |

**Returns:**
```json
{ "display": "flex", "color": "rgb(0, 0, 0)", "fontSize": "16px" }
```

---

## UI Interaction

### webview_find_element
Finds elements matching a CSS selector.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `selector` | string | ✅ | CSS selector |
| `appIdentifier` | string | ❌ | Target specific app |

**Returns:**
```json
{ "found": true, "count": 1, "elements": [{"outerHTML": "<button>Click me</button>", "boundingRect": {"x": 10, "y": 20, "width": 100, "height": 40}}] }
```

**Gotchas:** outerHTML is truncated to 5000 characters (increased from 200 in v0.3.1).

---

### webview_interact
Performs UI interactions on elements.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `selector` | string | ✅ | CSS selector for target element |
| `action` | string | ✅ | `"click"`, `"type"`, `"scroll"`, `"hover"` |
| `value` | string | ❌ | Text to type (for `"type"` action) |
| `appIdentifier` | string | ❌ | Target specific app |

**Returns:**
```json
{ "success": true, "action": "click", "selector": "#submit-btn" }
```

---

### webview_keyboard
Sends keyboard input to the focused element.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `keys` | string | ✅ | Key sequence (e.g., `"Enter"`, `"a"`, `"Tab"`) |
| `modifiers` | string[] | ❌ | `["ctrl"]`, `["shift"]`, `["alt"]`, `["meta"]` |
| `appIdentifier` | string | ❌ | Target specific app |

---

### webview_wait_for
Waits for an element condition to be met.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `selector` | string | ✅ | CSS selector |
| `condition` | string | ❌ | `"visible"`, `"hidden"`, `"exists"`, `"removed"` |
| `timeout` | number | ❌ | Timeout in milliseconds |
| `appIdentifier` | string | ❌ | Target specific app |

**When to use:** After interactions that trigger async UI updates. Wait for the result before taking the next action.

---

### webview_select_element
Agent-initiated visual element picker. Shows an overlay; user clicks to select.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `appIdentifier` | string | ❌ | Target specific app |

**Returns:** Rich metadata: tag, id, classes, attributes, text content, bounding rect, CSS selector, XPath, computed styles, parent chain, plus element screenshot.

**When to use:** When you need the user to point at a specific element (e.g., "which button do you mean?").

---

### webview_get_pointed_element
Gets element metadata for an element the user previously pointed at via Alt+Shift+Click.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `appIdentifier` | string | ❌ | Target specific app |

**Returns:** Same metadata structure as `webview_select_element`.

---

### webview_execute_js
Executes arbitrary JavaScript in the WebView context.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `script` | string | ✅ | JavaScript code to execute |
| `appIdentifier` | string | ❌ | Target specific app |

**Returns:** The return value of the script (must explicitly return a value).

**Gotchas:** Script must return a value. Wrap in an IIFE if needed: `(function() { ...; return result; })()`

---

## IPC / Backend

### ipc_execute_command
Invokes a Tauri command as if called from the frontend.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `command` | string | ✅ | Tauri command name |
| `args` | object | ❌ | Arguments as JSON object |
| `appIdentifier` | string | ❌ | Target specific app |

**Returns:** The command's return value (serialized JSON).

**When to use:** To test Rust backend commands without clicking through UI.

**Gotchas:** Can call ANY registered command — no frontend validation layer. Use on trusted machines only.

---

### ipc_emit_event
Emits a Tauri event.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `event` | string | ✅ | Event name |
| `payload` | any | ❌ | Event payload (any JSON) |
| `appIdentifier` | string | ❌ | Target specific app |

---

### ipc_get_backend_state
Gets application metadata and backend state.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `appIdentifier` | string | ❌ | Target specific app |

**Returns:** App config, version, registered plugins, and capabilities.

---

### ipc_monitor
Starts or stops IPC call interception.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `action` | string | ✅ | `"start"` or `"stop"` |
| `appIdentifier` | string | ❌ | Target specific app |

**When to use:** Before testing a workflow, start monitoring. After the workflow, use `ipc_get_captured` to see all IPC calls.

---

### ipc_get_captured
Returns captured IPC events since monitoring started.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `clear` | boolean | ❌ | Clear the capture buffer after reading |
| `appIdentifier` | string | ❌ | Target specific app |

**Returns:** Array of captured IPC events with command names, arguments, responses, and timing.

---

### read_logs
Reads application logs.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `lines` | number | ❌ | Number of recent lines to return |
| `level` | string | ❌ | Filter by log level |
| `appIdentifier` | string | ❌ | Target specific app |

---

## Window Management

### manage_window
Lists, inspects, or resizes windows.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `action` | string | ✅ | `"list"`, `"info"`, or `"resize"` |
| `label` | string | ❌ | Window label (required for "info" and "resize") |
| `width` | number | ❌ | New width (for "resize") |
| `height` | number | ❌ | New height (for "resize") |
| `appIdentifier` | string | ❌ | Target specific app |

**Returns (list):**
```json
[{"label": "main", "title": "My App", "focused": true}]
```

**Returns (info):**
```json
{"label": "main", "title": "My App", "width": 800, "height": 600, "x": 100, "y": 100, "focused": true, "visible": true}
```
