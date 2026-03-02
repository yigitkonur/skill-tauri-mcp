# IPC & Backend Tools — Deep Dive

> Reference for all IPC monitoring, command execution, event emission, and logging tools in tauri-plugin-mcp-bridge v0.9.0.
> All tool names have **no prefix** (since v0.8.0).

---

## ipc_execute_command

Sends a Tauri `invoke` command directly via the WebSocket connection, bypassing the frontend entirely.

### How It Works

1. MCP client sends the command via WebSocket to the plugin's server
2. Plugin receives the JSON command and dispatches it via `dispatch_command()`
3. Dispatched as `invoke_tauri` — calls the Rust backend command handler
4. Result (or error) is returned via WebSocket

### Parameters

| Parameter | Type | Description |
|---|---|---|
| `command` | string | The registered Tauri command name |
| `args` | object | Arguments to pass to the command (any JSON) |

### Return Structure

**Success:**
```json
{
  "success": true,
  "result": { "greeting": "Hello, World!" }
}
```

**Failure:**
```json
{
  "success": false,
  "error": "Command not found: unknown_command"
}
```

### Examples

#### Calling a greet command
```json
{
  "tool": "ipc_execute_command",
  "arguments": {
    "command": "greet",
    "args": { "name": "Claude" }
  }
}
```

#### Calling a file-read command
```json
{
  "tool": "ipc_execute_command",
  "arguments": {
    "command": "read_file",
    "args": { "path": "/tmp/config.json" }
  }
}
```

#### Calling a database query command
```json
{
  "tool": "ipc_execute_command",
  "arguments": {
    "command": "db_query",
    "args": { "sql": "SELECT * FROM users LIMIT 10" }
  }
}
```

### Critical Security Note

**ipc_execute_command is equivalent to full backend access:**
- Calls ANY registered Tauri command with ANY arguments
- No frontend validation — bypasses all JS-side input checks, form validation, rate limiting
- No UI confirmation dialogs — commands execute immediately
- Has the same permissions as the Tauri app's Rust backend
- Can read/write files, query databases, make network calls — whatever the app's commands allow

Use this tool to test backend commands directly, but understand that it skips all frontend safety rails.

---

## ipc_monitor

Dual-layer IPC monitoring system that captures both frontend and backend IPC traffic.

### Architecture

The monitor operates on **two layers simultaneously**:

1. **JS-side interception**: Injects `window.__MCP_START_IPC_MONITOR__` into the webview, which patches `window.__TAURI__.invoke()` to capture all outgoing calls and their responses
2. **Rust-side state tracking**: Monitors IPC at the plugin level for commands routed through the bridge

### Parameters

| Parameter | Type | Description |
|---|---|---|
| `action` | string | `"start"` to begin monitoring, `"stop"` to end monitoring |

### Usage Pattern

**Always start monitoring BEFORE the action you want to observe:**

```
1. ipc_monitor({action: "start"})     ← Start capturing
2. ... user action or webview_interact ...  ← IPC calls happen here
3. ipc_get_captured()                  ← Read what was captured
4. ipc_monitor({action: "stop"})       ← Clean up
```

If you start monitoring after the action, you will miss the IPC calls.

### What Gets Captured

- All `invoke()` calls from JS to Rust
- Command names and their full argument payloads
- Response data or error messages
- Timing information (when the call was made, when it returned)

---

## ipc_get_captured

Returns the array of IPC events captured since `ipc_monitor` was started.

### Parameters

| Parameter | Type | Description |
|---|---|---|
| `filter` | string | Optional. Filter captured events by command name (substring match). |

### Return Structure

```json
[
  {
    "id": 1,
    "type": "invoke",
    "command": "get_todos",
    "args": { "filter": "active" },
    "timestamp": "2024-12-01T10:30:00.123Z",
    "response": {
      "success": true,
      "result": [
        { "id": 1, "text": "Buy groceries", "done": false },
        { "id": 3, "text": "Write docs", "done": false }
      ]
    },
    "duration_ms": 12
  },
  {
    "id": 2,
    "type": "invoke",
    "command": "update_todo",
    "args": { "id": 1, "done": true },
    "timestamp": "2024-12-01T10:30:05.456Z",
    "response": {
      "success": true,
      "result": null
    },
    "duration_ms": 8
  }
]
```

### Filtering

```json
{
  "tool": "ipc_get_captured",
  "arguments": { "filter": "todo" }
}
```

Returns only events where the command name contains "todo".

### Tips

- Call immediately after the action you're observing — events are stored in memory
- Use `filter` to narrow down when many IPC calls are captured
- Check `duration_ms` to identify slow backend operations
- Compare `args` with expected values to debug data flow issues

---

## ipc_emit_event

Emits a **Tauri event** (not a DOM event) into the app's event system.

### Important Distinction

These are **Tauri events** — the ones handled by `listen()` and `emit()` in Tauri's event system. They are NOT browser DOM events (click, keydown, etc.).

### Parameters

| Parameter | Type | Description |
|---|---|---|
| `eventName` | string | The Tauri event name to emit |
| `payload` | any | JSON payload attached to the event |

### How It Works

- Events emitted this way are **indistinguishable from real app events**
- All listeners registered with `listen(eventName, ...)` will fire
- Both frontend (JS) and backend (Rust) listeners will receive the event

