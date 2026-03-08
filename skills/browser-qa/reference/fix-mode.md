# Fix Mode (--fix)

Reproduce a known bug, fix it, and validate the fix. Structured cycle: understand → reproduce → evidence → fix → validate → regression check.

## Contents
- [F.1: Understand the Bug](#f1-understand-the-bug)
- [F.2: Reproduce](#f2-reproduce)
- [F.3: Gather Evidence](#f3-gather-evidence)
- [F.4: Fix](#f4-fix)
- [F.5: Validate](#f5-validate)
- [F.6: Regression Check](#f6-regression-check)
- [F.7: Report](#f7-report)

## F.1: Understand the Bug

Parse the `--fix` description to extract:
- **What's broken**: Error message, wrong behavior, visual glitch, crash
- **Where**: Screen, route, page, or element
- **How to trigger**: User actions that cause the bug

If reproduction steps aren't clear → ask user for more detail.

**Search codebase** for related code (using Phase 1 context):
- Grep for error message text, component names, route handlers, API endpoints
- Identify likely source files
- Build a focused codebase context block (more targeted than the general Phase 1 block)

## F.2: Reproduce

Navigate to the affected area and trigger the bug:

1. Navigate to the relevant route
2. Follow reproduction steps (click, type, submit, etc.)
3. After each action: screenshot + check console + check network
4. **Goal**: Capture concrete evidence matching the description

**If bug does NOT reproduce**:
- Try alternative paths (different data, different state, different navigation order)
- Check console for related but differently-worded errors
- Ask user for more specific steps
- If still can't reproduce after 2 attempts → report as "not reproducible"

## F.3: Gather Evidence

Once reproduced, collect into a structured package:

1. **Screenshots**: Before, during, and after the bug manifests
2. **Console errors**: Exact text and stack trace
3. **Network failures**: URL, method, status code, response excerpt
4. **DOM state**: Relevant element attributes, missing/unexpected elements
5. **Reproduction steps**: Precise, numbered, as performed

## F.4: Fix

Spawn a fix agent using the [canonical template](fix-agents.md) with:
- Full evidence package from F.3
- Focused codebase context from F.1
- Clear success criteria

Up to **2 fix attempts**. Second attempt includes the first agent's changes + still-failing evidence.

## F.5: Validate

After fix agent completes:

1. Reload page → navigate to affected route
2. Re-run exact reproduction steps from F.2
3. Verify: specific error gone, network succeeds, UI correct
4. Screenshot the fixed state as proof

Fix works → proceed to F.6. Fix fails after 2 attempts → report as "fix attempted, still broken."

## F.6: Regression Check

Quick smoke test on the affected screen:
- `read_page` — page renders correctly
- `read_console_messages` — no new errors
- `read_network_requests` — no new failures
- Screenshot — page looks correct

If Phase 1 identified other screens sharing code with the fixed area, smoke-check those too.

Report any new issues as regressions.

## F.7: Report

### Fixed
```
Bug Fix: "[description]"
═══════════════════════════════════
Status: ✅ FIXED

Reproduction:
  1. Navigate to /settings
  2. Change "Theme" dropdown to "Dark"
  3. Click "Save"
  → TypeError: Cannot read properties of undefined (reading 'theme')

Root cause: Settings handler accessed `user.preferences.theme` but
`user.preferences` was null for users who never set preferences.

Fix applied:
  File: src/api/settings.ts:47
  Change: Added null coalescing — `user.preferences?.theme ?? 'light'`

Validation:
  ✅ Reproduction steps complete without error
  ✅ Settings save successfully
  ✅ Smoke test on /settings — no regressions

Files changed:
  - src/api/settings.ts (line 47)
```

### Still broken
```
Status: ❌ FIX ATTEMPTED, STILL BROKEN

Reproduction: [confirmed]
Fix attempts: 2
  Attempt 1: [what was tried, why it didn't work]
  Attempt 2: [what was tried, why it didn't work]

Recommendation: [suggestion for manual investigation]
```

### Not reproducible
```
Status: ⚠️ NOT REPRODUCIBLE

Attempted reproduction:
  1. [steps tried]
  Result: Bug did not manifest.

Possible explanations:
  - Bug may be intermittent or timing-dependent
  - Bug may require specific data/state not present
  - Bug may have been fixed by a recent change
```
