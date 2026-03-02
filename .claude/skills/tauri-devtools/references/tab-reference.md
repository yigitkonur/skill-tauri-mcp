# CrabNebula DevTools — Tab Reference

## Console Tab

### What It Shows
All Rust `tracing` events captured by the DevTools subscriber. This includes:
- `tracing::trace!()`, `debug!()`, `info!()`, `warn!()`, `error!()` macro outputs
- `log::` macro outputs (if piped through devtools via `attach_logger`)
- Span enter/exit events with timing

### Columns
| Column | Description |
|---|---|
| Timestamp | When the event was emitted (relative to app start) |
| Level | TRACE, DEBUG, INFO, WARN, or ERROR |
| Target | The Rust module path (e.g., `my_app::commands::file_ops`) |
| Message | The formatted log message |
| Fields | Structured key-value pairs from `tracing` span/event fields |

### Filtering
- **By level:** Click level badges to toggle visibility (TRACE/DEBUG/INFO/WARN/ERROR)
- **By target/crate:** Type a crate or module name in the filter box to show only events from that source
- **By text:** Free-text search across message content

### Jump-to-Source
Click on a target path to copy the module path. In the Premium desktop app, this can link directly to the source file.

### Export
Console events can be copied as text. The web UI supports selecting and copying ranges.

---

## Calls Tab

### What It Shows
Every Tauri IPC `invoke()` call between the frontend JavaScript and the Rust backend. Each call is represented as a tracing span with full lifecycle data.

### Columns
| Column | Description |
|---|---|
| Command | The Tauri command name being invoked |
| Arguments | Serialized arguments passed from JS to Rust |
| Response | The return value (or error) from the Rust handler |
| Duration | Wall-clock time from invoke start to response |
| Status | Success (✓) or Error (✗) |
| Nesting | Indentation showing parent-child span relationships |

### Filtering
- **By command name:** Filter to see only specific commands
- **By duration:** Sort by duration to find slow commands
- **By status:** Show only errors or only successes

### Interpreting Timing
- **< 1ms:** Fast — no concern
- **1-10ms:** Normal for simple commands
- **10-100ms:** Worth investigating — check for I/O or blocking operations
- **> 100ms:** Slow — likely involves file I/O, network calls, or heavy computation
- **> 1000ms:** Critical — user will perceive delay; needs optimization

### Span Nesting
A command handler that invokes other internal operations will show nested spans. This is the waterfall view — use it to identify which sub-operation is the bottleneck within a slow command.

---

## Config Tab

### What It Shows
The resolved Tauri application configuration — the final merged result of:
1. `tauri.conf.json` (base configuration)
2. Platform-specific overrides
3. Environment variable overrides
4. Plugin configurations

### Sections
| Section | What It Contains |
|---|---|
| App | Window definitions, `withGlobalTauri`, macOS private API settings |
| Build | Dev server URL, dist directory, build commands |
| Security | CSP settings, capabilities, asset protocol, freeze prototype |
| Plugins | Registered plugin configurations |
| Bundle | App identifier, icons, targets, resources |

### When to Use
- Verify that capabilities/permissions are correctly configured
- Check that window properties (size, title, resizable) are as expected
- Confirm security settings (CSP, asset protocol scope)
- Verify plugin registration

---

## Sources Tab

### What It Shows
The application's source files and bundled assets as resolved by the Tauri runtime. This includes:
- Frontend files (HTML, CSS, JS) as served by the asset protocol
- Static assets (images, fonts, data files) in the bundle
- Resource resolution paths

### Features
- Browse the file tree of bundled assets
- View file contents directly
- Verify that expected files are included in the build
- Check resolved paths for assets referenced in code

### When to Use
- An asset (image, font, JSON file) isn't loading in the app
- Need to verify the build output includes expected files
- Path resolution differs between `cargo tauri dev` and `cargo tauri build`
- CSP is blocking an asset and you need to verify its protocol/path
