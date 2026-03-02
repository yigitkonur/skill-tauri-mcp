# Tauri Observability & MCP Bridge Skills

## When to Activate

These two skills work together to give you full-stack debugging and interaction capability for Tauri v2 applications. Activate them when the user is working with a Tauri desktop or mobile app and needs help beyond what browser devtools can provide.

### tauri-devtools skill

**Activate when you see any of these triggers:**
- user mentions "tauri" + any debugging keyword (logs, errors, crashes, slow, performance, tracing)
- user asks about rust-side behavior in a tauri app
- user sees a frontend error but needs to find the backend cause
- user wants to inspect ipc calls, command timings, or payload sizes
- user mentions "crabnebula devtools" or "tauri devtools"
- user asks about tauri.conf.json, permissions, or capability bundles at runtime
- user wants structured rust logs with span context (not just println output)
- user needs to debug tauri plugin initialization or event listener registration
- user asks "why is this command slow" or "why does this call fail"
- user mentions console tab, calls tab, config tab, or sources tab

**What to do:**
1. Read `tauri-devtools/SKILL.md` for the full workflow
2. Use the tab decision guide to pick the right devtools tab
3. Follow the agent workflows for the specific debugging scenario
4. Reference `tauri-devtools/references/` files for deep technical detail

### tauri-mcp-bridge skill

**Activate when you see any of these triggers:**
- user wants to interact with a running tauri app (screenshot, click, type, scroll)
- user asks you to "look at" or "check" their running tauri app
- user wants to test their tauri app's ui behavior
- user wants to invoke tauri commands from outside the app
- user wants to monitor ipc traffic on a live app
- user asks about mcp-server-tauri or hypothesi/mcp-server-tauri
- user needs to set up the tauri mcp bridge plugin
- user wants to emit tauri events for testing
- user needs to debug on android emulator or ios simulator
- user asks about webview automation in tauri
- user mentions "tauri mcp", "tauri bridge", or "tauri automation"
- user wants to capture and replay ipc sequences
- user asks about screenshot, dom snapshot, or element picking in tauri

**What to do:**
1. Read `tauri-mcp-bridge/SKILL.md` for the full tool catalog and installation guide
2. If the bridge isn't installed yet, follow the 4-phase installation checklist
3. Use the dynamic install guide for non-standard project structures
4. Reference `tauri-mcp-bridge/references/` files for tool-specific deep dives

## How the Two Skills Complement Each Other

The skills cover different layers of the same problem:

| scenario | devtools skill | mcp bridge skill |
|----------|---------------|-----------------|
| "command returns wrong data" | read the rust handler's tracing spans to see where the logic diverges | invoke the command with test inputs and inspect the response |
| "ui doesn't update after action" | check if the event was emitted from rust | click the trigger, take screenshots before/after, check dom state |
| "app is slow" | read ipc timing spans in the calls tab to find the bottleneck | monitor ipc traffic while reproducing the slow interaction |
| "permission denied error" | inspect the config tab for capability bundle issues | call the command directly to isolate frontend vs backend cause |
| "layout looks wrong on mobile" | n/a (devtools is desktop-only for now) | connect to android/ios device and take native screenshots |

**Typical combined workflow:**
1. user reports a bug → use mcp bridge to reproduce it (screenshot, click, inspect dom)
2. bug confirmed → use devtools to read rust logs and ipc spans
3. root cause found → fix the code
4. use mcp bridge to verify the fix (screenshot, invoke command, check response)

## Key Constraints

- **tauri v2 only** — tauri v1 uses a different plugin system. if you detect tauri v1, tell the user to use the `devtools` crate directly.
- **devtools is dev-only** — never include `tauri-plugin-devtools` in production builds. it's behind `#[cfg(debug_assertions)]` for a reason.
- **mcp bridge requires the plugin installed** — the 20 tools won't work until the user has completed the 4-phase installation (cargo dependency, rust init, capabilities, mcp config).
- **screenshots on macos need screen recording permission** — if screenshots return blank, guide the user to system preferences → privacy → screen recording.
- **android needs adb port forwarding** — `adb forward tcp:3001 tcp:3001` before connecting.
