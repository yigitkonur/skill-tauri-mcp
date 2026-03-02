# Visual & Webview Tools — Deep Dive

> Reference for all visual inspection, DOM analysis, and webview interaction tools in tauri-plugin-mcp-bridge v0.9.0.
> All tool names have **no prefix** (since v0.8.0).

---

## webview_screenshot

Captures a screenshot of the active webview and returns it as base64-encoded image data.

### Platform Implementation Matrix

| Platform | Method | Notes |
|---|---|---|
| **macOS** | WKWebView native snapshot | Requires Screen Recording permission |
| **Windows** | webview2-com capture | Works out of the box |
| **Android** | JNI `WebView.draw()` to Canvas → Bitmap | Requires `adb forward` for port access |
| **iOS** | `UIView` snapshotting | Simulator-based; uses native API |
| **Linux** | html2canvas-pro JS injection | Fallback — cannot capture outside viewport |

### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `maxWidth` | number | (none) | Resizes using the `image` crate after capture. Smaller = faster transfer. |
| `filePath` | string | (none) | Saves to disk AND returns base64. Both happen when specified. |
| `format` | string | `"png"` | `"png"` (lossless, larger) or `"jpeg"` (lossy, smaller) |

### Return Structure

```json
{
  "type": "image",
  "data": "iVBORw0KGgoAAAANSUhEUgAA...",
  "mimeType": "image/png"
}
```

When `filePath` is set, the image is written to disk at that path AND returned as base64 in the response.

### macOS Screen Recording Permission

This is the **#1 gotcha** on macOS:

1. Go to **System Settings → Privacy & Security → Screen Recording**
2. Enable the **terminal application** you are running (Terminal.app, iTerm2, Warp, etc.)
3. **Fully restart** the terminal application — toggling the permission is not enough
4. Re-run `cargo tauri dev`

**Symptom when missing:** Screenshot returns a **blank or black image** with no error. The tool succeeds but the image content is empty. This is because macOS silently returns a blank capture when permission is not granted.

### Linux html2canvas-pro Fallback

On Linux, native screenshot is not implemented. The plugin injects html2canvas-pro into the webview:

- **Cannot capture content outside the current viewport** — scroll first if needed
- **May miss some CSS features** — complex filters, backdrop-filter, some newer properties
- **oklch() CSS support** was broken before v0.8.1 — upgrade if colors render incorrectly

### Tips

- Use `maxWidth: 800` for faster responses when full resolution is not needed
- Use `format: "jpeg"` for iterative debugging (smaller payload, faster round-trips)
- Take "before and after" screenshots to verify UI changes
- If screenshot is blank on macOS, ALWAYS check Screen Recording permission first

---

## webview_dom_snapshot

Captures a structured snapshot of the DOM as YAML. Two types available.

### Accessibility Type

Returns an accessibility tree computed using the `aria-api` library. Each node includes:

- **role**: ARIA role (button, link, heading, textbox, etc.)
- **name**: Accessible name (computed from labels, aria-label, text content)
- **states**: Active states (focused, expanded, checked, disabled, etc.)
- **ref**: A unique reference ID that can be used as a selector in other tools

Example output:

```yaml
- role: navigation
  name: "Main menu"
  ref: ref-1
  children:
    - role: link
      name: "Home"
      ref: ref-2
      states: []
    - role: link
      name: "Settings"
      ref: ref-3
      states: [focused]
    - role: button
      name: "Log out"
      ref: ref-4
      states: []
- role: main
  name: ""
  ref: ref-5
  children:
    - role: heading
      name: "Dashboard"
      ref: ref-6
      level: 1
    - role: textbox
      name: "Search"
      ref: ref-7
      states: [focused]
      value: ""
```

**Key insight:** The `ref` values (e.g., `ref-7`) can be passed directly to other tools using `strategy: "ref"`. This is the most reliable way to target elements discovered via accessibility snapshots.

### Structure Type

Returns the DOM hierarchy in YAML format:

