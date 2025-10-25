# Behavioral Assertions

This document defines testable behavioral assertions that specify how Task-Tactician must behave. These assertions guide implementation, testing, and validation.

## Event Processing Assertions

### EP-1: Event Idempotency

**Assertion**: Processing the same event multiple times produces identical results.

**Given**: A workflow event with message ID "msg-123" for issue #456
**When**: Event is processed successfully
**And**: Same event with message ID "msg-123" is received again
**Then**: Deduplication check identifies duplicate
**And**: Event is acknowledged without reprocessing
**And**: System state remains unchanged

**Acceptance Criteria**:

- Same message ID detected within deduplication window (10 minutes)
- No duplicate GitHub API calls made
- Logs indicate duplicate detection with correlation ID

### EP-2: Ordered Event Processing per Repository

**Assertion**: Events for the same repository process in the order received.

**Given**: Three events for repository "owner/repo" with sequence: E1, E2, E3
**When**: Events arrive in message queue
**Then**: E1 completes processing before E2 starts
**And**: E2 completes processing before E3 starts
**And**: Order maintained regardless of processing time

**Acceptance Criteria**:

- Message queue session key set to repository identifier
- Event timestamps in logs show sequential processing
- Concurrent events for different repositories allowed

### EP-3: Event Validation

**Assertion**: Invalid event payloads are rejected without processing.

**Given**: Event payload missing required field "repository.name"
**When**: Event Processor attempts validation
**Then**: Validation fails with specific error message
**And**: Event moved to dead letter queue
**And**: Error logged with correlation ID and validation details
**And**: No workflow processing attempted

**Acceptance Criteria**:

- All required fields validated before processing
- Validation errors descriptive (field name, reason)
- Dead letter queue receives rejected events

### EP-4: Event Processing Timeout

**Assertion**: Long-running event processing does not block queue.

**Given**: Event processing takes longer than message lock timeout
**When**: Message lock expires (5 minutes)
**Then**: Message becomes visible to other processors
**And**: Original processor completes or fails gracefully
**And**: Result determines message acknowledgment or retry

**Acceptance Criteria**:

- Message lock renewed during processing if needed
- Timeout handling prevents duplicate processing
- Logs indicate timeout scenario

## Workflow Processing Assertions

### WF-1: Branch Creation on InProgress State

**Assertion**: Branch created when issue transitions to InProgress state.

**Given**: Issue #123 titled "Add user authentication"
**And**: Issue has label "feature"
**And**: No branch exists for issue #123
**When**: Issue labeled with "in-progress"
**Then**: Branch created with name "feat/123-add-user-authentication"
**And**: Branch created from repository default branch (main)
**And**: Issue retains "in-progress" label

**Acceptance Criteria**:

- Branch existence check before creation
- Branch name follows configured pattern
- Graceful handling if branch already exists (log warning, no error)

### WF-2: No Branch Creation on Assignment Alone

**Assertion**: Issue assignment does not trigger branch creation.

**Given**: Issue #456 in Open state
**And**: Configuration allows auto_create_branch
**When**: User assigned to issue #456
**Then**: No branch created
**And**: "in-progress" label not applied (unless configured otherwise)
**And**: Issue remains in Open state

**Acceptance Criteria**:

- Assignment event processed without branch creation
- Workflow requires explicit InProgress transition
- Logs indicate assignment event received but no action taken

### WF-3: Branch Naming with Label Prefix

**Assertion**: Branch prefix determined by issue labels.

**Given**: Issue #789 with label "bug"
**And**: Configuration maps "bug" → "fix/" prefix
**When**: Branch created for issue
**Then**: Branch name is "fix/789-{slug}"
**And**: Default prefix "feat/" not used

**Acceptance Criteria**:

- Label-to-prefix mapping from configuration applied
- Multiple labels: first matching label determines prefix
- No matching labels: default prefix used

### WF-4: Branch Name Slug Generation

**Assertion**: Branch slug generated correctly from issue title.

**Given**: Issue title "Add OAuth2 Integration (with SSO!!!)"
**When**: Branch name generated
**Then**: Slug is "add-oauth2-integration-with-sso"
**And**: Total branch name ≤ 65 characters
**And**: Slug does not end with hyphen
**And**: Only alphanumeric characters and hyphens in slug

**Acceptance Criteria**:

- Non-alphanumeric characters replaced with hyphens
- Consecutive hyphens collapsed to single hyphen
- Slug truncated if necessary to meet length constraint
- Case converted to lowercase

### WF-5: Label Discovery and Fallback

**Assertion**: Workflow adapts to repository's existing labels with fallback to defaults.

