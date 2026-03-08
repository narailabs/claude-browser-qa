# Interaction Stability & Error Recovery

## Contents
- [Wait-for-Ready Protocol](#wait-for-ready-protocol)
- [Skeleton Screen Detection](#skeleton-screen-detection)
- [Click Retry Sequence](#click-retry-sequence)
- [Type/Input Retry Sequence](#typeinput-retry-sequence)
- [Navigation Retry Sequence](#navigation-retry-sequence)
- [General Error Recovery](#general-error-recovery)
- [Retry Limits](#retry-limits)

## Wait-for-Ready Protocol

Follow before AND after every element interaction (click, type, submit).

### Before interaction

1. **Page ready**: `read_page` — check for spinners, skeleton screens, "Loading..." text, progress bars. If loading: `wait(2s)` then re-check (up to 3 times, 6s max). Still loading after 6s → log perf observation, proceed anyway.
2. **Element visible**: `find` the element → `screenshot` to confirm visible (not behind modal, not scrolled away). If below fold: scroll first.
3. **Animations settled**: If previous action triggered animation, `wait(1s)` then screenshot to confirm settled.

### After interaction

1. **Wait for update**: `wait(1s)` → `read_page` to check content changed. If form/button triggered network request: `read_network_requests` to confirm.
2. **Loading states**: If interaction triggered a spinner/skeleton, wait for resolution (same as before-interaction step 1).
3. **Toasts/notifications**: Screenshot to capture transient success/error messages before they disappear.

## Skeleton Screen Detection

Skeleton screens indicate content still loading. Patterns:
- Rectangular grey blocks where text should be
- Circular grey blocks where avatars should be
- Pulsing/shimmer animation on placeholders

When detected: wait and re-check. Do NOT interact with skeleton-showing elements.

## Click Retry Sequence

4 attempts max, escalating:

1. **Retry by ref**: Click using element ref → wait 2s → try once more
2. **Retry after wait**: `wait(3s)` → retry (element may still be loading)
3. **Retry by find**: Re-locate via `find(query="[button text]")` → click new ref
4. **Retry by coordinate**: Screenshot → visually locate → `click(coordinate=[x, y])`
5. **Log failure**: "Element '[name]' not interactable on [screen]" + screenshot

## Type/Input Retry Sequence

1. **Click first**: Click the input field, then type
2. **Clear and retype**: Click field → Ctrl+A → type value
3. **Find and retry**: Re-locate via `find(query="[label/placeholder]")` → click → type
4. **Log failure**: Record as bug

## Navigation Retry Sequence

1. **Wait and retry**: `wait(5s)` → `navigate` again
2. **Check redirect**: `read_page` — page may have loaded at different URL
3. **Hard reload**: `navigate` to exact URL once more
4. **Log failure**: Screenshot + console errors, mark screen inaccessible, continue

## General Error Recovery

| Problem | Action |
|---------|--------|
| Page won't load | Navigation retry sequence |
| Element not found | Click retry sequence |
| Click has no effect | Click retry sequence |
| App crashes | Screenshot + console → log critical bug → navigate to reload → continue |
| Auth required mid-test | Ask user for credentials |
| Modal/dialog blocks | Find close/dismiss/X button → click. No close button → press Escape |
| Fix agent fails | Log bug as open, continue testing |
| Extension disconnects | `tabs_context_mcp` to reconnect. Still fails → ask user |
| JS alert/confirm | Click OK/Accept to dismiss |
| Infinite loading | `wait(10s)` → screenshot → still loading → log as bug → navigate away |

## Retry Limits

- Per-element: 4 attempts max
- Per-page navigation: 3 attempts max
- Session-wide: if >5 consecutive failures across different screens, pause and ask user whether to continue