```yaml
- tag: div
  id: "app"
  classes: ["container", "dark-mode"]
  children:
    - tag: nav
      classes: ["main-nav"]
      children:
        - tag: a
          href: "/"
          text: "Home"
        - tag: a
          href: "/settings"
          text: "Settings"
    - tag: main
      children:
        - tag: h1
          text: "Dashboard"
        - tag: input
          type: "text"
          placeholder: "Search..."
```

### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `type` | string | `"accessibility"` | `"accessibility"` or `"structure"` |
| `selector` | string | (none) | CSS selector to scope snapshot to a subtree |

### When to Use Which Type

- **accessibility**: When you need semantic meaning, ARIA roles, or ref-based selectors
- **structure**: When you need raw DOM structure, tag names, attributes, classes

---

## webview_get_styles

Returns **computed** (resolved) CSS values for a matched element.

### Important Distinction

This returns the browser's **computed values**, not the declared/authored CSS. For example:
- Declared: `width: 50%` → Computed: `width: 480px`
- Declared: `color: var(--primary)` → Computed: `color: rgb(59, 130, 246)`
- Declared: `font-size: 1.5rem` → Computed: `font-size: 24px`

### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `selector` | string | (required) | CSS selector for the target element |
| `strategy` | string | `"css"` | Selector strategy: css, xpath, text, aria-label, ref |
| `properties` | string[] | (all) | Specific CSS properties to return. Omit for all. |

### Example Return

```json
{
  "display": "flex",
  "flexDirection": "column",
  "backgroundColor": "rgb(255, 255, 255)",
  "padding": "16px",
  "borderRadius": "8px",
  "boxShadow": "rgba(0, 0, 0, 0.1) 0px 1px 3px 0px"
}
```

### Tips

- Request specific properties for faster response: `properties: ["display", "color", "backgroundColor"]`
- Use to verify theme/dark-mode changes
- Computed styles always return in canonical format (rgb not hex, px not rem)

---

## webview_find_element

Locates an element and returns its HTML and position.

### Return Structure

- `outerHTML`: The element's full HTML (truncated to **5000 characters** since v0.3.1)
- `boundingRect`: `{x, y, width, height}` — position and size in viewport coordinates

### Selector Strategies

| Strategy | Input | Example | Best For |
|---|---|---|---|
| `css` (default) | CSS selector | `"#submit-btn"` | IDs, classes, attributes |
| `xpath` | XPath expression | `"//button[@type='submit']"` | Complex structural queries |
| `text` | Visible text content | `"Save Changes"` | When you see text but don't know the selector |
| `aria-label` | ARIA label value | `"Close dialog"` | Accessible elements |
| `ref` | Ref ID from dom_snapshot | `"ref-7"` | After accessibility snapshot |

### Text Strategy Example

```json
{
  "selector": "Save Changes",
  "strategy": "text"
}
```

Finds the first element whose `textContent` matches. Useful when you can see the button text in a screenshot but don't know its CSS selector.

### Ref Strategy Example

After getting a dom_snapshot (accessibility type) that includes `ref: ref-7` for a search input:

```json
{
  "selector": "ref-7",
  "strategy": "ref"
}
```

This is the most reliable approach: snapshot → find ref → use ref in subsequent tools.

---

## webview_select_element

Interactive element selection — shows a visual overlay in the app for the user to click.

### How It Works

1. A highlight overlay appears on the app
2. As the user hovers, elements are highlighted with a colored border
3. User clicks to select an element
4. Tool returns rich metadata about the clicked element

### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `timeout` | number | `60000` | Range: 5000–120000ms. How long to wait for user click. |

### Return Structure

The return is the richest of any element tool:

```json
{
  "tag": "button",
  "id": "submit-form",
  "classes": ["btn", "btn-primary"],
  "attributes": {"type": "submit", "data-testid": "submit"},
  "text": "Save Changes",
  "boundingRect": {"x": 200, "y": 400, "width": 120, "height": 40},
  "cssSelector": "button#submit-form.btn.btn-primary",
  "xpath": "/html/body/div[2]/form/button",
  "computedStyles": {"backgroundColor": "rgb(59, 130, 246)", "...": "..."},
  "parentChain": ["form#user-form", "div.container", "div#app", "body", "html"],
  "screenshot": "iVBORw0KGgoAAAA..."
}
```

Includes an **element-level screenshot** (cropped to just that element).

