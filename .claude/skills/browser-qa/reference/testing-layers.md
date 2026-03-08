# Functional Testing Procedures

## Contents
- [Layer 5: Functional Testing](#layer-5-functional-testing)
- [Form Testing](#form-testing)
- [CRUD Lifecycle Testing](#crud-lifecycle-testing)
- [Button Classification & Testing](#button-classification--testing)
- [Action Verification Protocol](#action-verification-protocol)
- [Search & Filter Testing](#search--filter-testing)
- [Form Validation Testing](#form-validation-testing) *(functional/full mode)*
- [Navigation Testing](#navigation-testing) *(functional/full mode)*
- [Permutation Testing](#permutation-testing) *(full mode only)*
- [Error State Testing](#error-state-testing) *(full mode only)*
- [State Persistence Testing](#state-persistence-testing) *(full mode only)*
- [Security Spot Checks](#security-spot-checks) *(full mode only)*
- [Coverage Tracking](#coverage-tracking)

## Layer 5: Functional Testing

Runs in `functional` or `full` mode. Follow the [Interaction Stability Protocol](interaction-protocol.md) before every interaction.

For each interactive element on a screen:

### Form Testing

1. Find inputs → fill with reasonable test data → submit
   - Text inputs: descriptive values ("Test Task", "test@example.com", "555-0100")
   - Selects: choose first non-default option
   - Checkboxes/radios: toggle them
   - Date inputs: use today's date or a near-future date
2. After submission: `read_page` to verify state changed
3. Check console for new errors, verify no network failures

### CRUD Lifecycle Testing

For each data entity on a screen (lists, tables, grids, cards), test the full create → read → update → delete lifecycle.

#### Discover Entities

- `read_page` → identify repeating data structures (table rows, list items, card grids)
- Note entity type (tasks, users, projects, etc.) and current item count
- Record which CRUD operations appear available (add button? edit links? delete icons?)

#### CREATE

1. Find "Add" / "New" / "Create" / "+" button → click
2. Fill creation form with distinctive test data: `"QA Test [entity] [timestamp]"` (e.g., `"QA Test Task 1741234567"`)
   - Use unique names so the item is easy to find later
   - Fill all required fields, plus at least one optional field
3. Submit → wait → verify:
   - Item appears in the list/table
   - Item count incremented by 1
   - No console errors, network request succeeded (2xx)
   - Screenshot the created item

#### READ

1. Click the created item to open its detail view (if one exists)
2. Verify all fields match what was entered (name, optional fields, timestamps)
3. Navigate back to the list → verify item still visible
4. If no detail view exists, verify the item's inline data in the list is correct

#### UPDATE

1. Find edit button/link on the created item → click
2. Modify one or more fields (e.g., rename `"QA Test Task"` → `"QA Updated Task"`)
3. Save → wait → verify:
   - Changes reflected in list view and/or detail view
   - No console errors, network request succeeded
   - Screenshot updated state
4. If inline editing exists (click-to-edit), test that path too

#### DELETE

1. Find delete button on the test item
2. **Ask user first** (destructive action): "May I delete the test item 'QA Updated Task' to verify delete works?"
3. If approved:
   - Click delete → confirm if dialog appears → wait
   - Verify item removed from list, count decremented
   - Screenshot the list without the item
4. If declined: note as "Delete untested (user declined)" in coverage report

#### Cleanup

After CRUD testing, report which test items were created and left in the app:
```
Test data created:
  - Task: "QA Updated Task" (if delete was declined)
  - Project: "QA Test Project 1741234567"
```

### Button Classification & Testing

Classify each button before clicking:

| Type | Examples | Action |
|------|----------|--------|
| **Safe** | Navigate, toggle, filter, sort, expand, collapse | Click and verify outcome |
| **Destructive** | Delete, remove, reset, clear | Ask user first |
| **External action** | Send, publish, connect, deploy, share | Ask user first |

### Action Verification Protocol

For every action button — not just "click and move on." Verify the action actually did something:

1. **Capture pre-state**: screenshot + `read_page` + note relevant data (counts, values, status labels, sort order)
2. **Execute action**: click the button
3. **Capture post-state**: wait → screenshot + `read_page` + check console + check network
4. **Verify outcome** — compare pre vs post, something must have changed:

   | Action type | Expected change |
   |-------------|----------------|
   | Toggle (on/off, expand/collapse) | State flipped visually |
   | Sort | Item order changed |
   | Filter | Visible items reduced or restored |
   | Status change (complete, archive, approve) | Status label/badge updated |
   | Export/download | Network request succeeded (2xx), or download triggered |
   | Refresh/reload | Data re-fetched (network request), content updated |

5. **No visible change + no error** → log as potential bug: "Action '[button name]' had no visible effect on [screen]"
6. **Verify reversibility** (when safe): if the action is reversible (toggle, sort direction), reverse it and verify original state restored

### Search & Filter Testing

If search/filter exists:
1. Type a query matching existing data → verify results update to show matches
2. Clear the query → verify full results restored
3. Try a query with no matches → verify empty state message appears
4. If multiple filter controls exist, test them in combination

## Form Validation Testing

*Functional/full mode.* For each form discovered, test validation rules after the normal happy-path test:

1. **Empty submission**: Clear all fields → click submit → verify validation messages appear and form does NOT submit
2. **Invalid email format**: If email field exists, enter "notanemail" → submit → check validation
3. **Long strings**: Enter 200+ character string in a text input → submit → check for truncation or errors
4. **Special characters**: Enter `<script>alert(1)</script>` in a text field → submit → verify it's escaped/rejected (not executed)
5. **Boundary values**: For number inputs, try 0, negative, very large numbers

Log missing validation as minor bugs. Log XSS execution (if the script tag actually runs) as critical security bug.

## Navigation Testing

*Functional/full mode.* After visiting several screens:

1. **Back/forward**: Use browser back → verify previous screen loads correctly (especially in SPAs). Then forward → verify return to the screen you came from.
2. **Deep linking**: Copy the current URL → navigate to a blank page → navigate to the copied URL → verify the correct screen loads with correct state.
3. **Page refresh**: On a screen with state (form data, selected tab, scroll position), refresh the page → verify state is preserved or gracefully reset.
4. **404 handling**: Navigate to a non-existent route (e.g., `/this-does-not-exist`) → verify the app shows a proper 404 page (not a blank screen or crash).

## Permutation Testing

*Full mode only.* Test multiple paths through interactive elements — not just the first option.

### Select/Dropdown Permutations

For each select element, test every option:
- Select option → verify UI updates accordingly (filtered data, changed view, etc.)
- Check console/network after each change
- If >10 options: test first 3, last 1, and any edge-case option (e.g., "All", "None", "Other")
- Reset to default after testing

### Checkbox/Toggle Permutations

- Test both checked and unchecked states
- For checkbox groups: test all-checked, all-unchecked, and at least one mixed state
- Verify each state change has the expected effect on the UI

### Form Input Variations

- Run the happy-path form submission a second time with different valid data
- This catches bugs that only appear with specific data patterns (e.g., special characters in names, very long values, unicode)

### Tab/View Permutations

- For tabbed interfaces: click every tab, verify each tab's content loads correctly
- For view toggles (list/grid/calendar/kanban): switch to each view, verify data displays correctly in each

### Multi-Step Flow Branches

- If a flow has conditional branches (e.g., "New user? Yes/No"), test each branch
- If a wizard has optional steps, test both with and without completing them
- If a form has "Advanced options" or collapsible sections, expand and test those fields

Track permutation coverage: `"Permutations tested: 23/28 (82%)"`

## Error State Testing

*Full mode only.* Test how the app handles failures:

1. **Simulated network error**: On an API-dependent screen, use `javascript_tool` to temporarily intercept fetch:
   ```js
   window.__originalFetch = window.fetch;
   window.fetch = () => Promise.reject(new Error('Network error'));
   ```
   Then trigger the action that calls the API. Check for: error boundary UI, user-friendly message, retry button.

2. **Restore fetch** immediately after observation:
   ```js
   window.fetch = window.__originalFetch;
   ```

3. **Empty states**: If the app shows lists/tables, check behavior when data is empty (may already be visible on a fresh install or test account).

Log missing error handling (blank screen, unhandled promise rejection) as major bugs.

## State Persistence Testing

*Full mode only.* Check localStorage/sessionStorage behavior:

1. After performing actions (toggle settings, fill preferences), reload the page → check if state persists as expected.
2. Use `javascript_tool` to inspect storage:
   ```js
   JSON.stringify(Object.keys(localStorage))
   ```
3. Flag sensitive data in localStorage (auth tokens, passwords, PII) as security observations.

## Security Spot Checks

*Full mode only.* Basic observations — not penetration testing. Report as advisory (like performance).

1. **Sensitive data in storage**: Check localStorage/sessionStorage for auth tokens, passwords, API keys. Flag as security observation.
2. **HTTP form submissions**: If the app is served over HTTPS but forms submit to HTTP endpoints, flag it.
3. **Mixed content**: Note any HTTP resources loaded on HTTPS pages (images, scripts, API calls).
4. **Open redirects**: If URL parameters control redirects (e.g., `?redirect=`), note the pattern.

Security findings go in a **Security Observations** section of the report. They do NOT trigger auto-fix (too sensitive for automated changes).

## Coverage Tracking

Track four coverage dimensions per screen and overall:

| Dimension | Formula | Target |
|-----------|---------|--------|
| **Element coverage** | interactive elements tested / elements found | ≥90% |
| **CRUD coverage** | entities with full CRUD tested / data entities found | 100% |
| **Action coverage** | actions with verified outcomes / actions found | ≥90% |
| **Permutation coverage** | paths tested / paths identified | ≥80% (full mode) |

### Per-Screen Reporting

After each screen, report inline:
```
Tasks screen: Elements 12/14 (86%) | CRUD ✅ | Actions 5/5 (100%) | Permutations 8/10 (80%)
```

CRUD status per entity: ✅ (all ops tested), ⚠️ (partial — note what's missing), ❌ (blocked)

### Overall Coverage

At the end of testing, compute and report:
```
Functional Coverage: 92%
  Element: 45/52 (87%) | CRUD: 4/4 entities (100%) | Actions: 18/20 (90%) | Permutations: 28/35 (80%)
```

### Coverage Gaps

When coverage drops below 90%, note each gap with its reason:
- "Delete skipped — user declined destructive action"
- "Admin panel skipped — requires admin role"
- "Payment form skipped — requires real credentials"
- "3 dropdown options untested — >10 options, sampled representative set"

Mark TodoWrite item as completed after each screen.