**Given**: Repository has labels "in progress", "bug", "enhancement" but not "blocked"
**When**: Label configuration validated for repository
**Then**: "in progress" mapped to InProgress state
**And**: "bug" mapped to fix/ branch prefix
**And**: "enhancement" mapped to feat/ branch prefix
**And**: "blocked" falls back to system default label
**And**: Validated configuration cached

**Acceptance Criteria**:

- Repository labels queried via GitHub API
- Case-insensitive matching for common variants
- Missing required labels use system defaults
- Configuration cached with TTL to reduce API calls

### WF-6: Label Validation Before Application

**Assertion**: Labels validated to exist before attempting to apply.

**Given**: Repository lacks "has-pr" label
**When**: Workflow attempts to apply "has-pr" label
**Then**: Label existence checked first
**And**: If missing, warning logged
**And**: Operation skipped or fallback label used
**And**: Event processing continues

**Acceptance Criteria**:

- Existence check performed before label application
- Missing labels logged as warnings (non-blocking)
- Fallback strategy: use default or skip gracefully

**Note**: GitHub's native PR-issue linking (via "closes #123" in PR description) is sufficient. Task-Tactician does not implement custom PR-issue linking workflows.

### WF-7: Workflow State Precedence

**Assertion**: Blocked state takes precedence over InProgress.

**Given**: Issue #101 with labels "in-progress" and "blocked"
**When**: Issue state resolved
**Then**: State determined as Blocked
**And**: Branch creation skipped (if attempted)
**And**: Workflow actions respect Blocked state

**Acceptance Criteria**:

- Label precedence: Blocked > HasPR > InProgress > Open
- Configuration defines label mappings
- State resolution pure function testable independently

## Configuration Assertions

### CF-1: Configuration Hierarchy

**Assertion**: Repository configuration overrides organization configuration.

**Given**: Organization config sets `auto_create_branch: true`
**And**: Repository config sets `auto_create_branch: false`
**When**: Configuration loaded for repository
**Then**: Merged config has `auto_create_branch: false`
**And**: Repository setting takes precedence

**Acceptance Criteria**:

- Deep merge algorithm applied
- Repository > Organization > System Default precedence
- Nested properties merged correctly

### CF-2: Configuration Caching

**Assertion**: Configuration cached to reduce GitHub API calls.

**Given**: Repository config loaded at T0
**And**: Cache TTL is 5 minutes
**When**: Config requested at T0 + 2 minutes
**Then**: Cached config returned
**And**: No GitHub API call made
**When**: Config requested at T0 + 6 minutes
**Then**: Fresh config fetched from GitHub
**And**: Cache updated

**Acceptance Criteria**:

- Cache keyed by repository identifier
- TTL enforced (5 minutes default)
- Cache invalidation on explicit request

### CF-3: Invalid Configuration Handling

**Assertion**: Invalid configuration does not stop processing.

**Given**: Repository config file has malformed YAML
**When**: Configuration Manager attempts to load config
**Then**: Parse error logged with details
**And**: Organization config checked as fallback
**And**: System defaults used if all sources fail
**And**: Processing continues with defaults

**Acceptance Criteria**:

- YAML parse errors caught and logged
- Schema validation errors caught and logged
- Graceful degradation to defaults
- Warning logs indicate configuration issue

### CF-4: Missing Configuration File

**Assertion**: Missing configuration file uses defaults gracefully.

**Given**: Repository has no `.github/task-tactician.yml` file
**When**: Configuration Manager fetches repository config
**Then**: 404 error logged as info (not error)
**And**: Organization config checked
**And**: System defaults used as final fallback
**And**: Processing continues normally

**Acceptance Criteria**:

- 404 treated as "config not found", not error
- Falls through to next level in hierarchy
- Logs indicate using defaults

## Error Handling Assertions

### EH-1: Transient Error Retry

**Assertion**: GitHub API errors are retried with exponential backoff.

**Given**: GitHub API returns 503 Service Unavailable
**When**: GitHub Client attempts operation
**Then**: Error classified as transient
**And**: Operation retried after 1 second (first retry)
**And**: If fails again, retried after 2 seconds (second retry)
**And**: If fails third time, retried after 4 seconds (third retry)
**And**: After 3 failed retries, error returned to caller
**And**: Jitter applied to retry delays to prevent thundering herd

**Acceptance Criteria**:

- Exponential backoff: delay doubles each retry
- Maximum 3 retry attempts
- Jitter adds randomness (±10%) to delay
- Final failure logged with all retry attempts

### EH-2: Non-Retryable Error Handling

**Assertion**: GitHub API 404 errors not retried.

