# Claude Browser QA

AI-powered QA agent for Claude Code. Intelligently browses your web app, discovers every screen and interactive element, tests them systematically, and automatically fixes bugs it finds.

## What It Does

- **Auto-discovers** all navigation routes, forms, buttons, and interactive elements from the DOM
- **CRUD lifecycle testing**: Creates items, verifies them, edits, deletes — full data interaction
- **Action verification**: Every button click verified with pre/post state comparison
- **Permutation testing**: Tests all dropdown options, toggle states, form variations, tabs, and flow branches
- **Expectations validation**: Analyzes docs and code to build expectations, then validates each with dedicated subagents
- **Verifies everything**: DOM content, console errors, network failures, visual rendering, accessibility, performance
- **Fixes bugs inline** by spawning fix agents when issues are found, then re-tests to confirm
- **Coverage tracking**: 4-dimension coverage (elements, CRUD, actions, permutations) targeting ≥90%
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
mkdir -p .claude/skills/browser-qa/reference
curl -o .claude/skills/browser-qa/SKILL.md \
  https://raw.githubusercontent.com/narailabs/claude-browser-qa/main/skills/browser-qa/SKILL.md
# Also download reference files for full functionality
for f in testing-layers accessibility performance responsive fix-agents expectations-validation workflow-mode fix-mode interaction-protocol reporting; do
  curl -o .claude/skills/browser-qa/reference/$f.md \
    https://raw.githubusercontent.com/narailabs/claude-browser-qa/main/skills/browser-qa/reference/$f.md
done
```

> **Manual install note:** With this method you won't get automatic updates. Re-run the commands to update.

## Usage

```
# Broad testing (prompts for mode selection — Full recommended)
/browser-qa http://localhost:3000
/browser-qa http://localhost:5173 --mode full
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
| **full** (default) | CRUD lifecycle + action verification + permutations + expectations validation + a11y + perf + responsive + dark mode. Targets ≥90% coverage. |
| **functional** | CRUD lifecycle + action verification + form validation + navigation testing. No permutations or a11y/perf/responsive. |
| **smoke** | Navigate each screen, verify DOM + console + network, take screenshots. No interactions. |
| **workflow** | Test a specific user journey (via `--workflow`), fix bugs, re-run to verify |
| **fix** | Reproduce and fix a known bug (via `--fix`), validate, regression check |

If `--mode` is not explicitly set, the skill prompts you to choose (Full is recommended).

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
1. **Phase 1 — Codebase Recon**: Reads your docs and explores the project structure
2. **Phase 2 — Setup**: Opens Chrome, navigates to your app, detects auth and SPA/MPA routing
3. **Phase 3 — Discover**: Maps all screens, navigation links, and interactive elements
4. **Phase 4 — Expectations Validation**: Analyzes code/docs to build expectations, validates each with dedicated subagents that test and auto-fix
5. **Phase 5 — Test**: CRUD lifecycle, action verification, form validation, permutations, a11y, performance
6. **Phase 6 — Auto-Fix**: Spawns fix agents for runtime bugs, re-tests to verify
7. **Phase 7 — Responsive**: Tests at 4 breakpoints + dark mode
8. **Phase 8 — Report**: Coverage summary, CRUD results, bug reports, expectations results

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

Functional Coverage: 92%
  Element: 45/52 (87%) | CRUD: 4/4 (100%) | Actions: 18/20 (90%) | Permutations: 28/35 (80%)

Expectations validated: 12/15
  ✅ Pass: 10 | 🔧 Fixed: 1 | ❌ Fail: 1

Runtime bugs found: 3
  ✅ Fixed inline: 2
  ❌ Still open: 1

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
