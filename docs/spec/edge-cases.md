# Edge Cases and Error Scenarios

This document catalogs non-standard flows, edge cases, and error scenarios that Task-Tactician must handle gracefully.

## Event Processing Edge Cases

### EC-1: Duplicate Event Delivery

**Scenario**: Same event delivered multiple times due to message queue retry

**Cause**: At-least-once delivery guarantee from Service Bus

**Detection**: Message ID matches entry in deduplication cache

**Handling**:

- Check deduplication cache before processing
- If duplicate found: acknowledge message, skip processing, log info
- If not duplicate: add to cache, process normally

**Outcome**: No duplicate operations; idempotent behavior

**Test Coverage**: Unit test for deduplication logic, integration test with replayed messages

---

### EC-2: Out-of-Order Event Delivery

**Scenario**: Events for same issue arrive out of chronological order

**Example**: "issue.labeled" event arrives before "issue.assigned" event

**Cause**: Network delays, queue processing variations

**Handling**:

- Session-based ordering ensures in-order delivery per repository
- Current state fetched from GitHub for each event (source of truth)
- Operations idempotent: applying already-present label is no-op

**Outcome**: Correct final state regardless of event order

**Test Coverage**: Integration test with events submitted out of order

---

### EC-3: Event for Deleted Resource

**Scenario**: Event references issue or PR that no longer exists

**Example**: "issue.labeled" event for issue #123, but issue was deleted

**Cause**: Race condition between event generation and resource deletion

**Handling**:

- GitHub API call returns 404 Not Found
- Log warning with correlation ID and resource details
- Mark event as processed (no retry)
- Do not dead letter (expected scenario)

**Outcome**: Event skipped gracefully, no error propagation

**Test Coverage**: Unit test for 404 handling, integration test with deleted resource

---

### EC-4: Malformed Event Payload

**Scenario**: Event payload missing required fields or has invalid data types

**Example**: Missing "repository.name" field

**Cause**: Queue-Keeper bug, schema mismatch, corrupted message

**Handling**:

- Event Processor validates payload against schema
- Validation failure logged with payload excerpt (sanitized)
- Event moved to dead letter queue
- Alert triggered for investigation

**Outcome**: Invalid events isolated, do not block valid events

**Test Coverage**: Unit test for validation logic with various malformed payloads

---

### EC-5: Very Large Event Payload

**Scenario**: Event payload exceeds expected size (e.g., issue with huge description)

**Cause**: User creates issue with extremely large description or many comments

**Handling**:

- Service Bus message size limit: 256 KB (Standard tier)
- If payload approaches limit: Queue-Keeper truncates or rejects
- Task-Tactician handles available data, may skip processing large fields

**Outcome**: Process what's available, log warning if data truncated

**Test Coverage**: Integration test with maximum-size payload

---

## Workflow Edge Cases

### EC-6: Branch Already Exists

**Scenario**: Attempt to create branch that already exists

**Cause**: Manual branch creation, previous failed event retry, concurrent events

**Handling**:

- Check branch existence before creation (GitHub API GET)
- If branch exists: log info (not error), return existing branch reference
- No API error thrown (idempotent operation)

**Outcome**: Workflow continues successfully, no error

**Test Coverage**: Unit test for idempotency, integration test with pre-existing branch

---

### EC-7: Issue Without Milestone

**Scenario**: Issue transitions to InProgress state without assigned milestone

**Cause**: User forgets to assign milestone before starting work

**Handling**:

- Detect missing milestone during InProgress transition
- Add comment to issue prompting user to assign milestone
- Mention assignee in comment
- Do not block branch creation (milestone not required for workflow)

**Outcome**: Branch created, user prompted to add milestone

**Test Coverage**: Integration test with issue lacking milestone

---

### EC-8: Issue With Multiple Type Labels

**Scenario**: Issue has multiple type labels (e.g., "bug" and "feature")

**Cause**: User applies multiple labels incorrectly

**Handling**:

- Branch prefix determination: use first matching label from configuration
- Configuration defines label priority order
- Log warning if multiple type labels detected

