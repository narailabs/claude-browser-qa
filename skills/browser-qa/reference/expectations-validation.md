# Expectations-Based Validation

Validate that the app works as intended — not just that it doesn't crash. Uses subagents to keep context clean: one discovery agent builds expectations from code/docs, then separate validation agents each test one expectation and fix issues they find.

## Contents
- [Overview](#overview)
- [Discovery Subagent](#discovery-subagent)
- [Expectations Schema](#expectations-schema)
- [Validation Subagents](#validation-subagents)
- [Orchestration](#orchestration)
- [Integration with Modes](#integration-with-modes)

## Overview

Traditional testing catches runtime errors (crashes, 404s, console errors). Expectations-based validation catches:
- **Missing features**: Docs say "users can export data" but no export button exists
- **Wrong data**: API returns 10 items but UI shows 5
- **Broken flows**: Signup → redirect to dashboard, but redirect goes to homepage instead
- **Stale UI**: Component code renders a "Premium" badge but it never appears
- **Dead routes**: Router defines `/settings/billing` but navigating there shows 404 or blank page

## Discovery Subagent

Spawn a `general-purpose` subagent to analyze the codebase and produce structured expectations. This agent has NO browser access — it only reads code and docs.

### Prompt Template

```
Agent(
  subagent_type: "general-purpose",
  description: "Discover expected app behaviors",
  prompt: """
    Analyze this web application to produce a list of testable expectations —
    things that SHOULD be true when the app is working correctly.

    APP URL: [url]

    [Paste CODEBASE CONTEXT block from Phase 1]

    ANALYSIS STRATEGY (adaptive depth — go deeper only when needed):

    LEVEL 1 — Docs (always):
    - Read README.md, CLAUDE.md, any docs/ directory
    - Extract: described features, user-facing capabilities, documented flows

    LEVEL 2 — Routes (always):
    - Find and read the router config (React Router, Next.js pages/app dir,
      Vue Router, SvelteKit routes, Express routes, etc.)
    - List every defined route with its expected screen/purpose
    - Note any route guards, redirects, or auth requirements

    LEVEL 3 — Components (if Level 1-2 produced fewer than 10 expectations):
    - Read key page/view components to understand what each screen renders
    - Extract: expected headings, data displays, form fields, buttons, states
    - Focus on pages, not shared components

    LEVEL 4 — API handlers (if a specific expectation needs backend context):
    - Read API route handlers to understand expected request/response behavior
    - Extract: expected status codes, response shapes, validation rules
    - Only read handlers relevant to expectations from earlier levels

    OUTPUT FORMAT — Return ONLY a JSON array of expectations:
    [
      {
        "id": "EXP-001",
        "category": "route|feature|data|flow|ui",
        "summary": "Short description of what should be true",
        "screen": "/route-or-screen",
        "steps": ["Navigate to /route", "Look for X", "Verify Y"],
        "success_criteria": "What 'passing' looks like",
        "source": "Where this expectation came from (file:line or doc section)",
        "priority": "high|medium|low"
      }
    ]

    GUIDELINES:
    - Each expectation must be independently testable in a browser
    - Be specific: "Dashboard shows user's name" not "Dashboard works"
    - Include the route/screen where each expectation can be verified
    - Include concrete verification steps a browser agent can follow
    - Prioritize: routes and features > data correctness > UI polish
    - Aim for 10-30 expectations depending on app complexity
    - Do NOT include expectations that require creating test data from scratch
      (the app should already have enough state to verify)
    - Do NOT read test files to generate expectations — test files test
      implementation, we want to test user-facing behavior
  """
)
```

### Handling Discovery Results

Parse the returned JSON array. If the agent returns fewer than 5 expectations or the output is malformed:
- Log a warning: "Discovery agent found few expectations — app may have limited docs/routes"
- Proceed with whatever expectations were found
- Consider asking the user: "The codebase had limited docs. Want me to test based on what I discovered, or can you describe key features?"

Filter expectations:
- Remove any that reference routes not found during Phase 3 (Discover)
- Remove duplicates (same screen + same success criteria)
- Sort by priority: high first

## Expectations Schema

Each expectation follows this structure:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique ID (EXP-001, EXP-002, ...) |
| `category` | enum | `route` (screen exists), `feature` (capability works), `data` (correct display), `flow` (multi-step journey), `ui` (visual/layout) |
| `summary` | string | Human-readable description |
| `screen` | string | Route or screen to navigate to |
| `steps` | string[] | Ordered steps to verify |
| `success_criteria` | string | What passing looks like |
| `source` | string | File/line or doc section where expectation was derived |
| `priority` | enum | `high` (core features), `medium` (secondary), `low` (polish) |

## Validation Subagents

Each expectation gets its own `general-purpose` subagent with a clean context. The subagent handles testing AND fixing.

### Prompt Template

```
Agent(
  subagent_type: "general-purpose",
  description: "Validate: [EXP-ID] [summary]",
  prompt: """
    Validate this expectation against a running web app, and fix any issues found.

    EXPECTATION:
      ID: [id]
      Category: [category]
      Summary: [summary]
      Screen: [screen]
      Steps:
        [numbered steps]
      Success criteria: [success_criteria]
      Source: [source]

    APP URL: [url]

    [Paste CODEBASE CONTEXT block]

    INSTRUCTIONS:

    1. CONNECT TO BROWSER
       - Call Claude_in_Chrome:tabs_context_mcp to get the tab group
       - Use tab ID [tab_id] (already open to the app)

    2. VALIDATE
       - Navigate to the screen: Claude_in_Chrome:navigate(url="[url][screen]")
       - Wait for page to load (2-3 seconds), then read_page
       - Follow each step in order
       - After each step: screenshot + read_page to capture state
       - Check the success criteria

    3. DETERMINE RESULT
       - PASS: Success criteria fully met. Return result.
       - FAIL: Success criteria not met. Proceed to step 4.
       - BLOCKED: Cannot reach the screen (auth, crash, missing route).
         Take screenshot, return result with evidence.

    4. IF FAIL — DIAGNOSE AND FIX
       - Capture evidence: screenshot, console errors, network requests
       - Read the source file noted in the expectation
       - Search codebase for the root cause (Grep for component names,
         route handlers, API endpoints)
       - Apply the minimal correct fix
       - Run typecheck/lint to verify no new errors

    5. RE-VALIDATE AFTER FIX
       - Reload the page: navigate to the screen again
       - Re-run the validation steps
       - If PASS now: return result with fix details
       - If still FAIL: return result noting fix attempted but insufficient

    RETURN FORMAT (plain text, not JSON):
      EXPECTATION: [id]
      RESULT: PASS | FAIL | FIXED | BLOCKED
      EVIDENCE: [what you observed — screenshot descriptions, DOM state]
      FIX: [if fixed: root cause, file changed, line. If not: "N/A"]
      NOTES: [any additional context]

    RULES:
    - Do NOT start a dev server
    - Do NOT modify test files
    - Only fix issues directly related to this expectation
    - If the fix requires changes beyond 2 files, mark as FAIL with notes
      rather than attempting a risky multi-file change
    - Maximum 1 fix attempt per expectation
  """
)
```

### Tab Management

All validation subagents share the same browser tab (passed via `tab_id`). Run them **sequentially** — not in parallel — to avoid tab conflicts.

Between validations, the main agent does NOT need to reset the tab. Each subagent navigates to its own screen.

## Orchestration

The main agent orchestrates the full flow:

### Step 1: Run Discovery
```
discovery_result = Agent(discovery subagent)
expectations = parse(discovery_result)
```

### Step 2: Filter and Prioritize
- Remove expectations for routes not found in Phase 3
- Sort by priority (high → medium → low)
- In `smoke` mode: only validate `high` priority
- In `functional` mode: validate `high` and `medium`
- In `full` mode: validate all

### Step 3: Dispatch Validation Agents
For each expectation, sequentially:
```
for exp in expectations:
    result = Agent(validation subagent for exp)
    store result
    update TodoWrite progress
```

Report progress: "Validating expectation 3/15: Dashboard shows user count..."

### Step 4: Collect Results
Aggregate all results into:
- Pass count / total
- List of failures with evidence
- List of fixes applied
- List of blocked expectations

Feed into Phase 7 report.

### Step 5: Handle Failures
For expectations that FAIL (not FIXED):
- Add to the bug registry with `bug_type: expectation_failure`
- Include in the final report
- These do NOT trigger Phase 5 auto-fix (the validation agent already attempted a fix)

## Integration with Modes

| Mode | Expectations behavior |
|------|----------------------|
| `smoke` | Discovery runs. Only `high` priority validated. |
| `functional` | Discovery runs. `high` + `medium` validated. |
| `full` | Discovery runs. All expectations validated. |
| `workflow` | No discovery. User-described flow IS the expectation. |
| `fix` | No discovery. User-described bug IS the expectation. |

The `--no-autofix` flag applies: if set, validation agents report FAIL but do NOT attempt fixes.
