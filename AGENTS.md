# Tauri Observability & MCP Bridge Skills

## Overview

This repository contains two companion skills for AI agents working with Tauri v2 applications:

1. **tauri-devtools** — teaches agents how to use CrabNebula DevTools for rust-side observability (console logs, ipc call tracing, config inspection, source browsing)
2. **tauri-mcp-bridge** — teaches agents how to connect to and interact with running Tauri apps through 20 MCP tools (screenshots, clicking, typing, ipc commands, event monitoring)

## Activation Rules

### Activate tauri-devtools when:
- the user is debugging a Tauri v2 app and needs rust-side visibility
- browser devtools isn't showing the information needed (rust logs, ipc timings, command payloads)
- the user mentions CrabNebula DevTools, tauri tracing, or structured logging
- the user needs to correlate frontend errors with backend behavior
- the user asks about ipc call performance, slow commands, or rust panics
- the user wants to inspect tauri.conf.json, permissions, or capabilities at runtime
- the user is debugging plugin initialization, event system behavior, or command registration
- the user asks what's happening "on the rust side"

### Activate tauri-mcp-bridge when:
- the user wants to interact with a running Tauri app (see it, click it, type in it)
- the user needs screenshots or dom snapshots of the app
- the user wants to invoke Tauri commands from outside the app
- the user needs to monitor ipc traffic in real time
- the user wants to test the app on Android emulator or iOS simulator
- the user asks about setting up mcp-server-tauri or the Tauri MCP plugin
- the user wants visual element inspection, css style checking, or accessibility tree dumps
- the user needs to emit Tauri events for testing or verification
- the user asks about automating Tauri app interaction

### Activate both skills together when:
- debugging requires both seeing the rust internals AND interacting with the running app
- reproducing a bug (mcp bridge) and then diagnosing the root cause (devtools)
- verifying a fix by checking both the rust behavior and the ui result
- performance investigation that needs ipc timing (devtools) and user flow reproduction (mcp bridge)

## Skill Files

### tauri-devtools
- `tauri-devtools/SKILL.md` — main skill file: architecture, installation, tab decision guide, 10 agent workflows, 8 anti-patterns
- `tauri-devtools/references/tab-reference.md` — per-tab columns, filtering, keyboard shortcuts
- `tauri-devtools/references/ipc-span-anatomy.md` — span field documentation with examples
- `tauri-devtools/references/architecture-deep-dive.md` — tracing subscriber pipeline and data flow
- `tauri-devtools/references/common-debugging-scenarios.md` — 12+ scenarios with symptoms and fixes
- `tauri-devtools/references/integration-patterns.md` — plugin compatibility and custom tracing

### tauri-mcp-bridge
- `tauri-mcp-bridge/SKILL.md` — main skill file: 20 tools with schemas, 4-phase install, verification
- `tauri-mcp-bridge/references/tool-reference.md` — complete tool reference with params and return types
- `tauri-mcp-bridge/references/installation-checklist.md` — 4-phase verifiable checklist
- `tauri-mcp-bridge/references/dynamic-install-guide.md` — adaptive detection and adaptation rules
- `tauri-mcp-bridge/references/visual-tools-deep-dive.md` — screenshot, dom snapshot, element picking
- `tauri-mcp-bridge/references/ipc-tools-deep-dive.md` — ipc monitoring, command execution, events
- `tauri-mcp-bridge/references/multi-app-and-mobile.md` — multi-app architecture, android/ios setup
- `tauri-mcp-bridge/references/security-and-troubleshooting.md` — security model and troubleshooting

## Combined Workflow Pattern

The most effective debugging pattern uses both skills in sequence:

1. **Reproduce** — use mcp bridge to take a screenshot, click through the ui, confirm the bug exists
2. **Diagnose** — use devtools to read rust logs, inspect ipc call timings, check the config tab for permission issues
3. **Fix** — edit the relevant rust or frontend code
4. **Verify** — use mcp bridge to re-run the interaction, take a new screenshot, confirm the fix works
5. **Monitor** — use mcp bridge ipc monitoring to ensure no regressions in related commands

## Constraints

- **Tauri v2 only** — these skills are designed for Tauri v2's plugin system. Tauri v1 uses a different architecture (`devtools` crate instead of `tauri-plugin-devtools`).
- **Development builds only** — CrabNebula DevTools should never be included in production builds. The plugin is gated behind `#[cfg(debug_assertions)]`.
- **MCP bridge requires plugin installation** — the 20 tools require completing a 4-phase setup: cargo dependency, rust initialization, capability permissions, and MCP client configuration.
- **Platform considerations** — macOS requires screen recording permission for screenshots; Android requires adb port forwarding; Linux falls back to JavaScript-based screenshots when native capture isn't available.