**Outcome**: Branch created with prefix from first matching label

**Test Coverage**: Unit test for label precedence logic

---

### EC-9: Issue Title With Special Characters

**Scenario**: Issue title contains emojis, unicode, or special characters

**Example**: "Add 🚀 OAuth2 Integration (SSO) - Phase #1"

**Handling**:

- Slug generation replaces all non-alphanumeric with hyphens
- Unicode characters converted to hyphens
- Result: "add-oauth2-integration-sso-phase-1"

**Outcome**: Valid branch name created, special chars removed

**Test Coverage**: Property-based test with various unicode characters

---

### EC-10: Very Long Issue Title

**Scenario**: Issue title exceeds slug max length

**Example**: 200-character title

**Handling**:

- Slug truncated to configured max length (default 50 chars)
- Truncation at word boundary if possible
- Ensure slug doesn't end with hyphen

**Outcome**: Branch name within length constraints (≤65 chars total)

**Test Coverage**: Unit test with various long titles

---

### EC-11: PR Referencing Non-Existent Issue

**Scenario**: PR description references "#999" but issue #999 doesn't exist

**Cause**: Typo, issue number incorrect, issue in different repository

**Handling**:

- Issue validation returns 404 Not Found
- Log warning: "Issue #999 referenced in PR #123 not found"
- Continue processing other valid issue references
- Do not fail PR processing

**Outcome**: Valid issues linked, invalid references skipped with warning

**Test Coverage**: Unit test for issue validation, integration test with invalid reference

---

### EC-12: Circular Label Dependencies

**Scenario**: Conflicting workflow states from labels (e.g., "in-progress" and "blocked" simultaneously)

**Cause**: User applies conflicting labels

**Handling**:

- State resolution applies precedence: Blocked > HasPR > InProgress > Open
- Highest precedence state wins
- Log info message about conflicting labels

**Outcome**: Consistent state determined, workflow proceeds

**Test Coverage**: Unit test for state precedence resolution

---

## Configuration Edge Cases

### EC-13: Invalid YAML Configuration

**Scenario**: Repository configuration file has syntax errors

**Cause**: Manual editing mistake, incomplete save

**Handling**:

- YAML parser throws error
- Error caught and logged with line number and details
- Fall back to organization config
- If organization config also invalid, use system defaults
- Log warning indicating configuration issue

**Outcome**: Processing continues with fallback configuration

**Test Coverage**: Unit test with various malformed YAML

---

### EC-14: Missing Required Configuration Field

**Scenario**: Configuration file missing required field (e.g., "labels.in_progress")

**Cause**: Incomplete configuration, schema mismatch

**Handling**:

- Schema validation detects missing field
- Warning logged with field name
- Use system default value for missing field
- Merge with fallback config for complete configuration

**Outcome**: Processing continues with defaults for missing fields

**Test Coverage**: Unit test for partial configuration merging

---

### EC-15: Configuration File Too Large

**Scenario**: Configuration file exceeds GitHub API content retrieval limit

**Cause**: Excessive configuration, embedded documentation

**Handling**:

- GitHub API returns file content (up to 1 MB typically)
- If exceeds limit: GitHub returns error or truncates
- Log error, fall back to organization config or defaults

**Outcome**: Fallback configuration used, processing continues

**Test Coverage**: Integration test with large configuration file

---

## GitHub API Edge Cases

### EC-16: GitHub API Rate Limit Exhaustion

**Scenario**: API quota exhausted (5000 requests/hour)

**Cause**: High event volume, inefficient API usage

**Handling**:

- Rate limiter tracks remaining quota from response headers
- Circuit breaker opens at 20% remaining (1000 requests)
- Non-critical operations queued
- Critical operations (branch creation) allowed through
- Circuit closes after rate limit reset (1 hour)

**Outcome**: Service degraded but functional, recovers after reset

**Test Coverage**: Integration test with simulated rate limit exhaustion

---

### EC-17: GitHub API Unavailability

**Scenario**: GitHub API returns 503 Service Unavailable

**Cause**: GitHub outage, maintenance

**Handling**:

