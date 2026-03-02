# Multi-App & Mobile Development — Deep Dive

> Reference for multi-app orchestration, mobile platform setup, and remote device testing with tauri-plugin-mcp-bridge v0.9.0.

---

## Multi-App Architecture

### Port Allocation

- **Base port**: 9223
- **Port range**: 9223–9322 (100 ports)
- When a Tauri app with the MCP bridge plugin starts, it scans from the base port upward for the first available port
- Each running app gets its own dedicated port and WebSocket server
- The plugin uses `Builder::new().base_port(9223).build()` by default

### Session Manager

- Internally, the MCP server maintains a `Map<port, SessionInfo>` for all connected apps
- Each session tracks: port, bundle ID, connection time, WebSocket state
- **Most-recently-connected app is the default target** for all tools

### Querying Connected Sessions

```json
{"tool": "driver_session", "arguments": {"action": "status"}}
```

Returns an array of all connected sessions:

```json
{
  "sessions": [
    {
      "port": 9223,
      "appIdentifier": "com.example.todo-app",
      "connectedAt": "2024-12-01T10:00:00Z",
      "status": "connected"
    },
    {
      "port": 9224,
      "appIdentifier": "com.example.settings-app",
      "connectedAt": "2024-12-01T10:05:00Z",
      "status": "connected"
    }
  ],
  "defaultApp": "com.example.settings-app"
}
```

### Stopping Sessions

- `driver_session({action: "stop"})` without an identifier: **stops ALL sessions**
- `driver_session({action: "stop", appIdentifier: "com.example.todo-app"})`: stops only that session

---

## Targeting Specific Apps

When multiple Tauri apps are connected, you need to specify which app a tool targets.

### By Port Number

```json
{
  "tool": "webview_screenshot",
  "arguments": {
    "appIdentifier": 9224
  }
}
```

### By Bundle ID

```json
{
  "tool": "webview_screenshot",
  "arguments": {
    "appIdentifier": "com.example.settings-app"
  }
}
```

### Default Behavior (No Identifier)

```json
{
  "tool": "webview_screenshot",
  "arguments": {}
}
```

Targets the **most recently connected** app. This works fine for single-app development.

### Which Tools Accept appIdentifier?

**All tools** accept the `appIdentifier` parameter. Pass it to any tool when you need to target a specific app in a multi-app scenario:

- `webview_screenshot`, `webview_dom_snapshot`, `webview_find_element`
- `webview_interact`, `webview_wait_for`, `webview_execute_js`
- `ipc_execute_command`, `ipc_monitor`, `ipc_get_captured`
- `ipc_emit_event`, `ipc_get_backend_state`
- `read_logs`, etc.

### Multi-App Workflow Example

Testing interaction between two apps:

```
1. driver_session status          → see both apps connected
2. webview_screenshot             → default app (most recent)
   appIdentifier: 9223
3. ipc_execute_command            → trigger action in app A
   appIdentifier: "com.example.app-a"
   command: "send_notification"
4. webview_screenshot             → check app B received it
   appIdentifier: "com.example.app-b"
```

---

## Android Development

### Prerequisites

- Android SDK installed (via Android Studio or standalone)
- `adb` (Android Debug Bridge) available in PATH
- A running Android emulator or connected physical device

### ADB Port Forwarding (Required)

The Tauri app runs on the Android device/emulator, but the MCP server connects from the host machine. You **must** forward the WebSocket port:

```bash
adb forward tcp:9223 tcp:9223
```

This forwards port 9223 from the host to the Android device. If your app uses a different port (auto-allocated), forward that port instead.

**For multiple apps:**
```bash
adb forward tcp:9223 tcp:9223
adb forward tcp:9224 tcp:9224
```

### Bind Address

The app **must** bind to `0.0.0.0` (the default). This is required because the WebSocket connection comes from outside the device's network namespace:

```rust
#[cfg(debug_assertions)]
app.handle().plugin(
    tauri_plugin_mcp_bridge::Builder::new()
        .bind_address("0.0.0.0")  // Required for Android
        .base_port(9223)
        .build()
)?;
```

### ADB Path Detection

The plugin detects `adb` using:
1. `ANDROID_HOME` environment variable → `$ANDROID_HOME/platform-tools/adb`
2. Common paths: `/usr/local/share/android-sdk/platform-tools/adb`, etc.
3. System PATH

### Android-Specific Tools

```json
{"tool": "read_logs", "arguments": {"source": "android", "lines": 50}}
```

Uses `adb logcat` filtered to the app's process. Shows both Java/Kotlin and Rust log output.

```json
{"tool": "list_devices", "arguments": {}}
```

Shows connected Android emulators and physical devices (same as `adb devices`).

### Android Troubleshooting

| Issue | Fix |
|---|---|
| WebSocket connection refused | Run `adb forward tcp:9223 tcp:9223` |
| adb not found | Set `ANDROID_HOME` or install Android SDK |
| Multiple devices, wrong one targeted | Use `adb -s <serial> forward ...` for specific device |
| Emulator unreachable | Ensure emulator is fully booted; check with `adb devices` |

