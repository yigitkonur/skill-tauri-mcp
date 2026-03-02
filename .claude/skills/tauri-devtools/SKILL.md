---
name: tauri-devtools
description: "Activate when a user works on a Tauri v2 app and needs to debug IPC calls, inspect Rust backend logs, view app configuration, trace command latency, or understand runtime behavior that browser DevTools cannot show. Use this skill whenever the user mentions debugging Tauri, slow commands, Rust logs not appearing, IPC inspection, or Tauri app configuration issues. This skill covers CrabNebula DevTools — the Rust-side and IPC observability layer that fills the gap browser DevTools leaves."
---

# CrabNebula DevTools for Tauri v2

## What CrabNebula DevTools Is (and Is Not)

CrabNebula DevTools is a Rust-side observability tool for Tauri v2 applications. It surfaces everything that happens **behind** the webview — Rust backend logs, IPC call payloads and timing, resolved app configuration, and bundled asset inspection. Browser DevTools cannot see any of this because Tauri's IPC layer operates outside the browser's network stack.

| Capability | Browser DevTools | CrabNebula DevTools |
|---|---|---|
| DOM/CSS inspection | ✅ | ❌ |
| JavaScript debugging | ✅ | ❌ |
| HTTP network requests | ✅ | ❌ |
| Rust backend logs | ❌ | ✅ |
| IPC call inspection (invoke) | ❌ | ✅ |
| IPC latency/timing spans | ❌ | ✅ |
| Tauri app configuration | ❌ | ✅ |
| Rust crate-level log filtering | ❌ | ✅ |
| Asset resolution inspection | ❌ | ✅ |

Neither replaces the other. Use browser DevTools for frontend (DOM, CSS, JS, HTTP). Use CrabNebula DevTools for backend (Rust logs, IPC, config, tracing spans).

## Architecture: How It Works

The plugin registers a `tracing` subscriber that intercepts all Rust `tracing` spans and events from your application. This data is streamed via a gRPC server (dynamically allocated port in range 6000–9000) to either:
- The **web UI** at https://devtools.crabnebula.dev (free, any browser)
- The **Premium desktop app** (embedded in-app, requires `tauri-plugin-devtools-app` crate)

```
┌─────────────────────────┐
│   Your Tauri App        │
│                         │
│  Rust Backend           │
│   ├─ tracing spans ─────┼──► tracing subscriber (devtools plugin)
│   ├─ log events ────────┼──► captures & forwards
│   └─ IPC calls ─────────┼──► timing + payload data
│                         │
│  gRPC Server            │
│   └─ port 6000-9000 ────┼──► streams to UI
└─────────────────────────┘
          │
          ▼
┌─────────────────────────┐
│  DevTools UI            │
│  (browser or desktop)   │
│   ├─ Console tab        │
│   ├─ Calls tab          │
│   ├─ Config tab         │
│   └─ Sources tab        │
└─────────────────────────┘
```

**Zero impact on release builds** — the `#[cfg(debug_assertions)]` gate ensures the plugin is completely stripped from production binaries.

## Free vs Premium Tier

| Feature | Free (web UI) | Premium (desktop app) |
|---|---|---|
| Console log viewer | ✅ | ✅ |
| IPC Calls tab | ✅ | ✅ |
| Config viewer | ✅ | ✅ |
| Sources tab | ✅ | ✅ |
| Embedded in-app panel | ❌ | ✅ (Cmd/Ctrl+Shift+M) |
| Offline usage | ❌ (needs browser) | ✅ |
| Auto-connect | ❌ (paste URL) | ✅ |
| Crate: | `tauri-plugin-devtools` | + `tauri-plugin-devtools-app` |

## Installation — Cargo.toml

```toml
[dependencies]
tauri-plugin-devtools = "2.0.0"
```

Install via CLI:
```bash
cargo add tauri-plugin-devtools
```

> **Important:** This crate should only be active in debug builds. The `#[cfg(debug_assertions)]` pattern below ensures it is completely excluded from release builds, preventing binary bloat and potential security exposure.

