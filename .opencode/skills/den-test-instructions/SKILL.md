---
name: den-test-instructions
description: |
  Boot up the Den dev stack and the OpenWork desktop app locally, then test common flows like sign-in, dashboard, and telemetry.

  Triggers when user mentions:
  - "test den"
  - "test the app locally"
  - "boot up den"
  - "den dev flow"
  - "test sign in"
---

# Den Local Test Instructions

## Prerequisites

- Docker running (Colima, Docker Desktop, or equivalent)
- Node 20+, pnpm 10+
- Ports 3005 (den-web), 8790 (den-api), 8789 (den-worker-proxy) free

## 1. Start the Den stack

From the OpenWork repo root (`_repos/openwork`):

```bash
pnpm dev:den
```

This starts MySQL (via Docker), runs schema migrations, and launches three services:
- **den-web** on `http://localhost:3005` (Next.js frontend)
- **den-api** on `http://localhost:8790` (Hono API server)
- **den-worker-proxy** on `http://localhost:8789`

Wait for `den-api listening on 8788` and `Ready in` from den-web before proceeding.

## 2. Seed the demo organization

In a separate terminal, from the same repo root:

```bash
pnpm dev:den:seed-demo
```

This creates:
- **Org:** Acme Robotics (`acme-robotics-demo`)
- **Owner:** `alex@acme.test` / `OpenWorkDemo123!`
- **17 members**, 12 teams, 3 pending invites
- **14 plugins** with skills, MCPs, and config objects
- **1 marketplace** with imported plugin catalog

## 3. Sign in via browser

1. Open `http://localhost:3005`
2. Click "Sign in"
3. Enter `alex@acme.test` and `OpenWorkDemo123!`
4. The OTP verification code is printed to the den-api terminal logs:
   ```
   [auth] dev verification email payload for alex@acme.test: {"verificationCode":"832998"}
   ```
5. Enter the code. You land on the dashboard.

## 4. Start the OpenWork desktop app pointing at local Den

In a separate terminal, from the OpenWork repo root:

```bash
VITE_DEN_BASE_URL=http://localhost:3005 pnpm dev
```

This starts the desktop app with the Den connection pointed at your local stack instead of `https://app.openworklabs.com`.

Available env vars:
- `VITE_DEN_BASE_URL` -- Den web app URL (default: `https://app.openworklabs.com`)
- `VITE_DEN_API_BASE_URL` -- Den API URL directly (optional, auto-derived from base URL)
- `VITE_DEN_REQUIRE_SIGNIN` -- Force sign-in on launch (`1`/`true`/`yes`/`on`)

## 5. Common test flows

### Sign-in flow
1. Open the app or `http://localhost:3005`
2. Sign in with `alex@acme.test` / `OpenWorkDemo123!`
3. Check the den-api terminal for the OTP code
4. Verify you land on the dashboard with "Acme Robotics" in the sidebar

### Dashboard
1. After sign-in, verify the dashboard shows org member count and pending invites
2. Navigate sidebar links: Members, Skill Hubs, Plugins, Billing, Org Settings
3. Verify each page loads without errors

### Telemetry (if the telemetry branch is active)
1. Check the dashboard home shows "OpenWork users" and "Pending invites" from real org data
2. Open the enterprise preview section to see the usage insights template
3. Verify `GET /api/den/v1/telemetry/adoption` returns real data:
   ```bash
   curl -s http://localhost:3005/api/den/v1/telemetry/adoption -H "Cookie: <session-cookie>" | jq
   ```

### Chrome MCP verification
When verifying UI flows with Chrome MCP:
1. Navigate to the page under test
2. Use `chrome-devtools_take_snapshot` to inspect the accessibility tree
3. Use `chrome-devtools_take_screenshot` to capture visual evidence
4. Check `chrome-devtools_list_console_messages` for errors
5. Check `chrome-devtools_list_network_requests` for failed API calls

## 6. Teardown

Stop the dev stack with `Ctrl+C` in the terminal running `pnpm dev:den`.

To stop MySQL and clean up Docker:
```bash
pnpm dev:den:mysql:down
```

## Common Gotchas

- **Port in use:** If port 3005/8790/8789 is already taken, the dev script will error. Kill the existing process or use env overrides: `DEN_WEB_PORT`, `DEN_API_PORT`, `DEN_WORKER_PROXY_PORT`.
- **Docker not running:** `pnpm dev:den` needs Docker for MySQL. Start Colima (`colima start`) or Docker Desktop first.
- **OTP not showing:** The verification code only prints in dev mode (`OPENWORK_DEV_MODE=1`). The dev script sets this automatically.
- **Stale session:** If sign-in stops working, clear cookies for `localhost:3005` in Chrome or use an isolated browser context.
