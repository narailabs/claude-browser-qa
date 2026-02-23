---
name: e2e-test
description: Use when testing a web app end-to-end via Chrome browser. Navigates all screens, discovers interactive elements, verifies functionality, checks console errors and network failures, finds bugs, and automatically launches fix agents inline. Requires Claude in Chrome extension connected.
disable-model-invocation: true
argument-hint: "[url] [--mode smoke|functional|full] [--record] [--no-autofix]"
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

If no URL is provided, ask the user.

## Prerequisites

Before starting, verify Chrome MCP connection:
1. Call `Claude_in_Chrome:tabs_context_mcp` — if it fails, ask user to connect the Chrome extension
2. The target app must be running (e.g., `pnpm dev` on localhost)

## Execution Checklist

Copy and track with TodoWrite:

```
- [ ] Codebase recon: Read CLAUDE.md and explore project structure
- [ ] Setup: Connect to Chrome, navigate to app, take baseline screenshot
- [ ] Discover: Map all screens, navigation links, interactive elements
- [ ] Test each screen: DOM verification, console errors, network failures, screenshots
- [ ] Report: Coverage summary, bugs fixed vs remaining
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

3. **Verify app is reachable**:
   - `Claude_in_Chrome:read_page` — check the page has content (not error/blank)
   - If login/auth wall detected → ask user for credentials via AskUserQuestion
   - `Claude_in_Chrome:computer(action=screenshot)` — baseline screenshot

4. **Optional**: Start GIF recording with `Claude_in_Chrome:gif_creator(action=start_recording)`

5. **Clear console**: `Claude_in_Chrome:read_console_messages(clear=true)` — establish clean baseline

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

### Coverage Tracking

After each screen, count and report:
- Elements found vs elements tested
- Report inline: "Tasks screen: 12/14 elements tested (86%), 1 console warning found"
- Mark TodoWrite item as completed

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

## Phase 4: Report

### Coverage Summary
```
E2E Test Complete
═══════════════════════════════════
Screens: 6/6 tested (100%)
Elements: 45/52 exercised (87%)
Bugs found: 3
  ✅ Fixed inline: 2
  ❌ Still open: 1 (fix failed)

Coverage gaps:
- Settings: API key form skipped (no credentials)
- Tasks: Delete button skipped (destructive, user declined)
```

### Bug Reports

For each bug (whether fixed or still open):
- **Status**: ✅ Fixed / ❌ Still open
- **What**: Brief description
- **Where**: Screen and element
- **Severity**: critical (app broken) / major (feature broken) / minor (cosmetic/warning)
- **Evidence**: Screenshot, console error text, failed network request
- **Fix**: What file/line was changed (if fixed), or why fix failed (if not)

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

## Error Recovery

| Problem | Action |
|---------|--------|
| Page won't load | `computer(action=wait, duration=5)`, retry once, then log failure and move on |
| Element not found | Try `find` with different query, try `read_page` with broader scope |
| Click has no effect | Try clicking by ref, try by coordinate, log if still fails |
| App crashes | Screenshot, capture console, log critical bug, reload and continue |
| Auth required mid-test | Ask user for credentials |
| Modal/dialog blocks | Dismiss it (find close button), then continue |
| Fix agent fails | Log bug as open, continue testing other screens |
| Chrome extension disconnects | Call `tabs_context_mcp` to reconnect, retry last action |

## Testing Modes

**smoke** (default): Navigate each screen → verify DOM + console + network + screenshot. No interactions. Auto-fix any bugs found.

**functional**: smoke + interact with all safe elements, fill forms, click buttons, verify state changes. Auto-fix bugs found.

**full**: functional + test edge cases (empty inputs, special characters, rapid navigation, back/forward, data persistence across views). Auto-fix bugs found.
