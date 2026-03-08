# Fix Agent Guidelines

## Contents
- [When to Spawn](#when-to-spawn)
- [Canonical Prompt Template](#canonical-prompt-template)
- [Evidence Gathering Checklist](#evidence-gathering-checklist)
- [Fix Agent Rules](#fix-agent-rules)
- [Verification After Fix](#verification-after-fix)
- [Multi-Attempt Strategy](#multi-attempt-strategy)

## When to Spawn

Spawn a fix agent immediately when a bug is confirmed — do not batch fixes. Skip if `--no-autofix` is set.

**Deduplication**: Only spawn for a bug the FIRST time it's encountered. Same bug on subsequent screens → skip (the first fix will resolve all instances).

## Canonical Prompt Template

Use this single template for all fix contexts (broad testing, workflow mode, fix mode):

```
Agent(
  subagent_type: "general-purpose",
  description: "Fix [brief bug name]",
  prompt: """
    Fix this bug found during testing of the web app at [URL].

    BUG: [description]
    SCREEN: [route/view where bug occurs]
    ELEMENT: [UI element involved, if applicable]
    SEVERITY: [critical/major/minor]

    EVIDENCE:
    - Console error: [exact text, or 'none']
    - Network failure: [URL + method + status, or 'none']
    - Visual issue: [describe what looks wrong, or 'none']
    - Reproduction steps:
      1. Navigate to [route]
      2. [action]
      3. [observe bug] → expected: [what should happen]

    LIKELY SOURCE FILES:
    - [files identified from codebase recon or grep]

    [Paste CODEBASE CONTEXT block from Phase 1]

    INSTRUCTIONS:
    1. Read the project's CLAUDE.md for conventions
    2. Read the likely source files above
    3. Find and fix the root cause — not a workaround
    4. Run typecheck/lint to verify no new errors
    5. Do NOT run the dev server or browser
    6. Report: root cause, fix applied, files changed
  """
)
```

Customize the template per bug by filling in the bracketed fields. The CODEBASE CONTEXT block is the same across all bugs.

## Evidence Gathering Checklist

Collect before spawning any fix agent:

1. **Screenshot** showing the problem
2. **Console errors**: Exact text and stack trace if available
3. **Network failures**: URL, HTTP method, status code, response excerpt
4. **DOM state**: Relevant element attributes, missing elements, unexpected content
5. **Reproduction steps**: Precise, numbered, as performed

## Fix Agent Rules

Fix agents MUST:
- Read CLAUDE.md for project conventions
- Find root cause before fixing
- Apply minimal correct fix
- Run typecheck/lint

Fix agents must NOT:
- Start the dev server
- Run browser tests
- Apply UI workarounds instead of root cause fixes
- Modify tests to pass around bugs

## Verification After Fix

After the fix agent completes:

1. **Reload**: Navigate to the affected route
2. **Reproduce**: Re-run exact reproduction steps
3. **Verify**: Specific error is gone, network requests succeed, UI behaves correctly
4. **Screenshot**: Capture fixed state as proof
5. **Mark result**: Bug status → "Fixed" or "Fix Attempted, Still Broken"

## Multi-Attempt Strategy

- **Broad testing / workflow mode**: 1 fix attempt per bug. If it fails, mark as open.
- **Fix mode (`--fix`)**: Up to 2 attempts. Second attempt gets the first agent's changes + still-failing evidence as additional context.
- If fix agent returns uncertain: provide more context and try once more. Still failing → mark as open.

## Parsing Fix Agent Results

When the fix agent returns:
- Extract: root cause description, files changed, lines modified
- If agent reports "could not reproduce" or "unclear root cause" → mark bug as open with the agent's notes
- If agent reports success → proceed to verification
