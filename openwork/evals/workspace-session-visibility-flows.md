# Workspace session visibility flows

End-to-end scenarios for sidebar session visibility across workspaces on
launch. A workspace the user has not opened in this app run must still show
its real session list in the sidebar — never a false "No tasks yet." empty
state while sessions exist on the server.

Regression source: the workspace-switch coherence fix (#3977) restricted the
boot-time background session load to only the routed workspace, so every other
workspace rendered the empty state until it was opened.

## Preflight

1. Start Electron through Daytona or locally (`pnpm dev`, CDP at
   `http://127.0.0.1:9825` with `OPENWORK_ELECTRON_REMOTE_DEBUG_PORT=9825`).
2. Ensure at least three local workspaces exist, each with at least one
   session. Seed by opening each workspace and sending a short message, then
   fully quit the app.

## Flow 1: Unopened workspaces show sessions on launch

**Goal:** After a cold launch, every workspace in the sidebar lists its
sessions even if the user never opens it.

### Steps

1. Cold-launch Electron (full app start, not a renderer reload).
2. Wait for the boot overlay to dismiss and the sidebar to render.
3. Without clicking into any other workspace, expand each collapsed workspace
   with its chevron ("Expand" button).

### CDP steering

- Use `browser_eval` to find each workspace row and click
  `button[aria-label="Expand"]` — snapshot UIDs are unreliable in the dynamic
  React sidebar.
- Establish ground truth first: read `openwork.server.port` and
  `openwork.server.token` from localStorage, then fetch
  `GET /workspace/<id>/sessions` for every id from `GET /workspaces` and record
  the counts.

### Verification

- Compare each expanded workspace's sidebar entries against the API session
  count captured in ground truth.
- Watch the dev server log: boot must issue
  `GET /workspace/<id>/sessions` for **every** workspace, not just the routed
  one (background loads may be staggered; give them a few seconds).

### Pass criteria

- No workspace with server-side sessions renders "No tasks yet." after boot
  settles.
- Session titles in the sidebar match the API list for each workspace.
- No console error during boot or expand.

## Flow 2: Empty state is honest while loading

**Goal:** A workspace whose session index has not loaded yet shows a loading
state (or nothing), never a definitive "No tasks yet."

### Steps

1. Cold-launch Electron.
2. Immediately (within the first ~2s) expand a non-routed workspace.

### Pass criteria

- Before its session list arrives, the workspace shows no false empty-state
  claim; after it arrives, sessions render without requiring the user to open
  the workspace.
- A workspace with genuinely zero sessions may show "No tasks yet." once its
  load has completed.

## Flow 3: Opening a workspace still refreshes it

**Goal:** The fix must not regress the switch path: opening a workspace still
loads/refreshes its sessions.

### Steps

1. From the launch state, click into a previously unopened workspace.
2. Confirm its sessions render and the route changes to that workspace.

### Pass criteria

- Sessions render after the switch.
- Rapid workspace switching stays coherent (no stale workspace becoming
  active) — see `apps/app/tests/route-refresh-control.test.ts` and the
  workspace-storm e2e spec in the product repo.