### When to Use

- Disambiguation: "Which button do you mean?" → let the user point at it
- Discovery: finding the right selector for a complex UI element
- Debugging: getting full context about a specific element

---

## webview_get_pointed_element

Passive element inspection — reads the element the user Alt+Shift+Clicked.

### How It Works

1. User **Alt+Shift+Clicks** an element in the Tauri app (before calling this tool)
2. This tool returns metadata about the last pointed element
3. No overlay is shown — this is a passive read

### Return Structure

Same rich metadata as `webview_select_element`:
- tag, id, classes, attributes, text
- boundingRect, cssSelector, xpath
- computedStyles, parentChain, screenshot

### Difference from webview_select_element

| | webview_select_element | webview_get_pointed_element |
|---|---|---|
| Shows overlay | Yes | No |
| Waits for click | Yes (with timeout) | No — reads last pointed |
| User action | Click during overlay | Alt+Shift+Click beforehand |
| Use case | "Point at it now" | "I already pointed at it" |

---

## webview_execute_js

Executes arbitrary JavaScript in the webview context.

### Key Behaviors

- Runs in the **webview's global scope** — has access to `window`, `document`, all page JS
- Has access to `window.__TAURI__` (since `withGlobalTauri: true` is required for the plugin)
- **Must explicitly return a value** — the last expression is NOT auto-returned
- Can interact with Tauri APIs directly via `window.__TAURI__`

### Wrapping Pattern

Always wrap in an IIFE (Immediately Invoked Function Expression) to ensure a return value:

```javascript
(function() {
  const items = document.querySelectorAll('.todo-item');
  const data = Array.from(items).map(el => ({
    text: el.textContent,
    done: el.classList.contains('completed')
  }));
  return JSON.stringify(data);
})()
```

**Without the wrapper**, the result will be `undefined`.

### Accessing Tauri APIs

```javascript
(function() {
  return window.__TAURI__.invoke('get_app_config');
})()
```

### Timeout

Long-running scripts may timeout. Keep JS execution focused and fast. For async operations:

```javascript
(async function() {
  const response = await fetch('/api/data');
  const data = await response.json();
  return JSON.stringify(data);
})()
```

---

## Complete Workflow: See → Understand → Interact → Verify

This is the standard visual debugging loop:

### Step 1: See — Take Initial Screenshot

```json
{"tool": "webview_screenshot", "arguments": {"maxWidth": 1024}}
```

Observe the current state of the UI.

### Step 2: Understand — Get DOM Snapshot

```json
{"tool": "webview_dom_snapshot", "arguments": {"type": "accessibility"}}
```

Get the semantic tree. Identify ref IDs for elements of interest.

### Step 3: Find — Locate Specific Element

```json
{"tool": "webview_find_element", "arguments": {"selector": "ref-7", "strategy": "ref"}}
```

Get the HTML and bounding rect for the target element.

### Step 4: Interact — Perform Action

```json
{"tool": "webview_interact", "arguments": {"selector": "ref-7", "strategy": "ref", "action": "click"}}
```

Click, type, scroll, or otherwise interact.

### Step 5: Wait — Let UI Update

```json
{"tool": "webview_wait_for", "arguments": {"selector": ".success-message", "state": "visible", "timeout": 5000}}
```

Wait for the expected result to appear.

### Step 6: Verify — Take Follow-up Screenshot

```json
{"tool": "webview_screenshot", "arguments": {"maxWidth": 1024}}
```

Confirm the action had the expected effect.

---

## Selector Strategy Quick Reference

| Strategy | Syntax | When to Use |
|---|---|---|
| `css` | `"#id"`, `".class"`, `"tag[attr]"` | Default; when you know the CSS selector |
| `xpath` | `"//div[@class='x']//button"` | Complex structural navigation |
| `text` | `"Submit Order"` | Visible text, unknown selector |
| `aria-label` | `"Close dialog"` | Accessible components |
| `ref` | `"ref-12"` | After dom_snapshot accessibility; most reliable |

**Best practice:** Use `dom_snapshot` (accessibility) → get ref IDs → use `ref` strategy. This avoids brittle CSS selectors and works even when classes change.
