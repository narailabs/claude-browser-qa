---
name: browser-qa
description: Use when testing a web app end-to-end via Chrome browser. Navigates all screens, discovers interactive elements, verifies functionality, checks console errors and network failures, finds bugs, and automatically launches fix agents inline. Also supports targeted workflow testing and bug fix cycles. Requires Claude in Chrome extension connected.
disable-model-invocation: true
argument-hint: "[url] [--mode smoke|functional|full] [--record] [--no-autofix] [--a11y] [--perf] [--responsive] [--skip-auth] [--focus \"#route\"] [--workflow \"...\"] [--fix \"...\"]"
---

# Intelligent E2E Testing

Systematically explore and test a web application using Chrome MCP browser tools. Auto-discovers screens, interactive elements, and testable surfaces from the DOM — no project-specific configuration needed. **Automatically spawns fix agents for every bug found**, then re-tests to confirm the fix. Goes beyond runtime errors — analyzes docs and code to build expectations of how the app should behave, then validates each expectation with dedicated subagents that test and fix independently.

**Arguments received**: $ARGUMENTS

Parse the arguments:
- First positional argument = URL to test (e.g., `http://localhost:5173`)
- `--mode smoke|functional|full` = testing depth (default: `full`)
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

**Mode selection**: If `--mode` is NOT explicitly set and neither `--workflow` nor `--fix` is used, ask the user to pick a mode before starting:

```
AskUserQuestion:
  question: "Which testing mode should I use?"
  header: "Test mode"
  options:
    - label: "Full (Recommended)"
      description: "CRUD lifecycle, action verification, permutations, a11y, perf, responsive, dark mode. Targets ≥90% coverage."
    - label: "Functional"
      description: "CRUD lifecycle, action verification, form validation, navigation testing. No permutations or a11y/perf/responsive."
    - label: "Smoke"
      description: "Navigate and observe only — no interactions. Quick pass for DOM, console, and network errors."
```

Map the answer to the mode and proceed. If the user picks "Other", use their custom input as the mode or clarify.

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

- **`--workflow` set** → [Workflow Mode](reference/workflow-mode.md). Skip Phases 3-5.
- **`--fix` set** → [Fix Mode](reference/fix-mode.md). Skip Phases 3-5.
- **Otherwise** → Continue to Phase 3 (broad sweep testing).

Flag interactions:
- `--workflow`/`--fix` imply `--mode functional`
- `--a11y`/`--perf` with `--workflow`/`--fix`: After the workflow/fix completes, run Layer 6/7 checks on the screens that were visited during execution
- `--responsive`: Re-run workflow at each breakpoint, or verify fix at all breakpoints
- `--no-autofix` + `--fix`: Contradictory — ask user to clarify
- `--focus`: Limits broad testing (Phases 3-7) to the specified route only. Ignored when `--workflow`/`--fix` is set

## Phase 3: Discover

Map the app's testable surfaces:

1. `read_page` — find nav elements, sidebar, header links
2. `find(query="navigation links")` — record each link's text and href
3. `read_page(filter=interactive)` — catalog buttons, inputs, selects per screen
4. Build TodoWrite entry per screen: `"Screen: Tasks (#tasks) — 14 interactive elements"`
5. Report: "Found N screens with M total interactive elements. Starting testing."

**`--focus` filtering**: If `--focus` is set, limit discovery and all subsequent phases (4-7) to only the matching route/screen. Other screens are skipped and noted as "out of focus scope" in the report.

## Phase 4: Expectations-Based Validation

Validate the app works as *intended* — not just that it doesn't crash. Uses subagents for clean context per validation. Full procedures in [expectations-validation.md](reference/expectations-validation.md).

### Step 1: Discover Expectations
Spawn a `general-purpose` subagent (no browser access) to analyze the codebase adaptively:
- **Level 1** (always): Read README/docs → extract described features and capabilities
- **Level 2** (always): Parse route config → list all expected screens
- **Level 3** (if <10 expectations): Read key page components → extract expected UI elements
- **Level 4** (targeted): Read API handlers only when needed for specific expectations

