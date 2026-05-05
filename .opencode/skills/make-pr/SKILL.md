---
name: make-pr
description: |
  Create a PR with structured evidence: build verification, UI screenshots uploaded to an ephemeral host, API test results, and a verification comment. No screenshots committed to the repo.

  Triggers when user mentions:
  - "create a pr"
  - "make a pr"
  - "open a pr"
  - "push and pr"
---

# Make PR

Standardized workflow for creating a pull request with evidence. Screenshots go to an ephemeral image host, not the repo.

## 1. Pre-flight checks

Before creating the PR, gather evidence.

### Build verification

Run the build for every changed package:

```bash
# Den web
pnpm --filter @openwork-ee/den-web build

# Den API (check for import errors)
pnpm --filter @openwork-ee/den-db build

# Desktop app
pnpm --filter @openwork/app build
```

Report which builds passed and which failed.

### UI verification (if UI changed)

1. Boot the relevant dev stack (see `den-test-instructions` skill for Den flows)
2. Navigate to the changed screen via Chrome MCP
3. Take screenshots:
   ```
   chrome-devtools_take_screenshot with filePath to /tmp/pr-evidence-<name>.png
   ```
4. Upload screenshots to an ephemeral host -- do NOT commit them to the repo:
   ```bash
   curl -F "file=@/tmp/pr-evidence-<name>.png" https://0x0.st
   ```
   This returns a URL like `https://0x0.st/abc.png`. Use this URL in the PR body.
5. If `0x0.st` is down, try `https://litterbox.catbox.moe/resources/internals/api.php` with `-F "reqtype=fileupload" -F "time=24h" -F "fileToUpload=@file.png"`.
6. If all hosts fail, note the blocker and describe what the screenshot shows in text.

### API verification (if API changed)

1. Boot the dev stack
2. Curl the new/changed endpoints
3. Capture request and response:
   ```bash
   curl -s http://localhost:3005/api/den/v1/your-endpoint | jq
   ```
4. Include the output in the PR verification section.

### Console/network check

After loading the changed UI:
```
chrome-devtools_list_console_messages with types ["error", "warn"]
chrome-devtools_list_network_requests with resourceTypes ["xhr", "fetch"]
```

Report any errors. If clean, say "No console errors, no failed requests."

## 2. Create the PR

Use `gh pr create` with this template structure:

```bash
gh pr create --base dev --title "<type>(<scope>): <description>" --body "$(cat <<'EOF'
## Summary

- <what changed, 1-3 bullets>

## Evidence

### Screenshots
<!-- Upload to 0x0.st, paste URLs here -->
![description](https://0x0.st/xxx.png)

### Build verification
- `pnpm --filter <pkg> build` -- passed/failed

### API verification
<!-- curl output or "no API changes" -->

### UI verification
<!-- "Verified via Chrome MCP" or "no UI changes" -->
- Console errors: none
- Failed network requests: none

## Test instructions

1. `pnpm dev:den`
2. `pnpm dev:den:seed-demo`
3. Sign in as `alex@acme.test` / `OpenWorkDemo123!`
4. Navigate to <path>
5. Verify <what to check>
EOF
)"
```

### PR title convention

Follow the repo's existing commit style:
- `feat(<scope>): <description>` -- new feature
- `fix(<scope>): <description>` -- bug fix
- `chore(<scope>): <description>` -- maintenance

### Base branch

Default to `dev` for `_repos/openwork`. Check with `git log --oneline -1 origin/dev` to confirm it exists.

## 3. Post-PR verification comment

After the PR is created, post a verification comment with e2e test results:

```bash
gh pr comment <number> --repo different-ai/openwork --body "$(cat <<'EOF'
## Verification

<what was tested, step by step, with actual output>
EOF
)"
```

## What NOT to do

- Do NOT commit screenshots (`.png`, `.jpg`) to the repo for PR evidence
- Do NOT skip the build check
- Do NOT create a PR without at least one form of evidence (build, screenshot, API test, or manual description)
- Do NOT put "mocked" or "preview" labels in screenshots meant for client-facing PRs
