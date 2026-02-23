---
name: browser-qa
description: Use when testing a web app end-to-end via Chrome browser. Navigates all screens, discovers interactive elements, verifies functionality, checks console errors and network failures, finds bugs, and automatically launches fix agents inline. Also supports targeted workflow testing and bug fix cycles. Requires Claude in Chrome extension connected.
disable-model-invocation: true
argument-hint: "[url] [--mode smoke|functional|full] [--record] [--no-autofix] [--a11y] [--perf] [--responsive] [--skip-auth] [--workflow \"...\"] [--fix \"...\"]"
---

# Intelligent E2E Testing

Systematically explore and test a web application using Chrome MCP browser tools. Auto-discovers screens, interactive elements, and testable surfaces from the DOM — no project-specific configuration needed. **Automatically spawns fix agents for every bug found**, then re-tests to confirm the fix.

**Arguments received**: $ARGUMENTS

Parse the arguments:
- First positional argument = URL to test (e.g., `http://localhost:5173`)
- `--mode smoke|functional|full` = testing depth (default: `smoke`)
- `--record` = enable GIF recording of test session
- `--focus "#route"` = test only a specific route/area
- `--no-autofix` = skip auto-fixing bugs, just report them
- `--a11y` = run accessibility checks on every screen (included automatically in `full` mode)
- `--perf` = run performance awareness checks on every screen (included automatically in `full` mode)
- `--responsive` = test layouts at mobile, tablet, and desktop breakpoints (included automatically in `full` mode)
- `--skip-auth` = skip login even if auth wall is detected; test only public pages
- `--workflow "description"` = test a specific user journey end-to-end. Describe the steps in natural language. Skips broad discovery — only tests the described flow. Implies `--mode functional`.
- `--fix "description"` = reproduce a known bug, fix it, and validate the fix. Describe the bug or paste an error message. Runs a structured reproduce → fix → verify cycle.

`--workflow` and `--fix` are mutually exclusive. If both are provided, ask the user which they intended.

If no URL is provided, ask the user.

**Example invocations:**
```
/browser-qa http://localhost:3000
/browser-qa http://localhost:3000 --mode full
/browser-qa http://localhost:3000 --workflow "register a new user with email and password, then verify the dashboard loads"
/browser-qa http://localhost:3000 --workflow "add item to cart, go to checkout, fill shipping form, submit order"
/browser-qa http://localhost:3000 --fix "clicking save on the settings page throws a TypeError in the console"
/browser-qa http://localhost:3000 --fix "submitting the contact form returns a 500 error"
```

## Prerequisites — MUST CHECK BEFORE ANYTHING ELSE

**CRITICAL: This skill requires the Claude in Chrome browser extension. Do NOT proceed past this section until the extension is confirmed working.**

### Step 1: Check Chrome Extension Connection

Immediately call:
```
Claude_in_Chrome:tabs_context_mcp
```

**If the call SUCCEEDS** (returns tab data): The extension is connected. Proceed to Step 2.

**If the call FAILS** (error, tool not found, "not connected", or any failure):

**STOP. Do not continue.** Present this message to the user:

```
⚠️  Claude in Chrome extension is not connected.

This skill requires the Claude in Chrome browser extension to interact with
web pages. Please install and connect it before running /browser-qa.

Setup steps:
  1. Install the extension from the Chrome Web Store:
     https://chromewebstore.google.com/detail/claude/fcoeoabgfenejglbffodgkkbkcdhcgfn
  2. Open Chrome and grant the extension permissions for the sites you want to test
  3. Make sure Chrome is running and the extension is active
  4. In Claude Code, the extension should auto-connect — if not, try restarting Claude Code

Once connected, run /browser-qa again.
```

Then ask:
```
AskUserQuestion: "The Chrome extension is required but not connected. Would you like to try again?"
Options:
  - "I've connected it — retry now"
  - "Cancel — I'll set it up later"
```

If user retries, call `Claude_in_Chrome:tabs_context_mcp` again. If it still fails, show the setup message again and stop.
If user cancels, end the skill execution — do NOT proceed to any testing phases.

### Step 2: Verify App is Running

The target app must be running (e.g., `pnpm dev` on localhost). This is verified in Phase 1 when navigating to the URL.

## Execution Checklist

Copy and track with TodoWrite:

```
- [ ] Codebase recon: Read CLAUDE.md and explore project structure
- [ ] Setup: Connect to Chrome, navigate to app, detect auth/SPA, take baseline screenshot
- [ ] Discover: Map all screens, navigation links, interactive elements
- [ ] Test each screen: DOM, console, network, visual, functional, accessibility, performance
- [ ] Auto-fix: Fix bugs inline as they are found (with deduplication)
- [ ] Responsive: Re-test key screens at mobile, tablet, and desktop breakpoints
- [ ] Report: Coverage summary, bugs fixed vs remaining, accessibility, performance
```

## Phase 0: Codebase Reconnaissance

Before touching the browser, understand the project so fix agents have proper context.

1. **Read project instructions**: Look for `CLAUDE.md`, `README.md`, or similar project docs in the repo root.
   - Use `Read` tool on `CLAUDE.md` (preferred) or `README.md`
   - Extract: tech stack, directory structure, build/test commands, key patterns

2. **Explore project structure**: Use `Glob` or `Bash(ls)` to understand the layout:
   - Identify frontend source directory (e.g., `src/ui/`, `src/app/`, `client/`, `frontend/`)
   - Identify backend/API source directory (e.g., `src/web/`, `src/api/`, `server/`, `routes/`)
   - Identify component directory patterns (e.g., `components/`, `views/`, `pages/`)
   - Note the framework (React, Svelte, Vue, Angular, etc.) and backend (Express, Fastify, Next.js, etc.)

3. **Build codebase context summary**: Compose a short reference block that will be injected into fix agent prompts later. Format:
   ```
   CODEBASE CONTEXT (from project analysis):
   - Tech stack: [framework] frontend, [backend] server
   - Frontend source: [path to UI/component files]
   - Backend source: [path to routes/API files]
   - Build/test commands: [from CLAUDE.md or package.json]
   - Key patterns: [anything notable from CLAUDE.md — e.g., "ESM imports must end in .js", "uses Zod validation"]
   ```

   Store this in a variable/memory — you'll paste it into every fix agent prompt in Phase 3.5.