---

## iOS Development

### Prerequisites

- Xcode installed (with iOS simulators)
- iOS simulator running

### Port Forwarding

**Not needed for simulators** — iOS simulators run on the same machine as the MCP server, so localhost connections work directly.

For **physical iOS devices**, you would need network-level routing (iOS doesn't support USB port forwarding like adb).

### iOS-Specific Tools

```json
{"tool": "read_logs", "arguments": {"source": "ios", "lines": 50}}
```

Reads the iOS simulator's log stream, filtered to the app.

```json
{"tool": "list_devices", "arguments": {}}
```

Shows available iOS simulators and their states (booted, shutdown).

### Screenshots on iOS

Uses native `UIView` snapshotting — works reliably on simulators. No special permissions needed.

### iOS Troubleshooting

| Issue | Fix |
|---|---|
| Simulator not detected | Open Xcode → Devices & Simulators → boot a simulator |
| Logs empty | Ensure app is running in the simulator |
| Connection fails | Check simulator is booted; verify port with `lsof -i :9223` |

---

## Remote Device Testing

For testing on devices not on the same machine (e.g., a phone on the same WiFi network, a remote build server).

### Starting a Remote Session

```json
{
  "tool": "driver_session",
  "arguments": {
    "action": "start",
    "host": "192.168.1.100",
    "port": 9223
  }
}
```

### Requirements

1. **App must bind to 0.0.0.0**: The default bind address. If the app binds to 127.0.0.1, remote connections are rejected.

```rust
// This works for remote:
Builder::new().bind_address("0.0.0.0").build()

// This blocks remote:
Builder::new().bind_address("127.0.0.1").build()
```

2. **Firewall must allow the WebSocket port**: Port 9223 (or whichever port the app is using) must be open for incoming TCP connections.

3. **Same network**: Both machines must be reachable via IP. VPN or SSH tunneling works too.

### Verifying Remote Connection

```json
{"tool": "driver_session", "arguments": {"action": "status"}}
```

Should show the remote session as connected. If it fails, check:
- Is the app running on the remote machine?
- Is the firewall blocking the port?
- Is the IP address correct?
- Is the bind address 0.0.0.0?

### SSH Tunnel Alternative

If direct network access is not possible, use an SSH tunnel:

```bash
ssh -L 9223:localhost:9223 user@remote-machine
```

Then connect to `localhost:9223` from the MCP server. The app on the remote machine can bind to `127.0.0.1` in this case since the tunnel terminates locally.

---

## Platform Screenshot Matrix

| Platform | Method | Requirement | Gotcha |
|---|---|---|---|
| **macOS** | WKWebView native snapshot | Screen Recording permission for terminal app | Blank/black image if permission not granted — no error |
| **Windows** | webview2-com capture | None | Works out of the box |
| **Linux** | html2canvas-pro (JS injection) | None | Cannot capture outside viewport; may miss some CSS |
| **Android** | JNI `WebView.draw()` to Canvas → Bitmap | `adb forward` for port access | Captures actual device/emulator screen |
| **iOS** | `UIView` snapshotting | Xcode + running simulator | Simulator only; no physical device support currently |

### Screenshot Tips by Platform

**macOS:**
- Grant Screen Recording to your terminal app, then FULLY RESTART the terminal
- If using multiple terminal apps, grant permission to the one running `cargo tauri dev`

**Linux:**
- Scroll content into view before capturing — html2canvas-pro only captures the viewport
- oklch() CSS colors require v0.8.1+ (earlier versions render them incorrectly)

**Android:**
- Screenshots capture the actual device screen, including system UI if visible
- Emulator screenshots may include the emulator chrome

**Windows:**
- Most reliable platform for screenshots — no special setup needed

---

## Platform-Specific Builder Configuration

### Desktop Development (macOS/Windows/Linux)

```rust
#[cfg(debug_assertions)]
app.handle().plugin(
    tauri_plugin_mcp_bridge::Builder::new()
        .bind_address("127.0.0.1")  // Local only — more secure
        .base_port(9223)
        .build()
)?;
```

### Mobile Development (Android/iOS)

```rust
#[cfg(debug_assertions)]
app.handle().plugin(
    tauri_plugin_mcp_bridge::Builder::new()
        .bind_address("0.0.0.0")  // Required for mobile device access
        .base_port(9223)
        .build()
)?;
```

### Multi-Environment (Desktop + Mobile)

```rust
#[cfg(debug_assertions)]
{
    let bind = if cfg!(any(target_os = "android", target_os = "ios")) {
        "0.0.0.0"
    } else {
        "127.0.0.1"
    };
    app.handle().plugin(
        tauri_plugin_mcp_bridge::Builder::new()
            .bind_address(bind)
            .base_port(9223)
            .build()
    )?;
}
```

This binds to all interfaces on mobile (required) and localhost-only on desktop (more secure).
