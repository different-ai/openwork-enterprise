# PRD: HandsFree + OpenCode Browser Integration

**Status:** Planned  
**Priority:** Medium  
**Depends on:** opencode-browser Chrome Web Store publication  

## Problem

HandsFree currently uses `chrome-devtools-mcp` for browser control. This works but has limitations:
- Launches a separate Chrome profile (or `--autoConnect` which is fragile)
- No CSS/aria/label selector support — only CDP uid-based clicking
- No shadow DOM traversal
- No download management
- No file upload support
- The Realtime model can't discover tool names without calling `mcp_list_tools` first

Meanwhile, `opencode-browser` already has 20+ browser automation tools with richer selectors, shadow DOM traversal, tab ownership, download/upload support, and runs in the user's actual Chrome profile with their cookies and logins.

## Proposal

Replace `chrome-devtools-mcp` with `opencode-browser` as HandsFree's browser control backend.

### Architecture

```
HandsFree Electron
  │
  ├─ Spawn opencode-browser broker (bin/broker.cjs)
  │   └─ Connects to Chrome extension via Native Messaging
  │
  └─ Talk to broker over Unix socket (~/.opencode-browser/broker.sock)
      JSON-lines protocol: { type: "request", id, op, ...args }
```

**Option A: Direct socket integration** (recommended)

Electron main connects to the broker Unix socket directly. No MCP wrapper needed. Tool calls from the Realtime model route through a thin adapter in Electron main that translates tool names to broker ops.

Pros:
- Simplest, fewest moving parts
- No extra process
- Electron already has Node.js `net` module for Unix sockets

Cons:
- HandsFree-specific code, not reusable as MCP

**Option B: MCP wrapper**

Write a small MCP server that proxies to the broker socket. HandsFree connects to it like any other MCP server.

Pros:
- Reusable by other MCP clients
- Fits existing HandsFree MCP plumbing

Cons:
- Extra process
- Extra translation layer

### Tool Mapping

| HandsFree tool | opencode-browser op | Notes |
|---|---|---|
| `browser_navigate(url)` | `browser_navigate` | |
| `browser_click(selector)` | `browser_click` | CSS + label + aria + text selectors |
| `browser_type(selector, text)` | `browser_type` | With `clear` option |
| `browser_screenshot()` | `browser_screenshot` | Returns base64 PNG |
| `browser_snapshot()` | `browser_snapshot` | Structured accessibility tree |
| `browser_scroll(selector)` | `browser_scroll` | Scroll element into view |
| `browser_tabs()` | `browser_get_tabs` | List all tabs |
| `browser_open(url)` | `browser_open_tab` | |
| `browser_close(tabId)` | `browser_close_tab` | |
| `browser_query(selector, mode)` | `browser_query` | Read text/value/html/exists |
| `browser_download(url)` | `browser_download` | |
| `browser_console()` | `browser_console` | Read console logs |

### CUA Integration

When `use_computer` delegates to gpt-5.5 for browser tasks, the CUA model uses screenshot-based clicking. With opencode-browser integration, we could add a hybrid mode:

1. CUA takes screenshot, identifies the target
2. Instead of coordinate-clicking, use `browser_snapshot` to get the accessibility tree
3. Match the visual target to an accessibility node
4. Click using `browser_click(selector)` — more reliable than pixel coordinates

This is a future optimization, not part of the initial integration.

### Prerequisites

1. **opencode-browser Chrome extension published on Chrome Web Store**
   - Build: `bun run build:cws` (script exists)
   - Missing: CWS screenshots (1-5 at 1280x800), hosted privacy policy URL
   - Missing: CWS developer account ($5 + identity verification)
   - Missing: Manual dashboard submission
   - No code changes needed — extension is technically ready

2. **Native host installed**
   - `bin/native-host.cjs` must be registered with Chrome
   - Installation script exists but needs to run on user's machine
   - Could be automated as part of HandsFree onboarding

3. **Broker running**
   - `bin/broker.cjs` must be running for the socket connection
   - HandsFree Electron can spawn this on launch

## Non-Goals

- Headless browser (agent-browser backend) — not needed, user has their own Chrome
- MCP wrapper — use direct socket integration first
- CUA hybrid mode — future optimization
- Replacing CUA for browser tasks — opencode-browser complements CUA, doesn't replace it

## Success Criteria

- HandsFree can navigate, click, type, and screenshot in Chrome using opencode-browser
- Works with user's existing Chrome profile (cookies, logins preserved)
- Tab ownership prevents conflicts with other automation
- No separate Chrome instance needed (unlike chrome-devtools-mcp)
- Selector-based clicking is more reliable than screenshot-coordinate clicking for web content

## Timeline

1. Publish opencode-browser extension to Chrome Web Store
2. Add broker spawn + socket connection to HandsFree Electron main
3. Expose browser tools to Realtime model
4. Test with CUA test page and real websites
5. Remove chrome-devtools-mcp dependency