4. **Mark TodoWrite item as completed** and proceed to Phase 1.

## Phase 1: Setup

1. **Get URL**: Parse from skill arguments, or ask the user:
   ```
   AskUserQuestion: "What URL should I test?"
   Options: "http://localhost:5173", "http://localhost:3000", "Other"
   ```

2. **Connect to Chrome**:
   - `Claude_in_Chrome:tabs_context_mcp` — get existing tabs
   - `Claude_in_Chrome:tabs_create_mcp` — create a fresh tab
   - `Claude_in_Chrome:navigate` to the app URL

3. **Verify app is reachable and check for auth wall**:
   - `Claude_in_Chrome:read_page` — check the page has content (not error/blank)
   - `Claude_in_Chrome:computer(action=screenshot)` — baseline screenshot

   **Auth wall detection**: Determine if the app requires authentication:
   - Look for: login forms (username/email + password inputs), "Sign in" / "Log in" buttons, OAuth buttons ("Sign in with Google"), redirect to `/login` or `/auth` path
   - `Claude_in_Chrome:find(query="login form, sign in, log in, email, password")` — search for auth-related elements

   **If `--skip-auth` was passed**: Skip login, test only publicly accessible pages. Note auth-gated screens as "skipped (requires auth)" in the coverage report.

   **If auth wall detected** (and `--skip-auth` not set):
   1. Ask user for credentials:
      ```
      AskUserQuestion: "The app requires login. Please provide credentials."
      Options: "I'll type them in the browser myself", "Use test credentials (I'll provide username and password)", "Skip auth — test only public pages"
      ```
   2. **If user provides credentials**:
      - `Claude_in_Chrome:find(query="email or username input")` → click → type username
      - `Claude_in_Chrome:find(query="password input")` → click → type password
      - `Claude_in_Chrome:find(query="sign in button, log in button, submit")` → click
      - `Claude_in_Chrome:computer(action=wait, duration=3)` — wait for auth to complete
      - `Claude_in_Chrome:read_page` — verify login succeeded (look for dashboard content, user name, absence of login form)
      - `Claude_in_Chrome:computer(action=screenshot)` — capture post-login state
      - If login failed (still on login page, error message visible) → ask user to verify credentials
   3. **If user types credentials manually**:
      - `Claude_in_Chrome:computer(action=wait, duration=15)` — give user time to log in
      - Ask user to confirm when done
      - Verify login succeeded as above
   4. **If user says skip auth**: Proceed with testing only publicly accessible pages. Note auth-gated screens as "skipped (requires auth)" in the coverage report.

   **OAuth/SSO flows**: If the app uses OAuth (Google, GitHub, etc.):
   - These typically open a popup or redirect to a third-party domain
   - Ask the user to complete the OAuth flow manually, then confirm when done
   - Do NOT attempt to automate third-party login pages

   **Session persistence**: After successful login:
   - The browser session (cookies/tokens) should persist across `Claude_in_Chrome:navigate` calls
   - If at any point during testing you encounter the login page again (session expired), repeat the auth flow — ask user if credentials are the same or changed
   - Note in the report if session expired during testing

4. **Optional**: Start GIF recording with `Claude_in_Chrome:gif_creator(action=start_recording)`

5. **Clear console**: `Claude_in_Chrome:read_console_messages(clear=true)` — establish clean baseline

6. **Detect SPA vs MPA routing**:

   Determine whether the app uses single-page app (SPA) routing or traditional multi-page (MPA) navigation. This affects how you navigate and wait for content.

   **Detection method**:
   - `Claude_in_Chrome:read_page` — look for clues:
     - Hash-based routes (`#/tasks`, `#/settings`) → SPA (hash routing)
     - Framework-specific root elements (`<div id="root">`, `<div id="app">`, `<div id="__next">`) → likely SPA
     - Full-page `<a href="/about">` links without JavaScript interception → likely MPA
   - Click one navigation link and observe:
     - If the browser tab URL changes but the page does NOT fully reload (no white flash, the shell/header persists) → SPA
     - If the browser fully reloads (white flash, all content re-renders) → MPA
   - `Claude_in_Chrome:read_console_messages` — look for framework boot messages (React, Vue, Svelte, Angular) → confirms SPA

   **Record the result**: Store `APP_TYPE: SPA` or `APP_TYPE: MPA` for use in later phases.

   **Navigation strategy per type**:

   | Behavior | SPA | MPA |
   |----------|-----|-----|
   | Navigate to screen | Click nav links or `Claude_in_Chrome:navigate` — both work | `Claude_in_Chrome:navigate` to full URL |
   | Wait after navigation | `Claude_in_Chrome:computer(action=wait, duration=2)` — wait for client-side render | `Claude_in_Chrome:computer(action=wait, duration=3)` — wait for full page load |
   | Verify page loaded | `Claude_in_Chrome:read_page` — check for new content in the main area | `Claude_in_Chrome:read_page` — check entire page rendered |
   | Console baseline | Console persists across navigations — only clear at start of each screen test | Console resets on each page load — read immediately after load |
   | Network baseline | Network log accumulates — note which requests are new since navigation | Network log resets per page — all requests are relevant |
   | Back/forward testing | `Claude_in_Chrome:navigate("javascript:history.back()")` or browser back — verify SPA handles popstate | Standard back navigation |

## Execution Mode Routing

After Phase 1 setup completes, determine the execution path:

- **If `--workflow` is set**: Go to **Phase W: Workflow Testing**. Skip Phase 2 (Discover) and Phase 3 (Test Each Screen).
- **If `--fix` is set**: Go to **Phase F: Bug Fix Cycle**. Skip Phase 2 (Discover) and Phase 3 (Test Each Screen).
- **Otherwise**: Continue with Phase 2 → Phase 3 as normal (broad sweep testing).