**Given**: GitHub API returns 404 Not Found for branch check
**When**: GitHub Client classifies error
**Then**: Error marked as non-retryable
**And**: Error returned immediately without retry
**And**: Caller handles NotFound appropriately
**And**: Log entry indicates 404 response

**Acceptance Criteria**:

- 404, 403, 400 errors not retried
- Classification based on HTTP status code
- Error includes context (repository, resource, operation)

### EH-3: Rate Limit Handling

**Assertion**: GitHub API rate limit triggers circuit breaker.

**Given**: GitHub API rate limit at 950/5000 remaining (19%)
**When**: Rate Limiter checks quota before next call
**Then**: Circuit breaker opens (stops new requests)
**And**: Non-urgent operations queued
**And**: Only critical operations allowed through
**And**: Warning logged about approaching rate limit

**Acceptance Criteria**:

- Circuit opens at 20% remaining quota
- Rate limit headers parsed from API responses
- Circuit closes when quota resets (1 hour window)
- Queued operations resume after reset

### EH-4: Dead Letter Queue for Failed Events

**Assertion**: Events that fail after retries move to dead letter queue.

**Given**: Event fails processing 3 times (max retries)
**When**: Final retry exhausted
**Then**: Event moved to dead letter queue
**And**: Failure reason and retry history included
**And**: Alert triggered for operations team
**And**: Processing continues with next event

**Acceptance Criteria**:

- Dead letter queue configured on message queue
- Failure metadata included (error, retry count, timestamps)
- Operations team notified of DLQ entries
- Main queue not blocked by failed events

## GitHub API Interaction Assertions

### GH-1: Branch Existence Check Before Creation

**Assertion**: Branch existence checked before creation attempt.

**Given**: Request to create branch "feat/123-test"
**When**: GitHub Client executes create branch operation
**Then**: GET request to check branch existence first
**And**: If branch exists, log info message and return existing branch
**And**: If branch does not exist, POST request to create branch
**And**: Creation only attempted if check returns 404

**Acceptance Criteria**:

- Idempotent operation: check then create
- Existing branch not error condition (log info, continue)
- Avoids unnecessary API errors

### GH-2: Label Application Idempotency

**Assertion**: Label applied only if not already present.

**Given**: Issue #123 already has "in-progress" label
**When**: Workflow attempts to apply "in-progress" label
**Then**: Current labels fetched from issue
**And**: Label presence checked
**And**: Apply operation skipped (label already present)
**And**: Log indicates label already applied
**And**: No API call made to apply label

**Acceptance Criteria**:

- Current state checked before modification
- Duplicate operations avoided
- Reduces unnecessary API calls

### GH-3: Comment Addition

**Assertion**: Comments added to issues for milestone prompts.

**Given**: Issue #456 transitions to InProgress without milestone
**When**: Workflow detects missing milestone
**Then**: Comment added to issue: "Please assign a milestone to this issue."
**And**: Assignee mentioned in comment
**And**: Comment includes context (issue number, workflow state)

**Acceptance Criteria**:

- Comment text configurable
- Assignee mention included (@username)
- Comment creation logged

## Observability Assertions

### OB-1: Correlation ID Propagation

**Assertion**: Correlation ID propagates through all operations for single event.

**Given**: Event received with correlation ID "corr-abc-123"
**When**: Event processed
**Then**: All log entries include correlation ID "corr-abc-123"
**And**: All GitHub API calls include correlation ID in headers/logs
**And**: Correlation ID enables end-to-end tracing

**Acceptance Criteria**:

- Correlation ID extracted from event or generated
- Included in structured log context
- Searchable in log aggregation tool

### OB-2: Structured Logging

**Assertion**: All logs use consistent structured format.

**Given**: Any operation being logged
**When**: Log entry created
**Then**: Log formatted as JSON
**And**: Contains standard fields: timestamp, level, message, correlation_id
**And**: Contains context fields: repository, resource_type, resource_id, operation
**And**: Error logs include error type and stack trace

**Acceptance Criteria**:

- JSON format for machine parsing
- Required fields always present
- Context fields enable filtering and grouping

### OB-3: Metrics Emission

**Assertion**: Key operations emit metrics for monitoring.

**Given**: Event processing completes successfully
**When**: Metrics recorded
**Then**: Counter incremented: `events_processed_total`
**And**: Histogram recorded: `event_processing_duration_seconds`
**And**: Counter incremented: `workflow_actions_executed_total{action_type="create_branch"}`

**Acceptance Criteria**:

- Metrics emitted for: events processed, errors, API calls, workflow actions
- Metrics include labels for aggregation (repository, action_type, error_type)
- Metrics accessible to monitoring system (Prometheus-compatible)

