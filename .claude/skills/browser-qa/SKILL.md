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
- `--a11y` = run accessibility checks (included in `full` mode)
- `--perf` = run performance checks (included in `full` mode)
- `--responsive` = test at mobile/tablet/desktop breakpoints (included in `full` mode)
- `--skip-auth` = skip login, test only public pages
- `--workflow "description"` = test a specific user journey. Implies `--mode functional`. Skips broad discovery.
- `--fix "description"` = reproduce and fix a known bug. Runs reproduce → fix → verify cycle.

`--workflow` and `--fix` are mutually exclusive. If both provided, ask which was intended.
If no URL provided, ask the user.

**Example invocations:**
```
/browser-qa http://localhost:3000
/browser-qa http://localhost:3000 --mode full
/browser-qa http://localhost:3000 --workflow "register user, then verify dashboard loads"
/browser-qa http://localhost:3000 --fix "clicking save on settings throws TypeError"
```

## Prerequisites

**CRITICAL: Do NOT proceed until the Chrome extension is confirmed working.**

Call `Claude_in_Chrome:tabs_context_mcp` immediately.

**If it succeeds**: Proceed.

**If it fails**: Stop. Show the user:
```
Chrome extension not connected. Install from:
https://chromewebstore.google.com/detail/claude/fcoeoabgfenejglbffodgkkbkcdhcgfn

Setup: Install → grant permissions → ensure Chrome is running → restart Claude Code if needed.
```
Ask if user wants to retry or cancel. If cancel → end execution.

## Phase 1: Codebase Reconnaissance

Before opening the browser, understand the project so fix agents have context.

1. Read project docs (`CLAUDE.md`, `README.md`) for tech stack, conventions, build/test commands
2. Explore project structure to identify frontend/backend source directories, component patterns, framework

3. Build a **CODEBASE CONTEXT** block for fix agent prompts:
   ```
   CODEBASE CONTEXT:
   - Tech stack: [framework] frontend, [backend] server
   - Frontend source: [path]
   - Backend source: [path]
   - Build/test commands: [from docs]
   - Key patterns: [notable conventions]
   ```
   Store this for injection into all fix agent prompts.

## Phase 2: Setup

1. **Connect to Chrome**: `tabs_context_mcp` → `tabs_create_mcp` → `navigate` to app URL

2. **Verify app reachable**: `read_page` + screenshot. If blank/error → app may not be running, notify user.

3. **Auth wall detection**: Look for login forms, "Sign in" buttons, OAuth buttons, `/login` redirects.

   - **`--skip-auth` set**: Test only public pages. Note auth-gated screens as "skipped" in report.
   - **Auth wall detected**:
     - Ask user: provide test credentials, type them in browser manually, or skip auth
     - If credentials provided: fill login form → submit → wait → verify login succeeded
     - If OAuth/SSO: ask user to complete manually, then confirm
     - After login, session persists via cookies/tokens. If session expires mid-test, re-auth.

4. **Optional**: If `--record`, start GIF recording via `gif_creator(action=start_recording)`

5. **Clear console baseline**: `read_console_messages(clear=true)`

6. **Detect SPA vs MPA**:

   | Signal | Indicates |
   |--------|-----------|
   | Hash routes (`#/tasks`) | SPA (hash routing) |
   | Framework root (`<div id="root">`) | Likely SPA |
   | Full page reload on link click | MPA |
   | Page shell persists on navigation | SPA |

   Record `APP_TYPE: SPA` or `MPA`. Navigation strategy differs:

   | Behavior | SPA | MPA |
   |----------|-----|-----|
   | Navigate | Click links or `navigate` | `navigate` to full URL |
   | Wait after nav | 2s (client render) | 3s (full page load) |
   | Console | Persists — clear per screen | Resets per page load |
   | Network | Accumulates — note new requests | Resets — all are relevant |

## Execution Mode Routing

