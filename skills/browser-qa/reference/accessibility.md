# Accessibility Checks

WCAG-informed checks using DOM inspection. No external tools required. Run with `--a11y` or `--mode full`.

## Contents
- [Images](#1-images-without-alt-text)
- [Form Labels](#2-form-labels)
- [Heading Hierarchy](#3-heading-hierarchy)
- [ARIA Landmarks](#4-aria-landmarks-and-attributes)
- [Keyboard Navigation](#5-keyboard-navigability)
- [Focus Management](#6-focus-management)
- [Color Contrast](#7-color-contrast)
- [ARIA Live Regions](#8-aria-live-regions)
- [Language Attribute](#9-language-attribute)
- [Reporting](#reporting)

## 1. Images Without Alt Text
- Find all `<img>` elements via `find(query="images")`
- Inspect each for `alt` attribute via `read_page`
- Missing or empty `alt=""` on non-decorative images → **major** (WCAG 1.1.1: Non-text Content)
- Decorative images should have `alt=""` (empty is correct) or `role="presentation"`

## 2. Form Labels
- Find all inputs via `read_page(filter=interactive)`
- Each `<input>`, `<select>`, `<textarea>` must have: `<label for="...">`, wrapping `<label>`, `aria-label`, or `aria-labelledby`
- Unlabeled form controls → **major** (WCAG 1.3.1: Info and Relationships)

## 3. Heading Hierarchy
- Find headings via `find(query="headings")`
- Check: no skipped levels (h1 → h3 with no h2)
- Check: exactly one `<h1>` per page
- Violations → **minor** (WCAG 1.3.1: Info and Relationships)

## 4. ARIA Landmarks and Attributes
- Check for `<main>`, `<nav>`, `<header>`, `<footer>` or equivalent `role` attributes
- Check interactive custom elements (divs with click handlers) for `role`, `tabindex`, `aria-*`
- Clickable `<div>` or `<span>` without `role="button"` and `tabindex` → **major** (WCAG 4.1.2: Name, Role, Value)

## 5. Keyboard Navigability
*Full mode only.*

1. Tab through interactive elements using `key(text="Tab")`
2. Verify focus moves in logical order with visible focus indicator
3. Screenshot to confirm focus indicator visible
4. Test Enter/Space activates focused button or link
5. Focus trapped (can't Tab out of a section) → **critical** (WCAG 2.1.2: No Keyboard Trap)

## 6. Focus Management
- Modal opens → verify focus moves into modal
- Modal closes → verify focus returns to trigger element
- In-page navigation → verify focus moves to new content area
- Poor focus management → **major** (WCAG 2.4.3: Focus Order)

## 7. Color Contrast
- Screenshot → visually inspect for text difficult to read (light on light, dark on dark)
- Flag low-contrast text → **minor** with note: "Visual inspection — confirm with a contrast checker tool"
- (WCAG 1.4.3: Contrast Minimum)

## 8. ARIA Live Regions
- For dynamic content updates (notifications, live data, form errors): check for `aria-live` attribute
- Status messages that appear without page reload should use `aria-live="polite"` or `role="status"`
- Missing live region on dynamic content → **minor** (WCAG 4.1.3: Status Messages)

## 9. Language Attribute
- Check `<html>` element for `lang` attribute via `javascript_tool`: `document.documentElement.lang`
- Missing or empty `lang` → **minor** (WCAG 3.1.1: Language of Page)

## Reporting

All accessibility bugs use `[A11Y]` prefix and include:
- Issue description
- WCAG criterion reference (e.g., "WCAG 2.1 SC 1.1.1")
- Affected screen and element
- Severity: critical / major / minor

Accessibility bugs DO trigger auto-fix agents (most are straightforward attribute additions).
