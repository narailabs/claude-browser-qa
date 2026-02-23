# CLAUDE.md - Claude Browser QA

## Overview

This repo contains a Claude Code skill: `/e2e-test` — an AI-powered QA agent that uses Chrome MCP browser tools to intelligently browse web applications, discover all screens and interactive elements, test them systematically, and automatically fix bugs it finds.

## Repo Structure

```
claude-browser-qa/
├── CLAUDE.md                    # This file
├── README.md                    # Installation and usage guide
└── .claude/
    └── skills/
        └── e2e-test/
            └── SKILL.md         # The complete skill definition
```

## How Skills Work

Claude Code skills are prompt documents stored at `.claude/skills/<name>/SKILL.md`. They are invoked via `/skill-name [args]` in Claude Code. This skill requires the **Claude in Chrome** extension to be connected.

## No Code

This is a prompt-only project. There is no source code to build, test, or lint. The skill is a markdown file with YAML frontmatter — all intelligence comes from the instructions + Claude's reasoning + Chrome MCP tools.