**Flag interactions with targeted modes:**
- `--workflow` and `--fix` both imply `--mode functional` (interactions are inherent)
- `--a11y`: Runs accessibility checks only on screens visited during the workflow/fix (not all screens)
- `--perf`: Checks performance of requests made during the workflow/fix
- `--responsive`: Re-runs the workflow at each breakpoint after desktop succeeds, or verifies the fix works at all breakpoints
- `--no-autofix` + `--workflow`: Valid — test the workflow but only report bugs, don't fix them
- `--no-autofix` + `--fix`: Contradictory — the purpose of `--fix` is to fix. If both provided, ask the user to clarify
- `--focus`: Ignored when `--workflow` or `--fix` is set (they already define scope)

---

## Phase W: Workflow Testing (--workflow)

Test a specific user journey end-to-end. The user describes the workflow in natural language, and you execute it step by step, fixing bugs encountered along the way.

### W.1: Parse the Workflow Description

Analyze the `--workflow` text to extract:
- **Starting point**: Which URL, route, or screen does the workflow begin at?
- **Action steps**: Ordered list of user actions (navigate, click, type, submit, select, scroll, verify)
- **Expected outcomes**: What should happen after each step? (page changes, content appears, redirect occurs)
- **Success criteria**: What does "workflow complete" look like? (final state, confirmation message, redirect to specific page)

If the description is too vague to derive concrete steps:
```
AskUserQuestion: "I need more detail to test this workflow. Can you describe the specific steps?"
Options: "Let me describe the steps", "Just navigate to the starting page and explore"
```

Build a TodoWrite checklist of the extracted steps:
```
- [ ] Step 1: Navigate to /register
- [ ] Step 2: Fill in registration form (name, email, password)
- [ ] Step 3: Click "Create Account"
- [ ] Step 4: Verify redirect to /dashboard
- [ ] Step 5: Verify welcome message is displayed
```

### W.2: Codebase-Informed Step Planning

Use the Phase 0 codebase context to correlate workflow steps with actual code:
- Which routes/components are involved in this workflow?
- Which API endpoints will be hit during form submissions or navigation?
- Are there relevant form validation rules, state management patterns, or middleware?
- What error handling exists along this path?

This helps anticipate where bugs are likely and gives better context to fix agents if issues are found.

### W.3: Execute the Workflow

Follow the **Interaction Stability: Wait-for-Ready Protocol** for every interaction.

For each step in the workflow:

1. **Perform the action**:
   - Navigation: `Claude_in_Chrome:navigate` or click a link/button
   - Form input: `Claude_in_Chrome:find` the field → `Claude_in_Chrome:computer(action=click)` → `Claude_in_Chrome:computer(action=type, text="...")`
   - Button click: `Claude_in_Chrome:find` the button → `Claude_in_Chrome:computer(action=click)`
   - Verification: `Claude_in_Chrome:read_page` to check content

2. **Capture evidence**:
   - `Claude_in_Chrome:computer(action=screenshot)` — screenshot after each step
   - `Claude_in_Chrome:read_console_messages` — check for errors
   - `Claude_in_Chrome:read_network_requests` — check for failures

3. **Verify the expected outcome**:
   - Did the page change as expected?
   - Is the expected content visible?
   - Are there any console errors or network failures?

4. **Handle results**:
   - **Step succeeds** → mark TodoWrite item complete, proceed to next step
   - **Step fails** → record the failure point with full evidence (screenshot, console, network), then go to W.4

### W.4: Fix Bugs Found in Workflow

When a step fails, use the same fix process as Phase 3.5 (spawn a fix agent with evidence and codebase context).

**Key difference from broad testing**: After each fix, **re-run the ENTIRE workflow from step 1** — not just the failed step. This catches regressions where a fix to step 3 might break step 1.

- Maximum **3 full re-runs** of the workflow to prevent infinite loops
- If the same step fails after a fix attempt, mark as unresolved and report
- If a previously-passing step fails after a fix (regression), spawn a fix agent for the regression too
- Track re-run count: "Workflow run 2/3 (after fixing step 3)"

### W.5: Workflow Report

```
Workflow Test: "[workflow description]"
═══════════════════════════════════
Steps: 5/5 passed ✅

Step 1: Navigate to /register ✅
Step 2: Fill in name, email, password ✅
Step 3: Click "Create Account" ✅
  → Bug found: 500 error on POST /api/register
  → Fix applied: Fixed null check in register handler (src/api/auth.ts:42)
  → Re-run 2/3: All steps pass ✅
Step 4: Verify redirect to /dashboard ✅
Step 5: Verify welcome message shown ✅

Bugs found: 1
  ✅ Fixed: POST /api/register 500 error (src/api/auth.ts:42)

Workflow re-runs: 2 (1 fix applied)
Workflow status: PASSING
```

If the workflow still fails after all fix attempts:
```
Workflow status: FAILING at step 3
  ❌ Step 3 "Click Create Account" → POST /api/register returns 500
  Fix attempted but did not resolve the issue.
  See bug report above for details.
```

---

## Phase F: Bug Fix Cycle (--fix)

Reproduce a known bug, fix it, and validate the fix. This is a structured reproduce → evidence → fix → validate → regression check cycle.

### F.1: Understand the Bug

Parse the `--fix` description to extract:
- **What's broken**: Error message, wrong behavior, visual glitch, crash
- **Where it happens**: Which screen, route, page, or element
- **How to trigger it**: User actions that cause the bug (if described)

If reproduction steps are not clear from the description:
```
AskUserQuestion: "Can you describe the exact steps to reproduce this bug?"
Options: "Let me describe the steps", "Just go to the page and look for the error"
```

**Search the codebase** for related code (using Phase 0 context):
- `Grep` for error message text, component names, route handlers, API endpoints mentioned in the bug description
- Identify the likely source files involved
- Build a focused codebase context block for the fix agent (more targeted than the general Phase 0 block)

### F.2: Reproduce the Bug

Navigate to the affected area and attempt to trigger the bug:

