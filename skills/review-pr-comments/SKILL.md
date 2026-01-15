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

Analyze and respond to pull request review comments using parallel investigation agents, then resolve threads once addressed.

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
      title
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

Filter to only unresolved threads (`isResolved: false`).

### Step 2: Cluster Threads for Parallel Investigation

Group related threads to optimize investigation and maintain context where it matters.

#### Clustering Algorithm

```
1. Group by file path (primary clustering)
   - All threads on the same file go together
   - These likely need shared context about that file's structure

2. Detect "same issue" patterns (merge clusters)
   - Scan comment bodies for: "same issue", "same here", "ditto", "+1",
     "similar to above", "related to", or matching technical keywords
   - If thread A references thread B's concern, merge their clusters

3. Handle orphan threads
   - Threads that don't cluster with others become single-thread clusters
   - These are investigated independently
```

#### Cluster Data Structure

Build an array of clusters:

```json
[
  {
    "cluster_id": "file:api/handler.go",
    "reason": "Same file (api/handler.go)",
    "threads": [
      {
        "id": "PRRT_123",
        "path": "api/handler.go",
        "line": 42,
        "comments": [
          {"id": "PRRC_456", "body": "Missing nil check", "author": "reviewer1"}
        ]
      },
      {
        "id": "PRRT_789",
        "path": "api/handler.go",
        "line": 67,
        "comments": [
          {"id": "PRRC_012", "body": "Same issue here", "author": "reviewer1"}
        ]
      }
    ]
  },
  {
    "cluster_id": "file:repo/query.sql",
    "reason": "Same file (repo/query.sql)",
    "threads": [
      {
        "id": "PRRT_345",
        "path": "repo/query.sql",
        "line": 15,
        "comments": [
          {"id": "PRRC_678", "body": "Missing index on email column", "author": "reviewer2"}
        ]
      }
    ]
  }
]
```

### Step 3: Launch Parallel Investigation Agents

For each cluster, launch a Task agent to investigate. Use parallel Task calls for efficiency.

#### Agent Definition

The investigation agent is defined in `agents/investigate-cluster/AGENT.md`. Read this file to understand the agent's expected behavior and output format.

#### Agent Dispatch

Use the Task tool with `subagent_type: "general-purpose"` for each cluster:

```
For each cluster in clusters:
  Launch Task with:
    - prompt: See template below
    - subagent_type: "general-purpose"
```

**Launch all agents in parallel** by making multiple Task tool calls in a single response.

#### Agent Prompt Template

```
Read agents/investigate-cluster/AGENT.md for your instructions.

Investigate this cluster of PR review comments:

{cluster_json}

Return your findings as JSON per the AGENT.md output format.
```

### Step 4: Aggregate Results

Collect findings from all agents and merge into a unified results array.

Parse the JSON output from each agent's response and combine:

```json
{
  "all_findings": [
    // ... findings from all clusters merged
  ],
  "summary": {
    "total_threads": 4,
    "resolved": 2,
    "valid": 1,
    "acceptable": 1,
    "needs_discussion": 0
  }
}
```

### Step 5: Present Findings to User

Display a summary table with all findings:

```
| # | File | Issue | Status | Proposed Response |
|---|------|-------|--------|-------------------|
| 1 | api/handler.go:42 | Missing nil check | RESOLVED | Already handled by GetParams() |
| 2 | api/handler.go:67 | Same issue here | RESOLVED | Same as above - GetParams() handles this |
| 3 | repo/query.sql:15 | Missing index | VALID | Need to add migration |
| 4 | utils/cache.go:23 | Race condition | ACCEPTABLE | Single-threaded access pattern |
```

Show summary counts:
```
Summary: 4 threads (2 RESOLVED, 1 VALID, 1 ACCEPTABLE)
```

Ask user: "Do you want me to reply to these comments and resolve the threads marked RESOLVED/ACCEPTABLE?"

### Step 6: Reply to Comments

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

### Step 7: Resolve Threads

After replying, resolve threads using GraphQL:

```bash
gh api graphql -f query='
mutation {
  resolveReviewThread(input: {threadId: "THREAD_ID"}) {
    thread { isResolved }
  }
}'
```

Batch multiple resolutions in one mutation:

```bash
gh api graphql -f query='
mutation {
  t1: resolveReviewThread(input: {threadId: "ID1"}) { thread { isResolved } }
  t2: resolveReviewThread(input: {threadId: "ID2"}) { thread { isResolved } }
}'
```

## Output Format

After completing all actions, summarize:

```
## PR Review Comments - Summary

**PR:** #1556 - feat: Add summary API endpoints
**Threads processed:** 4
**Clusters investigated:** 2 (parallel)
**Resolved:** 3
**Needs action:** 1

### Actions Taken
- Investigated 2 clusters in parallel
- Replied to 4 review comments
- Resolved 3 threads (issues already addressed or acceptable)
- 1 thread requires code changes (see below)

### Outstanding Items
- [ ] Add index on `users.email` (thread #2)
```
