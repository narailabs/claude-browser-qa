# Responsive & Dark Mode Testing

Run with `--responsive` or `--mode full`. Re-tests discovered screens at common breakpoints and color schemes.

## Contents
- [Breakpoints](#breakpoints)
- [Test Procedure](#test-procedure)
- [Layout Issue Checklist](#layout-issue-checklist)
- [Responsive Navigation](#responsive-navigation)
- [Dark Mode Testing](#dark-mode-testing)
- [Reporting](#reporting)

## Breakpoints

| Name | Width | Height |
|------|-------|--------|
| Mobile (portrait) | 375px | 812px |
| Mobile (landscape) | 812px | 375px |
| Tablet | 768px | 1024px |
| Desktop | 1280px | 900px |

Start with mobile portrait (most likely to reveal issues), then tablet. Desktop is the baseline and was already tested. Mobile landscape catches orientation-specific bugs.

## Test Procedure

For each breakpoint:

1. **Resize viewport**: `resize_window(width, height)` or `computer(action=resize, ...)`
2. **Re-test each screen** (abbreviated — not full Layer 1-7):
   - Navigate to the screen's route
   - Screenshot — capture the layout
   - `read_page` — check expected content is visible
3. **Check for layout issues** (see checklist below)
4. **Log responsive bugs** with breakpoint: "[Mobile 375px] Header text overlaps navigation on Tasks screen"

After all breakpoints: **restore desktop viewport** (`resize_window(1280, 900)`).

## Layout Issue Checklist

Inspect screenshots for:

- **Horizontal overflow**: Content extends beyond viewport width (horizontal scrollbar)
- **Overlapping elements**: Text or buttons overlapping each other
- **Missing elements**: Content present at desktop but missing and not accessible via mobile menu
- **Unreadable text**: Text too small to read at viewport width
- **Broken images**: Images overflowing containers or distorted
- **Touch targets**: Buttons/links smaller than ~44x44px (hard to tap on mobile)
- **Cut-off content**: Text or UI elements clipped by container edges

Severity: `major` for broken functionality, `minor` for cosmetic issues.

## Responsive Navigation

- If desktop had a full nav bar, mobile should have hamburger menu or equivalent
- If hamburger exists: click it → screenshot → verify menu opens and links work
- Missing mobile navigation → **major**

## Dark Mode Testing

Test both light and dark color schemes to catch theme-related bugs.

### Detection and Toggling

1. **Check for in-app toggle**: `find(query="dark mode toggle, theme switch, appearance")` — if found, use it
2. **If no toggle found**: Use `javascript_tool` to emulate via media query:
   ```js
   // Check current preference
   window.matchMedia('(prefers-color-scheme: dark)').matches
   ```
   Then use `preview_resize(colorScheme="dark")` if available, or note that dark mode testing requires an in-app toggle.

### What to Check

After switching to dark mode:

1. **Screenshot** — visually scan for:
   - Invisible text (dark text on dark background)
   - Invisible icons or UI elements
   - Hard-to-read text due to poor contrast
   - Elements with hardcoded light-theme colors (white backgrounds, light borders)
2. **Check for missing theme variables**: UI elements that didn't respond to the theme change (e.g., a card with white background in dark mode)
3. **Switch back to light mode** and verify the UI returns to normal

Dark mode bugs: severity `minor` (cosmetic) unless text is completely invisible (`major`).

## Reporting

Responsive issues go in a **Responsive Issues** section:
- Breakpoint (e.g., "Mobile 375px")
- Issue description with screenshot reference
- Screen affected
- Severity: major / minor
- Status: Fixed / Open

Responsive bugs DO trigger auto-fix agents (typically CSS fixes). Include breakpoint and viewport screenshot in the fix agent prompt.
