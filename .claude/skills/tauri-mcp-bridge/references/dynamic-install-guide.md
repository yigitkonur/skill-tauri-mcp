# Dynamic Installation Detection Engine

This guide teaches the agent HOW TO ADAPT MCP bridge installation to any Tauri v2 app — not just a specific project.

---

## Phase A — Preflight Detection

### Check 1: Tauri Version Gate

Search for `tauri = ` in `src-tauri/Cargo.toml`:

```bash
grep 'tauri = ' src-tauri/Cargo.toml
```

- If version contains `"^1"` or `"= 1"` or `"1."`: **STOP** — Tauri v1 is not supported by mcp-bridge. Inform the user.
- If version contains `"^2"` or `"= 2"` or `"2."`: **Proceed.**

### Check 2: Entry Point (lib.rs vs main.rs)

Search for `tauri::Builder::default()`:

```bash
grep -r "tauri::Builder::default()" src-tauri/src/
```

- The file containing `Builder::default()` is where all plugin registration goes.
- Common locations: `src-tauri/src/lib.rs` (Tauri v2 default) or `src-tauri/src/main.rs` (older projects).
- If found in `lib.rs`, the entry function is typically `pub fn run()` or `#[cfg_attr(mobile, tauri::mobile_entry_point)] pub fn run()`.

### Check 3: Existing Plugin Pattern

Search for `#[cfg(debug_assertions)]` in the entry point file:

```bash
grep "cfg(debug_assertions)" src-tauri/src/lib.rs
```

- If exists: **Match the existing style exactly.** If other plugins use the multi-line block form:
  ```rust
  #[cfg(debug_assertions)]
  {
      builder = builder.plugin(some_plugin::init());
  }
  ```
  Then add mcp-bridge in the same block.
- If NOT exists: Create a fresh block with the mcp-bridge registration.

### Check 4: Capabilities File Location

Search for `"identifier"` in `src-tauri/capabilities/`:

```bash
grep '"identifier"' src-tauri/capabilities/*.json
```

- Confirm which file is the active capability file (usually `default.json`).
- The active file contains `"core:default"` in its permissions.
- Check if the `"windows"` array label matches the actual window labels in `tauri.conf.json`.

### Check 5: tauri.conf.json Structure

Read `src-tauri/tauri.conf.json`:

- Confirm `"app": {}` section exists at the top level.
- Check if `"withGlobalTauri"` is already set (idempotency guard — don't add it twice).
- Note the actual window label(s) in `"app.windows[].label"` — may not be `"main"`.

---

## Phase B — Adaptation Rules

### Rule 1: Cargo.toml Version Pins

**Always** use:
```bash
cargo add tauri-plugin-mcp-bridge
```
This pulls the latest compatible version. Never hardcode a specific version unless the user has a constraint.

After running `cargo add`, verify with:
```
grep "tauri-plugin-mcp-bridge" src-tauri/Cargo.toml
```

### Rule 2: Multiple Capabilities Files

If multiple `.json` files exist in `src-tauri/capabilities/`:

1. Find the one containing `"core:default"` — that is the active permissions file.
2. Add `"mcp-bridge:default"` to THAT file's `"permissions"` array.
3. Do NOT create a new capabilities file unless specifically requested.

### Rule 3: Custom Window Labels

If `tauri.conf.json` → `app.windows` shows a label other than `"main"`:

1. Read the actual label(s): `"settings-panel"`, `"editor"`, etc.
2. In the capabilities file, replace `"windows": ["main"]` with the actual label(s).
3. If the capabilities file uses `"windows": ["*"]`, leave it — it already matches all windows.

### Rule 4: Monorepo / Workspace Layouts

If `src-tauri/` is not at the standard location relative to the project root:

1. Search for `tauri.conf.json` to find the actual Tauri project path:
   ```
   find . -name "tauri.conf.json" -not -path "*/node_modules/*"
   ```
2. All subsequent file edits use the discovered path instead of assuming `src-tauri/`.
3. Common monorepo patterns:
   - `apps/desktop/src-tauri/`
   - `packages/app/src-tauri/`
   - `client/src-tauri/`

### Rule 5: Mobile App Projects

If `tauri.conf.json` has `"android"` or `"ios"` sections, or if `src-tauri-android/` exists:

1. Keep `bind_address` as `"0.0.0.0"` (required for remote device connections).
2. Add note for the user: Run `adb forward tcp:9223 tcp:9223` for Android testing.
3. Do NOT change to `"127.0.0.1"` — mobile devices can't connect to localhost on the host machine.

### Rule 6: Existing Plugins Check (Idempotency)

Before adding `tauri-plugin-mcp-bridge`:

```
grep "tauri-plugin-mcp-bridge" src-tauri/Cargo.toml
```

If already present: **Skip the Cargo.toml edit.** Report as "already installed."

Similarly, check if `"mcp-bridge:default"` is already in capabilities:

```
grep "mcp-bridge" src-tauri/capabilities/*.json
```

If present: **Skip the capabilities edit.** Report as "already configured."

---

## Phase C — Verification After Each Edit

### After Cargo.toml Edit
```bash
grep "tauri-plugin-mcp-bridge" src-tauri/Cargo.toml
```
✅ Must show the crate in `[dependencies]`.

### After lib.rs / main.rs Edit
```bash
grep "tauri_plugin_mcp_bridge::init()" src-tauri/src/lib.rs
# or
grep "tauri_plugin_mcp_bridge::init()" src-tauri/src/main.rs
```
✅ Must show the init() call.

```bash
grep "cfg(debug_assertions)" src-tauri/src/lib.rs
```
✅ Must show the debug gate wrapping the plugin registration.

### After tauri.conf.json Edit
```bash
grep "withGlobalTauri" src-tauri/tauri.conf.json
```
✅ Must show `"withGlobalTauri": true` in the `"app"` section context.

### After Capabilities Edit
```bash
grep "mcp-bridge:default" src-tauri/capabilities/*.json
```
✅ Must show the permission in the active capabilities file.

### Final Verification
```bash
cargo tauri dev
```
✅ Terminal must show: `MCP Bridge WebSocket server listening on <address>:<port>`

Then in AI client:
1. `driver_session({action: "start"})` → must connect
2. `webview_screenshot` → must return a non-blank image

**Full success = non-blank screenshot returned.**