1. `Claude_in_Chrome:navigate` to the relevant route
2. Follow the reproduction steps (click, type, submit, etc.)
3. After each action:
   - `Claude_in_Chrome:computer(action=screenshot)` — capture state
   - `Claude_in_Chrome:read_console_messages` — check for the reported error
   - `Claude_in_Chrome:read_network_requests` — check for failures
4. **Goal**: Confirm the bug exists by capturing concrete evidence that matches the description

**If the bug does NOT reproduce**:
- Try alternative reproduction paths (different test data, different app state, different navigation order)
- Check console for errors that might be related but have a different message
- Ask the user:
  ```
  AskUserQuestion: "I wasn't able to reproduce this bug. Can you provide more specific steps?"
  Options: "Let me provide more detail", "Try again with different data", "The bug might be intermittent — skip reproduction"
  ```
- If still can't reproduce after 2 attempts → report as "not reproducible" and stop

### F.3: Gather Evidence

Once the bug is reproduced, collect all evidence into a structured package:
- **Screenshots**: Before, during, and after the bug manifests
- **Console errors**: Exact error text, stack trace if available
- **Network failures**: URL, HTTP method, status code, response body excerpt if available
- **DOM state**: Relevant element attributes, missing elements, unexpected content
- **Reproduction steps**: Precise, numbered, as performed in F.2

### F.4: Fix the Bug

Spawn a fix agent (same as Phase 3.5) with:
- The full evidence package from F.3
- The focused codebase context from F.1 (specific file paths and relevant code)
- Clear success criteria: "After the fix, performing these reproduction steps should complete without the error/issue"

```
Task(
  subagent_type: "general-purpose",
  description: "Fix [brief bug description]",
  prompt: """
    Fix this bug reported by a user:

    BUG: [description from --fix]
    REPRODUCED: Yes — confirmed in browser

    EVIDENCE:
    - Console error: [exact text]
    - Network failure: [URL + status, or 'none']
    - Visual issue: [describe what's wrong]
    - Reproduction steps:
      1. [step]
      2. [step]
      3. [observe bug]

    LIKELY SOURCE FILES:
    - [files identified in F.1]

    [Paste CODEBASE CONTEXT from Phase 0]

    INSTRUCTIONS:
    1. Read the project's CLAUDE.md for conventions
    2. Read the likely source files above
    3. Find and fix the root cause
    4. Run typecheck/lint to verify no new errors
    5. Do NOT run the dev server or browser
    6. Report: root cause, fix applied, files changed
  """
)
```

### F.5: Validate the Fix

After the fix agent completes:
1. **Reload the page**: `Claude_in_Chrome:navigate` to the affected route
2. **Re-run exact reproduction steps** from F.2
3. **Check for the specific error**: Is the console error gone? Does the network request succeed? Does the UI behave correctly now?
4. **Screenshot**: Capture the fixed state as proof

**If fix works**: Proceed to F.6 (regression check)

**If fix fails**:
- Provide the fix agent's changes + the still-failing evidence to a second fix agent with additional context
- Maximum **2 fix attempts** total
- If still failing after 2 attempts → report as "fix attempted, still broken" with details of what was tried

### F.6: Regression Check

Run a quick smoke test on the affected screen to ensure the fix didn't break anything:
- **DOM verification**: `Claude_in_Chrome:read_page` — page still renders correctly
- **Console errors**: `Claude_in_Chrome:read_console_messages` — no new errors introduced
- **Network failures**: `Claude_in_Chrome:read_network_requests` — no new failures
- **Visual screenshot**: `Claude_in_Chrome:computer(action=screenshot)` — page looks correct

If Phase 0 identified other screens that share code with the fixed area (same component, same API endpoint), navigate to those screens and run a quick smoke check too.

Report any new issues introduced by the fix as regressions.

### F.7: Fix Report

```
Bug Fix: "[bug description from --fix]"
═══════════════════════════════════
Status: ✅ FIXED

Reproduction:
  1. Navigate to /settings
  2. Change the "Theme" dropdown to "Dark"
  3. Click "Save"
  → TypeError: Cannot read properties of undefined (reading 'theme')

Root cause: Settings update handler accessed `user.preferences.theme`
but `user.preferences` was null for users who never set preferences.

Fix applied:
  File: src/api/settings.ts:47
  Change: Added null coalescing — `user.preferences?.theme ?? 'light'`

Validation:
  ✅ Reproduction steps now complete without error
  ✅ Settings save successfully
  ✅ Smoke test on /settings — no regressions

Files changed:
  - src/api/settings.ts (line 47)
```

If the fix failed:
```
Status: ❌ FIX ATTEMPTED, STILL BROKEN

Reproduction: [confirmed — bug reproduces]
Fix attempts: 2
  Attempt 1: [what was tried, why it didn't work]
  Attempt 2: [what was tried, why it didn't work]

Recommendation: [suggestion for manual investigation]
```

If the bug was not reproducible:
```
Status: ⚠️ NOT REPRODUCIBLE

Attempted reproduction:
  1. [steps tried]
  2. [steps tried]
  Result: Bug did not manifest. Page behaved as expected.

Possible explanations:
  - Bug may be intermittent or timing-dependent
  - Bug may require specific data/state not present in current environment
  - Bug may have been fixed by a recent change
```

---

## Phase 2: Discover

Map the app's testable surface automatically:

1. **Find navigation**:
   - `Claude_in_Chrome:read_page` — look for nav elements, sidebar, header links
   - `Claude_in_Chrome:find(query="navigation links")` — find all nav items
   - Record each link's text and href/route

2. **Find interactive elements** (per visible screen):
   - `Claude_in_Chrome:read_page(filter=interactive)` — catalog buttons, inputs, selects, checkboxes
   - Count total interactive elements

3. **Build test plan**: Create a TodoWrite entry per discovered screen:
   ```
   - [ ] Screen: Tasks (#tasks) — 14 interactive elements
   - [ ] Screen: Agents (#agents) — 8 interactive elements
   - [ ] Screen: Settings (#settings) — 6 interactive elements
   ```

4. **Report to user**: "Found N screens with M total interactive elements. Starting testing."

## Interaction Stability: Wait-for-Ready Protocol

