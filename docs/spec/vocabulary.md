# Domain Vocabulary

This document defines the core concepts and terminology used throughout Task-Tactician's architecture and implementation.

## Core Domain Concepts

### Workflow Event

A normalized representation of a GitHub event received via the message queue.

- **Identified by**: MessageId (unique identifier from message queue), CorrelationId (for tracing)
- **Contains**: Event type, action, timestamp, repository context, affected resource (issue or PR)
- **Lifecycle**: Received → Validated → Processed → Acknowledged
- **Invariants**: Must be idempotent to process; same MessageId processed multiple times produces identical outcome

### Issue

A GitHub issue representing a work item.

- **Identified by**: IssueNumber (unique within repository)
- **Contains**: Title, description, state (open/closed), labels, assignees, milestone
- **State Transitions**: Open → InProgress → HasPR → Closed
- **Business Rules**: Cannot transition to InProgress without assignee; branch creation triggers on InProgress state

### Pull Request

A GitHub pull request proposing code changes.

- **Identified by**: PRNumber (unique within repository)
- **Contains**: Title, description, source branch, target branch, state, linked issues
- **Lifecycle**: Open → Review → Approved → Merged/Closed
- **Business Rules**: Must link to at least one issue; milestone should sync with linked issues

### Branch

A Git branch created for development work.

- **Identified by**: BranchName (unique within repository)
- **Naming Convention**: `{prefix}/{issue-number}-{slug}`
- **Contains**: Prefix (feat, fix, docs, etc.), issue reference, descriptive slug
- **Constraints**: Name ≤ 65 characters; slug derived from issue title; must not already exist

### Workflow State

The current phase of an issue in the development lifecycle.

- **Values**: Open, InProgress, Blocked, HasPR, Done
- **Tracked by**: Issue labels applied by Task-Tactician
- **Transitions**: Triggered by GitHub events (assignment, labeling, PR creation, PR merge)
- **Invariants**: Issue cannot be InProgress and Blocked simultaneously

### Label

A GitHub label providing metadata about issues and PRs.

- **Purpose**: Indicates workflow state, issue type, priority, or other classification
- **Standard Labels**: `in-progress`, `blocked`, `has-pr` (configured per repository)
- **Constraints**: Uses existing repository labels; does not create new labels

### Configuration

Hierarchical settings controlling workflow behavior.

- **Levels** (priority order): Repository → Organization → System Default
- **Contains**: Label names, branch naming patterns, workflow enablement flags
- **Loading**: Retrieved from GitHub repository files (`.github/task-tactician.yml`)
- **Caching**: Cached with TTL to reduce GitHub API calls

### Repository

A GitHub repository where Task-Tactician operates.

- **Identified by**: Owner and repository name
- **Contains**: Default branch name, configuration file location
- **Scope**: Unit of configuration and event processing

### Milestone

A GitHub milestone representing a sprint or release.

- **Identified by**: Milestone number or title
- **Contains**: Title, description, due date, associated issues
- **Business Rules**: Issues should have milestone before moving to InProgress state

## Event Processing Concepts

### Event Deduplication

Preventing duplicate processing of the same event.

- **Mechanism**: Track processed MessageIds within TTL window
- **Key**: Combination of webhook delivery ID and repository ID
- **Storage**: In-memory cache (no persistent state)
- **Purpose**: Ensures idempotent behavior for at-least-once delivery semantics

### Ordered Processing

Ensuring events for the same issue/PR process in sequence.

- **Mechanism**: Message queue sessions keyed by repository
- **Guarantee**: Events for same repository process in order received
- **Purpose**: Prevents race conditions in workflow state transitions

### Idempotent Operation

An operation that produces the same result regardless of how many times it's executed.

- **Examples**: Creating a branch (check existence first), applying a label (check if already applied)
- **Pattern**: Check current state → Compare with desired state → Apply changes only if different
- **Error Recovery**: Enables safe retry on transient failures

## Workflow Concepts

### Issue Assignment

