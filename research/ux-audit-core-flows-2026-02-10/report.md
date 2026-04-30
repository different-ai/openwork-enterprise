# OpenWork UI/UX Core-Flow Audit (Docker + Chrome MCP)

Date: 2026-02-10

This is a hands-on UI/UX pass over OpenWork's web UI using the Docker dev stack and Chrome MCP.

## Environment

- Repo (openwork-enterprise) SHA: `56f095d627b38b2365ed1f7fe55550d6f1b31938`
- OpenWork repo (`_repos/openwork`) SHA (base): `017412ca496c7302a6d895d1b8813cbeb5084883` (`origin/dev`)
- Docker context: `colima`

Dev stacks exercised during this audit:

- Baseline stack (used for findings + before screenshots):
  - Web UI: `http://localhost:62077`
  - OpenWork server: `http://localhost:62076`
  - Compose project: `openwork-dev-8550c7fb`
- PR verification stacks (used for after screenshots):
  - `openwork-dev-07c3e4c7` (tip gating)
  - `openwork-dev-bb125c98` (browser quickstart prompt)

## Core Flows Tried

- Start a local OpenWork app/server stack and open `/session`
- Create sessions ("New task") and send messages
- Open model picker and change model
- Navigate: Automations / Skills / Plugins / Apps / Settings
- Add a suggested plugin (opencode-scheduler) and reload engine
- Trigger the "Automate your browser" quickstart

## Findings (UI/UX)

The items below are intentionally written as "issue tickets": repro + impact + fix direction + evidence.

### 1) "Automate your browser" quickstart fails with `Invalid Tool` (model tries to call `ls`)

- Repro:
  - Open `/session`.
  - Click `Automate your browser`.
  - Observe an `Invalid Tool` error early in the run.
- Impact:
  - This is a headline onboarding flow; failing immediately undermines trust and blocks the user.
- Root cause (from session message JSON):
  - The model attempts tool `ls`, but this environment exposes `bash` (not `ls`) for shell commands.
- Fix direction:
  - Make the quickstart prompt tool-safe: explicitly instruct the agent to use `bash` for `ls`.
  - Optional: add a UI hint in the error card that translates `ls -> bash` for recovery.
- Evidence:
  - `research/ux-audit-core-flows-2026-02-10/media/01-automate-browser-invalid-tool.png`
  - PR: https://github.com/different-ai/openwork/pull/523

### 2) Random "Connect Notion MCP" CTA appears across unrelated pages (session/dashboard)

- Repro:
  - Browse Sessions / Automations / Skills / Settings.
  - Observe a floating/rotating CTA prompting Notion MCP connect.
- Impact:
  - Feels like a random interruption; breaks focus on the active task.
  - Adds noise to the navigation mental model (Apps are the place to connect Apps).
- Fix direction:
  - Gate MCP connection tips so they only appear within Apps (Dashboard -> Apps).
  - Keep the Apps page as the canonical place to connect Notion.
- Evidence:
  - `research/ux-audit-core-flows-2026-02-10/media/02-floating-notion-cta.png`
  - `research/ux-audit-core-flows-2026-02-10/media/07-notion-cta-on-automations.png`
  - `research/ux-audit-core-flows-2026-02-10/media/08-settings-advanced-reset-copy.png`
  - PR: https://github.com/different-ai/openwork/pull/522

### 3) Model picker is overwhelming (flat list; weak discovery)

- Repro:
  - Open the model picker in the composer.
  - Observe a very large, mostly ungrouped list.
- Impact:
  - Hard to choose a model confidently; slow scanning; easy to pick the wrong provider/model.
  - Likely causes performance/jank on lower-end devices.
- Fix direction:
  - Add provider grouping + collapsible sections.
  - Add "favorites" / "recent" pinned section and a default filter.
- Evidence:
  - `research/ux-audit-core-flows-2026-02-10/media/03-model-picker-overwhelming.png`

### 4) Reload toast copy is scary + generic ("Stops all active tasks")

- Repro:
  - Install skill-creator on Skills, or add opencode-scheduler on Plugins.
  - Observe reload toast copy includes "Stops all active tasks".
- Impact:
  - Overly alarming language without explaining what "tasks" are affected or whether a safe defer exists.
  - Same copy appears for multiple actions; lacks context and clarity.
- Fix direction:
  - Make copy context-aware ("Reload engine to load new plugins/skills").
  - Clarify what is affected (sessions/runs) and whether reload is queued until idle.
- Evidence:
  - `research/ux-audit-core-flows-2026-02-10/media/05-skill-creator-installed-reload-toast.png`
  - `research/ux-audit-core-flows-2026-02-10/media/06-plugin-added-reload-toast-copy.png`

### 5) Todo/run progress rendering is broken (stray `[` and duplicated counters)

- Repro:
  - Send a message that triggers the agent to emit todos/progress.
  - Observe stray bracket(s) and repeated "N todos" counters interleaved with tool output.
- Impact:
  - High confusion: looks like a rendering bug and makes the run status hard to follow.
- Fix direction:
  - Audit message/part rendering for todo events; ensure JSON or list delimiters never leak to UI.
  - Add a snapshot test for todo rendering with real event payloads.
- Evidence:
  - `research/ux-audit-core-flows-2026-02-10/media/09-stray-todos-bracket-render.png`

### 6) Settings > Advanced reset language remains ambiguous in web/remote mode

- Repro:
  - Go to Settings -> Advanced.
  - Observe reset actions describe "restarts the app".
- Impact:
  - In web mode, "restart the app" is ambiguous (browser tab? remote server? engine?).
  - Risk of destructive actions while exploring.
- Fix direction:
  - Make copy explicit for web mode (what restarts, what data is cleared).
  - Consider hiding desktop-only reset actions on web.
- Evidence:
  - `research/ux-audit-core-flows-2026-02-10/media/08-settings-advanced-reset-copy.png`

## Media Index

- `research/ux-audit-core-flows-2026-02-10/media/01-automate-browser-invalid-tool.png`
- `research/ux-audit-core-flows-2026-02-10/media/02-floating-notion-cta.png`
- `research/ux-audit-core-flows-2026-02-10/media/03-model-picker-overwhelming.png`
- `research/ux-audit-core-flows-2026-02-10/media/04-skills-install-skill-creator-inert.png`
- `research/ux-audit-core-flows-2026-02-10/media/05-skill-creator-installed-reload-toast.png`
- `research/ux-audit-core-flows-2026-02-10/media/06-plugin-added-reload-toast-copy.png`
- `research/ux-audit-core-flows-2026-02-10/media/07-notion-cta-on-automations.png`
- `research/ux-audit-core-flows-2026-02-10/media/08-settings-advanced-reset-copy.png`
- `research/ux-audit-core-flows-2026-02-10/media/09-stray-todos-bracket-render.png`
- `research/ux-audit-core-flows-2026-02-10/media/pr1-after-no-notion-tip-session.png`
- `research/ux-audit-core-flows-2026-02-10/media/pr1-after-no-notion-tip-automations.png`
- `research/ux-audit-core-flows-2026-02-10/media/pr1-after-apps-has-notion-connect.png`
- `research/ux-audit-core-flows-2026-02-10/media/pr2-after-automate-browser-no-invalid-tool.png`