Before interacting with ANY element during testing (click, type, submit), follow this stability protocol. This prevents false failures caused by interacting with elements that are still loading or animating.

### Before Every Interaction

1. **Wait for page to be ready**:
   - `Claude_in_Chrome:read_page` — check if the page content has fully rendered
   - Look for loading indicators: spinners, skeleton screens, "Loading..." text, progress bars, shimmer/placeholder elements
   - If loading indicators are present: `Claude_in_Chrome:computer(action=wait, duration=2)` then re-check (up to 3 times, max 6 seconds total)
   - If still loading after 6 seconds → log as performance observation and proceed anyway

2. **Verify element is visible and stable**:
   - `Claude_in_Chrome:find(query="[element description]")` — confirm the element exists in the DOM
   - `Claude_in_Chrome:computer(action=screenshot)` — visually confirm the element is visible on screen (not hidden behind a modal, not scrolled out of view, not covered by another element)
   - If the element is below the fold: scroll to it first

3. **Wait for animations to settle**:
   - If the previous action triggered an animation (page transition, accordion expanding, dropdown opening), wait briefly: `Claude_in_Chrome:computer(action=wait, duration=1)`
   - Then take a screenshot to confirm the UI has settled before proceeding

### After Every Interaction

1. **Wait for response**:
   - `Claude_in_Chrome:computer(action=wait, duration=1)` — brief pause for UI to update
   - `Claude_in_Chrome:read_page` — check if the page content changed as expected
   - If a form submission or button click should trigger a network request: `Claude_in_Chrome:read_network_requests` to confirm the request was made

2. **Check for new loading states**:
   - If the interaction triggered a loading spinner or skeleton screen, wait for it to resolve (same protocol as step 1 above)

3. **Detect toasts/notifications**:
   - After form submissions or actions, look for success/error toast notifications
   - `Claude_in_Chrome:computer(action=screenshot)` — capture any transient notifications before they disappear
   - Note toast content as evidence in bug reports or as confirmation of successful action

### Skeleton Screen Detection

Skeleton screens (grey placeholder boxes mimicking content layout) indicate content is still loading. Common patterns:
- Rectangular grey blocks where text should be
- Circular grey blocks where avatars should be
- Pulsing/shimmer animation on placeholder elements

When detected: wait and re-check. Do NOT interact with or test elements that are still showing skeleton placeholders — the real content has not loaded yet.

## Phase 3: Test Each Screen

For each screen, mark its TodoWrite item as `in_progress`, then run all verification layers.

### Layer 1: DOM Verification
- `Claude_in_Chrome:navigate` to the screen's route
- `Claude_in_Chrome:read_page` — verify page has meaningful content (not empty/error)
- `Claude_in_Chrome:find` — check for expected heading, main content area
- If page appears blank or broken → log as bug, take screenshot, continue

### Layer 2: Console Errors
- `Claude_in_Chrome:read_console_messages(pattern="error|Error|warn|Warning|exception", clear=true)`
- Any errors found → log as bugs with the error text
- Warnings → log as minor issues

### Layer 3: Network Failures
- `Claude_in_Chrome:read_network_requests`
- Check for any 4xx or 5xx status codes
- Check for failed/aborted requests
- Log failures as bugs with URL and status

### Layer 4: Visual Evidence
- `Claude_in_Chrome:computer(action=screenshot)` — capture current state
- Visually assess: does the page look correct? Any obvious rendering issues?

### Layer 5: Functional Testing (--mode functional or full)

**Important**: Before each interaction below, follow the **Interaction Stability: Wait-for-Ready Protocol** above. Do not click or type until the element is confirmed visible and the page is stable.

For each interactive element on the screen:

1. **Forms**: Find inputs → fill with reasonable test data → submit
   - Text inputs: use descriptive test values ("Test Task", "test@example.com")
   - Selects: choose first non-default option
   - Checkboxes: toggle them
   - After submission: verify state changed (read_page again)

2. **Buttons**:
   - **Safe buttons** (navigation, toggle, filter): click and verify outcome
   - **Destructive buttons** (delete, remove, reset): ASK USER FIRST
     ```
     AskUserQuestion: "Found a 'Delete All' button. Should I test it?"
     Options: "Yes, test it", "Skip it"
     ```
   - **External action buttons** (connect, send, publish): ASK USER FIRST

3. **Search/Filter**: If search exists, type a query, verify results update

4. **After each interaction**:
   - Check console for new errors
   - Verify no network failures
   - Take screenshot if state changed

### Layer 6: Accessibility Checks (--a11y or --mode full)

Run these WCAG-informed checks on every screen. Use DOM inspection — no external tools needed.

1. **Images without alt text**:
   - `Claude_in_Chrome:find(query="images")` — find all `<img>` elements
   - `Claude_in_Chrome:read_page` — inspect each image element for `alt` attribute
   - Missing or empty `alt=""` on non-decorative images → log as accessibility bug (severity: major)

2. **Form labels**:
   - `Claude_in_Chrome:read_page(filter=interactive)` — find all form inputs
   - For each `<input>`, `<select>`, `<textarea>`: verify it has an associated `<label>` (via `for` attribute or wrapping), or an `aria-label` / `aria-labelledby` attribute
   - Unlabeled form controls → log as accessibility bug (severity: major)

3. **Heading hierarchy**:
   - `Claude_in_Chrome:find(query="headings")` — find all `<h1>` through `<h6>` elements
   - Check that headings do not skip levels (e.g., `<h1>` followed by `<h3>` with no `<h2>`)
   - Check that there is exactly one `<h1>` per page
   - Violations → log as accessibility bug (severity: minor)

4. **ARIA landmarks and attributes**:
   - `Claude_in_Chrome:read_page` — check for presence of `<main>`, `<nav>`, `<header>`, `<footer>` or equivalent `role` attributes
   - Check interactive custom elements (divs with click handlers) for `role`, `tabindex`, and `aria-*` attributes
   - Clickable `<div>` or `<span>` without `role="button"` and `tabindex` → log as accessibility bug (severity: major)

