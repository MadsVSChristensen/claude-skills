---
name: investigate-cluster
description: Internal agent for investigating a cluster of related PR review comments. Launched by coordinator skills, not user-invocable.
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
---

# Investigate Comment Cluster

Investigate a cluster of related PR review comments and return structured findings.

## Input

You will receive a JSON cluster object with this structure:

```json
{
  "cluster_id": "file:api/handler.go",
  "reason": "Same file",
  "threads": [
    {
      "id": "PRRT_123",
      "path": "api/handler.go",
      "line": 42,
      "comments": [
        {
          "id": "PRRC_456",
          "body": "Missing nil check here",
          "author": "reviewer1"
        }
      ]
    }
  ]
}
```

## Investigation Process

For each thread in the cluster:

### 1. Read the Relevant Code

Read the file at the specified path, focusing on the line number mentioned:

```bash
# Read surrounding context (20 lines before and after)
```

Use the Read tool to examine the code at `path` around `line`.

### 2. Verify the Claim

Don't assume the reviewer is correct. Investigate whether the issue actually exists:

| Comment Type | What to Check |
|--------------|---------------|
| Missing functionality | Search codebase for existing implementations |
| Bug/vulnerability | Trace the actual code path to verify |
| Performance concern | Consider data scale, whether optimization is premature |
| Missing tests | Search for existing test coverage |
| Code style/pattern | Check project conventions |

### 3. Check for Existing Solutions

The issue may already be handled elsewhere:
- Utility functions
- Middleware or interceptors
- Parent callers
- Framework guarantees

Use Grep to search for related patterns if needed.

### 4. Categorize the Finding

Assign one of these statuses:

- **RESOLVED** - Issue doesn't exist or is already fixed
- **VALID** - Issue exists and should be addressed
- **ACCEPTABLE** - Issue exists but is acceptable (document why)
- **NEEDS_DISCUSSION** - Requires user input to decide

### 5. Draft a Response

Write a concise response that:
- Directly addresses the reviewer's concern
- Provides evidence (code references, search results)
- Explains the reasoning

## Output Format

Return your findings as a JSON code block:

```json
{
  "cluster_id": "file:api/handler.go",
  "findings": [
    {
      "thread_id": "PRRT_123",
      "comment_id": "PRRC_456",
      "file": "api/handler.go:42",
      "issue": "Missing nil check",
      "status": "RESOLVED",
      "response": "The nil check is handled by GetParams() on line 38, which returns an error if params is nil before this code path is reached.",
      "evidence": "See api/handler.go:38 - GetParams() validates input"
    }
  ]
}
```

## Important Guidelines

1. **Be thorough but focused** - Investigate each comment fully, but stay within the cluster's scope
2. **Provide evidence** - Reference specific lines, files, or search results
3. **Be objective** - If the reviewer is right, say so. If they're wrong, explain why with evidence
4. **Consider context** - Related comments in the same cluster may inform each other
5. **Output valid JSON** - The coordinator will parse your output programmatically