For Premium (embedded desktop app), also add:
```toml
tauri-plugin-devtools-app = "2.0.0"
```

## Installation — src-tauri/src/lib.rs

### Basic Pattern (recommended)

```rust
#[cfg_attr(mobile, tauri::mobile_entry_point)]
pub fn run() {
    #[cfg(debug_assertions)]
    let devtools = tauri_plugin_devtools::init();

    let mut builder = tauri::Builder::default();

    #[cfg(debug_assertions)]
    {
        builder = builder.plugin(devtools);
    }

    builder
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

### Builder Pattern (custom config / compatibility with tauri-plugin-log)

```rust
pub fn run() {
    tauri::Builder::default()
        .setup(|app| {
            let (tauri_plugin_log, max_level, logger) =
                tauri_plugin_log::Builder::new().split(app.handle())?;

            #[cfg(debug_assertions)]
            {
                let mut devtools_builder = tauri_plugin_devtools::Builder::default();
                devtools_builder.attach_logger(logger);
                app.handle().plugin(devtools_builder.init())?;
            }

            #[cfg(not(debug_assertions))]
            {
                tauri_plugin_log::attach_logger(max_level, logger);
            }

            app.handle().plugin(tauri_plugin_log)?;
            Ok(())
        })
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

### Alternative: Separate debug/release logging

```rust
pub fn run() {
    let mut builder = tauri::Builder::default();

    #[cfg(debug_assertions)]
    {
        let devtools = tauri_plugin_devtools::init();
        builder = builder.plugin(devtools);
    }

    #[cfg(not(debug_assertions))]
    {
        use tauri_plugin_log::{Builder, Target, TargetKind};
        let log_plugin = Builder::default()
            .targets([
                Target::new(TargetKind::Stdout),
                Target::new(TargetKind::LogDir { file_name: None }),
                Target::new(TargetKind::Webview),
            ])
            .build();
        builder = builder.plugin(log_plugin);
    }

    builder
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

## Platform-Specific Setup

**Android:**
The gRPC server runs inside the app on the device/emulator. To connect from your desktop browser:
```bash
adb forward tcp:PORT tcp:PORT
```
Replace `PORT` with the actual port shown in the terminal output when the app starts.

**macOS:** No special setup required for DevTools (no screen recording permission needed — that is only required for the MCP bridge plugin's screenshot tool).

**iOS:** DevTools prints the connection URL after a short delay (~3 seconds). Use the URL to connect from your desktop browser.

## What Each Tab Surfaces (Agent Decision Guide)

### Console Tab
**Data:** All Rust `tracing` events — log messages from `tracing::info!()`, `tracing::error!()`, `tracing::debug!()`, and `log::` macros piped through the tracing subscriber. Filterable by log level (TRACE, DEBUG, INFO, WARN, ERROR) and by crate/module origin.

**When to direct user here:** When the user reports a Rust-side error they can't find, when they need to trace the sequence of events leading to a bug, or when a specific crate is generating excessive log output.

**Bug class caught:** Silent panics, unhandled Result errors logged at ERROR level, missing tracing spans indicating code paths not being reached, and crate-level noise drowning out important messages.

### Calls Tab
**Data:** Every IPC `invoke()` call between the frontend and Rust backend. Shows: command name, arguments (serialized), response, timing/duration as spans, and nesting for commands that trigger sub-commands.

**When to direct user here:** When the user says a Tauri command is slow, when IPC calls fail silently, when the user wants to see what data is being passed to Rust commands, or when debugging the order of IPC calls.

**Bug class caught:** Slow commands (visible via span duration), incorrect argument serialization, commands being called in wrong order, redundant IPC calls, and commands that never return.

### Config Tab
**Data:** The resolved Tauri application configuration — the merged result of `tauri.conf.json`, any environment overrides, and plugin configurations. Shows window settings, security settings, capability permissions, and plugin registrations.

**When to direct user here:** When the user suspects a misconfigured capability, when a plugin isn't being loaded, when window properties are wrong, or when security settings need verification.

**Bug class caught:** Missing permissions causing silent command failures, incorrect window sizes/titles, wrong CSP settings blocking resources, and plugins not being registered.

### Sources Tab
**Data:** Application source files and bundled assets as they exist in the running app. Shows the resolved paths for assets, the actual file contents being served, and resource resolution chains.

**When to direct user here:** When the user reports a bundled image/asset not loading, when they need to verify that a file was correctly included in the build, or when asset path resolution is behaving unexpectedly.

**Bug class caught:** Missing assets, incorrect asset paths, files not being included in the bundle, and path resolution differences between dev and production builds.

## Common Agent Workflows

### 1. "My Tauri command is slow"
→ Open the **Calls tab**. Filter by the command name. Look at the span duration. If the span shows >100ms, the bottleneck is in the Rust handler. If the span is fast but the UI feels slow, the bottleneck is in the frontend (use browser DevTools for that). Check for nested spans — a slow parent span might contain a slow child (e.g., database query, file I/O).

### 2. "I see a Rust error but can't find it"
→ Open the **Console tab**. Set filter to ERROR level. The error will include the source location (file, line number). If the error comes from a dependency crate, filter by that crate name. Look at the timestamps to correlate with user actions.

### 3. "My bundled image isn't loading"
→ Open the **Sources tab**. Navigate to the expected asset path. If the file isn't listed, it wasn't included in the bundle — check `tauri.conf.json` bundle settings. If it IS listed but shows the wrong content, the path resolution is incorrect. Compare the path the frontend is requesting (browser DevTools Network tab) with the path DevTools Sources shows.

### 4. "I don't know if my capability config is right"
→ Open the **Config tab**. Find the `security.capabilities` section. Verify the permissions list includes the required permission identifiers for the plugins being used. Cross-reference with the error messages in the Console tab — a missing capability usually produces a specific permission error.

### 5. "Which crate is spamming my terminal?"
→ Open the **Console tab**. Look at the source/origin column for each log entry. Filter by different crate names until you identify the noisy one. Then either: (a) reduce its log level in the `RUST_LOG` environment variable, or (b) add a tracing filter in code.

## Anti-Patterns Agents Make

### 1. Adding `println!()` to Rust instead of reading the Console tab
DevTools already captures all `tracing` output. Adding `println!()` bypasses the tracing subscriber entirely and won't appear in DevTools. Use `tracing::info!()` or `tracing::debug!()` macros instead, which DevTools captures automatically.

### 2. Guessing IPC latency without the Calls tab spans
Agents often add timing code (`Instant::now()` / `.elapsed()`) to both JS and Rust sides to measure IPC performance. The Calls tab already provides precise timing spans for every IPC call — including serialization overhead — without any code changes.

### 3. Assuming browser DevTools Network tab shows IPC calls
Tauri IPC calls do NOT appear in the browser's Network tab. They use a custom protocol (ipc:// on macOS, https://ipc.localhost on Windows). Only the CrabNebula DevTools Calls tab shows IPC traffic.

### 4. Wrapping plugin registration in release builds (security hole)
Never do this:
```rust
// WRONG — ships debug tooling to users
builder = builder.plugin(tauri_plugin_devtools::init());
```
Always wrap in `#[cfg(debug_assertions)]`:
```rust
// CORRECT
#[cfg(debug_assertions)]
{
    builder = builder.plugin(tauri_plugin_devtools::init());
}
```

### 5. Forgetting `#[cfg(debug_assertions)]` and shipping debug tooling to users
The devtools plugin starts a gRPC server that exposes application internals. In a release build, this would:
- Bloat the binary with the tracing subscriber, gRPC server, and all devtools dependencies
- Open a network port that exposes application data
- Potentially conflict with production logging systems

Always gate with `#[cfg(debug_assertions)]`.