5. **Keyboard navigability** (--mode full only):
   - Use `Claude_in_Chrome:computer(action=type, text="[Tab]")` to tab through interactive elements
   - Verify focus moves visibly through all interactive elements in a logical order
   - `Claude_in_Chrome:computer(action=screenshot)` — check that a visible focus indicator appears on the focused element
   - If focus gets trapped or skips important elements → log as accessibility bug (severity: critical)
   - Test that pressing Enter/Space activates the focused button or link

6. **Focus management after interactions**:
   - After modal dialogs open: verify focus moves into the modal
   - After modal dialogs close: verify focus returns to the trigger element
   - After in-page navigation: verify focus moves to the new content area
   - Focus not managed properly → log as accessibility bug (severity: major)

7. **Color contrast (best-effort)**:
   - `Claude_in_Chrome:computer(action=screenshot)` — visually inspect the screenshot for text that appears very light on a light background or very dark on a dark background
   - Flag any text that appears difficult to read due to low contrast
   - Log as accessibility bug (severity: minor) with a note: "Visual inspection — confirm with a contrast checker tool"

Accessibility bugs should be reported in the Phase 4 report with an `[A11Y]` prefix and reference the relevant WCAG criterion where possible (e.g., "WCAG 2.1 SC 1.1.1: Non-text Content").

### Layer 7: Performance Awareness (--perf or --mode full)

These are observational checks using the Chrome MCP tools — not a full profiling suite, but useful for catching obvious problems.

1. **Slow network requests**:
   - `Claude_in_Chrome:read_network_requests` — review all requests made during page load
   - Flag any request that took longer than 3 seconds as a performance warning
   - Flag any request that took longer than 10 seconds as a performance bug (severity: minor)
   - Report the URL, method, and duration

2. **Large response payloads**:
   - From the same network request data, check response sizes
   - Flag any single response larger than 1 MB as a performance warning
   - Flag any single response larger than 5 MB as a performance bug (severity: minor)
   - Pay special attention to API/JSON responses — large JSON payloads often indicate missing pagination or over-fetching

3. **Excessive DOM size**:
   - `Claude_in_Chrome:read_page` — assess the overall page complexity
   - If the page DOM appears extremely deep (heavily nested structures) or contains an unusually large number of elements (e.g., hundreds of repeated rows without virtualization), flag as a performance observation
   - Note: this is a qualitative assessment since exact DOM node counts are not available — flag only obvious cases

4. **Too many network requests**:
   - `Claude_in_Chrome:read_network_requests` — count total requests per page load
   - More than 50 requests on a single page → flag as a performance warning
   - More than 100 requests → flag as a performance bug (severity: minor)
   - Note duplicate or redundant requests to the same endpoint

5. **Uncompressed or uncached assets**:
   - From network request data, look for large JS/CSS bundles
   - Note if the same resources appear to be re-fetched on navigation (no caching)

Performance findings should be reported in Phase 4 under a separate **Performance Observations** section. These are advisory — they should NOT trigger auto-fix agents (performance optimizations are too complex for automated fixes). Severity is always `minor` or `observation`.

### Coverage Tracking

After each screen, count and report:
- Elements found vs elements tested
- Report inline: "Tasks screen: 12/14 elements tested (86%), 1 console warning found"
- Mark TodoWrite item as completed

### Bug Deduplication

Maintain a running bug registry throughout testing to avoid reporting the same issue multiple times.

**Deduplication key**: Each bug is identified by a composite key of:
- `error_signature`: The core error message text (strip variable parts like timestamps, IDs, or line numbers — keep the constant portion)
- `bug_type`: One of `console_error`, `network_failure`, `dom_issue`, `visual_issue`, `functional_issue`, `a11y_issue`, `perf_issue`, `responsive_issue`

**Before logging any new bug**:
1. Compute its deduplication key
2. Check if a bug with the same key already exists in your registry
3. If **duplicate found**: Do NOT create a new bug entry. Instead, append the current screen/route to the existing bug's `affected_screens` list
4. If **new bug**: Create a new entry with `affected_screens: [current_screen]`

**Bug registry format** (maintain in memory):
```
BUG REGISTRY:
- BUG-001: {key: "TypeError: Cannot read properties of undefined", type: console_error, severity: major, affected_screens: ["#tasks", "#agents", "#settings"], first_seen: "#tasks"}
- BUG-002: {key: "GET /api/users 404", type: network_failure, severity: major, affected_screens: ["#dashboard"], first_seen: "#dashboard"}
```

**Auto-fix deduplication**: Only spawn a fix agent for a bug the FIRST time it is encountered. When the same bug appears on subsequent screens, skip the fix agent — the fix from the first occurrence (if successful) will resolve all instances.

**Reporting deduplication** (Phase 4): In the final report, list each unique bug once with all its affected screens, not once per screen. Format:
```
BUG-001: TypeError: Cannot read properties of undefined (reading 'map')
  Severity: major | Type: console_error
  Affected screens: #tasks, #agents, #settings (3 screens)
  Status: Fixed (fix applied when first found on #tasks)
```

---

## Phase 3.5: Auto-Fix Bugs (Inline, Per Bug Found)

**This runs immediately when any bug is confirmed — do not wait until Phase 4.**

Unless `--no-autofix` was passed, for each bug discovered:

### Step 1 — Gather Evidence
Before spawning a fix agent, collect:
- Screenshot showing the problem
- Exact console errors (copy text)
- Failed network request URL + status
- Exact reproduction steps
- Which screen/element is affected

### Step 2 — Spawn Fix Agent

Use the `Task` tool with subagent type `general-purpose`:

