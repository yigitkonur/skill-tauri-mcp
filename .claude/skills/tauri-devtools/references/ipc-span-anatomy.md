# IPC Span Anatomy — CrabNebula DevTools

## What Is an IPC Span?

When a frontend JavaScript call like `invoke('my_command', { key: 'value' })` reaches the Tauri backend, the DevTools plugin wraps the entire execution in a `tracing` span. This span captures the full lifecycle of the IPC call.

## Span Fields

| Field | Type | Description |
|---|---|---|
| `name` | String | The Tauri command name (e.g., `"my_command"`) |
| `arguments` | JSON | Serialized input parameters from the frontend |
| `response` | JSON | Serialized return value (on success) or error (on failure) |
| `start_time` | Timestamp | When the command handler began executing |
| `end_time` | Timestamp | When the command handler returned |
| `duration` | Duration | `end_time - start_time` (wall-clock time) |
| `status` | Enum | `OK` or `ERROR` |
| `parent_span` | Option | Reference to parent span if this was a nested call |
| `child_spans` | List | Any sub-operations that created their own spans |

## Example: Normal Span

```
Command: get_user_profile
Arguments: { "user_id": 42 }
Duration: 3ms
Status: OK
Response: { "name": "Alice", "email": "alice@example.com" }

Timeline:
├─ get_user_profile [3ms] ✓
│  └─ db_query [2ms] ✓
```

**Interpretation:** Fast command with a single database sub-operation. The 1ms overhead (3ms total - 2ms db) is serialization + Tauri IPC overhead. This is healthy.

## Example: Slow Span

```
Command: export_report
Arguments: { "format": "pdf", "pages": 50 }
Duration: 2,340ms
Status: OK
Response: { "path": "/tmp/report.pdf", "size": 1048576 }

Timeline:
├─ export_report [2340ms] ✓
│  ├─ fetch_data [45ms] ✓
│  ├─ render_pages [2100ms] ✓  ← BOTTLENECK
│  │  ├─ render_page [40ms] ✓  (×50 pages)
│  └─ write_file [195ms] ✓
```

**Interpretation:** The `render_pages` sub-span takes 2100ms of the 2340ms total — this is the bottleneck. The agent should investigate the rendering logic, not the data fetching or file writing.

**Agent action:** Look at `render_page` — 40ms × 50 pages = 2000ms. Consider: Can pages be rendered in parallel? Can output be streamed? Is there unnecessary recomputation per page?

## Example: Error Span

```
Command: save_settings
Arguments: { "theme": "dark", "language": "fr" }
Duration: 12ms
Status: ERROR
Error: "PermissionDenied: Cannot write to /etc/app/settings.json"

Timeline:
├─ save_settings [12ms] ✗
│  └─ write_file [10ms] ✗
```

**Interpretation:** The command failed at the file write step. The error message clearly indicates a permission issue — the app is trying to write to a system directory instead of the user's app data directory.

**Agent action:** Change the file path to use `app.path().app_data_dir()` instead of a hardcoded system path.

## Reading Span Duration Context

| Duration | Classification | Typical Cause | Agent Response |
|---|---|---|---|
| < 1ms | Instant | In-memory operations, simple calculations | No action needed |
| 1-10ms | Fast | Simple DB queries, small file reads, serialization | Normal for most commands |
| 10-100ms | Moderate | Complex queries, file operations, image processing | Acceptable for user-initiated actions |
| 100-500ms | Slow | Network calls, large file I/O, heavy computation | User will notice; consider async/background |
| 500ms-2s | Very slow | Report generation, batch operations, external APIs | Must show progress indicator to user |
| > 2s | Critical | Blocking the main thread, unresponsive UI | Immediate investigation required |

## Nested Spans — Waterfall Reading

Nested spans show the call hierarchy within a command handler. Read the waterfall from top to bottom:

1. **Find the slowest top-level span** — this is the overall bottleneck
2. **Look at its children** — which child takes the most time?
3. **Recurse** — drill into the slowest child until you find the leaf operation
4. **The leaf operation is your optimization target** — it's the actual work being done

If a span has many children of similar duration, the bottleneck is the NUMBER of operations, not any single one — consider batching or parallelizing.

## Missing Spans

If you expect to see a span but it doesn't appear:
- The command handler doesn't include `#[tracing::instrument]` — add it
- The `tracing` subscriber wasn't initialized early enough — ensure `tauri_plugin_devtools::init()` is called before any other plugin registrations
- The command is crashing before the span completes — check the Console tab for panics
