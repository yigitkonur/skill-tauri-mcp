# MCP Bridge Installation Checklist

A comprehensive 4-phase checklist an agent can run through on ANY Tauri v2 app. Each item is verifiable via file inspection or command output.

---

## Pre-Flight Checks

- [ ] **Verify Tauri v2:** Read `src-tauri/Cargo.toml` — look for `tauri = "2"` or `tauri = "^2"`. If it shows `"1"` or `"^1"`, STOP — v1 is not supported.
- [ ] **Locate entry point:** Find `tauri::Builder::default()` — could be in `src-tauri/src/lib.rs` or `src-tauri/src/main.rs`. This is where plugin registration goes.
- [ ] **Detect monorepo:** Check if `src-tauri/` is at project root or nested (e.g., `packages/desktop/src-tauri/`, `apps/tauri/src-tauri/`). All `src-tauri/` references below are relative to the Tauri project root.
- [ ] **Check for mobile entry point:** Look for `#[cfg_attr(mobile, tauri::mobile_entry_point)]` in `lib.rs` — if present, the app supports mobile builds. Plugin registration goes inside the `run()` function.
- [ ] **Check existing MCP bridge:** Search for `tauri-plugin-mcp-bridge` in `Cargo.toml` and `mcp-bridge` in capabilities files. If both exist, check version and skip to verification.
- [ ] **Verify Node.js:** `node --version` — must be v20 or later for the MCP server.

---

## Phase 1: AI Client Configuration

- [ ] **Run auto-install:** `npx -y install-mcp @hypothesi/tauri-mcp-server --client <client-name>`
  - Valid clients: `claude-code`, `cursor`, `windsurf`, `vscode`, `cline`, `roo-cline`, `claude`, `zed`, `goose`, `warp`, `codex`
- [ ] **Verify config created/updated** at the correct path:
  - Claude Code: `~/.claude/claude_desktop_config.json` or project `.mcp.json`
  - Cursor: `.cursor/mcp.json` in project root
  - VS Code: `.vscode/mcp.json` in project root
  - Windsurf: `~/.windsurf/mcp.json`
  - Claude Desktop (macOS): `~/Library/Application Support/Claude/claude_desktop_config.json`
  - Claude Desktop (Windows): `%APPDATA%\Claude\claude_desktop_config.json`
- [ ] **Idempotency check:** If config already contains `"tauri"` MCP server entry, verify the package is `@hypothesi/tauri-mcp-server` and skip adding.
- [ ] **Fully quit and relaunch** the AI client — window reload is NOT sufficient
- [ ] **Verify tools loaded:** Ask AI client to list tools — expect 20 Tauri tools

---

## Phase 2: Rust Plugin

### 2a. Add Crate Dependency

- [ ] **Navigate to Tauri project root** (the directory containing `src-tauri/`)
- [ ] **Idempotency check:** `grep "tauri-plugin-mcp-bridge" src-tauri/Cargo.toml`
  - If found, check version: `0.9` or higher → skip. Older → update version.
  - If not found → continue.
- [ ] **Add dependency:** `cd src-tauri && cargo add tauri-plugin-mcp-bridge`
- [ ] **Verify:** `grep "tauri-plugin-mcp-bridge" Cargo.toml` → should show `= "0.9"` or similar

**Alternative — Feature flag approach (optional):**
```toml
# In src-tauri/Cargo.toml:
[features]
mcp-debug = ["tauri-plugin-mcp-bridge"]

[dependencies]
tauri-plugin-mcp-bridge = { version = "0.9", optional = true }
```

### 2b. Register Plugin

- [ ] **Idempotency check:** `grep "tauri_plugin_mcp_bridge" src-tauri/src/lib.rs src-tauri/src/main.rs`
  - If found → verify it's inside a `#[cfg(debug_assertions)]` or `#[cfg(feature = "mcp-debug")]` block → skip.
  - If not found → continue.

- [ ] **Determine builder pattern** used in the entry point:

  **Pattern A — Reassigned builder (most common):**
  ```rust
  let mut builder = tauri::Builder::default();
  // ... existing plugins ...
  #[cfg(debug_assertions)]
  {
      builder = builder.plugin(tauri_plugin_mcp_bridge::init());
  }
  builder.run(tauri::generate_context!()).expect("...");
  ```

  **Pattern B — Chained builder (needs refactoring):**
  ```rust
  // BEFORE:
  tauri::Builder::default()
      .plugin(some_plugin::init())
      .run(tauri::generate_context!()).expect("...");

  // AFTER: Break chain to insert conditional
  let mut builder = tauri::Builder::default()
      .plugin(some_plugin::init());
  #[cfg(debug_assertions)]
  {
      builder = builder.plugin(tauri_plugin_mcp_bridge::init());
  }
  builder.run(tauri::generate_context!()).expect("...");
  ```

  **Pattern C — Mobile entry point:**
  ```rust
  #[cfg_attr(mobile, tauri::mobile_entry_point)]
  pub fn run() {
      let mut builder = tauri::Builder::default();
      #[cfg(debug_assertions)]
      {
          builder = builder.plugin(tauri_plugin_mcp_bridge::init());
      }
      builder.run(tauri::generate_context!()).expect("...");
  }
  ```

  **Pattern D — Feature flag (if using the optional dependency approach):**
  ```rust
  #[cfg(feature = "mcp-debug")]
  {
      builder = builder.plugin(tauri_plugin_mcp_bridge::init());
  }
  ```

- [ ] **Verify guard:** The plugin init MUST be inside `#[cfg(debug_assertions)]` or `#[cfg(feature = "...")]` — NEVER unconditional in production code.

### 2c. Custom Configuration (optional)

