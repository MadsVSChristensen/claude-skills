# Changelog

All notable changes to this project will be documented in this file.

## [1.1.0] - 2026-01-15

### Added
- **agents/** folder for reusable agent definitions
- `investigate-cluster` agent for investigating PR comment clusters

### Changed
- `review-pr-comments` skill now uses parallel cluster-based investigation:
  - Groups related comments by file path and semantic similarity
  - Launches parallel agents per cluster for faster processing
  - Maintains shared context for related comments (e.g., "same issue here")

## [1.0.0] - 2026-01-14

### Added
- Initial marketplace structure following Anthropic skills format
- **pr-workflow-skills** plugin with:
  - `review-pr-comments` skill - Analyze, respond to, and resolve PR review comments
