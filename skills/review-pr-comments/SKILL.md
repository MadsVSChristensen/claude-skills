---
name: review-pr-comments
description: Analyze, respond to, and resolve PR review comments. Use when user wants to address reviewer feedback, reply to PR comments, resolve review threads, or investigate whether review comments are valid.
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
  - Task
  - AskUserQuestion
---

# Review PR Comments

Analyze and respond to pull request review comments, then resolve threads once addressed.

## Arguments

The skill accepts a PR number as the argument:
- `/review-pr-comments 1556` - Process PR #1556
- `/review-pr-comments https://github.com/owner/repo/pull/1556` - Process from URL

## Workflow

### Step 1: Fetch Unresolved Review Threads

Use GraphQL to get all unresolved review threads:

```bash
gh api graphql -f query='
{
  repository(owner: "OWNER", name: "REPO") {
    pullRequest(number: PR_NUMBER) {
      reviewThreads(first: 50) {
        nodes {
          id
          isResolved
          comments(first: 10) {
            nodes {
              id
              body
              author { login }
              createdAt
              path
              line
            }
          }
        }
      }
    }
  }
}'
```

Extract owner/repo from:
1. The argument if it's a URL
2. Otherwise, use `gh repo view --json owner,name`

### Step 2: Analyze Each Comment

For each unresolved thread:

1. **Read the relevant code** at the file path and line mentioned
2. **Investigate the claim**:
   - If it mentions a missing index, check if the index exists in migrations
   - If it mentions a bug, check if the code actually has that bug
   - If it mentions a pattern issue, evaluate if the concern is valid at current scale
3. **Categorize the finding**:
   - `RESOLVED` - Issue doesn't exist or is already fixed
   - `VALID` - Issue exists and should be addressed
   - `ACCEPTABLE` - Issue exists but is acceptable (document why)
   - `NEEDS_DISCUSSION` - Requires user input

### Step 3: Present Findings to User

Display a summary table:

```
| Thread | File | Issue | Status | Proposed Response |
|--------|------|-------|--------|-------------------|
| 1 | api/handler.go:42 | Missing nil check | RESOLVED | Already handled by GetParams() |
| 2 | repo/query.sql:15 | Missing index | VALID | Need to add migration |
```

Ask user: "Do you want me to reply to these comments and resolve the threads marked RESOLVED/ACCEPTABLE?"

### Step 4: Reply to Comments

For each thread the user approves, reply using:

```bash
gh api repos/OWNER/REPO/pulls/PR_NUMBER/comments/COMMENT_ID/replies \
  -X POST \
  -f body="**Status:** Your response here"
```

Format responses based on status:
- `RESOLVED`: "**Resolved:** [explanation of why this is not an issue]"
- `VALID`: "**Acknowledged:** [what will be done to fix it]"
- `ACCEPTABLE`: "**Acceptable:** [reasoning for why this is OK as-is]"

### Step 5: Resolve Threads

After replying, resolve threads using GraphQL:

```bash
gh api graphql -f query='
mutation {
  resolveReviewThread(input: {threadId: "THREAD_ID"}) {
    thread { isResolved }
  }
}'
```

You can batch multiple resolutions in one mutation:

```bash
gh api graphql -f query='
mutation {
  t1: resolveReviewThread(input: {threadId: "ID1"}) { thread { isResolved } }
  t2: resolveReviewThread(input: {threadId: "ID2"}) { thread { isResolved } }
}'
```

## Investigation Guidelines

When analyzing review comments:

### General Approach
1. **Read the relevant code** - Always read the file and line mentioned in the comment
2. **Verify the claim** - Don't assume the reviewer is correct; investigate whether the issue actually exists
3. **Check for existing solutions** - The issue may already be handled elsewhere (utility functions, middleware, other layers)
4. **Consider context** - Evaluate whether the concern is valid at the current scale/usage

### Common Investigation Patterns

| Comment Type | What to Check |
|--------------|---------------|
| Missing functionality | Search codebase for existing implementations |
| Bug/vulnerability | Trace the actual code path to verify the claim |
| Performance concern | Consider data scale, pagination, and whether optimization is premature |
| Missing tests | Search for existing test coverage |
| Code style/pattern | Check project conventions and consistency |

### Key Questions to Answer
- Does the issue actually exist, or is it already handled?
- If it exists, is it a real problem at current scale?
- What's the cost/benefit of addressing it now vs later?

## Output Format

After completing all actions, summarize:

```
## PR Review Comments - Summary

**PR:** #1556 - feat: Add summary API endpoints
**Threads processed:** 4
**Resolved:** 3
**Needs action:** 1

### Actions Taken
- Replied to 4 review comments
- Resolved 3 threads (issues already addressed or acceptable)
- 1 thread requires code changes (see below)

### Outstanding Items
- [ ] Add index on `users.email` (thread #2)
```