GitHub event where a user is assigned to an issue.

- **Trigger**: `issues.assigned` event
- **Workflow Actions**: May apply `in-progress` label, may create branch (if configured)
- **Business Rule**: Assignment alone does not trigger workflow progression; used for triage purposes

### Issue In-Progress

State indicating active development work on an issue.

- **Detection**: Issue labeled with `in-progress` OR moved to "In Progress" project board column
- **Workflow Actions**: Create feature branch if none exists, ensure assignee present
- **Constraints**: Milestone should be assigned before transition

### Branch Creation

Automated creation of a feature branch for an issue.

- **Trigger**: Issue transitions to InProgress state and no branch exists
- **Naming**: Follows configured pattern with prefix, issue number, and slug
- **Prefix Mapping**: Determined by issue labels (bug → fix/, feature → feat/) with fallback to defaults
- **Source**: Created from repository's default branch (main/master)
- **Idempotency**: Check existence before creation; log if already exists

### Label Discovery

Process of identifying available labels in a repository and mapping them to workflow states.

- **Purpose**: Adapt to repository's existing label schema rather than requiring specific labels
- **Discovery**: Query repository labels via GitHub API
- **Mapping**: Match discovered labels to workflow states (InProgress, Blocked, etc.)
- **Fallback**: Use system default labels if preferred labels don't exist
- **Caching**: Cache validated label configuration to reduce API calls
- **Examples**: 
  - Repository has "in progress" → maps to InProgress state
  - Repository lacks "blocked" → falls back to system default "blocked"

### Label Validation

Verification that required workflow labels exist in repository.

- **Required Labels**: Labels needed for workflow state transitions
- **Validation**: Check existence before attempting to apply
- **Missing Labels**: Log warning and use closest alternative or skip operation
- **Default Labels**: System-provided fallbacks when repository labels unavailable

## Error Handling Concepts

### Transient Error

Temporary failure that may succeed on retry.

- **Examples**: GitHub API rate limit, network timeout, service unavailable (5xx)
- **Response**: Exponential backoff retry (max 3 attempts)
- **Dead Letter**: Move to dead letter queue after exhausting retries

### Configuration Error

Error due to invalid or missing configuration.

- **Examples**: Malformed YAML, missing required field, invalid regex pattern
- **Response**: Log error, use system defaults, continue processing
- **Alert**: Notify operations team for manual intervention

### Business Logic Error

Error due to workflow rule violation or invalid state.

- **Examples**: Branch already exists, issue lacks milestone, circular dependency
- **Response**: Log warning with context, skip operation, mark event as processed
- **Outcome**: Does not block other events; provides debugging information

### GitHub API Error

Error from GitHub API operations.

- **Categories**: Not Found (404), Forbidden (403), Rate Limit (429), Server Error (5xx)
- **Handling**: Distinguish between retryable (5xx, 429) and non-retryable (404, 403)
- **Context**: Include repository, resource type, operation for debugging

## Data Flow Concepts

### Event Payload

Structured data describing a GitHub event.

- **Format**: JSON matching GitHub webhook schema
- **Envelope**: Message metadata (ID, correlation ID, timestamp)
- **Validation**: Schema validation before processing
- **Content**: Event type, action, resource data (issue/PR), repository context

### Configuration Hierarchy

Layered configuration where specific settings override general ones.

- **Merge Strategy**: Deep merge with repository config taking precedence
- **Resolution**: Check repository → Check organization → Use system default
- **Caching**: Cache merged result per repository with TTL
- **Invalidation**: Cache expires after TTL or on configuration change event

### Workflow Action

Discrete operation executed as part of workflow processing.

- **Types**: CreateBranch, ApplyLabel, RemoveLabel, AddComment, UpdateMilestone
- **Execution**: Applied in sequence after event processing determines required actions
- **Idempotency**: Each action checks current state before executing
- **Logging**: All actions logged with correlation ID for audit trail

## Integration Concepts

### GitHub App Authentication

