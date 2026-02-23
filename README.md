# Claude Browser QA

AI-powered QA agent for Claude Code. Intelligently browses your web app, discovers every screen and interactive element, tests them systematically, and automatically fixes bugs it finds.

## What It Does

- **Auto-discovers** all navigation routes, forms, buttons, and interactive elements from the DOM
- **Tests systematically** with multiple depth modes: smoke, functional, and full
- **Verifies everything**: DOM content, console errors, network failures, visual rendering, accessibility, performance
- **Fixes bugs inline** by spawning fix agents when issues are found, then re-tests to confirm
- **Tracks coverage**: reports screens tested, elements exercised, and gaps
- **Targeted workflow testing**: test a specific user journey end-to-end with `--workflow`
- **Bug fix cycles**: reproduce a known bug, fix it, and validate with `--fix`
- **Records GIF** of the test session (optional)

No project-specific configuration needed — it works on any web app.

## Prerequisites

1. [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed
2. [Claude in Chrome](https://chromewebstore.google.com/detail/claude/fcoeoabgfenejglbffodgkkbkcdhcgfn) extension installed and connected
3. Your web app running locally (e.g., `npm run dev`)

> **Note:** The skill will check for the Chrome extension at startup. If it's not connected, it will show setup instructions and stop — it won't proceed without it.

## Install

### Option 1: From the Narai Marketplace (recommended)

Add the Narai marketplace to Claude Code, then install the plugin:

```
/plugin marketplace add narailabs/narai-claude-plugins
/plugin install browser-qa@narai
```

This installs the plugin with automatic updates when new versions are released.

### Option 2: Install plugin directly from GitHub

```
/plugin install narailabs/claude-browser-qa
```

### Option 3: Manual install (copy into your project)

Copy the skill file into your project's `.claude/skills/` directory:

```bash
# From your project root
mkdir -p .claude/skills/browser-qa
curl -o .claude/skills/browser-qa/SKILL.md \
  https://raw.githubusercontent.com/narailabs/claude-browser-qa/main/skills/browser-qa/SKILL.md
```

Or clone and copy:

```bash
git clone https://github.com/narailabs/claude-browser-qa.git /tmp/browser-qa
cp -r /tmp/browser-qa/skills/browser-qa .claude/skills/
```

> **Manual install note:** With this method you won't get automatic updates. Re-run the curl/copy command to update.

## Usage

```
# Broad testing
/browser-qa http://localhost:3000
/browser-qa http://localhost:5173 --mode functional
/browser-qa http://localhost:8080 --mode full --record
/browser-qa http://localhost:3000 --focus "#settings"

# Targeted workflow testing
/browser-qa http://localhost:3000 --workflow "register a new user, then verify the dashboard loads"
/browser-qa http://localhost:3000 --workflow "add item to cart, go to checkout, fill shipping, submit"

# Bug fix cycles
/browser-qa http://localhost:3000 --fix "save button on settings throws a TypeError"
/browser-qa http://localhost:3000 --fix "submitting contact form returns a 500 error"
```

### Modes

| Mode | What It Does |
|------|-------------|
| **smoke** (default) | Navigate each screen, verify DOM + console + network, take screenshots |
| **functional** | Smoke + interact with forms, click buttons, verify state changes |
| **full** | Functional + edge cases + accessibility + performance + responsive testing |
| **workflow** | Test a specific user journey (via `--workflow`), fix bugs, re-run to verify |
| **fix** | Reproduce and fix a known bug (via `--fix`), validate, regression check |

### Options

| Flag | Description |
|------|-------------|
| `--mode` | Testing depth: `smoke`, `functional`, or `full` |
| `--record` | Record a GIF of the test session |
| `--focus "#route"` | Test only a specific route/area |
| `--no-autofix` | Report bugs without attempting to fix them |
| `--workflow "..."` | Test a specific user journey end-to-end |
| `--fix "..."` | Reproduce a known bug, fix it, and validate |
| `--a11y` | Run accessibility checks (included in `full` mode) |
| `--perf` | Run performance awareness checks (included in `full` mode) |
| `--responsive` | Test at mobile, tablet, and desktop breakpoints (included in `full` mode) |
| `--skip-auth` | Skip login, test only public pages |

## How It Works

**Broad testing** (default — no `--workflow` or `--fix`):
1. **Phase 0 — Codebase Recon**: Reads your `CLAUDE.md` and explores the project structure
2. **Phase 1 — Setup**: Opens Chrome, navigates to your app, detects auth and SPA/MPA routing
3. **Phase 2 — Discover**: Maps all screens, navigation links, and interactive elements from the DOM
4. **Phase 3 — Test**: For each screen, runs verification layers (DOM, console, network, visual, functional, accessibility, performance)
5. **Phase 3.5 — Auto-Fix**: When a bug is found, spawns a fix agent with full context, then re-tests to verify
6. **Phase 4 — Report**: Coverage summary, bug reports with screenshots and reproduction steps

**Workflow testing** (`--workflow "..."`):
1. Parses your workflow description into concrete steps
2. Executes each step, checking for errors at every point
3. Fixes bugs found, then re-runs the entire workflow to catch regressions
4. Reports step-by-step pass/fail status

**Bug fix** (`--fix "..."`):
1. Searches the codebase for related code
2. Reproduces the bug in the browser with evidence gathering
3. Spawns a fix agent with full context (up to 2 attempts)
4. Validates the fix and runs a regression check

## Example Output

```
E2E Test Complete
═══════════════════════════════════
Screens: 6/6 tested (100%)
Elements: 45/52 exercised (87%)
Bugs found: 3
  ✅ Fixed inline: 2
  ❌ Still open: 1 (fix failed)

Coverage gaps:
- Settings: API key form skipped (no credentials)
- Tasks: Delete button skipped (destructive, user declined)
```

## Safety

The skill always asks before:
- Clicking destructive buttons (delete, remove, reset)
- Clicking external-action buttons (send, publish, deploy)
- Filling forms that need real credentials

## License

MIT