- **`--workflow` set** → [Workflow Mode](reference/workflow-mode.md). Skip Phases 3-4.
- **`--fix` set** → [Fix Mode](reference/fix-mode.md). Skip Phases 3-4.
- **Otherwise** → Continue to Phase 3 (broad sweep testing).

Flag interactions:
- `--workflow`/`--fix` imply `--mode functional`
- `--a11y`/`--perf`: Run only on screens visited during workflow/fix
- `--responsive`: Re-run workflow at each breakpoint, or verify fix at all breakpoints
- `--no-autofix` + `--fix`: Contradictory — ask user to clarify
- `--focus`: Ignored when `--workflow`/`--fix` is set

## Phase 3: Discover

Map the app's testable surfaces:

1. `read_page` — find nav elements, sidebar, header links
2. `find(query="navigation links")` — record each link's text and href
3. `read_page(filter=interactive)` — catalog buttons, inputs, selects per screen
4. Build TodoWrite entry per screen: `"Screen: Tasks (#tasks) — 14 interactive elements"`
5. Report: "Found N screens with M total interactive elements. Starting testing."

## Phase 4: Test Each Screen

For each screen, run verification layers. Follow the [Interaction Stability Protocol](reference/interaction-protocol.md) for all interactions.

### Layer 1: DOM Verification
- Navigate to screen route
- `read_page` — verify meaningful content (not empty/error)
- Check for expected heading, main content area
- Blank/broken → log as bug + screenshot

### Layer 2: Console Errors
- `read_console_messages(pattern="error|Error|warn|Warning|exception", clear=true)`
- Errors → log as bugs. Warnings → log as minor issues.

### Layer 3: Network Failures
- `read_network_requests` — check for 4xx/5xx, failed/aborted requests
- Log failures with URL and status

### Layer 4: Visual Evidence
- Screenshot current state
- Visually assess for rendering issues

### Layer 5: Functional Testing (`functional`/`full` mode)

For each interactive element, follow the detailed procedures in [testing-layers.md](reference/testing-layers.md):
- **Forms**: Fill with test data → submit → verify state changed
- **Buttons**: Classify (safe/destructive/external) → click safe ones, ask about destructive/external
- **Search/Filter**: Type query → verify results update
- After each interaction: check console, network, screenshot if changed

New in `functional`/`full` mode:
- **Form validation testing**: Empty submit, invalid formats, special characters
- **Navigation testing**: Back/forward, deep linking, page refresh

New in `full` mode only:
- **Error state testing**: Simulate network failures, check error boundaries
- **State persistence testing**: localStorage/sessionStorage after reload
- **Security spot checks**: Sensitive data in storage, HTTP forms, mixed content

### Layer 6: Accessibility (`--a11y` or `full` mode)

Run WCAG-informed checks per [accessibility.md](reference/accessibility.md):
1. Images without alt text (WCAG 1.1.1)
2. Unlabeled form controls (WCAG 1.3.1)
3. Heading hierarchy (WCAG 1.3.1)
4. ARIA landmarks and attributes (WCAG 4.1.2)
5. Keyboard navigability — `full` mode only (WCAG 2.1.2)
6. Focus management (WCAG 2.4.3)
7. Color contrast — visual (WCAG 1.4.3)
8. ARIA live regions (WCAG 4.1.3)
9. Language attribute (WCAG 3.1.1)

### Layer 7: Performance (`--perf` or `full` mode)

Observational checks per [performance.md](reference/performance.md):
1. Slow requests (>3s warning, >10s bug)
2. Large payloads (>1MB warning, >5MB bug)
3. Excessive DOM size
4. Too many requests (>50 warning, >100 bug)
5. Uncompressed/uncached assets
6. Duplicate API calls
7. Render-blocking resources

Performance findings are **advisory** — they do NOT trigger auto-fix.

## Bug Deduplication

Maintain a running bug registry to avoid duplicate reports.

**Dedup key**: `error_signature` (constant part of error text) + `bug_type` (console_error, network_failure, dom_issue, visual_issue, functional_issue, a11y_issue, perf_issue, responsive_issue)