The agent returns a JSON array of structured expectations — each with an ID, category, screen, verification steps, and success criteria.

### Step 2: Filter by Mode
- `smoke`: Only `high` priority expectations
- `functional`: `high` + `medium` priority
- `full`: All expectations

### Step 3: Validate Each Expectation
For each expectation, spawn a **separate** `general-purpose` subagent with clean context. Each validation agent:
1. Connects to the browser (same tab, passed via tab ID)
2. Navigates to the expectation's screen
3. Follows the verification steps
4. Checks success criteria
5. **If FAIL**: Reads source code, diagnoses root cause, applies fix, re-validates
6. Returns: PASS / FAIL / FIXED / BLOCKED with evidence

Run sequentially (agents share the browser tab). Report progress: "Validating 3/15: Dashboard shows user count..."

### Step 4: Collect Results
- Aggregate pass/fail/fixed/blocked counts
- Failed expectations (not fixed) → add to bug registry as `expectation_failure`
- Feed all results into Phase 8 report

`--no-autofix`: Validation agents report failures but skip fix attempts.

## Phase 5: Test Each Screen

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
- **CRUD lifecycle**: For each data entity — create item → verify → edit → verify → delete (ask first) → verify
- **Forms**: Fill with test data → submit → verify state changed
- **Actions**: Click → capture pre/post state → verify outcome changed → verify reversibility
- **Buttons**: Classify (safe/destructive/external) → click safe ones, ask about destructive/external
- **Search/Filter**: Type query → verify results update → clear → verify reset
- After each interaction: check console, network, screenshot if changed

New in `functional`/`full` mode:
- **Form validation testing**: Empty submit, invalid formats, special characters
- **Navigation testing**: Back/forward, deep linking, page refresh

New in `full` mode only:
- **Permutation testing**: Test all select options, toggle states, form input variations, tab views, flow branches
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

## Phase 6: Auto-Fix Bugs

**Runs immediately when any bug is confirmed during Phase 5** (unless `--no-autofix`). Note: bugs found during Phase 4 (expectations validation) are already handled by the validation subagents — this phase covers runtime bugs from Phase 5.

For each bug, follow the process in [fix-agents.md](reference/fix-agents.md):

1. **Gather evidence**: Screenshot, console errors, network failures, reproduction steps
2. **Spawn fix agent**: Use the canonical prompt template with bug details + CODEBASE CONTEXT
3. **Verify fix**: Reload → re-run reproduction steps → check error is gone → screenshot
4. **Mark result**: Bug → "Fixed" or "Fix Attempted, Still Broken"
5. **Continue testing**: Resume where you left off

## Phase 7: Responsive Testing

Runs with `--responsive` or `--mode full`. Follow procedures in [responsive.md](reference/responsive.md).

Re-test screens at breakpoints: Mobile (375px), Mobile landscape (812x375), Tablet (768px), Desktop (1280px).

For each: resize → navigate → screenshot → check for layout issues (overflow, overlap, missing elements, unreadable text, small touch targets).

**Dark mode testing**: Toggle color scheme via in-app toggle or media query emulation. Check for invisible text, missing theme variables, hardcoded colors.

Responsive bugs trigger auto-fix. Restore desktop viewport after testing.

## Phase 8: Report

Generate a comprehensive report per [reporting.md](reference/reporting.md):

- **Coverage summary**: App type, auth, screens/elements tested, bug counts
- **Expectations validation**: Pass/fail/fixed/blocked counts, detailed results per expectation
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
| OAuth/SSO flows (user does manually) | | Clicking safe buttons (non-destructive, in-app) |
| | | Spawning fix agents |
| | | A11y / perf / responsive checks |
| | | Retrying failed interactions |

## Testing Modes

**smoke**: Navigate each screen → DOM + console + network + screenshot. Expectations validation (high priority only). No interactions. Auto-fix. Deduplication active. Coverage target: element coverage only.

**functional**: smoke + interact with safe elements, fill forms, click buttons, verify state. CRUD lifecycle for all data entities. Action verification with pre/post state comparison. Expectations validation (high + medium priority). Form validation + navigation testing. Auto-fix. Stability protocol active. Coverage target: ≥90% element + action coverage, 100% CRUD coverage.