```
Task(
  subagent_type: "general-purpose",
  description: "Fix [brief bug name]",
  prompt: """
    Fix this bug found during e2e testing of the web app at [URL].

    BUG: [description]
    SCREEN: [which route/view]
    ELEMENT: [which UI element]
    SEVERITY: [critical/major/minor]

    EVIDENCE:
    - Console error: [paste exact error text, or 'none']
    - Network failure: [URL + status, or 'none']
    - Visual issue: [describe what looks wrong]
    - Reproduction steps:
      1. Navigate to [route]
      2. [action]
      3. [what you observe] → expected: [what should happen]

    [Paste the CODEBASE CONTEXT block you built in Phase 0 here]

    INSTRUCTIONS:
    1. Read the project's CLAUDE.md (if it exists) for project conventions
    2. Use superpowers:systematic-debugging to find root cause first
    3. Read the relevant source files
    4. Apply the minimal correct fix
    5. Run the project's typecheck/lint command (from CLAUDE.md or package.json) to verify no errors
    6. Do NOT run the dev server or take browser actions
    7. Report what you found and fixed (file, line numbers, change made)
  """
)
```

### Step 3 — Wait for Agent Result

The fix agent runs in the foreground (default). When it returns:
- Read its result summary
- Note which files it changed

### Step 4 — Re-Test to Verify Fix

After the fix agent completes, verify the fix worked:
1. Reload the page: `Claude_in_Chrome:navigate` back to the affected route
2. Repeat the exact reproduction steps that revealed the bug
3. Check console and network for the specific error
4. Take a screenshot to confirm correct state

**If fix worked**: Mark bug as ✅ Fixed in your running bug list
**If fix failed**: Log as ❌ Fix Attempted, Still Broken — move on, report at end

### Step 5 — Continue Testing

Resume testing where you left off (next element or next screen).

---

## Phase 3.7: Responsive / Viewport Testing (--responsive or --mode full)

Re-test discovered screens at common breakpoints to find layout issues.

### Breakpoints
| Name    | Width   |
|---------|---------|
| Mobile  | 375px   |
| Tablet  | 768px   |
| Desktop | 1280px  |

### Procedure

For each breakpoint, starting with mobile (most likely to reveal issues):

1. **Resize viewport**:
   - `Claude_in_Chrome:computer(action=resize, width=375, height=812)` (mobile)
   - If the `resize` action is not available, use `Claude_in_Chrome:navigate` with a viewport parameter, or note in the report that viewport resizing was not possible and skip this phase

2. **Re-test each screen** (abbreviated — not full Layer 1-7):
   - `Claude_in_Chrome:navigate` to the screen's route
   - `Claude_in_Chrome:computer(action=screenshot)` — capture the layout at this viewport
   - `Claude_in_Chrome:read_page` — check for content that should be visible

3. **Check for layout issues** by inspecting the screenshot:
   - **Horizontal overflow**: content extending beyond the viewport width (horizontal scrollbar)
   - **Overlapping elements**: text or buttons overlapping each other
   - **Missing elements**: navigation items, buttons, or content that was present at desktop but is missing and not accessible via a mobile menu
   - **Unreadable text**: text too small to read at the viewport width
   - **Broken images**: images that overflow their containers or become distorted
   - **Touch target size**: buttons or links that appear too small to tap (less than ~44x44px)

4. **Check responsive navigation**:
   - If desktop had a full nav bar, verify mobile has a hamburger menu or equivalent
   - If a hamburger menu exists, click it and verify it works
   - `Claude_in_Chrome:computer(action=click)` on the menu toggle
   - `Claude_in_Chrome:computer(action=screenshot)` — verify menu opened

5. **Log responsive bugs** with the breakpoint noted:
   - Format: "[Mobile 375px] Header text overlaps navigation on Tasks screen"
   - Severity: `major` for broken functionality, `minor` for cosmetic issues

6. **Repeat for each breakpoint**, then restore desktop viewport:
   - `Claude_in_Chrome:computer(action=resize, width=1280, height=900)`

Responsive bugs SHOULD trigger auto-fix agents (via Phase 3.5 process) since they are typically CSS fixes. Include the breakpoint and a screenshot at the affected viewport size in the fix agent prompt.

---

## Phase 4: Report

### Coverage Summary
```
E2E Test Complete
═══════════════════════════════════
App type: SPA / MPA
Auth: Logged in as [user] / No auth / Skipped
Screens: 6/6 tested (100%)
Elements: 45/52 exercised (87%)

Bugs found: 5 unique (across 8 total occurrences)
  ✅ Fixed inline: 3
  ❌ Still open: 2

Accessibility issues: 4
  3 major (missing labels, no keyboard access)
  1 minor (heading hierarchy)

Performance observations: 2
  1 slow request (>3s), 1 large payload (>1MB)

Responsive issues: 1
  1 major at mobile 375px (nav overflow)

Coverage gaps:
- Settings: API key form skipped (no credentials)
- Tasks: Delete button skipped (destructive, user declined)
```

### Bug Reports

For each bug (whether fixed or still open), list each unique bug once with all affected screens:
- **Status**: ✅ Fixed / ❌ Still open
- **What**: Brief description
- **Where**: Affected screens (list all, from bug deduplication registry)
- **Severity**: critical (app broken) / major (feature broken) / minor (cosmetic/warning)
- **Evidence**: Screenshot, console error text, failed network request
- **Fix**: What file/line was changed (if fixed), or why fix failed (if not)

### Accessibility Report (if --a11y or --mode full)

For each accessibility issue:
- **Issue**: Description (e.g., "Image missing alt text")
- **WCAG Criterion**: Reference (e.g., "1.1.1 Non-text Content")
- **Where**: Screen and element
- **Severity**: critical / major / minor
- **Status**: ✅ Fixed / ❌ Open

### Performance Observations (if --perf or --mode full)

For each performance finding:
- **Observation**: Description (e.g., "GET /api/tasks took 4.2 seconds")
- **Where**: Screen and request/element
- **Recommendation**: Brief suggestion (e.g., "Consider pagination for this endpoint")
- Note: Performance observations do NOT trigger auto-fix agents

### Responsive Issues (if --responsive or --mode full)

For each responsive issue:
- **Breakpoint**: Mobile 375px / Tablet 768px / Desktop 1280px
- **Issue**: Description with screenshot reference
- **Where**: Screen
- **Severity**: major / minor
- **Status**: ✅ Fixed / ❌ Open

### Next Steps (for remaining open bugs)
```
AskUserQuestion: "N bugs remain unfixed. What would you like to do?"
Options:
  "Try fixing them again" — respawn fix agents with more context
  "Just log them" — done, I'll fix later
  "Re-test a specific screen" — re-run testing on one screen
```

