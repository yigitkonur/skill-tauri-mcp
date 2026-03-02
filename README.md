# skill-tauri-mcp

```bash
npx skills add yigitkonur/skill-tauri-mcp
```

> [copilot review setup](https://github.com/yigitkonur/skill-copilot-review) · [devin review setup](https://github.com/yigitkonur/skill-devin-review-init) · [greptile review setup](https://github.com/yigitkonur/skill-greptile-init) · [mcp server testing](https://github.com/yigitkonur/skill-mcp-server-tester) · [mcp-use code review](https://github.com/yigitkonur/skill-mcp-use) · [design dna extraction](https://github.com/yigitkonur/skill-design-soul-saas) · [snapshot to nextjs](https://github.com/yigitkonur/skill-snapshot-to-nextjs)

two companion skills that give ai agents deep observability and live control over tauri v2 apps. one plugs into crabnebula devtools for rust-side debugging — the console/calls/config/sources tabs that browser devtools fundamentally cannot show. the other bridges mcp so agents can screenshot, click, type, call rust commands, and monitor ipc on a running app through 20 tools.

## what it does

most ai agents working with tauri hit two walls. first, they can't see what's happening on the rust side — browser devtools only shows the webview, not the command handlers, event pipelines, or tracing spans that make tauri apps work. second, they can't interact with a running tauri app the way they can with a browser — no screenshots, no clicking, no command invocation.

this skill package solves both problems:

**tauri-devtools** teaches agents how to use crabnebula devtools — the only tool that gives visibility into the rust side of tauri apps. it covers the console tab (structured rust logs with tracing span context), the calls tab (every ipc invocation with timing and payloads), the config tab (live tauri.conf.json, permissions, capability bundles), and the sources tab (frontend asset inspection). agents learn when to check each tab, how to read span hierarchies, and how to correlate frontend errors with backend causes.

**tauri-mcp-bridge** teaches agents how to connect to a running tauri app through the hypothesi/mcp-server-tauri bridge. once connected, agents get 20 tools that cover visual inspection (screenshots, dom snapshots, element finding), interaction (clicking, typing, scrolling, keyboard input), window management, ipc command execution, ipc monitoring, event emission, and log reading. the skill covers installation (a 4-phase process that modifies cargo.toml, lib.rs, capabilities, and mcp config), all tool schemas with parameters and return types, and platform-specific setup for macos, linux, android, and ios.

together, they give an agent full-stack tauri debugging capability — see the rust side through devtools, drive the app side through mcp.

### the 20 mcp tools

| # | tool | category | what it does |
|---|------|----------|-------------|
| 1 | `get_setup_instructions` | setup | returns step-by-step plugin setup instructions |
| 2 | `driver_session` | setup | start/stop/status the bridge connection |
| 3 | `list_devices` | mobile | lists android emulators and ios simulators |
| 4 | `webview_screenshot` | visual | captures native screenshot of the webview |
| 5 | `webview_dom_snapshot` | visual | accessibility tree or dom structure as yaml |
| 6 | `webview_get_styles` | visual | computed css styles for an element |
| 7 | `webview_find_element` | visual | finds elements by css/xpath/text/aria-label/ref |
| 8 | `webview_interact` | interaction | click, type, scroll, hover, clear, select, focus |
| 9 | `webview_keyboard` | interaction | press, down, up, type keyboard input |
| 10 | `webview_wait_for` | interaction | waits for element visible/hidden/attached/detached |
| 11 | `webview_execute_js` | interaction | runs arbitrary javascript in the webview |
| 12 | `webview_select_element` | interaction | visual picker overlay — user clicks, agent gets metadata |
| 13 | `webview_get_pointed_element` | interaction | gets element from alt+shift+click |
| 14 | `manage_window` | window | list, info, or resize tauri windows |
| 15 | `ipc_execute_command` | ipc | invoke any registered tauri command |
| 16 | `ipc_monitor` | ipc | start/stop ipc call interception |
| 17 | `ipc_get_captured` | ipc | get captured ipc traffic with timing |
| 18 | `ipc_emit_event` | ipc | emit tauri events for testing |
| 19 | `ipc_get_backend_state` | ipc | get app metadata, version, environment |
| 20 | `read_logs` | monitoring | read console/android/ios/system logs |

### what makes this different

- **grounded in real source code** — every tool name, schema, and parameter extracted from the actual hypothesi/mcp-server-tauri codebase. not a summary of docs, not a guess at what the api might look like. the skill files reference specific source files, actual type definitions, and real default values.

- **two complementary skills** — devtools fills the rust/ipc blind spot browser devtools leaves. mcp bridge lets the agent see and drive the app. neither replaces the other. an agent debugging a tauri command failure would use devtools to read the rust panic in console, then use mcp bridge to replay the frontend action that triggered it.

- **adaptive installation** — the dynamic install guide teaches agents how to detect tauri version, find entry points, handle monorepos, adapt to custom window labels, and verify each edit step by step. not a copy-paste tutorial — a decision tree that works across different project structures.

- **breaking change aware** — documents every version change from v0.1.0 through v0.9.0 including the permission rename (`tauri-plugin-mcp` → `mcp`), tool name prefix removal (`tauri_` prefix dropped), and html2canvas replacement with native screenshot api.

- **platform-specific** — covers macos screen recording permission for screenshots, android adb port forwarding for device testing, linux javascript fallback when native screenshots aren't available, and ios simulator setup with proper device listing.

## usage

```
"my tauri app's greet command returns an error but the frontend just shows 'something went wrong' — what's actually happening on the rust side?"
```

```
"take a screenshot of my running tauri app and tell me if the sidebar layout matches the figma spec"
```

```
"set up crabnebula devtools in my tauri project so i can see rust logs and ipc call timings"
```

```
"monitor all ipc calls while i click through the settings page, then show me which commands are slow"
```

### file overview

| file | lines | what it covers |
|------|-------|---------------|
| `SKILL.md` | 475 | crabnebula devtools: architecture, installation, tab decision guide, 10 agent workflows, 8 anti-patterns, troubleshooting |
| `references/devtools-tab-reference.md` | 176 | per-tab documentation: columns, filtering, keyboard shortcuts |
| `references/devtools-ipc-span-anatomy.md` | 328 | span fields, normal/slow/error examples, custom spans, waterfall reading |
| `references/devtools-architecture-deep-dive.md` | 188 | tracing subscriber pipeline, data flow, component roles |
| `references/devtools-common-debugging-scenarios.md` | 214 | 14 scenarios with symptoms, tabs, root causes, fixes |
| `references/devtools-integration-patterns.md` | 323 | tauri-plugin-log compatibility, custom tracing, RUST_LOG |
| `SKILL-mcp-bridge.md` | 964 | mcp bridge: 20 tools with schemas, 4-phase install, verification, 12 workflows, 24-row troubleshooting |
| `references/bridge-tool-reference.md` | 609 | complete tool reference: params, types, annotations, return examples |
| `references/bridge-installation-checklist.md` | 254 | 4-phase verifiable checklist with idempotency and mobile detection |
| `references/bridge-dynamic-install-guide.md` | 188 | adaptive detection: 5 checks + 6 adaptation rules |
| `references/bridge-visual-tools-deep-dive.md` | 434 | screenshot internals, dom snapshots, element picking, see→interact workflow |
| `references/bridge-ipc-tools-deep-dive.md` | 423 | ipc monitoring workflow, command execution, event emission, backend state |
| `references/bridge-multi-app-and-mobile.md` | 364 | multi-app architecture, android/ios setup, remote device testing |
| `references/bridge-security-and-troubleshooting.md` | 240 | security model, 20-row troubleshooting matrix |

**total: 5,172 lines across 14 skill files** (+ 260 lines in CLAUDE.md, AGENTS.md, README.md = **5,432 lines**).

### scope

**built for:** tauri v2 apps that need rust-side debugging, ipc inspection, or ai agent interaction. covers both the observability gap (what browser devtools can't see) and the automation gap (agents that need to see and drive running apps).

**not for:** tauri v1 apps (use `devtools` crate instead of `tauri-plugin-devtools`). not for browser-only web apps — use chrome devtools mcp or playwright mcp instead.

## license

mit