## Performance Assertions

### PF-1: Event Processing Latency

**Assertion**: Events process within target latency.

**Given**: Standard issue assignment event
**When**: Event processing completes
**Then**: Total duration < 5 seconds (P95)
**And**: GitHub API calls complete < 2 seconds (P95)
**And**: Configuration loading < 500ms (P95, cache hit)

**Acceptance Criteria**:

- Latency measured from event receipt to acknowledgment
- P95 latency tracked in metrics
- Alerts configured for latency threshold breaches

### PF-2: Concurrent Event Processing

**Assertion**: System processes multiple events concurrently.

**Given**: 10 events for different repositories arrive simultaneously
**When**: Events processed
**Then**: All 10 events process in parallel
**And**: No queuing delay between events
**And**: Each event respects repository-level ordering
**And**: Throughput scales with available concurrency

**Acceptance Criteria**:

- Session-based ordering only for same repository
- Different repositories process independently
- Concurrency limited by function runtime scaling

## Security Assertions

### SE-1: GitHub App Authentication

**Assertion**: All GitHub API calls use valid installation tokens.

**Given**: GitHub API call required
**When**: GitHub Client authenticates
**Then**: JWT token generated from private key
**And**: Installation token requested using JWT
**And**: Installation token cached for reuse (max 1 hour)
**And**: API calls use installation token in Authorization header

**Acceptance Criteria**:

- Private key retrieved from secure vault
- JWT token signed correctly with private key
- Installation tokens cached to reduce token requests
- Expired tokens regenerated automatically

### SE-2: Secret Storage

**Assertion**: GitHub App private key stored securely.

**Given**: Application startup
**When**: GitHub Client initializes
**Then**: Private key retrieved from Azure Key Vault
**And**: Managed identity used for Key Vault authentication
**And**: Private key cached in memory only (never persisted)
**And**: Key rotation supported without code changes

**Acceptance Criteria**:

- No secrets in code or configuration files
- Secure vault access via managed identity
- Secrets cached in memory with TTL

## Edge Case Assertions

### EC-1: Branch Already Exists

**Assertion**: Graceful handling when branch creation fails due to existing branch.

**Given**: Issue #123 transitions to InProgress
**And**: Branch "feat/123-test" already exists
**When**: Workflow attempts branch creation
**Then**: Existence check returns branch exists
**And**: Info log recorded (not error)
**And**: No API error generated
**And**: Workflow continues (does not fail)

**Acceptance Criteria**:

- Idempotent check prevents error
- Existing branch not treated as failure
- Workflow completes successfully

### EC-2: Issue Not Found

**Assertion**: Missing issue reference handled gracefully.

**Given**: PR description references "#999" (does not exist)
**When**: PR-Issue Linker validates issue
**Then**: Issue lookup returns 404
**And**: Warning logged: "Issue #999 not found"
**And**: Other valid issue references still processed
**And**: PR processing continues

**Acceptance Criteria**:

- Invalid references logged but do not block processing
- Partial success: valid references processed
- Error context includes PR number and invalid reference

### EC-3: Configuration File Parse Error

**Assertion**: Malformed configuration does not stop event processing.

**Given**: Repository config has invalid YAML syntax
**When**: Configuration Manager loads config
**Then**: Parse error caught and logged
**And**: System defaults used for repository
**And**: Event processing continues
**And**: Warning indicates configuration issue

**Acceptance Criteria**:

- Parse errors do not propagate to event processing
- Defaults provide safe fallback
- Logs include parse error details for debugging

## Enhanced Workflow Assertions (MVP)

### ST-1: Stale Issue Detection

**Assertion**: Issues inactive for configured period are flagged as stale.

**Given**: Issue has no activity (comments, state changes, updates) for 30 days
**And**: Issue does not have exclusion labels ("on-hold", "blocked")
**And**: Repository config enables stale detection
**When**: Stale detection runs
**Then**: "stale" label applied to issue
**And**: Comment added explaining staleness
**And**: Cycle time event NOT recorded (issue not closed)

**Acceptance Criteria**:

- Activity check includes comments, label changes, status updates
- Exclusion labels prevent stale marking
- Comment provides context and next steps
- Can be disabled per repository

### ST-2: Stale Issue Exclusions

**Assertion**: Issues with exclusion labels are not marked stale.

**Given**: Issue inactive for 45 days
**And**: Issue has "blocked" label
**When**: Stale detection runs
**Then**: Issue NOT marked stale
**And**: No stale label applied
**And**: No stale comment added

**Acceptance Criteria**:

- Exclusion labels configurable per repository
- Default exclusions: "on-hold", "blocked", "wip"