- Error classified as transient
- Exponential backoff retry (max 3 attempts)
- If all retries fail: event requeued in message queue
- Circuit breaker opens after sustained failures
- Service recovers when GitHub recovers

**Outcome**: Events buffered in queue, processed after GitHub recovery

**Test Coverage**: Integration test with simulated GitHub outage

---

### EC-18: GitHub API Slow Response

**Scenario**: GitHub API calls take >30 seconds (timeout threshold)

**Cause**: GitHub performance degradation, network issues

**Handling**:

- HTTP client timeout set to 30 seconds
- Timeout treated as transient error
- Retry with exponential backoff
- If consistent timeouts: circuit breaker opens

**Outcome**: Slow operations retried, circuit breaker prevents cascade

**Test Coverage**: Integration test with simulated slow responses

---

### EC-19: GitHub Authentication Failure

**Scenario**: Installation token invalid or expired

**Cause**: Token expiration, GitHub App uninstalled, permissions revoked

**Handling**:

- GitHub API returns 401 Unauthorized
- Attempt to regenerate installation token
- If regeneration fails: GitHub App may be uninstalled
- Log error with installation ID
- Alert operations team

**Outcome**: Processing blocked until authentication resolved

**Test Coverage**: Unit test for token regeneration, integration test with expired token

---

### EC-20: GitHub Permission Denied

**Scenario**: GitHub App lacks required permissions for operation

**Example**: Attempt to create branch but "contents: write" permission missing

**Cause**: GitHub App configuration error, permissions changed

**Handling**:

- GitHub API returns 403 Forbidden
- Error classified as non-retryable (configuration issue)
- Log error with operation and required permission
- Alert operations team to update GitHub App permissions
- Event moved to dead letter queue

**Outcome**: Event fails, requires manual intervention to fix permissions

**Test Coverage**: Integration test with insufficient permissions

---

## Message Queue Edge Cases

### EC-21: Message Lock Timeout

**Scenario**: Event processing exceeds message lock timeout (5 minutes)

**Cause**: Slow GitHub API calls, complex workflow

**Handling**:

- Lock renewal attempted during long processing
- If renewal fails: message becomes visible to other instances
- Original processor completes or times out
- Message processed exactly once (deduplication handles duplicates)

**Outcome**: Event processed once, no duplicate operations

**Test Coverage**: Integration test with long-running operation

---

### EC-22: Dead Letter Queue Full

**Scenario**: Dead letter queue accumulates many failed messages

**Cause**: Sustained processing failures, configuration issues

**Handling**:

- Monitor dead letter queue depth metric
- Alert triggered at threshold (>10 messages)
- Operations team investigates failed messages
- Manual intervention to resolve root cause and replay messages

**Outcome**: Failed events preserved for investigation and replay

**Test Coverage**: Integration test with DLQ monitoring

---

### EC-23: Service Bus Unavailability

**Scenario**: Cannot connect to Service Bus queue

**Cause**: Azure outage, network issue, configuration error

**Handling**:

- Function trigger fails to receive messages
- Azure Functions runtime retries connection
- Events buffer in Queue-Keeper until Service Bus recovers
- Automatic recovery when Service Bus available

**Outcome**: Events processed after recovery, no data loss

**Test Coverage**: Manual test (simulate by stopping Service Bus)

---

## Concurrency Edge Cases

### EC-24: Concurrent Events for Same Issue

**Scenario**: Multiple events for same issue arrive simultaneously

**Example**: Issue assigned and labeled "in-progress" in quick succession

**Handling**:

- Session-based ordering ensures sequential processing per repository
- Events processed one at a time for same repository
- Current state fetched from GitHub for each event
- Idempotent operations prevent conflicts

**Outcome**: Correct final state, no race conditions

**Test Coverage**: Integration test with concurrent events

---

### EC-25: Concurrent Branch Creation Attempts

**Scenario**: Multiple processors attempt to create same branch simultaneously

**Cause**: Deduplication failure, race condition

**Handling**:

- First request succeeds (branch created)
- Second request checks existence first, finds existing branch
- Second request returns existing branch (idempotent)
- No error thrown

