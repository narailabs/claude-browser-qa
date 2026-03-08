# Workflow Mode (--workflow)

Test a specific user journey end-to-end. The user describes the workflow in natural language; execute it step by step, fixing bugs encountered along the way.

## Contents
- [W.1: Parse Workflow](#w1-parse-workflow)
- [W.2: Codebase-Informed Planning](#w2-codebase-informed-planning)
- [W.3: Execute](#w3-execute)
- [W.4: Fix Bugs](#w4-fix-bugs)
- [W.5: Report](#w5-report)
- [Workflow Chaining](#workflow-chaining)

## W.1: Parse Workflow

Analyze the `--workflow` text to extract:
- **Starting point**: URL, route, or screen where the workflow begins
- **Action steps**: Ordered list of user actions (navigate, click, type, submit, select, verify)
- **Expected outcomes**: What should happen after each step
- **Success criteria**: What "workflow complete" looks like (final state, confirmation message, redirect)

If description is too vague → ask user for more specific steps.

Build a TodoWrite checklist:
```
- [ ] Step 1: Navigate to /register
- [ ] Step 2: Fill in registration form (name, email, password)
- [ ] Step 3: Click "Create Account"
- [ ] Step 4: Verify redirect to /dashboard
- [ ] Step 5: Verify welcome message displayed
```

## W.2: Codebase-Informed Planning

Use Phase 1 codebase context to correlate workflow steps with actual code:
- Which routes/components are involved?
- Which API endpoints will be hit?
- Any relevant validation rules, state management, or middleware?
- What error handling exists along this path?

This helps anticipate likely bugs and provides better fix context.

## W.3: Execute

Follow the [Interaction Stability Protocol](interaction-protocol.md) for every interaction.

For each step:

1. **Perform action**: Navigate, fill form fields, click buttons, verify content
2. **Capture evidence**: Screenshot + `read_console_messages` + `read_network_requests`
3. **Verify outcome**: Page changed as expected? Content visible? No errors?
4. **Handle result**:
   - Succeeds → mark TodoWrite item complete, next step
   - Fails → record failure with full evidence, go to W.4

## W.4: Fix Bugs

When a step fails, spawn a fix agent using the [canonical template](fix-agents.md).

**Key difference from broad testing**: After each fix, **re-run the ENTIRE workflow from step 1** — not just the failed step. This catches regressions.

- Maximum **3 full re-runs** to prevent infinite loops
- Same step fails after fix → mark as unresolved
- Previously-passing step fails after fix (regression) → spawn fix for regression too
- Track: "Workflow run 2/3 (after fixing step 3)"

## W.5: Report

```
Workflow Test: "[workflow description]"
═══════════════════════════════════
Steps: 5/5 passed

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

If workflow still fails after all fix attempts:
```
Workflow status: FAILING at step 3
  ❌ Step 3 "Click Create Account" → POST /api/register returns 500
  Fix attempted but did not resolve the issue.
```

## Workflow Chaining

If the user provides multiple workflows separated by semicolons or as a list:
```
--workflow "register user; login; add item to cart; checkout"
```

Execute each workflow sequentially. If one fails, report it and ask the user whether to continue to the next or stop.

Between workflows:
- Reset browser state: navigate to the app root
- Clear console: `read_console_messages(clear=true)`
- Note any state dependencies between workflows (e.g., "register" must succeed before "login")

Report each workflow's status independently, then provide combined summary.