### ML-1: Milestone Prompting on Issue Creation

**Assertion**: Issues created without milestone receive prompt to assign one.

**Given**: New issue created without milestone
**And**: Repository has active milestones
**And**: Repository config enables milestone prompting
**When**: Issue creation event processed
**Then**: Comment added suggesting milestone assignment
**And**: Comment lists available milestones
**And**: Event processing continues (non-blocking)

**Acceptance Criteria**:

- Prompt is informational, not mandatory
- Lists currently active milestones
- Does not prompt if no milestones exist
- Can be disabled per repository

### ML-2: No Milestone Prompt if Already Assigned

**Assertion**: Issues created with milestone do not receive prompt.

**Given**: New issue created with milestone assigned
**When**: Issue creation event processed
**Then**: No milestone prompt comment added
**And**: Event processing continues normally

**Acceptance Criteria**:

- Detection checks milestone field at creation
- No duplicate prompts

### DP-1: Dependency Detection and Parsing

**Assertion**: Issue dependencies are detected and recorded.

**Given**: Issue body contains "Depends on #123"
**And**: Issue #123 exists in repository
**When**: Issue creation or update event processed
**Then**: Dependency relationship recorded
**And**: Log entry indicates dependency detected
**And**: Metric recorded for dependency graph

**Acceptance Criteria**:

- Supports keywords: "Depends on", "Blocks", "Blocked by", "Requires"
- Case-insensitive matching
- Multiple dependencies in single issue supported
- Invalid references logged as warnings

### DP-2: Invalid Dependency Reference

**Assertion**: Invalid dependency references are logged but do not block processing.

**Given**: Issue body contains "Depends on #99999"
**And**: Issue #99999 does not exist
**When**: Issue creation event processed
**Then**: Warning logged indicating invalid reference
**And**: Dependency not recorded
**And**: Event processing continues
**And**: No error thrown

**Acceptance Criteria**:

- Invalid references do not stop workflow
- Log includes issue number and invalid reference
- Same-repository validation only (no cross-repo)

### DP-3: Dependency Notation Formats

**Assertion**: Multiple dependency notation formats are recognized.

**Given**: Issue body contains various formats

- "Depends on #123"
- "Blocks #456"
- "Blocked by #789"
- "Requires #012"
**When**: Issue event processed
**Then**: All dependencies detected
**And**: Each recorded with correct relationship type

**Acceptance Criteria**:

- Standardized keywords recognized
- Both "blocks" and "blocked by" supported
- Relationship direction recorded correctly

### CT-1: Cycle Time Recording on Issue Close

**Assertion**: Cycle time is calculated and recorded when issue closes.

**Given**: Issue created at 2025-01-15T10:00:00Z
**And**: Issue closed at 2025-01-20T14:30:00Z
**When**: Issue closed event processed
**Then**: Cycle time calculated as 5 days, 4.5 hours
**And**: Metric recorded with repository and issue type labels
**And**: Log entry includes cycle time value
**And**: Metric queryable for analytics

**Acceptance Criteria**:

- Calculation: closed_at - created_at
- Recorded in seconds for precision
- Tagged with repository, issue type (from labels)
- Stored as metric for aggregation

### CT-2: Cycle Time Not Recorded for Reopened Issues

**Assertion**: Reopening issue does not record duplicate cycle time.

**Given**: Issue previously closed (cycle time already recorded)
**And**: Issue reopened
**When**: Issue reopened event processed
**Then**: No new cycle time metric recorded
**And**: Previous cycle time metric remains
**And**: Log indicates issue reopened

**Acceptance Criteria**:

- Only first closure records cycle time
- Reopening does not overwrite or duplicate
- Analytics can track reopen rate separately

### CT-3: Cycle Time Aggregation Metrics

**Assertion**: Cycle time metrics support percentile aggregation.

**Given**: Multiple issues closed with various cycle times
**When**: Metrics queried for repository
**Then**: P50, P95, P99 percentiles calculable
**And**: Metrics can be filtered by issue type
**And**: Metrics can be grouped by time period

**Acceptance Criteria**:

- Metrics platform supports percentile queries
- Can filter by labels (issue type, priority)
- Can group by day/week/month

## Assertion Testing Strategy

Each assertion above should have:

1. **Unit Test**: For pure logic (branch name generation, state resolution)
2. **Integration Test**: For component interactions (workflow + GitHub client)
3. **Contract Test**: For interface implementations (IGitHubOperations contract)
4. **End-to-End Test**: For complete workflows (event → actions → GitHub)

**Coverage Target**: All behavioral assertions must have corresponding automated tests.
