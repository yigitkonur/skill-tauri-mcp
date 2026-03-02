# Security Model & Troubleshooting — Deep Dive

> Reference for security considerations, threat model, and comprehensive troubleshooting for tauri-plugin-mcp-bridge v0.9.0.

---

## Security Model

### The Debug-Only Contract

The **single most important security boundary** in the MCP bridge plugin is:

```rust
#[cfg(debug_assertions)]
```

This compile-time gate ensures:

- **Debug builds** (`cargo tauri dev`, `cargo build`): WebSocket server starts, all 20 MCP tools are available, full app control is exposed
- **Release builds** (`cargo tauri build`, `cargo build --release`): Plugin code is **completely stripped** from the binary. Not disabled — removed entirely. The WebSocket server does not exist, the tools do not exist, the attack surface is zero.

### Why This Matters

```rust
// ✅ CORRECT — debug-only
#[cfg(debug_assertions)]
app.handle().plugin(tauri_plugin_mcp_bridge::Builder::new().build())?;

// ❌ NEVER DO THIS — feature flag instead of debug gate
#[cfg(feature = "mcp")]
app.handle().plugin(tauri_plugin_mcp_bridge::Builder::new().build())?;
// Feature flags can accidentally be enabled in release builds!

// ❌ NEVER DO THIS — unconditional registration
app.handle().plugin(tauri_plugin_mcp_bridge::Builder::new().build())?;
// This ships the full MCP bridge to production!
```

**Rules:**
1. ALWAYS wrap plugin registration in `#[cfg(debug_assertions)]`
2. NEVER use feature flags as a substitute — they can be enabled in release profiles
3. NEVER accept PRs that remove or weaken the debug guard
4. Verify release builds don't contain the plugin: search the binary for "mcp-bridge" strings

---

### Network Exposure

#### Bind Address Implications

| Bind Address | Who Can Connect | Use Case |
|---|---|---|
| `0.0.0.0` (default) | Any device on the network | Mobile development, remote testing |
| `127.0.0.1` | Only local processes | Desktop-only development (more secure) |

#### Threat Surface

- **No authentication**: The WebSocket server accepts ANY connection without credentials, tokens, or handshakes. Any process that can reach the port is fully trusted.
- **Predictable ports**: Port range 9223–9322 is well-known. An attacker or malware scanning local ports will find the server.
- **Full app control**: Once connected, the client can execute commands, read the DOM, take screenshots, inject JS, emit events — everything.
- **LAN exposure with 0.0.0.0**: Any device on the same network (coworker, shared WiFi, etc.) can connect.

#### Real Scenarios to Consider

1. **Malicious browser extension**: A compromised extension could scan localhost ports and find the MCP server
2. **Shared WiFi**: On a café or conference network, other devices can reach your MCP server if bound to 0.0.0.0
3. **Other local processes**: Any application on your machine can connect to the WebSocket — no isolation
4. **Port scanning**: Automated scanners will find the server on known port ranges

---

### Command Execution Risks

Each tool category has specific security implications:

#### ipc_execute_command
- Calls **ANY** registered Tauri command with **ANY** arguments
- Bypasses all frontend validation (form checks, rate limiting, input sanitization)
- Has the same privileges as the Rust backend
- Can read/write files, query databases, make network requests — whatever the app allows

#### webview_execute_js
- Runs **arbitrary JavaScript** in the webview context
- Has access to `window.__TAURI__` (full Tauri API)
- Can read DOM content, cookies, localStorage, sessionStorage
- Can make fetch requests with the app's credentials/cookies
- Can modify the page, inject content, exfiltrate data

#### ipc_emit_event
- Triggers **any Tauri event** — events are indistinguishable from real app events
- Can cause state transitions, data mutations, navigation
- Could trigger destructive operations (delete, reset, logout) if such event handlers exist

#### webview_interact
- Simulates user actions: click, type, scroll, hover, clear, select, focus
- Can click destructive buttons (delete, submit, pay)
- Can type into any input field (passwords, payment info)
- No confirmation — actions execute immediately

#### webview_screenshot
- Captures **everything visible** in the webview
- May include PII: names, emails, addresses, financial data
- Screenshot data is sent to the AI service for analysis
- Saved files persist on disk

---

### Mitigation Rules