## When to Ask the User

**ALWAYS ask before:**
- Clicking destructive buttons (delete, remove, reset, clear)
- Clicking external-action buttons (send, publish, connect, deploy)
- Filling forms that need real credentials (API keys, tokens, passwords)
- Any action that could modify external state
- Completing OAuth/SSO login flows (ask user to do it manually)

**Ask once then remember:**
- Auth credentials for login
- Whether to skip or test destructive actions (applies to all similar buttons)

**Never ask about:**
- Navigation between screens
- Reading page content
- Taking screenshots
- Checking console/network
- Filling forms with obviously test data (names, descriptions)
- Spawning fix agents for bugs (do this automatically)
- Running accessibility checks
- Running performance checks
- Resizing viewport for responsive testing
- Retrying failed interactions (follow retry sequence automatically)

## Fix Agent Guidelines

**Which subagent to use**: `general-purpose` for most bugs (has all tools). Use a second `general-purpose` agent for complex multi-file issues if the first attempt fails.

**What makes a good fix agent prompt**:
- Exact error message or visual symptom (not vague descriptions)
- The reproduction steps — precise, numbered
- The codebase context block from Phase 0 — so it knows where to look
- Instruction to read CLAUDE.md and use `superpowers:systematic-debugging` so it finds root cause
- Clear success criteria: what "fixed" looks like

**What fix agents should NOT do**:
- Start the dev server
- Run browser tests
- Make UI changes to workaround rather than fix root cause
- Modify tests to pass around bugs

**If fix agent returns with uncertainty**: Provide more context and try again once. If still failing, mark as open.

## Error Recovery & Retry Strategy

### Structured Retry Logic for Interactions

When any element interaction (click, type, submit) fails, follow this escalation sequence before logging a failure:

**Click retry sequence** (try each step, move to next on failure):
1. **Retry by ref**: `Claude_in_Chrome:computer(action=click)` using the element's ref — wait 2 seconds, try once more
2. **Retry after wait**: `Claude_in_Chrome:computer(action=wait, duration=3)` then retry — the element may still be loading or animating
3. **Retry by find**: `Claude_in_Chrome:find(query="[button text or label]")` to re-locate the element, then click the newly found ref
4. **Retry by coordinate**: `Claude_in_Chrome:computer(action=screenshot)` → visually identify the element's position → `Claude_in_Chrome:computer(action=click, coordinate=[x, y])` using pixel coordinates
5. **Log failure**: If all 4 attempts fail, log as a bug: "Element '[name]' not interactable on [screen]" with a screenshot

**Type/input retry sequence**:
1. **Click the field first**: `Claude_in_Chrome:computer(action=click)` on the input, then `Claude_in_Chrome:computer(action=type, text="...")`
2. **Clear and retype**: Click the field, select all (`Ctrl+A`), then type the value
3. **Find and retry**: `Claude_in_Chrome:find(query="[input label or placeholder]")` to re-locate, click, then type
4. **Log failure**: If all attempts fail, log as a bug

**Navigation retry sequence**:
1. **Wait and retry**: `Claude_in_Chrome:computer(action=wait, duration=5)` then `Claude_in_Chrome:navigate` again
2. **Check if redirected**: `Claude_in_Chrome:read_page` — maybe the page loaded but at a different URL (redirect)
3. **Hard reload**: `Claude_in_Chrome:navigate` to the exact URL one more time
4. **Log failure**: Screenshot, capture console errors, mark screen as inaccessible, continue to next screen

### General Error Recovery Table

| Problem | Action |
|---------|--------|
| Page won't load | Follow navigation retry sequence above |
| Element not found | Follow click retry sequence above |
| Click has no effect | Follow click retry sequence above |
| App crashes | Screenshot, capture console, log critical bug, `Claude_in_Chrome:navigate` to reload, continue |
| Auth required mid-test | Ask user for credentials via AskUserQuestion |
| Modal/dialog blocks | `Claude_in_Chrome:find(query="close button, dismiss, X button")` → click to dismiss. If no close button found, press Escape: `Claude_in_Chrome:computer(action=type, text="[Escape]")` |
| Fix agent fails | Log bug as open, continue testing other screens |
| Chrome extension disconnects | `Claude_in_Chrome:tabs_context_mcp` to reconnect. If that fails, ask user to reconnect extension |
| JavaScript alert/confirm dialog | `Claude_in_Chrome:computer(action=click)` on OK/Accept to dismiss |
| Infinite loading | `Claude_in_Chrome:computer(action=wait, duration=10)`, screenshot, if still loading log as a bug and navigate away |

### Maximum Retry Limits

- Per-element interaction retries: 4 attempts max (as described in sequences above)
- Per-page navigation retries: 3 attempts max
- Total test session: if more than 5 consecutive failures occur across different screens, pause and ask the user if they want to continue

## Testing Modes

**smoke** (default): Navigate each screen → verify DOM + console + network + screenshot. No interactions. Auto-fix any bugs found. Bug deduplication active.

**functional**: smoke + interact with all safe elements, fill forms, click buttons, verify state changes. Auto-fix bugs found. Interaction stability protocol active.

**full**: functional + edge cases (empty inputs, special characters, rapid navigation, back/forward, data persistence across views) + accessibility checks (Layer 6) + performance awareness (Layer 7) + responsive viewport testing (Phase 3.7). Auto-fix bugs found.

**workflow**: Test a specific user journey. `--workflow "description"`. Skips broad discovery. Parses the description into steps, executes them sequentially, fixes bugs encountered, re-runs entire workflow from step 1 after each fix to catch regressions. Report shows step-by-step pass/fail status.

**fix**: Reproduce and fix a known bug. `--fix "description"`. Skips broad discovery. Structured cycle: understand → reproduce → gather evidence → fix → validate → regression check. Up to 2 fix attempts. Report shows fix status, root cause, and validation results.

Individual layers can also be enabled independently via flags: `--a11y`, `--perf`, `--responsive`.