Before logging any bug:
1. Compute its dedup key
2. If duplicate exists → append current screen to existing bug's `affected_screens`
3. If new → create entry with `affected_screens: [current_screen]`

**Auto-fix dedup**: Only spawn fix agent the FIRST time a bug is encountered.

Registry format:
```
BUG REGISTRY:
- BUG-001: {key: "TypeError: Cannot read properties of undefined", type: console_error, severity: major, affected_screens: ["#tasks", "#agents"], first_seen: "#tasks"}
```

## Phase 5: Auto-Fix Bugs

**Runs immediately when any bug is confirmed** (unless `--no-autofix`).

For each bug, follow the process in [fix-agents.md](reference/fix-agents.md):

1. **Gather evidence**: Screenshot, console errors, network failures, reproduction steps
2. **Spawn fix agent**: Use the canonical prompt template with bug details + CODEBASE CONTEXT
3. **Verify fix**: Reload → re-run reproduction steps → check error is gone → screenshot
4. **Mark result**: Bug → "Fixed" or "Fix Attempted, Still Broken"
5. **Continue testing**: Resume where you left off

## Phase 6: Responsive Testing

Runs with `--responsive` or `--mode full`. Follow procedures in [responsive.md](reference/responsive.md).

Re-test screens at breakpoints: Mobile (375px), Mobile landscape (812x375), Tablet (768px), Desktop (1280px).

For each: resize → navigate → screenshot → check for layout issues (overflow, overlap, missing elements, unreadable text, small touch targets).

**Dark mode testing**: Toggle color scheme via in-app toggle or media query emulation. Check for invisible text, missing theme variables, hardcoded colors.

Responsive bugs trigger auto-fix. Restore desktop viewport after testing.

## Phase 7: Report

Generate a comprehensive report per [reporting.md](reference/reporting.md):

- **Coverage summary**: App type, auth, screens/elements tested, bug counts
- **Bug reports**: Each unique bug once with all affected screens, status, evidence, fix details
- **Accessibility report** (if `--a11y`/`full`): Issues with WCAG references
- **Performance observations** (if `--perf`/`full`): Advisory findings
- **Responsive issues** (if `--responsive`/`full`): Per-breakpoint findings
- **Security observations** (if `full`): Advisory findings
- **Next steps**: Ask user about remaining open bugs

If `--record`: Stop GIF recording, export, and offer to the user.

## Safety Guardrails

| Always ask before | Ask once, then remember | Never ask about |
|---|---|---|
| Destructive buttons (delete, remove, reset) | Auth credentials | Navigation / reading pages |
| External actions (send, publish, deploy) | Destructive action preference | Screenshots / console / network |
| Real credentials (API keys, passwords) | | Filling forms with test data |
| OAuth/SSO flows (user does manually) | | Spawning fix agents |
| | | A11y / perf / responsive checks |
| | | Retrying failed interactions |

## Testing Modes

**smoke** (default): Navigate each screen → DOM + console + network + screenshot. No interactions. Auto-fix. Deduplication active.

**functional**: smoke + interact with safe elements, fill forms, click buttons, verify state. Form validation + navigation testing. Auto-fix. Stability protocol active.

**full**: functional + error state testing + state persistence + security spot checks + accessibility (Layer 6) + performance (Layer 7) + responsive (Phase 6) + dark mode. Auto-fix.

**workflow**: Test specific user journey via `--workflow`. See [workflow-mode.md](reference/workflow-mode.md). Parses description into steps, executes sequentially, fixes bugs, re-runs entire workflow after each fix (max 3 re-runs).

**fix**: Reproduce and fix known bug via `--fix`. See [fix-mode.md](reference/fix-mode.md). Structured cycle: understand → reproduce → evidence → fix (up to 2 attempts) → validate → regression check.

Individual layers can also be enabled via flags: `--a11y`, `--perf`, `--responsive`.
