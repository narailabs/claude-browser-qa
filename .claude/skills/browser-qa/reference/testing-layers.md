# Functional Testing Procedures

## Contents
- [Layer 5: Functional Testing](#layer-5-functional-testing)
- [Form Testing](#form-testing)
- [Button Classification & Testing](#button-classification--testing)
- [Search & Filter Testing](#search--filter-testing)
- [Form Validation Testing](#form-validation-testing) *(functional/full mode)*
- [Navigation Testing](#navigation-testing) *(functional/full mode)*
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

### Button Classification & Testing

Classify each button before clicking:

| Type | Examples | Action |
|------|----------|--------|
| **Safe** | Navigate, toggle, filter, sort, expand, collapse | Click and verify outcome |
| **Destructive** | Delete, remove, reset, clear | Ask user first |
| **External action** | Send, publish, connect, deploy, share | Ask user first |

### Search & Filter Testing

If search/filter exists:
1. Type a query, verify results update
2. Clear the query, verify results reset
3. Try a query with no results, verify empty state

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

After each screen, count and report:
- Elements found vs elements tested
- Inline format: "Tasks screen: 12/14 elements tested (86%), 1 console warning found"
- Mark TodoWrite item as completed