### Examples

#### Simulating a data update event
```json
{
  "tool": "ipc_emit_event",
  "arguments": {
    "eventName": "data-updated",
    "payload": { "table": "users", "action": "insert", "id": 42 }
  }
}
```

#### Triggering a navigation event
```json
{
  "tool": "ipc_emit_event",
  "arguments": {
    "eventName": "navigate",
    "payload": { "path": "/settings" }
  }
}
```

### Use Cases

- Test event handlers without clicking through UI flows
- Simulate backend-originated events (e.g., "new data available")
- Trigger state transitions that are normally async or external
- Reproduce event-driven bugs with specific payloads

### Caution

Since events are indistinguishable from real ones, they can cause real state changes. Be careful emitting events that trigger destructive operations (delete, reset, etc.).

---

## ipc_get_backend_state

Returns metadata about the connected Tauri application.

### Return Structure

```json
{
  "tauriVersion": "2.1.0",
  "appVersion": "1.0.0",
  "appName": "my-tauri-app",
  "environment": "development",
  "plugins": [
    "mcp-bridge",
    "sql",
    "store",
    "fs"
  ],
  "windows": ["main"],
  "platform": {
    "os": "macos",
    "arch": "aarch64"
  }
}
```

### When to Use

- **First call after session start**: Verify connection is working and understand the app
- **Debugging**: Check which plugins are registered, what platform is running
- **Multi-app scenarios**: Confirm you're connected to the right app

### Tips

- If `plugins` doesn't include `"mcp-bridge"`, the plugin is not properly registered
- Use `appVersion` to correlate with expected behavior
- `environment` confirms debug vs release build

---

## read_logs

Reads log output from various sources depending on platform.

### Source Types

| Source | Platform | What It Reads | How |
|---|---|---|---|
| `console` | All | WebView `console.log`, `console.warn`, `console.error` | JS injection captures console output |
| `android` | Android | Device/emulator logs | Runs `adb logcat` filtered to app |
| `ios` | iOS | Simulator logs | Reads iOS simulator log stream |
| `system` | Desktop | App's stdout/stderr | Captures process output |

### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `source` | string | `"console"` | Log source: `"console"`, `"android"`, `"ios"`, `"system"` |
| `filter` | string | (none) | Regex or keyword filter applied to log lines |
| `since` | string | (none) | ISO 8601 timestamp — only show logs after this time |
| `lines` | number | `50` | Number of log lines to return |

### Examples

#### Get recent console errors
```json
{
  "tool": "read_logs",
  "arguments": {
    "source": "console",
    "filter": "error",
    "lines": 20
  }
}
```

#### Get Android logs since a timestamp
```json
{
  "tool": "read_logs",
  "arguments": {
    "source": "android",
    "since": "2024-12-01T10:30:00Z",
    "lines": 100
  }
}
```

### Tips

- Use `console` source for JS-level debugging (most common)
- Use `system` source for Rust-level panics and println! output
- Set `filter` to narrow down noisy logs
- Increase `lines` if you need more history (default 50 may miss earlier entries)

---

## Complete IPC Debugging Workflow

When debugging how a feature works under the hood:

### Step 1: Understand the App

```json
{"tool": "ipc_get_backend_state", "arguments": {}}
```

Review: app name, version, registered plugins, platform. Confirms connection is live.

### Step 2: Start Monitoring

```json
{"tool": "ipc_monitor", "arguments": {"action": "start"}}
```

Begin capturing all IPC traffic. Do this BEFORE the action.

### Step 3: Trigger the Action

Either instruct the user to perform an action, or trigger it programmatically:

```json
{"tool": "webview_interact", "arguments": {"selector": "#save-button", "action": "click"}}
```

### Step 4: Capture the IPC Calls

```json
{"tool": "ipc_get_captured", "arguments": {}}
```

Review: which commands were called, what arguments were passed, what responses came back, how long each took.

### Step 5: Analyze Results

Look for:
- **Unexpected commands**: is the frontend calling something you didn't expect?
- **Missing commands**: is a call not being made that should be?
- **Error responses**: which commands are failing and why?
- **Slow calls**: which commands have high `duration_ms`?
- **Argument mismatches**: is the frontend sending wrong data?

### Step 6: Check Logs for Backend Context

```json
{"tool": "read_logs", "arguments": {"source": "console", "filter": "error", "lines": 30}}
```

Correlate IPC errors with console output.

### Step 7: Stop Monitoring

```json
{"tool": "ipc_monitor", "arguments": {"action": "stop"}}
```

Clean up the monitor hooks.

---

## IPC vs DOM: Which Tool for What?

| Goal | Tool | Why |
|---|---|---|
| Call a Rust command | `ipc_execute_command` | Direct backend invocation |
| See what commands the UI calls | `ipc_monitor` + `ipc_get_captured` | Intercepts JS→Rust calls |
| Trigger a Tauri event | `ipc_emit_event` | Tests event listeners |
| Trigger a DOM event (click/type) | `webview_interact` | Simulates user actions |
| Run arbitrary JS | `webview_execute_js` | Direct webview scripting |
| Read backend metadata | `ipc_get_backend_state` | App info + connection check |
| Read logs | `read_logs` | Console, system, mobile logs |
