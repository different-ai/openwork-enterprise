---
name: using-openwork-with-chrome-mcp
description: |
  Drive the OpenWork dev renderer with Chrome MCP and verify workspace UI flows.

  Triggers when user mentions:
  - "use Chrome MCP with OpenWork"
  - "hook up to pnpm dev renderer"
  - "add an OpenWork workspace through the UI"
---

# Skill: using-openwork-with-chrome-mcp

Use this when you need to control the running OpenWork app UI through Chrome MCP, especially the local `pnpm dev` renderer at `localhost:5173`.

## Quick Usage (Already Configured)

### 1. Attach to the OpenWork renderer

1. Run `chrome-devtools_list_pages`.
2. Select or navigate to the OpenWork renderer, usually:
   - `http://localhost:5173/#/...`
3. Run `chrome-devtools_take_snapshot` and confirm the page title is `OpenWork`.

### 2. Prefer UI actions first

Use snapshots and visible element UIDs:

1. Click `Add workspace`.
2. Click `Local workspace`.
3. Click, fill, or press only visible controls from the latest snapshot.
4. Verify with another snapshot, not assumptions.

## Native Folder Picker Limitation

Chrome MCP cannot reliably control the native OS folder picker opened by `Select folder`.

If the user explicitly provides a folder path and asks to continue through the UI, inject the path into the visible `CreateWorkspaceModal` React state, then click the now-enabled `Create Workspace` button.

### Inject selected folder into the modal

Run this with `chrome-devtools_evaluate_script`, replacing `path` with the explicit folder path:

```js
() => {
  const path = "/absolute/path/to/folder";

  function findFiber(el) {
    const key = Object.keys(el).find((k) => k.startsWith("__reactFiber$"));
    return key ? el[key] : null;
  }

  const placeholder = Array.from(document.querySelectorAll("span")).find(
    (el) => el.textContent?.trim() === "No folder selected yet.",
  );
  if (!placeholder) return { ok: false, reason: "placeholder not found" };

  let fiber = findFiber(placeholder);
  while (fiber && (fiber.elementType?.name || fiber.type?.name) !== "CreateWorkspaceModal") {
    fiber = fiber.return;
  }
  if (!fiber) return { ok: false, reason: "CreateWorkspaceModal fiber not found" };

  const hooks = [];
  let hook = fiber.memoizedState;
  while (hook) {
    hooks.push(hook);
    hook = hook.next;
  }

  const selectedFolderHook = hooks[2];
  const pickingFolderHook = hooks[3];
  if (!selectedFolderHook?.queue?.dispatch) {
    return { ok: false, reason: "selectedFolder dispatch not found" };
  }

  selectedFolderHook.queue.dispatch(path);
  pickingFolderHook?.queue?.dispatch?.(false);

  return { ok: true, dispatchedPath: path };
}
```

After injection:

1. Take a fresh snapshot.
2. Confirm the absolute path is visible where `No folder selected yet.` was.
3. Click `Create Workspace`.
4. Wait for the new workspace name/path to appear in the sidebar or settings panel.

## Verification Checklist

- The new workspace appears in the left sidebar.
- The settings or workspace page shows the expected folder path.
- Console/network errors are checked if the UI stalls.
- If the UI navigates to settings, verify the selected workspace name and root path there.

## Safety Notes

- Only inject a path that the user explicitly requested or that you just created for the test.
- Do not use injection to bypass a security or permission decision.
- Always interact with the UI after injection so the app's normal submit path creates the workspace.

## First-Time Setup (If Not Configured)

1. Start the OpenWork dev renderer, for example from `_repos/openwork/apps/app` with `pnpm dev`.
2. Ensure Chrome MCP can list pages.
3. Navigate Chrome MCP to `http://localhost:5173` if no OpenWork renderer page is open.
