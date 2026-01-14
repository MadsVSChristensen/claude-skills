# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Claude Code skills marketplace - a collection of reusable automation skills that Claude can invoke via slash commands. Skills are Markdown files with YAML frontmatter containing instructions for Claude to follow.

## Architecture

```
Marketplace (mads-claude-skills)
└── Plugin (pr-workflow-skills)
    └── Skill (review-pr-comments)
```

- **Marketplace**: Top-level container defined in `.claude-plugin/marketplace.json`
- **Plugins**: Groups of related skills (e.g., PR workflows, code analysis)
- **Skills**: Individual SKILL.md files in `skills/<skill-name>/` directories

## Adding New Skills

1. Create `skills/<skill-name>/SKILL.md` with YAML frontmatter:
   ```yaml
   ---
   name: skill-name
   description: When Claude should use this skill
   allowed-tools:
     - Bash
     - Read
   ---
   ```

2. Register the skill in `.claude-plugin/marketplace.json` under the appropriate plugin's `skills` array

## Skill File Format

Skills are instruction sets for Claude, not compiled code. Each SKILL.md contains:
- YAML frontmatter with `name`, `description`, and `allowed-tools`
- Step-by-step workflow instructions
- Example commands and expected outputs
- Investigation guidelines for decision-making

## Installation

```
/plugin marketplace add MadsVSChristensen/claude-skills
/plugin install pr-workflow-skills@mads-claude-skills
```

## Key Patterns

- Skills use `gh api` (GitHub CLI) for GitHub integration, preferring GraphQL for complex operations
- User approval checkpoints before making changes (AskUserQuestion tool)
- Summary tables for presenting analysis results
- Status categorization (RESOLVED, VALID, ACCEPTABLE, NEEDS_DISCUSSION) for review workflows