OAuth-based authentication for GitHub API access.

- **Mechanism**: JWT token generated from private key → Installation token request
- **Scope**: Installation token scoped to specific organization/repository
- **Lifespan**: Installation tokens expire after 1 hour; regenerate as needed
- **Storage**: Private key in secure vault; installation tokens cached in memory

### Message Queue Session

Mechanism for ordered message delivery within a partition.

- **Session Key**: Repository identifier
- **Guarantee**: Messages with same session key process in order
- **Timeout**: Session locks timeout after 5 minutes to prevent deadlock
- **Use Case**: Ensures sequential processing of events for same repository

### Rate Limit Management

Handling GitHub API rate limits to prevent throttling.

- **Primary Limit**: 5,000 requests/hour per installation
- **Strategy**: Track remaining requests, implement circuit breaker at 20% remaining
- **Backoff**: Exponential backoff with jitter on 429 responses
- **Priority**: Queue non-urgent operations to preserve quota for critical actions

## Observability Concepts

### Correlation ID

Unique identifier tracking a request through the system.

- **Source**: Generated from incoming event or message ID
- **Propagation**: Included in all log entries and API calls for single event
- **Purpose**: Enables tracing event processing across components
- **Format**: UUID v4

### Structured Logging

Logging with consistent, machine-parseable format.

- **Format**: JSON with standard fields (timestamp, level, message, context)
- **Context**: Includes correlation ID, repository, resource type, operation
- **Levels**: Debug, Info, Warn, Error
- **Destination**: Application Insights / CloudWatch Logs

### Metrics

Quantitative measurements of system behavior.

- **Categories**: Event processing throughput, API call rates, error rates, latency
- **Aggregation**: Time-series data with percentiles (P50, P95, P99)
- **Alerting**: Thresholds configured for operational alerts
- **Purpose**: Performance monitoring and capacity planning

## Enhanced Workflow Concepts (MVP)

### Stale Issue

Issue that has had no activity for configured period.

- **Detection**: No comments, state changes, or updates for N days (configurable, default 30)
- **Action**: Apply "stale" label to flag for triage
- **Exclusions**: Issues with specific labels (e.g., "on-hold", "blocked") not marked stale
- **Purpose**: Surface inactive issues for review/closure

### Milestone

Target release or timeframe for issue completion.

- **Representation**: GitHub milestone assigned to issue
- **Prompting**: When issue created without milestone, add comment suggesting assignment
- **Validation**: Milestone must exist in repository
- **Configuration**: Can disable prompting per repository

### Issue Dependency

Relationship indicating one issue depends on another.

- **Notation**: Special syntax in issue body (e.g., "Depends on #123", "Blocks #456")
- **Detection**: Parse issue description for dependency keywords
- **Validation**: Referenced issues must exist in same repository
- **Purpose**: Track work dependencies for planning

### Cycle Time

Duration from issue creation to closure.

- **Measurement**: Timestamp of "closed" event minus timestamp of "opened" event
- **Recording**: Logged as metric when issue closes
- **Aggregation**: Calculate P50, P95, P99 across repository
- **Purpose**: Track team velocity and identify bottlenecks

## Business Rules Summary

1. **Issue assignment alone does not trigger branch creation** - only InProgress state does
2. **InProgress state detected by label or project board column** - not by assignee
3. **Branch naming follows configurable pattern** - prefix based on issue type labels
4. **Configuration hierarchy**: Repository > Organization > System defaults
5. **All operations must be idempotent** - safe to replay events
6. **Labels must exist in repository** - Task-Tactician does not create new labels
7. **PR-issue linking supports multiple detection methods** - branch name, title, description
8. **Workflow state transitions are event-driven** - no polling or scheduled jobs
9. **Stale issues flagged after configured inactivity period** - default 30 days
10. **Milestone prompting occurs on issue creation** - non-blocking suggestion
11. **Dependencies tracked via issue description parsing** - simple keyword detection
12. **Cycle time recorded on issue closure** - for analytics and planning