**Outcome**: Branch created once, both requests succeed

**Test Coverage**: Integration test with concurrent creation attempts

---

## Data Validation Edge Cases

### EC-26: Empty Issue Title

**Scenario**: Issue has empty or whitespace-only title

**Cause**: GitHub API allows empty titles in some cases

**Handling**:

- Slug generation fails validation
- Use fallback slug: "issue-{number}"
- Log warning about empty title

**Outcome**: Branch created with fallback name

**Test Coverage**: Unit test for empty title handling

---

### EC-27: Issue Number Overflow

**Scenario**: Repository has issue number >2^32 (unlikely but theoretically possible)

**Cause**: Very old, very active repository

**Handling**:

- Issue number type supports large integers
- No overflow in modern systems
- Branch name may exceed length limit if number very large

**Outcome**: Handled correctly up to system limits

**Test Coverage**: Unit test with large issue numbers

---

### EC-28: Repository Name With Special Characters

**Scenario**: Repository name contains hyphens, underscores

**Example**: "my-awesome-project_v2"

**Handling**:

- Repository name used as-is from GitHub API
- Session key includes special characters (valid)
- No special handling required

**Outcome**: Processed normally

**Test Coverage**: Integration test with special char repository names

---

## Recovery Scenarios

### EC-29: Partial Operation Failure

**Scenario**: Branch created successfully, but label application fails

**Cause**: GitHub API partial failure, rate limit mid-operation

**Handling**:

- Operations executed sequentially, each idempotent
- Failed operation logged with context
- Event reprocessed: branch creation skipped (exists), label applied
- Final state consistent after retry

**Outcome**: Eventually consistent state achieved

**Test Coverage**: Integration test with simulated partial failure

---

### EC-30: Configuration Change During Processing

**Scenario**: Repository configuration updated while event being processed

**Cause**: User commits configuration change

**Handling**:

- Configuration cached with TTL (5 minutes)
- Processing uses cached config (consistency within event)
- Next event (after cache expiry) uses new config
- No mid-processing config changes

**Outcome**: Eventual consistency, no mid-processing surprises

**Test Coverage**: Integration test with configuration change during processing

---

## Monitoring Edge Cases

### EC-31: Correlation ID Missing

**Scenario**: Event arrives without correlation ID

**Cause**: Queue-Keeper bug, legacy event format

**Handling**:

- Generate new correlation ID (UUID)
- Log warning about missing correlation ID
- Use generated ID for all subsequent logging

**Outcome**: Tracing possible with generated ID

**Test Coverage**: Unit test for correlation ID generation

---

### EC-32: Log Volume Spike

**Scenario**: Excessive logging causes Application Insights quota exceeded

**Cause**: Debug logging enabled in production, error loop

**Handling**:

- Log levels enforced (debug disabled in production)
- Sampling applied in Application Insights (not every log ingested)
- Alert on high ingestion rate
- Operations team investigates and adjusts log levels

**Outcome**: Quota managed, critical logs retained

**Test Coverage**: Manual test (monitor log volume in staging)

---

## Summary of Handling Strategies

| Error Category | Handling Strategy |
|----------------|------------------|
| Transient Errors | Exponential backoff retry (max 3 attempts) |
| Configuration Errors | Fall back to defaults, log warning, continue |
| Business Logic Errors | Log warning, skip operation, continue |
| Validation Errors | Reject event, dead letter, alert |
| Rate Limit | Circuit breaker, queue operations, wait for reset |
| Idempotency | Check current state, skip if already done |
| Concurrency | Session-based ordering, idempotent operations |
| Partial Failures | Retry entire event, idempotent operations ensure consistency |

## Testing Coverage

All edge cases documented above must have:

1. **Unit Test**: For isolated logic (validation, transformation, classification)
2. **Integration Test**: For component interactions (GitHub API, queue, configuration)
3. **Manual Test**: For scenarios requiring live services (Azure outages, GitHub downtime)

**Coverage Target**: 100% of documented edge cases validated by automated tests (where possible)