**full** (default): functional + all expectations validated + permutation testing (all select options, toggle states, form variations, tab views, flow branches) + error state testing + state persistence + security spot checks + accessibility (Layer 6) + performance (Layer 7) + responsive (Phase 7) + dark mode. Auto-fix. Coverage target: ≥90% all dimensions, ≥80% permutation coverage.

**workflow**: Test specific user journey via `--workflow`. See [workflow-mode.md](reference/workflow-mode.md). Parses description into steps, executes sequentially, fixes bugs, re-runs entire workflow after each fix (max 3 re-runs). No expectations discovery (the workflow IS the expectation).

**fix**: Reproduce and fix known bug via `--fix`. See [fix-mode.md](reference/fix-mode.md). Structured cycle: understand → reproduce → evidence → fix (up to 2 attempts) → validate → regression check. No expectations discovery (the bug IS the expectation).

Individual layers can also be enabled via flags: `--a11y`, `--perf`, `--responsive`.

---

## Feedback Loop — Self-Improvement Protocol

> This skill improves itself over time. The following protocol runs automatically.

### @BOOT — Load Learned Context

At the start of every run:

1. Read `feedback/patterns.md` — apply all listed patterns as constraints on this run.
2. Read the last 5 entries of `feedback/run-log.jsonl` — note recent corrections to avoid repeating them.
3. Pay special attention to patterns about: element interaction timing (wait durations, retry strategies), false-positive bug classification (benign console warnings vs real errors, expected 404s vs broken endpoints), expectations discovery accuracy (over/under-generating expectations from codebase analysis), fix agent prompt effectiveness, and subagent coordination (tab sharing, context passing).

Do not mention the feedback loop to the user unless asked. Apply patterns silently.

### @REVIEW — Self-Assess After Completion

After completing the user's task (before saying "done"):

1. **Corrections check**: Did the user correct, redirect, or redo anything during this run? List each correction.
2. **Approach diff**: Compare initial approach to final output. What changed and why?
3. **Friction points**: Where did the process feel clumsy, slow, or uncertain?
4. **Domain-specific review**:
   - Did element interaction timing cause false failures (clicking too early, not waiting long enough)?
   - Were any console errors or network failures misclassified as bugs (false positives) or missed (false negatives)?
   - Did expectations discovery generate accurate, testable expectations — or were there too many false/trivial/untestable ones?
   - Did fix agent prompts produce effective fixes on the first attempt, or did they need retries?
   - Did validation subagents have sufficient context, or did they fail due to missing codebase info?
   - Was bug deduplication accurate, or were duplicates reported or distinct bugs merged?

Produce a structured internal review (not shown to user yet).

### @EVOLVE — Propose Improvements (User-Gated)

After @REVIEW, present findings to the user:

> "I completed the task. Before wrapping up, I noticed some patterns that could improve how I handle browser QA testing in the future. Here's what I'd like to update:"

For EACH proposed change, present:

| Field | Content |
|-------|---------|
| **What** | The exact text to add/modify in `feedback/patterns.md` |
| **Why** | What triggered this — user correction, self-review finding, or repeated pattern |
| **Impact** | How this makes future runs better |

Then ask:
> "Would you like me to apply any of these? You can approve, modify, or reject each one."

**Rules:**
- NEVER write to `feedback/patterns.md` without user approval
- NEVER modify SKILL.md content above the Feedback Loop section
- NEVER change the skill's name, description, or core purpose
- Append approved patterns to the appropriate section in `feedback/patterns.md`
- Log the full run summary to `feedback/run-log.jsonl`
- If a proposed change would alter what the skill DOES (vs how it does it), flag it: "⚠️ This change may affect scope — proceed?"

### Scope Guard

The feedback loop may ONLY modify:
- `feedback/patterns.md` — learned behavioral patterns
- `feedback/run-log.jsonl` — run history

It may NEVER modify:
- SKILL.md frontmatter (name, description)
- SKILL.md sections above the Feedback Loop section
- Any files in reference/ or assets/
- The skill's core purpose or scope
