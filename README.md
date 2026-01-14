# Mads' Claude Skills Marketplace

Personal collection of Claude Code skills for PR reviews, code analysis, and workflow automation.

## Installation

Install from the marketplace:

```bash
claude mcp add-skill-marketplace /path/to/claude-skills
```

Or install individual plugins:

```bash
claude plugins add /path/to/claude-skills --plugin pr-workflow-skills
```

## Available Skills

### `/review-pr-comments`

Analyze, respond to, and resolve PR review comments.

**Usage:**
```
/review-pr-comments 1556
/review-pr-comments https://github.com/owner/repo/pull/1556
```

**What it does:**
1. Fetches all unresolved review threads on the PR
2. Investigates each comment against the codebase
3. Categorizes findings (resolved, valid, acceptable, needs discussion)
4. Presents summary for user approval
5. Replies to comments with appropriate responses
6. Resolves threads that are addressed

**Example workflow:**
```
User: /review-pr-comments 1556

Claude: Found 4 unresolved review threads. Analyzing...

| Thread | File | Issue | Status |
|--------|------|-------|--------|
| 1 | api/handler.go:42 | Missing nil check | RESOLVED |
| 2 | repo/query.sql:15 | Correlated subquery | ACCEPTABLE |

Reply and resolve these threads? [Yes/No]

User: Yes

Claude: Done. Replied to 4 comments and resolved 4 threads.
```

## Adding New Skills

1. Create a new folder under `skills/` with a `SKILL.md` file:

```
skills/
└── my-new-skill/
    └── SKILL.md
```

2. Add the skill to the appropriate plugin in `.claude-plugin/marketplace.json`:

```json
{
  "plugins": [
    {
      "name": "pr-workflow-skills",
      "skills": [
        "./skills/review-pr-comments",
        "./skills/my-new-skill"
      ]
    }
  ]
}
```

Or create a new plugin group for unrelated skills.

### SKILL.md Format

```yaml
---
name: my-skill-name
description: What it does and when Claude should use it
allowed-tools:
  - Bash
  - Read
---

# My Skill Name

Instructions for Claude to follow...
```

## License

MIT
