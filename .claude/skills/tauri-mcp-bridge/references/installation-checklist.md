# MCP Bridge Installation Checklist

A 4-phase checklist an agent can run through on ANY Tauri v2 app. Each item is verifiable via file inspection.

---

## Pre-Flight: Verify Tauri v2

- [ ] **Check Tauri version:** Read `src-tauri/Cargo.toml` — look for `tauri = "2"` or `tauri = "^2"`. If it shows `"1"` or `"^1"`, STOP — v1 is not supported.
- [ ] **Check entry point:** Find `tauri::Builder::default()` — could be in `src-tauri/src/lib.rs` or `src-tauri/src/main.rs`. This is where plugin registration goes.

---

## Phase 1: AI Client Configuration

- [ ] Run: `npx -y install-mcp @hypothesi/tauri-mcp-server --client <client-name>`
- [ ] Verify MCP config file was created/updated at the correct location for your client
- [ ] **Fully quit and relaunch** the AI client (window reload is NOT enough)
- [ ] Ask the AI client to list available tools — should show 20 Tauri tools

**Verification:** Ask the AI: "What Tauri MCP tools do you have?" — expect 20 tools listed.

---

## Phase 2: Rust Plugin

- [ ] Run: `cargo add tauri-plugin-mcp-bridge` (in the `src-tauri/` directory)
- [ ] Verify `Cargo.toml` contains `tauri-plugin-mcp-bridge = "0.9"` (or newer) in `[dependencies]`
- [ ] Add plugin registration to lib.rs/main.rs inside `#[cfg(debug_assertions)]` block:
  ```rust
  #[cfg(debug_assertions)]
  {
      builder = builder.plugin(tauri_plugin_mcp_bridge::init());
  }
  ```
- [ ] Verify the `#[cfg(debug_assertions)]` guard is present — plugin MUST NOT be in release builds

**Verification:** `grep "tauri_plugin_mcp_bridge" src-tauri/src/lib.rs` (or main.rs) — should return the init() line.

---

## Phase 3: tauri.conf.json

- [ ] Open `src-tauri/tauri.conf.json`
- [ ] Find the `"app"` section (top-level key)
- [ ] Add `"withGlobalTauri": true` inside `"app"` (NOT in "build", NOT in "tauri")
- [ ] Verify: `grep "withGlobalTauri" src-tauri/tauri.conf.json` — should show `true` and be inside `"app"` context

**Verification:** Parse the JSON and confirm `json.app.withGlobalTauri === true`.

---

## Phase 4: Capabilities

- [ ] Find the active capabilities file: look in `src-tauri/capabilities/` for the file containing `"core:default"`
- [ ] Add `"mcp-bridge:default"` to the `"permissions"` array
- [ ] **DO NOT** use `"mcp-bridge:allow-all"` (deprecated in v0.7.0)
- [ ] If `"windows"` array exists, verify it contains the correct window label(s)

**Verification:** `grep "mcp-bridge:default" src-tauri/capabilities/*.json` — should return a match.

---

## Final Verification

- [ ] Run `cargo tauri dev`
- [ ] Check terminal for: `MCP Bridge WebSocket server listening on 0.0.0.0:9223` (or similar port)
- [ ] In AI client, call `driver_session({action: "start"})`
- [ ] Call `webview_screenshot` — **should return a non-blank image**

✅ **Full success = `webview_screenshot` returns a visible, non-blank image of the app UI.**

---

## Platform-Specific Checks

### macOS
- [ ] If screenshot is blank: Grant Screen Recording permission to the terminal app
  - System Settings → Privacy & Security → Screen Recording → Enable your terminal
  - **Fully quit and relaunch** the terminal after granting
  
### Android
- [ ] Run `adb forward tcp:9223 tcp:9223`
- [ ] Keep `bind_address` as `"0.0.0.0"` in Builder config

### Windows
- [ ] No special setup needed
