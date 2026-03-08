# CLAUDE.md - Claude Browser QA

## Overview

This repo contains a Claude Code plugin: `/browser-qa` — an AI-powered QA agent that uses Chrome browser tools to intelligently browse web applications, discover all screens and interactive elements, test them systematically, and automatically fix bugs it finds. Also supports targeted workflow testing (`--workflow`) and bug fix cycles (`--fix`).

## Repo Structure

```
claude-browser-qa/
├── CLAUDE.md                    # This file
├── README.md                    # Installation and usage guide
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest (name, version, author)
├── skills/
│   └── browser-qa/
│       ├── SKILL.md             # Main skill definition (plugin copy)
│       └── reference/           # Detailed procedures loaded on demand
│           ├── testing-layers.md       # Functional testing procedures
│           ├── accessibility.md        # WCAG accessibility checks
│           ├── performance.md          # Performance observation checks
│           ├── responsive.md           # Viewport + dark mode testing
│           ├── fix-agents.md           # Fix agent templates & guidelines
│           ├── workflow-mode.md        # --workflow mode procedures
│           ├── fix-mode.md             # --fix mode procedures
│           ├── interaction-protocol.md # Wait-for-ready & retry logic
│           └── reporting.md            # Report templates for all modes
├── .claude/
│   └── skills/
│       └── browser-qa/          # Local dev copy (kept in sync with skills/)
│           ├── SKILL.md
│           └── reference/
└── .github/
    └── workflows/
        └── claude.yml           # Claude Code GitHub Action
```

## How It Works

This repo is structured as a **Claude Code plugin** distributed via the **Narai marketplace**.

- **Plugin**: Defined by `.claude-plugin/plugin.json`. The skill lives in `skills/browser-qa/SKILL.md`.
- **Marketplace**: Listed in the separate [narailabs/narai-claude-plugins](https://github.com/narailabs/narai-claude-plugins) marketplace repo. Users add it with `/plugin marketplace add narailabs/narai-claude-plugins` and install with `/plugin install browser-qa@narai`.
- **Manual install**: Users can also copy `skills/browser-qa/SKILL.md` into their project's `.claude/skills/browser-qa/` directory.
- **Local dev**: The `.claude/skills/` copy lets you test the skill locally in this repo. Keep it in sync with `skills/browser-qa/SKILL.md`.

## Prerequisites

This skill requires the **Claude in Chrome** browser extension. The skill checks for it at startup and will block execution with setup instructions if it's not connected.

## No Code

This is a prompt-only project. There is no source code to build, test, or lint. The skill is a markdown file with YAML frontmatter — all intelligence comes from the instructions + Claude's reasoning + Chrome browser tools.