1. **ALWAYS** wrap plugin registration in `#[cfg(debug_assertions)]` — this is non-negotiable
2. **Use `127.0.0.1`** when not testing on mobile devices — reduces network exposure
3. **Keep `mcp-bridge:default`** in development capabilities only — never in production capability files
4. **Never ship** a `capabilities/*.json` file containing mcp-bridge permissions to production
5. **Don't run untrusted AI clients** against apps with sensitive data (the AI client has full access)
6. **Be aware** screenshots may contain user data — they are sent to the AI provider
7. **ipc_execute_command** bypasses all frontend validation — test with care
8. **Consider granular permissions** instead of `mcp-bridge:default` for shared development environments where multiple developers use the same app

---

## Comprehensive Troubleshooting Guide

### Quick Diagnostic Steps

Before diving into specific issues, always check:

```
1. Is the Tauri app running? (cargo tauri dev)
2. Is the MCP server configured in the AI client?
3. Can you reach the WebSocket port? (curl or browser to ws://localhost:9223)
4. Are there errors in the terminal running cargo tauri dev?
```

### Issue Reference Table

| # | Symptom | Root Cause | Diagnostic | Fix |
|---|---|---|---|---|
| 1 | "No tools available" in AI client | MCP server not configured | Check AI client's MCP config file (e.g., `~/.claude/config.json`, VS Code settings) | Run `npx -y install-mcp @hypothesi/tauri-mcp-server --client <name>` to auto-configure |
| 2 | Tools listed but "connection refused" | Tauri app not running or not reachable | Check terminal for `cargo tauri dev` output; verify port with `lsof -i :9223` | Start the app with `cargo tauri dev` |
| 3 | "WebSocket connection failed" on port 9223 | MCP bridge plugin not registered in the app | Check `src-tauri/src/lib.rs` for plugin init call | Add plugin inside `#[cfg(debug_assertions)]` block |
| 4 | "window.__TAURI__ is undefined" | `withGlobalTauri` not enabled | Check `src-tauri/tauri.conf.json` → `app` section | Add `"withGlobalTauri": true` under `"app"` |
| 5 | Screenshot blank/black (macOS) | Screen Recording permission not granted | System Settings → Privacy & Security → Screen Recording | Enable for your terminal app → **fully restart** the terminal application |
| 6 | Screenshot shows partial/broken content (Linux) | html2canvas-pro JS fallback limitations | Expected behavior — native screenshot not available on Linux | Scroll content into view; accept minor CSS rendering differences |
| 7 | "Permission denied" on IPC command calls | Missing capabilities in app config | Check `src-tauri/capabilities/*.json` for mcp-bridge permissions | Add `"mcp-bridge:default"` to the appropriate capability file |
| 8 | "mcp-bridge:allow-all" not recognized | Outdated permission identifier string | Breaking change in v0.7.0 — permission was renamed | Replace `"mcp-bridge:allow-all"` with `"mcp-bridge:default"` |
| 9 | Tool names with `tauri_` prefix fail | Using pre-v0.8.0 tool names | Breaking change in v0.8.0 — prefix was removed | Remove the `tauri_` prefix from all tool names (e.g., `tauri_webview_screenshot` → `webview_screenshot`) |
| 10 | Port 9223 already in use | Another Tauri app or process using the port | Check with `lsof -i :9223` or `netstat -an \| grep 9223` | The plugin auto-scans the next 100 ports. Or set a custom base: `Builder::new().base_port(9300)` |
| 11 | oklch() CSS colors render incorrectly in screenshots | Old html2canvas-pro version (pre-v0.8.1) | Only affects Linux (JS fallback screenshots) | Update to tauri-plugin-mcp-bridge v0.8.1 or later |
| 12 | Commands fail intermittently on app startup | Window not ready when first command arrives | Race condition between WebSocket connection and webview initialization (fixed in v0.8.2) | Update to v0.8.2+ which adds retry backoff for commands during startup |
| 13 | Android device unreachable | Missing `adb forward` for WebSocket port | Run `adb devices` to verify device is connected | Run `adb forward tcp:9223 tcp:9223` to forward the port |
| 14 | "Module not found" when starting MCP server | Node.js version too old | Run `node --version` — requires 20+ | Upgrade to Node.js 20 or later |
| 15 | Stale/outdated tools after npm update | AI client caching old MCP server process | The AI client may hold the old process in memory | Fully quit the AI client application and relaunch it |
| 16 | Wrong app targeted in multi-app setup | Default targets most-recently-connected app | Multiple Tauri apps are running; commands go to the wrong one | Pass `appIdentifier` (port number or bundle ID) explicitly to every tool call |
| 17 | `webview_interact` click does nothing | Element not visible, not in viewport, or obscured | Use `webview_find_element` to check bounding rect; element may be hidden or behind overlay | Scroll element into view; use `webview_wait_for` with `state: "visible"` before interacting |
| 18 | JS execution returns `undefined` | Script doesn't explicitly return a value | The last expression is not auto-returned like in browser console | Wrap in IIFE: `(function(){ ...; return result; })()` |
| 19 | IPC monitor shows no captured events | Monitor not started before the action | `ipc_get_captured` returns empty because monitoring was not active | Call `ipc_monitor({action: "start"})` BEFORE the action you want to observe |
| 20 | `dom_snapshot` shows empty or minimal tree | Page content hasn't finished rendering | DOM snapshot was taken before async content loaded | Use `webview_wait_for` to wait for a key element, then take the snapshot |
| 21 | `webview_wait_for` times out | Element never appears or selector is wrong | Element may have a different selector than expected, or the UI flow didn't trigger | Verify selector with `webview_find_element` first; increase timeout; check if prerequisite action succeeded |
| 22 | `webview_select_element` times out | User didn't click within the timeout period | Default timeout is 60 seconds; user may not have noticed the overlay | Increase `timeout` parameter; ensure user knows to click an element in the overlay |
| 23 | `webview_get_pointed_element` returns nothing | User hasn't Alt+Shift+Clicked an element | This tool reads the LAST pointed element — requires prior user action | Instruct user to Alt+Shift+Click an element in the app first, then call the tool |
| 24 | Multiple tools fail with "session not found" | WebSocket connection dropped | App may have crashed, been restarted, or network interrupted | Check if app is still running; call `driver_session({action: "status"})` to verify; restart session if needed |

