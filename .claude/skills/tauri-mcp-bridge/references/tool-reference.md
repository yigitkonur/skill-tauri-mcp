# MCP Bridge Tool Reference

All 20 tools exposed by `@hypothesi/tauri-mcp-server` v0.9.0. Tool names have NO `tauri_` prefix (removed in v0.8.0).

**Error format (all tools):**
```json
{"type": "text", "text": "Error: <description of what went wrong>"}
```

**Common parameters across tools:**
- `appIdentifier` (string | number, optional) — port number or bundle ID to target a specific running Tauri app
- `windowId` (string, optional) — window label to target a specific window in multi-window apps

**Selector strategy** (tools that accept `strategy` parameter):
| Strategy | Value | Selector Example | Description |
|---|---|---|---|
| CSS | `"css"` (default) | `"#my-btn"`, `".card > h2"` | Standard CSS selectors |
| XPath | `"xpath"` | `"//button[@type='submit']"` | XPath 1.0 expressions |
| Text | `"text"` | `"Sign In"` | Match by exact text content |
| ARIA Label | `"aria-label"` | `"Close dialog"` | Match by aria-label attribute |
| Ref ID | `"ref"` | `"ref-42"` | Use ref IDs from dom_snapshot accessibility output |

---

## 1. get_setup_instructions

**Category:** Setup & Config
**Description:** Returns step-by-step setup instructions for the MCP bridge plugin.
**Annotations:** `readOnlyHint: true`, `destructiveHint: false`, `openWorldHint: false`

**Parameters:** None