- [ ] **If needed:** Use Builder pattern for custom bind address or port:
  ```rust
  use tauri_plugin_mcp_bridge::Builder;
  #[cfg(debug_assertions)]
  {
      let mcp = Builder::new()
          .bind_address("127.0.0.1")  // default: "0.0.0.0"
          .base_port(9323)            // default: 9223
          .build();
      builder = builder.plugin(mcp);
  }
  ```
- [ ] **Security note:** Default `0.0.0.0` binds to all interfaces (needed for mobile testing). Use `127.0.0.1` for localhost-only when not testing on mobile devices.

---

## Phase 3: tauri.conf.json

- [ ] **Locate config:** Find `src-tauri/tauri.conf.json`
- [ ] **Idempotency check:** `grep "withGlobalTauri" src-tauri/tauri.conf.json`
  - If found and `true` → skip this phase.
  - If found and `false` → change to `true`.
  - If not found → add it.
- [ ] **Add `withGlobalTauri: true`** inside the `"app"` section (NOT `"build"`, NOT `"tauri"`)
- [ ] **Verify placement:** Parse the JSON and confirm `json.app.withGlobalTauri === true`

**Anti-patterns to avoid:**
- ❌ `"build": { "withGlobalTauri": true }` — wrong section
- ❌ `"tauri": { "withGlobalTauri": true }` — Tauri v1 style, doesn't exist in v2
- ✅ `"app": { "withGlobalTauri": true, ... }` — correct

---

## Phase 4: Capabilities

### 4a. Find Active Capabilities File

- [ ] **Look in `src-tauri/capabilities/`** for JSON files
- [ ] **Common names:** `default.json`, `main.json`, `desktop.json`, `migrated.json`
- [ ] **Identify correct file:** The one containing `"core:default"` in its `"permissions"` array
- [ ] **If no capabilities directory exists:** Create `src-tauri/capabilities/default.json`:
  ```json
  {
    "identifier": "default",
    "description": "Default capabilities",
    "windows": ["main"],
    "permissions": ["core:default", "mcp-bridge:default"]
  }
  ```

### 4b. Add Permission

- [ ] **Idempotency check:** `grep "mcp-bridge" src-tauri/capabilities/*.json`
  - If `"mcp-bridge:default"` found → skip.
  - If `"mcp-bridge:allow-all"` found → replace with `"mcp-bridge:default"` (breaking change v0.7.0).
  - If not found → add `"mcp-bridge:default"` to the `"permissions"` array.
- [ ] **Verify window labels match:** Compare `"windows"` array in capabilities with `"app.windows[].label"` in `tauri.conf.json`. They must match.
  - Default window label is `"main"` but some projects use custom labels.
  - If the app has multiple windows, include all labels that need MCP bridge access.

### 4c. Verify

- [ ] `grep "mcp-bridge:default" src-tauri/capabilities/*.json` → should return exactly one match

---

## Final Verification

### Step 1: Build and Run

- [ ] `cargo tauri dev`
- [ ] **Expected terminal output:**
  ```
  MCP Bridge WebSocket server listening on 0.0.0.0:9223
  ```
  (Port may differ if 9223 is busy — any port in 9223–9322 range is valid)

### Step 2: AI Client Connection

- [ ] Call `driver_session({action: "start"})` — should succeed
- [ ] Call `driver_session({action: "status"})` — should show connected session

### Step 3: Screenshot Test

- [ ] Call `webview_screenshot` — should return a non-blank image
- [ ] **If blank/black (macOS):** Grant Screen Recording permission to terminal app, then fully restart terminal and AI client
- [ ] **If blank (Linux):** Expected — falls back to html2canvas-pro JS capture

### Step 4: DOM Test

- [ ] Call `webview_dom_snapshot({type: "accessibility"})` — should return YAML with elements and ref IDs
- [ ] **If "window.__TAURI__ is undefined":** `withGlobalTauri` not set or in wrong section

### Step 5: IPC Test

- [ ] Call `ipc_get_backend_state` — should return app metadata and registered commands
- [ ] **If permission error:** `mcp-bridge:default` missing from capabilities

---

## Platform-Specific Checks

### macOS
- [ ] If screenshot is blank: Grant Screen Recording permission to the terminal app
  - System Settings → Privacy & Security → Screen Recording → Enable your terminal
  - **Fully quit and relaunch** the terminal after granting permission

### Android
- [ ] Run `adb forward tcp:9223 tcp:9223`
- [ ] Keep `bind_address` as `"0.0.0.0"` in Builder config (required for device access)
- [ ] Use `list_devices` to verify device connectivity
- [ ] Connect with `driver_session({action: "start", host: "<device-ip>", port: 9223})`

### iOS
- [ ] Ensure simulator is booted or device is connected
- [ ] Use `list_devices` to verify device is detected
- [ ] Use `read_logs({source: "ios"})` for iOS-specific log access

### Windows
- [ ] No special setup needed — all tools work out of the box

### Linux
- [ ] Native screenshots not yet implemented (webkit2gtk/glib version conflicts)
- [ ] Screenshots fall back to html2canvas-pro (JS-based, may miss off-viewport elements)

---

## Quick Verification Commands

```bash
# Check all installation artifacts exist:
grep "tauri-plugin-mcp-bridge" src-tauri/Cargo.toml
grep "tauri_plugin_mcp_bridge" src-tauri/src/lib.rs src-tauri/src/main.rs 2>/dev/null
grep "withGlobalTauri" src-tauri/tauri.conf.json
grep "mcp-bridge" src-tauri/capabilities/*.json 2>/dev/null
```

✅ **Full success = all 4 greps return matches AND `webview_screenshot` returns a visible, non-blank image.**