---

### Diagnostic Commands

Quick commands to diagnose common issues:

#### Check if the WebSocket port is listening
```bash
lsof -i :9223
# or
netstat -an | grep 9223
```

#### Check if the Tauri app is running
```bash
ps aux | grep "tauri"
```

#### Check ADB devices (Android)
```bash
adb devices
```

#### Check ADB port forwarding
```bash
adb forward --list
```

#### Verify MCP server is installed
```bash
npx @hypothesi/tauri-mcp-server --version
```

#### Test WebSocket connection manually
```bash
# Using websocat (install with: cargo install websocat)
echo '{"type":"ping"}' | websocat ws://localhost:9223
```

---

### Version-Specific Breaking Changes

Keep these in mind when upgrading:

| Version | Change | Migration |
|---|---|---|
| **v0.7.0** | Permission renamed | `"mcp-bridge:allow-all"` → `"mcp-bridge:default"` |
| **v0.8.0** | Tool prefix removed | `tauri_webview_screenshot` → `webview_screenshot` (all 20 tools) |
| **v0.8.1** | oklch() CSS fix | Update npm + Cargo dependencies |
| **v0.8.2** | Startup race condition fix | Update npm + Cargo dependencies |
| **v0.9.0** | Current stable | Latest — no migration needed |

---

### When to Restart What

| Changed | Restart |
|---|---|
| `tauri.conf.json` | `cargo tauri dev` (full restart) |
| `lib.rs` (plugin setup) | `cargo tauri dev` (full restart) |
| `capabilities/*.json` | `cargo tauri dev` (full restart) |
| MCP server config | AI client (full quit + relaunch) |
| npm package updated | AI client (full quit + relaunch) |
| Cargo dependency updated | `cargo tauri dev` (full restart) |
| Screen Recording permission (macOS) | Terminal app (full quit + relaunch) |

---

### Escalation Path

If standard troubleshooting doesn't resolve the issue:

1. **Check versions**: `cargo tree | grep mcp-bridge` and `npm list @hypothesi/tauri-mcp-server`
2. **Enable verbose logging**: Set `RUST_LOG=debug` before running `cargo tauri dev`
3. **Check GitHub Issues**: Search the plugin repository for similar issues
4. **Collect diagnostics**: Terminal output from `cargo tauri dev`, AI client logs, MCP server logs
5. **Minimal reproduction**: Create a fresh Tauri app with only the MCP bridge plugin to isolate the issue