**Return example:**
```json
{
  "type": "text",
  "text": "# MCP Bridge Setup Instructions\n\n## Step 1: Add the Rust plugin\n\nAdd to src-tauri/Cargo.toml:\n```toml\ntauri-plugin-mcp-bridge = \"0.9\"\n```\n\n## Step 2: Register the plugin\n\nIn src-tauri/src/lib.rs (inside your run function):\n```rust\n#[cfg(debug_assertions)]\n{\n    builder = builder.plugin(tauri_plugin_mcp_bridge::init());\n}\n```\n\n## Step 3: Enable withGlobalTauri\n\nIn src-tauri/tauri.conf.json, add to the \"app\" section:\n```json\n\"withGlobalTauri\": true\n```\n\n## Step 4: Add capabilities\n\nIn src-tauri/capabilities/default.json, add to permissions:\n```json\n\"mcp-bridge:default\"\n```\n\n## Step 5: Run your app\n\n```bash\ncargo tauri dev\n```\n\nLook for: MCP Bridge WebSocket server listening on 0.0.0.0:9223"
}
```

---

## 2. driver_session

**Category:** Setup & Config
**Description:** Manages bridge session lifecycle. MUST be called with `action: "start"` before any other tool.
**Annotations:** `readOnlyHint: false`, `destructiveHint: false`, `openWorldHint: false`

**Schema:**
| Parameter | Type | Required | Default | Constraints |
|---|---|---|---|---|
| `action` | string | ✅ | — | `"start"` \| `"stop"` \| `"status"` |
| `host` | string | ❌ | `"127.0.0.1"` | Valid hostname or IP |
| `port` | number | ❌ | `9223` | 1–65535 |
| `appIdentifier` | string \| number | ❌ | — | Port number or bundle ID |

**Return example (start — success):**
```json
{"type": "text", "text": "Connected to Tauri app on port 9223 (com.example.myapp)"}
```

**Return example (status):**
```json
{"type": "text", "text": "Active sessions:\n- Port 9223: com.example.myapp (connected)\n- Port 9224: com.example.other (connected)"}
```

**Return example (error):**
```json
{"type": "text", "text": "Error: Could not connect to Tauri app at 127.0.0.1:9223. Is the app running with cargo tauri dev?"}
```

---

## 3. list_devices

**Category:** Setup & Config
**Description:** Lists connected Android emulators/devices and iOS simulators.
**Annotations:** `readOnlyHint: true`, `destructiveHint: false`, `openWorldHint: false`

**Parameters:** None

**Return example:**
```json
{
  "type": "text",
  "text": "Android devices:\n- emulator-5554: Pixel_6_API_34\n- 1A2B3C4D: Physical Device\n\niOS simulators:\n- AAAA-BBBB-CCCC: iPhone 15 Pro (Booted)\n- DDDD-EEEE-FFFF: iPad Air (Shutdown)"
}
```

---

## 4. webview_screenshot

**Category:** UI Automation & WebView
**Description:** Captures a screenshot of the current WebView state.
**Annotations:** `readOnlyHint: true`, `destructiveHint: false`, `openWorldHint: false`

**Schema:**
| Parameter | Type | Required | Default | Constraints |
|---|---|---|---|---|
| `maxWidth` | number | ❌ | — | Positive integer, pixels |
| `filePath` | string | ❌ | — | Valid file system path |
| `format` | string | ❌ | `"png"` | `"png"` \| `"jpeg"` |
| `quality` | number | ❌ | — | 1–100 (jpeg only) |
| `windowId` | string | ❌ | — | Window label |
| `appIdentifier` | string \| number | ❌ | — | Port or bundle ID |

**Return example (inline):**
```json
{
  "type": "image",
  "data": "iVBORw0KGgoAAAANSUhEUgAA...",
  "mimeType": "image/png"
}
```

**Return example (saved to disk):**
```json
{"type": "text", "text": "Screenshot saved to /tmp/screenshot.png (1920x1080)"}
```

**Platform notes:**
- macOS: Native capture, requires Screen Recording permission on terminal app
- Linux: Falls back to html2canvas-pro (JS-based capture)
- Windows: Native capture, no special permissions

---

## 5. webview_dom_snapshot

**Category:** UI Automation & WebView
**Description:** Returns YAML accessibility tree or DOM structure snapshot.
**Annotations:** `readOnlyHint: true`, `destructiveHint: false`, `openWorldHint: false`

**Schema:**
| Parameter | Type | Required | Default | Constraints |
|---|---|---|---|---|
| `type` | string | ❌ | `"accessibility"` | `"accessibility"` \| `"structure"` |
| `selector` | string | ❌ | — | Root element to snapshot from |
| `strategy` | string | ❌ | `"css"` | `"css"` \| `"xpath"` \| `"text"` \| `"aria-label"` \| `"ref"` |
| `windowId` | string | ❌ | — | Window label |
| `appIdentifier` | string \| number | ❌ | — | Port or bundle ID |

**Return example (accessibility):**
```json
{
  "type": "text",
  "text": "- role: main\n  name: \"App Container\"\n  ref: ref-1\n  children:\n    - role: navigation\n      name: \"Main Nav\"\n      ref: ref-2\n      children:\n        - role: link\n          name: \"Home\"\n          ref: ref-3\n        - role: link\n          name: \"Settings\"\n          ref: ref-4\n    - role: region\n      name: \"Content\"\n      ref: ref-5\n      children:\n        - role: heading\n          name: \"Dashboard\"\n          level: 1\n          ref: ref-6\n        - role: button\n          name: \"Refresh\"\n          ref: ref-7\n          state: enabled"
}
```

**Return example (structure):**
```json
{
  "type": "text",
  "text": "- tag: div\n  id: app\n  class: container\n  children:\n    - tag: nav\n      class: sidebar\n      children:\n        - tag: a\n          href: /home\n          text: Home\n        - tag: a\n          href: /settings\n          text: Settings"
}
```

---

## 6. webview_get_styles

**Category:** UI Automation & WebView
**Description:** Gets computed CSS styles for an element.
**Annotations:** `readOnlyHint: true`, `destructiveHint: false`, `openWorldHint: false`

**Schema:**
| Parameter | Type | Required | Default | Constraints |
|---|---|---|---|---|
| `selector` | string | ✅ | — | Target element |
| `properties` | string[] | ❌ | all computed | CSS property names |
| `strategy` | string | ❌ | `"css"` | `"css"` \| `"xpath"` \| `"text"` \| `"aria-label"` \| `"ref"` |
| `windowId` | string | ❌ | — | Window label |
| `appIdentifier` | string \| number | ❌ | — | Port or bundle ID |

**Return example:**
```json
{
  "type": "text",
  "text": "{\n  \"display\": \"flex\",\n  \"flexDirection\": \"column\",\n  \"color\": \"rgb(0, 0, 0)\",\n  \"fontSize\": \"16px\",\n  \"backgroundColor\": \"rgba(0, 0, 0, 0)\"\n}"
}
```

---

## 7. webview_find_element

**Category:** UI Automation & WebView
**Description:** Finds elements matching a selector.
**Annotations:** `readOnlyHint: true`, `destructiveHint: false`, `openWorldHint: false`

**Schema:**
| Parameter | Type | Required | Default | Constraints |
|---|---|---|---|---|
| `selector` | string | ✅ | — | Element selector |
| `strategy` | string | ❌ | `"css"` | `"css"` \| `"xpath"` \| `"text"` \| `"aria-label"` \| `"ref"` |
| `windowId` | string | ❌ | — | Window label |
| `appIdentifier` | string \| number | ❌ | — | Port or bundle ID |

**Return example:**
```json
{
  "type": "text",
  "text": "Found 1 element(s):\n\n[1] <button id=\"submit-btn\" class=\"btn primary\" type=\"submit\">Save Changes</button>\n    Bounding rect: {x: 450, y: 320, width: 120, height: 40}\n    Visible: true"
}
```

**Note:** outerHTML is truncated to 5000 characters per element.

---

## 8. webview_interact

**Category:** UI Automation & WebView
**Description:** Performs UI interactions — click, type, scroll, hover, clear, select, focus.
**Annotations:** `readOnlyHint: false`, `destructiveHint: false`, `openWorldHint: false`

**Schema:**
| Parameter | Type | Required | Default | Constraints |
|---|---|---|---|---|
| `selector` | string | ✅ | — | Target element |
| `action` | string | ✅ | — | `"click"` \| `"type"` \| `"scroll"` \| `"hover"` \| `"clear"` \| `"select"` \| `"focus"` |
| `value` | string | ❌ | — | Text for type, option for select |
| `scrollDirection` | string | ❌ | — | `"up"` \| `"down"` |
| `scrollAmount` | number | ❌ | — | Pixels to scroll |
| `strategy` | string | ❌ | `"css"` | `"css"` \| `"xpath"` \| `"text"` \| `"aria-label"` \| `"ref"` |
| `windowId` | string | ❌ | — | Window label |
| `appIdentifier` | string \| number | ❌ | — | Port or bundle ID |

**Return example (click):**
```json
{"type": "text", "text": "Clicked element: #submit-btn"}
```

**Return example (type):**
```json
{"type": "text", "text": "Typed 'hello@example.com' into #email-input"}
```

**Return example (scroll):**
```json
{"type": "text", "text": "Scrolled down 300px on .content-area"}
```

---

## 9. webview_keyboard

**Category:** UI Automation & WebView
**Description:** Sends keyboard input — press, down, up, or type actions.
**Annotations:** `readOnlyHint: false`, `destructiveHint: false`, `openWorldHint: false`

**Schema:**
| Parameter | Type | Required | Default | Constraints |
|---|---|---|---|---|
| `action` | string | ✅ | — | `"press"` \| `"down"` \| `"up"` \| `"type"` |
| `key` | string | ✅ | — | Key name: `"Enter"`, `"Tab"`, `"Escape"`, `"a"`, `"F5"`, etc. |
| `modifiers` | string[] | ❌ | `[]` | `"ctrl"` \| `"shift"` \| `"alt"` \| `"meta"` |
| `windowId` | string | ❌ | — | Window label |
| `appIdentifier` | string \| number | ❌ | — | Port or bundle ID |

**Return example:**
```json
{"type": "text", "text": "Key press: Enter"}
```

**Return example (with modifiers):**
```json
{"type": "text", "text": "Key press: ctrl+s"}
```

**Common key combinations:**
- `{action: "press", key: "Enter"}` — Submit form
- `{action: "press", key: "Tab"}` — Move to next field
- `{action: "press", key: "Escape"}` — Close modal
- `{action: "press", key: "a", modifiers: ["ctrl"]}` — Select all
- `{action: "press", key: "s", modifiers: ["ctrl"]}` — Save (Ctrl+S)
- `{action: "press", key: "z", modifiers: ["ctrl"]}` — Undo

---

## 10. webview_wait_for

**Category:** UI Automation & WebView
**Description:** Waits for an element to reach a specified condition.
**Annotations:** `readOnlyHint: true`, `destructiveHint: false`, `openWorldHint: false`

**Schema:**
| Parameter | Type | Required | Default | Constraints |
|---|---|---|---|---|
| `selector` | string | ✅ | — | Element to wait for |
| `state` | string | ❌ | `"visible"` | `"visible"` \| `"hidden"` \| `"attached"` \| `"detached"` |
| `timeout` | number | ❌ | — | Milliseconds (positive integer) |
| `strategy` | string | ❌ | `"css"` | `"css"` \| `"xpath"` \| `"text"` \| `"aria-label"` \| `"ref"` |
| `windowId` | string | ❌ | — | Window label |
| `appIdentifier` | string \| number | ❌ | — | Port or bundle ID |

**State values:**
- `"visible"` — Element exists in DOM and is visible (not `display:none`, not `visibility:hidden`)
- `"hidden"` — Element exists but is not visible
- `"attached"` — Element exists in the DOM (may or may not be visible)
- `"detached"` — Element has been removed from the DOM

**Return example (success):**
```json
{"type": "text", "text": "Element .success-message became visible after 1250ms"}
```

**Return example (timeout):**
```json
{"type": "text", "text": "Error: Timeout waiting for .success-message to become visible after 5000ms"}
```

---

## 11. webview_execute_js

**Category:** UI Automation & WebView
**Description:** Executes arbitrary JavaScript in the WebView context.
**Annotations:** `readOnlyHint: false`, `destructiveHint: false`, `openWorldHint: false`

**Schema:**
| Parameter | Type | Required | Default | Constraints |
|---|---|---|---|---|
| `script` | string | ✅ | — | Valid JavaScript code |
| `windowId` | string | ❌ | — | Window label |
| `appIdentifier` | string \| number | ❌ | — | Port or bundle ID |

**Return example:**
```json
{"type": "text", "text": "Result: {\"url\":\"http://localhost:1420\",\"title\":\"My App\",\"readyState\":\"complete\"}"}
```

**Important:** Script MUST return a value. Use IIFE pattern for complex scripts:
```javascript
(function() {
  const data = document.querySelector('#data-container').dataset;
  return JSON.stringify(data);
})()
```

---

## 12. webview_select_element

**Category:** UI Automation & WebView
**Description:** Activates a visual element picker overlay. User clicks an element, agent receives metadata.
**Annotations:** `readOnlyHint: true`, `destructiveHint: false`, `openWorldHint: false`

**Schema:**
| Parameter | Type | Required | Default | Constraints |
|---|---|---|---|---|
| `timeout` | number | ❌ | `60000` | 5000–120000 (ms) |
| `windowId` | string | ❌ | — | Window label |
| `appIdentifier` | string \| number | ❌ | — | Port or bundle ID |

**Return example:**
```json
{
  "type": "text",
  "text": "Selected element:\n  Tag: button\n  ID: save-btn\n  Classes: btn, btn-primary\n  Text: Save Changes\n  Bounding rect: {x: 450, y: 320, width: 120, height: 40}\n  CSS selector: #save-btn\n  XPath: /html/body/div/main/form/button[2]\n  Computed styles: {display: 'inline-flex', backgroundColor: 'rgb(59, 130, 246)', ...}\n  Parent chain: form.settings-form > div.actions > button#save-btn\n  [Element screenshot attached]"
}
```

---

## 13. webview_get_pointed_element

**Category:** UI Automation & WebView
**Description:** Gets element metadata from a previous Alt+Shift+Click by the user.
**Annotations:** `readOnlyHint: true`, `destructiveHint: false`, `openWorldHint: false`

**Schema:**
| Parameter | Type | Required | Default | Constraints |
|---|---|---|---|---|
| `windowId` | string | ❌ | — | Window label |
| `appIdentifier` | string \| number | ❌ | — | Port or bundle ID |

**Return:** Same metadata structure as `webview_select_element`.

**Return example (no element pointed):**
```json
{"type": "text", "text": "Error: No element has been pointed at. Ask the user to Alt+Shift+Click on an element first."}
```

---

## 14. manage_window

**Category:** Window Management
**Description:** Lists, inspects, or resizes application windows.
**Annotations:** `readOnlyHint: false` (for resize), `destructiveHint: false`, `openWorldHint: false`

**Schema:**
| Parameter | Type | Required | Default | Constraints |
|---|---|---|---|---|
| `action` | string | ✅ | — | `"list"` \| `"info"` \| `"resize"` |
| `windowId` | string | ❌ | — | Window label (required for info/resize) |
| `width` | number | ❌ | — | Positive integer (for resize) |
| `height` | number | ❌ | — | Positive integer (for resize) |
| `logical` | boolean | ❌ | `true` | Logical (true) vs physical (false) pixels |
| `appIdentifier` | string \| number | ❌ | — | Port or bundle ID |

**Return example (list):**
```json
{"type": "text", "text": "Windows:\n- main: \"My App\" (800x600, focused)\n- settings: \"Settings\" (400x300)"}
```

**Return example (info):**
```json
{"type": "text", "text": "Window 'main':\n  Title: My App\n  Size: 800x600 (logical)\n  Position: (100, 100)\n  Focused: true\n  Visible: true\n  Fullscreen: false\n  Maximized: false"}
```

**Return example (resize):**
```json
{"type": "text", "text": "Resized window 'main' to 1024x768"}
```

---

## 15. ipc_execute_command

**Category:** IPC & Plugin
**Description:** Invokes any registered Tauri command as if called from the frontend.
**Annotations:** `readOnlyHint: false`, `destructiveHint: false`, `openWorldHint: false`

**Schema:**
| Parameter | Type | Required | Default | Constraints |
|---|---|---|---|---|
| `command` | string | ✅ | — | Registered Tauri command name |
| `args` | unknown | ❌ | — | JSON-serializable arguments |
| `appIdentifier` | string \| number | ❌ | — | Port or bundle ID |

**Return example:**
```json
{"type": "text", "text": "Command result:\n{\"id\": 42, \"name\": \"John Doe\", \"email\": \"john@example.com\"}"}
```

**Return example (error):**
```json
{"type": "text", "text": "Error: Command 'get_user' failed: user not found"}
```

---

## 16. ipc_monitor

**Category:** IPC & Plugin
**Description:** Starts or stops IPC call interception.
**Annotations:** `readOnlyHint: false`, `destructiveHint: false`, `openWorldHint: false`

**Schema:**
| Parameter | Type | Required | Default | Constraints |
|---|---|---|---|---|
| `action` | string | ✅ | — | `"start"` \| `"stop"` |
| `appIdentifier` | string \| number | ❌ | — | Port or bundle ID |

**Return example (start):**
```json
{"type": "text", "text": "IPC monitoring started. Use ipc_get_captured to retrieve captured events."}
```

**Return example (stop):**
```json
{"type": "text", "text": "IPC monitoring stopped. Captured 15 events."}
```

---

## 17. ipc_get_captured

**Category:** IPC & Plugin
**Description:** Returns captured IPC events since monitoring was started.
**Annotations:** `readOnlyHint: true`, `destructiveHint: false`, `openWorldHint: false`

**Schema:**
| Parameter | Type | Required | Default | Constraints |
|---|---|---|---|---|
| `filter` | string | ❌ | — | Filter by command name substring |
| `appIdentifier` | string \| number | ❌ | — | Port or bundle ID |

**Return example:**
```json
{
  "type": "text",
  "text": "Captured IPC events (3):\n\n[1] Command: get_settings\n    Args: {}\n    Response: {\"theme\": \"dark\", \"language\": \"en\"}\n    Time: 2024-01-15T10:30:01.123Z\n\n[2] Command: save_settings\n    Args: {\"theme\": \"light\"}\n    Response: {\"success\": true}\n    Time: 2024-01-15T10:30:02.456Z\n\n[3] Command: get_settings\n    Args: {}\n    Response: {\"theme\": \"light\", \"language\": \"en\"}\n    Time: 2024-01-15T10:30:02.789Z"
}
```

---

## 18. ipc_emit_event

**Category:** IPC & Plugin
**Description:** Emits a Tauri event to the application.
**Annotations:** `readOnlyHint: false`, `destructiveHint: false`, `openWorldHint: false`

**Schema:**
| Parameter | Type | Required | Default | Constraints |
|---|---|---|---|---|
| `eventName` | string | ✅ | — | Event name |
| `payload` | unknown | ❌ | — | JSON-serializable payload |
| `appIdentifier` | string \| number | ❌ | — | Port or bundle ID |

**Return example:**
```json
{"type": "text", "text": "Event 'user:logout' emitted with payload: {\"reason\": \"session_expired\"}"}
```

---

## 19. ipc_get_backend_state

**Category:** IPC & Plugin
**Description:** Gets application metadata, version, environment, and registered plugins.
**Annotations:** `readOnlyHint: true`, `destructiveHint: false`, `openWorldHint: false`

**Schema:**
| Parameter | Type | Required | Default | Constraints |
|---|---|---|---|---|
| `appIdentifier` | string \| number | ❌ | — | Port or bundle ID |

**Return example:**
```json
{
  "type": "text",
  "text": "Backend state:\n  App: com.example.myapp v1.0.0\n  Tauri: 2.1.0\n  Environment: development\n  Plugins: [mcp-bridge, shell, dialog, fs]\n  Commands: [get_user, save_settings, get_settings, delete_user]\n  Windows: [main, settings]"
}
```

---

## 20. read_logs

**Category:** Log Reading
**Description:** Reads logs from multiple sources — WebView console, Android logcat, iOS logs, or system logs.
**Annotations:** `readOnlyHint: true`, `destructiveHint: false`, `openWorldHint: false`

**Schema:**
| Parameter | Type | Required | Default | Constraints |
|---|---|---|---|---|
| `source` | string | ✅ | — | `"console"` \| `"android"` \| `"ios"` \| `"system"` |
| `lines` | number | ❌ | `50` | Positive integer |
| `filter` | string | ❌ | — | Substring match filter |
| `since` | string | ❌ | — | ISO 8601 timestamp |
| `windowId` | string | ❌ | — | Window label |
| `appIdentifier` | string \| number | ❌ | — | Port or bundle ID |

**Source types:**
- `"console"` — WebView `console.log()`, `console.warn()`, `console.error()` output
- `"android"` — Android logcat output (requires connected device/emulator via ADB)
- `"ios"` — iOS device or simulator logs
- `"system"` — System-level application logs

**Return example (console):**
```json
{
  "type": "text",
  "text": "[LOG] 10:30:01 App initialized\n[LOG] 10:30:02 Fetching user data...\n[WARN] 10:30:03 API response slow (2.1s)\n[LOG] 10:30:03 User data loaded: 42 records\n[ERROR] 10:30:05 Failed to load avatar: 404"
}
```

**Return example (android):**
```json
{
  "type": "text",
  "text": "01-15 10:30:01.123 D/TauriApp: WebView initialized\n01-15 10:30:02.456 I/TauriApp: Bridge connected on port 9223\n01-15 10:30:03.789 W/TauriApp: Slow IPC call: get_settings (500ms)\n01-15 10:30:04.012 E/TauriApp: Uncaught exception in WebView"
}
```

**Return example (filtered):**
```json
{
  "type": "text",
  "text": "[ERROR] 10:30:05 Failed to load avatar: 404\n[ERROR] 10:31:12 WebSocket reconnect failed: timeout"
}
```

---

## Tool Annotations Summary

| Tool | readOnlyHint | destructiveHint | openWorldHint |
|---|---|---|---|
| `get_setup_instructions` | true | false | false |
| `driver_session` | false | false | false |
| `list_devices` | true | false | false |
| `webview_screenshot` | true | false | false |
| `webview_dom_snapshot` | true | false | false |
| `webview_get_styles` | true | false | false |
| `webview_find_element` | true | false | false |
| `webview_interact` | false | false | false |
| `webview_keyboard` | false | false | false |
| `webview_wait_for` | true | false | false |
| `webview_execute_js` | false | false | false |
| `webview_select_element` | true | false | false |
| `webview_get_pointed_element` | true | false | false |
| `manage_window` | false | false | false |
| `ipc_execute_command` | false | false | false |
| `ipc_monitor` | false | false | false |
| `ipc_get_captured` | true | false | false |
| `ipc_emit_event` | false | false | false |
| `ipc_get_backend_state` | true | false | false |
| `read_logs` | true | false | false |

**Read-only tools** (safe to call anytime): `get_setup_instructions`, `list_devices`, `webview_screenshot`, `webview_dom_snapshot`, `webview_get_styles`, `webview_find_element`, `webview_wait_for`, `webview_select_element`, `webview_get_pointed_element`, `ipc_get_captured`, `ipc_get_backend_state`, `read_logs`

**Side-effect tools** (modify state or trigger actions): `driver_session`, `webview_interact`, `webview_keyboard`, `webview_execute_js`, `manage_window` (resize), `ipc_execute_command`, `ipc_monitor`, `ipc_emit_event`
