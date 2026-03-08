# Report Templates

## Contents
- [Broad Testing Report](#broad-testing-report)
- [CRUD Test Results](#crud-test-results)
- [Expectations Validation Report](#expectations-validation-report)
- [Bug Report Format](#bug-report-format)
- [Accessibility Report](#accessibility-report)
- [Performance Report](#performance-report)
- [Responsive Report](#responsive-report)
- [Security Observations](#security-observations)
- [Next Steps](#next-steps)
- [Markdown File Output](#markdown-file-output)

## Broad Testing Report

The Phase 8 report for broad testing (smoke, functional, full modes):

```
E2E Test Complete
═══════════════════════════════════
App type: SPA / MPA
Auth: Logged in as [user] / No auth / Skipped
Mode: smoke / functional / full
Screens: 6/6 tested (100%)

Functional Coverage: 92%
  Element: 45/52 (87%) | CRUD: 4/4 entities (100%) | Actions: 18/20 (90%) | Permutations: 28/35 (80%)

Expectations validated: 12/15
  ✅ Pass: 10
  🔧 Fixed: 1
  ❌ Fail: 1

Runtime bugs found: 5 unique (across 8 total occurrences)
  ✅ Fixed inline: 3
  ❌ Still open: 2

Accessibility issues: 4
  3 major, 1 minor

Performance observations: 2

Responsive issues: 1

Security observations: 0

Coverage gaps:
- Settings: API key form skipped (no credentials)
- Tasks: Delete button skipped (destructive, user declined)
- Admin panel: requires admin role — not tested

Test data created:
  - Task: "QA Updated Task 1741234567" (delete declined)
  - Project: "QA Test Project 1741234567" (deleted ✅)
```

## CRUD Test Results

For each data entity discovered, report the CRUD lifecycle result:

```
CRUD Test Results
═══════════════════════════════════
Tasks (CRUD ✅):
  Create: ✅ Created "QA Test Task 1741234567"
  Read:   ✅ Detail view shows all fields correctly
  Update: ✅ Renamed to "QA Updated Task", change persisted
  Delete: ✅ Deleted, removed from list (user approved)

Projects (CRUD ⚠️ partial):
  Create: ✅ Created "QA Test Project"
  Read:   ✅ Detail view correct
  Update: ❌ BUG — Edit button opens blank modal (BUG-003)
  Delete: ⏭️ Skipped (user declined)

Users (CRUD ⚠️ partial):
  Create: ⏭️ Skipped (requires admin role)
  Read:   ✅ User list displays correctly
  Update: ⏭️ Skipped (requires admin role)
  Delete: ⏭️ Skipped (requires admin role)
```

## Expectations Validation Report

```
Expectations Validation
═══════════════════════════════════
Discovered: 15 expectations from codebase analysis
Validated: 12 (high + medium priority)
Skipped: 3 (low priority, smoke mode)

EXP-001: Dashboard shows logged-in user's name ✅ PASS
  Screen: /dashboard | Source: src/pages/Dashboard.tsx:12

EXP-002: Task list displays all user tasks ✅ PASS
  Screen: /tasks | Source: README.md "Features" section

EXP-003: Settings page saves theme preference 🔧 FIXED
  Screen: /settings | Source: src/pages/Settings.tsx:45
  Issue: Theme preference saved but not applied on reload
  Fix: Added theme initialization in App.tsx useEffect (src/App.tsx:18)

EXP-004: Export button generates CSV download ❌ FAIL
  Screen: /tasks | Source: README.md "Export data as CSV"
  Issue: Export button exists but click triggers "Not implemented" console error
  Fix attempted: Could not determine intended implementation from codebase

EXP-005: /settings/billing route renders billing page ⚠️ BLOCKED
  Screen: /settings/billing | Source: src/router.tsx:28
  Issue: Route defined but navigating shows blank page (auth redirect loop)
```

## Bug Report Format

List each unique bug ONCE with all affected screens (from dedup registry):

```
BUG-001: TypeError: Cannot read properties of undefined (reading 'map')
  Severity: major | Type: console_error
  Affected screens: #tasks, #agents, #settings (3 screens)
  Evidence: [console error text]
  Status: ✅ Fixed — src/components/TaskList.tsx:23 (added null check)

BUG-002: GET /api/users 404
  Severity: major | Type: network_failure
  Affected screens: #dashboard
  Evidence: GET http://localhost:3000/api/users → 404
  Status: ❌ Still open — fix agent could not determine root cause
```

## Accessibility Report

For `--a11y` or `--mode full`:

```
Accessibility Issues
═══════════════════════════════════
[A11Y] Missing alt text on product images (WCAG 1.1.1)
  Screen: #products | Element: img.product-image | Severity: major | Status: ✅ Fixed

[A11Y] Form input without label (WCAG 1.3.1)
  Screen: #settings | Element: input#api-key | Severity: major | Status: ❌ Open

[A11Y] Heading hierarchy skips h2 (WCAG 1.3.1)
  Screen: #dashboard | Severity: minor | Status: ❌ Open
```

## Performance Report

For `--perf` or `--mode full`:

```
Performance Observations (advisory — no auto-fix)
═══════════════════════════════════
[PERF] GET /api/tasks took 4.2s (threshold: 3s)
  Screen: #tasks | Recommendation: Consider pagination

[PERF] Response payload 2.3 MB for GET /api/analytics
  Screen: #dashboard | Recommendation: Paginate or lazy-load data

[PERF] 73 network requests on page load
  Screen: #dashboard | Recommendation: Bundle or lazy-load assets
```

## Responsive Report

For `--responsive` or `--mode full`:

```
Responsive Issues
═══════════════════════════════════
[Mobile 375px] Nav bar overflows horizontally on Tasks screen
  Severity: major | Status: ✅ Fixed (CSS overflow-x fix)

[Tablet 768px] Sidebar overlaps main content on Dashboard
  Severity: minor | Status: ❌ Open
```

## Security Observations

For `--mode full` (security spot checks):

```
Security Observations (advisory — no auto-fix)
═══════════════════════════════════
[SEC] Auth token stored in localStorage
  Screen: all | Note: Consider httpOnly cookies for sensitive tokens

[SEC] Form submits to HTTP endpoint
  Screen: #contact | Note: Use HTTPS for form submissions
```

## Next Steps

After reporting, ask the user:
```
"N bugs remain unfixed. What would you like to do?"
Options:
  "Try fixing them again" — respawn fix agents with more context
  "Just log them" — done, user will fix later
  "Re-test a specific screen" — re-run testing on one screen
```

## Markdown File Output

If the user requests a written report, or if there are many findings (>10 total issues), offer to write the report to a markdown file:

```
"Would you like me to save this report to a file?"
Options: "Yes, save to qa-report.md", "No, console output is fine"
```

If yes: write the full report to `qa-report.md` (or user-specified path) in the project root using the Write tool.
