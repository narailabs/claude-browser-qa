# Claude Browser QA

AI-powered QA agent for Claude Code. Intelligently browses your web app, discovers every screen and interactive element, tests them systematically, and automatically fixes bugs it finds.

## What It Does

- **Auto-discovers** all navigation routes, forms, buttons, and interactive elements from the DOM
- **Tests systematically** with three depth modes: smoke, functional, and full
- **Verifies everything**: DOM content, console errors, network failures, visual rendering
- **Fixes bugs inline** by spawning fix agents when issues are found, then re-tests to confirm
- **Tracks coverage**: reports screens tested, elements exercised, and gaps
- **Records GIF** of the test session (optional)

No project-specific configuration needed — it works on any web app.

## Prerequisites

1. [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed
2. [Claude in Chrome](https://chromewebstore.google.com/detail/claude-in-chrome/) extension connected
3. Your web app running locally (e.g., `npm run dev`)

## Install

Copy the skill into your project:

```bash
# From your project root
mkdir -p .claude/skills/e2e-test
curl -o .claude/skills/e2e-test/SKILL.md \
  https://raw.githubusercontent.com/narailabs/claude-browser-qa/main/.claude/skills/e2e-test/SKILL.md
```

Or clone and copy:

```bash
git clone https://github.com/narailabs/claude-browser-qa.git /tmp/browser-qa
cp -r /tmp/browser-qa/.claude/skills/e2e-test .claude/skills/
```

## Usage

```
/e2e-test http://localhost:3000
/e2e-test http://localhost:5173 --mode functional
/e2e-test http://localhost:8080 --mode full --record
/e2e-test http://localhost:3000 --focus "#settings"
/e2e-test http://localhost:3000 --no-autofix
```

### Modes

| Mode | What It Does |
|------|-------------|
| **smoke** (default) | Navigate each screen, verify DOM + console + network, take screenshots |
| **functional** | Smoke + interact with forms, click buttons, verify state changes |
| **full** | Functional + edge cases (empty inputs, special characters, rapid navigation, data persistence) |

### Options

| Flag | Description |
|------|-------------|
| `--mode` | Testing depth: `smoke`, `functional`, or `full` |
| `--record` | Record a GIF of the test session |
| `--focus "#route"` | Test only a specific route/area |
| `--no-autofix` | Report bugs without attempting to fix them |

## How It Works

1. **Phase 0 — Codebase Recon**: Reads your `CLAUDE.md` and explores the project structure to understand the tech stack
2. **Phase 1 — Setup**: Opens Chrome, navigates to your app, takes a baseline screenshot
3. **Phase 2 — Discover**: Maps all screens, navigation links, and interactive elements from the DOM
4. **Phase 3 — Test**: For each screen, runs 5 verification layers (DOM, console, network, visual, functional)
5. **Phase 3.5 — Auto-Fix**: When a bug is found, spawns a fix agent with full context, then re-tests to verify
6. **Phase 4 — Report**: Coverage summary, bug reports with screenshots and reproduction steps

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
