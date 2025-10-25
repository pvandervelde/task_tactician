# Component Responsibilities (RDD)

This document defines component responsibilities using Responsibility-Driven Design (RDD). Each component's responsibilities are described in terms of what it **knows** (data) and what it **does** (operations), along with its **collaborators** and **roles** in collaborations.

## Event Processor

### Responsibilities

**Knows:**

- Message queue connection details (queue name, connection string)
- Event schema and validation rules
- Processed message IDs within deduplication window
- Current processing state (active, paused, error)

**Does:**

- Receives messages from queue (Azure Service Bus / AWS SQS)
- Validates event payload structure and required fields
- Checks for duplicate events using message ID
- Extracts correlation ID for request tracing
- Routes events to appropriate workflow handler
- Acknowledges successfully processed messages
- Moves failed messages to dead letter queue after retry exhaustion
- Maintains ordered processing per repository (via queue sessions)

### Collaborators

- **Message Queue** (receives events from)
- **Workflow Engine** (delegates processing to)
- **Deduplication Cache** (queries for duplicate detection)
- **Logging Service** (records processing events)

### Roles

- **Consumer**: Receives and acknowledges queue messages
- **Validator**: Ensures event payload meets schema requirements
- **Router**: Directs events to correct workflow handler
- **Coordinator**: Manages event lifecycle from receipt to completion

---

## Workflow Engine

### Responsibilities

**Knows:**

- Workflow rules for issue lifecycle management
- State transition logic (Open → InProgress → HasPR → Done)
- Business rules for each workflow type
- Valid state combinations and transitions

**Does:**

- Determines required actions based on event type and current state
- Executes issue workflow (assignment, labeling, branch creation)
- Executes PR workflow (issue linking, label syncing)
- Validates business rules before applying actions
- Generates workflow actions (CreateBranch, ApplyLabel, etc.)
- Sequences actions in dependency order
- Handles workflow-specific errors

### Collaborators

- **Configuration Manager** (retrieves workflow settings from)
- **GitHub Client** (executes operations via)
- **Issue State Resolver** (determines current issue state from)
- **Branch Name Generator** (creates branch names with)
- **Label Discovery Service** (validates labels and mappings via)

### Roles

- **Orchestrator**: Coordinates multi-step workflow execution
- **Rule Engine**: Applies business logic to determine actions
- **State Machine**: Manages issue lifecycle state transitions
- **Validator**: Enforces workflow preconditions and constraints

---

## Configuration Manager

### Responsibilities

**Knows:**

- Configuration hierarchy (Repository → Organization → System Default)
- Configuration cache state and TTL
- Configuration file locations in GitHub repositories
- Merge strategy for hierarchical configuration

**Does:**

- Loads repository-specific configuration from `.github/task-tactician.yml`
- Loads organization-level configuration from metadata repository
- Provides system default configuration
- Merges configurations according to precedence rules
- Caches merged configuration per repository
- Invalidates cache after TTL expiration
- Validates configuration schema and types
- Provides configuration values to workflow engine

### Collaborators

- **GitHub Client** (fetches configuration files from)
- **Configuration Cache** (stores merged configs in)
- **Configuration Validator** (validates schemas with)
- **Workflow Engine** (supplies settings to)

### Roles

- **Provider**: Supplies configuration to consuming components
- **Aggregator**: Merges multiple configuration sources
- **Cache Manager**: Maintains configuration cache lifecycle
- **Validator**: Ensures configuration correctness

---

## GitHub Client

### Responsibilities

**Knows:**

- GitHub API base URL and endpoints
- GitHub App credentials (app ID, installation ID)
- Current rate limit status and remaining quota
- Authentication token cache and expiration

**Does:**

- Authenticates as GitHub App using JWT and private key
- Obtains installation tokens for API operations
- Creates branches from default branch
- Applies and removes issue labels
- Retrieves issue and PR data
- Adds comments to issues and PRs
- Fetches file contents from repositories
- Handles GitHub API rate limiting
- Retries transient API failures with exponential backoff
- Implements circuit breaker for sustained failures

### Collaborators

- **Secret Store** (retrieves private key from)
- **Rate Limiter** (coordinates API quota with)
- **Retry Policy** (applies failure recovery via)
- **Logging Service** (records API operations to)

### Roles

- **API Adapter**: Translates domain operations to GitHub API calls
- **Authenticator**: Manages GitHub App authentication flow
- **Circuit Breaker**: Prevents cascading failures during outages
- **Rate Manager**: Respects and enforces API rate limits

---

## Branch Name Generator

### Responsibilities

**Knows:**

- Branch naming pattern configuration (prefix, issue number, slug)
- Slug generation rules (max length, character transformations)
- Label-to-prefix mappings (bug → fix/, feature → feat/)
- Default prefix when no label matches

**Does:**

- Extracts issue number from issue data
- Generates slug from issue title (kebab-case, max 50 chars)
- Determines branch prefix based on issue labels
- Constructs complete branch name following pattern
- Validates branch name constraints (length, characters)
- Ensures branch name uniqueness

### Collaborators

- **Configuration Manager** (retrieves naming patterns from)
- **Issue** (extracts data from)

### Roles

- **Generator**: Creates branch names from issue data
- **Formatter**: Applies naming conventions and transformations
- **Validator**: Ensures generated names meet constraints

---

## Label Discovery Service

### Responsibilities

**Knows:**

- Repository's available labels (queried from GitHub)
- Workflow state label mappings (configuration preferences)
- System default labels (fallback values)
- Label validation cache (TTL-based)

**Does:**

- Queries repository for all available labels
- Maps discovered labels to workflow states (InProgress, Blocked, etc.)
- Maps label categories to branch prefixes (bug → fix/, feature → feat/)
- Falls back to system defaults when preferred labels missing
- Validates labels exist before application attempts
- Caches validated label configuration to reduce API calls

### Collaborators

- **GitHub Client** (queries repository labels via)
- **Configuration Manager** (gets label preferences from)
- **Workflow Engine** (provides validated labels to)

### Roles

- **Discoverer**: Identifies available labels in repository
- **Mapper**: Creates mappings between labels and workflow concepts
- **Validator**: Ensures labels exist before use
- **Cache Manager**: Reduces API calls through caching

**Note**: GitHub's native PR-issue linking (via keywords in PR description) handles issue-PR relationships. Task-Tactician does not implement custom PR-issue linking.

---

## Issue State Resolver

### Responsibilities

**Knows:**

- Mapping between labels and workflow states
- Project board column names indicating states
- State precedence rules (Blocked overrides InProgress)

**Does:**

- Determines current workflow state from issue labels
- Checks project board column for state indication
- Resolves conflicting state signals
- Identifies state transitions from events

### Collaborators

- **Configuration Manager** (retrieves label mappings from)
- **Issue** (reads labels and project board status from)

### Roles

- **Resolver**: Determines current state from multiple signals
- **Interpreter**: Translates labels/columns to workflow states
- **Arbitrator**: Resolves conflicting state indicators

---

## Deduplication Cache

### Responsibilities

**Knows:**

- Processed message IDs within TTL window
- Deduplication key format (webhook ID + repository ID)
- TTL for deduplication entries (default 10 minutes)

**Does:**

- Stores processed message IDs
- Checks if message ID already processed
- Expires old entries after TTL
- Provides fast lookup for duplicate detection

### Collaborators

- **Event Processor** (serves duplicate checks for)

### Roles

- **Cache**: Stores temporary deduplication state
- **Expirer**: Removes stale entries automatically
- **Lookup**: Provides fast duplicate detection

---

## Retry Policy

### Responsibilities

**Knows:**

- Error classifications (transient, configuration, business logic)
- Retry attempt limits (max 3 attempts)
- Backoff strategy (exponential with jitter)
- Maximum delay between retries

**Does:**

- Classifies errors as retryable or non-retryable
- Calculates delay before next retry attempt
- Applies jitter to prevent thundering herd
- Limits total retry attempts
- Decides when to give up and dead letter

### Collaborators

- **GitHub Client** (applies retry logic for)
- **Event Processor** (coordinates message retry with)

### Roles

- **Classifier**: Categorizes errors by type
- **Calculator**: Determines retry delays
- **Limiter**: Enforces retry attempt limits
- **Strategist**: Applies backoff and jitter algorithms

---

## Rate Limiter

### Responsibilities

**Knows:**

- Current GitHub API rate limit consumption
- Remaining quota and reset time
- Circuit breaker thresholds (20% remaining triggers circuit)
- Queued operations awaiting quota

**Does:**

- Tracks API calls and decrements quota
- Monitors rate limit headers from API responses
- Opens circuit when quota critically low
- Queues non-urgent operations
- Delays operations approaching rate limit
- Resets quota tracking on reset time

### Collaborators

- **GitHub Client** (coordinates API calls with)
- **Circuit Breaker** (triggers protection via)

### Roles

- **Monitor**: Tracks rate limit consumption
- **Throttler**: Delays operations to stay within quota
- **Queue Manager**: Prioritizes and queues operations
- **Protector**: Prevents quota exhaustion

---

## Secret Store

### Responsibilities

**Knows:**

- Azure Key Vault / AWS Secrets Manager location
- Secret names and versions
- Access policies and permissions

**Does:**

- Retrieves GitHub App private key
- Caches secrets in memory with TTL
- Handles secret rotation
- Provides secrets to authorized components

### Collaborators

- **GitHub Client** (provides private key to)
- **Cloud Provider IAM** (authenticates access via)

### Roles

- **Vault**: Secures sensitive credentials
- **Provider**: Supplies secrets on demand
- **Rotator**: Manages secret lifecycle and renewal

---

## Stale Issue Detector

### Responsibilities

**Knows:**

- Staleness threshold (configurable, default 30 days)
- Exclusion labels that prevent stale marking
- Last activity timestamp for issues
- Repository staleness configuration

**Does:**

- Checks issue activity against threshold
- Applies "stale" label to inactive issues
- Adds informational comment explaining staleness
- Respects exclusion labels (e.g., "blocked", "on-hold")
- Logs stale detections

### Collaborators

- **GitHub Client** (queries issue activity, applies labels via)
- **Configuration Manager** (gets staleness settings from)
- **Logging Service** (records detections to)

### Roles

- **Detector**: Identifies inactive issues
- **Labeler**: Marks stale issues for triage
- **Commentator**: Provides context on staleness

---

## Milestone Prompter

### Responsibilities

**Knows:**

- Available milestones in repository
- Issue milestone status
- Milestone prompting configuration

**Does:**

- Detects issues created without milestone
- Adds comment suggesting milestone assignment
- Lists currently active milestones in prompt
- Skips prompting if disabled or no milestones exist
- Logs prompting actions

### Collaborators

- **GitHub Client** (queries milestones, adds comments via)
- **Configuration Manager** (checks if prompting enabled from)
- **Event Processor** (receives issue creation events from)

### Roles

- **Prompter**: Suggests milestone assignment
- **Advisor**: Provides available options
- **Validator**: Checks if prompt needed

---

## Dependency Tracker

### Responsibilities

**Knows:**

- Dependency notation keywords ("Depends on", "Blocks", etc.)
- Issue description parsing rules
- Detected dependency relationships

**Does:**

- Parses issue descriptions for dependency references
- Validates referenced issues exist
- Records dependency relationships as metrics
- Logs invalid references as warnings
- Supports multiple notation formats

### Collaborators

- **GitHub Client** (validates issue existence via)
- **Metrics Collector** (records dependencies to)
- **Logging Service** (logs detections and errors to)

### Roles

- **Parser**: Extracts dependency references from text
- **Validator**: Confirms referenced issues exist
- **Recorder**: Tracks dependency relationships

---

## Cycle Time Tracker

### Responsibilities

**Knows:**

- Issue creation timestamps
- Issue closure timestamps
- Issue type labels

**Does:**

- Calculates cycle time on issue closure (closed_at - created_at)
- Records cycle time as metric with labels (repository, issue type)
- Skips duplicate recordings for reopened issues
- Logs cycle time values

### Collaborators

- **Metrics Collector** (records cycle time to)
- **Event Processor** (receives issue closed events from)
- **Logging Service** (logs cycle times to)

### Roles

- **Calculator**: Computes time durations
- **Recorder**: Captures metrics for analytics
- **Analyzer**: Enables velocity tracking

---

## Logging Service

### Responsibilities

**Knows:**

- Log destination (Application Insights, CloudWatch)
- Log format (structured JSON)
- Correlation ID for current request
- Configured log level

**Does:**

- Writes structured log entries
- Includes correlation ID in all entries
- Adds contextual metadata (repository, issue number, operation)
- Routes logs to appropriate destination
- Filters logs by configured level

### Collaborators

- **All Components** (receive logging from)
- **Application Insights / CloudWatch** (sends logs to)

### Roles

- **Recorder**: Captures operational events
- **Formatter**: Structures logs consistently
- **Correlator**: Maintains request tracing context
- **Router**: Directs logs to destination

---

## Component Interaction Summary

### Primary Flow: Issue Assignment Event

```
Event Processor
  ├─> Deduplication Cache (check duplicate)
  ├─> Workflow Engine (process event)
  │     ├─> Configuration Manager (get workflow settings)
  │     │     └─> GitHub Client (fetch config file)
  │     ├─> Issue State Resolver (determine current state)
  │     ├─> Branch Name Generator (create branch name)
  │     └─> GitHub Client (create branch, apply labels)
  │           ├─> Secret Store (get private key)
  │           └─> Rate Limiter (check quota)
  └─> Logging Service (record completion)
```

### Key Collaboration Patterns

1. **Configuration Retrieval**: Components ask Configuration Manager, which caches and merges from multiple sources
2. **GitHub Operations**: Components request via GitHub Client, which handles auth, rate limits, and retries
3. **Error Handling**: Retry Policy classifies errors and determines retry strategy for all components
4. **Observability**: All components collaborate with Logging Service to maintain audit trail
5. **State Management**: Issue State Resolver centralizes state determination logic used by multiple workflows

## Architectural Principles

1. **Single Responsibility**: Each component has one clear purpose
2. **Dependency Inversion**: Components depend on abstractions (interfaces), not concrete implementations
3. **Separation of Concerns**: Business logic (Workflow Engine) separate from infrastructure (GitHub Client, Message Queue)
4. **Statelessness**: No persistent state; GitHub is source of truth
5. **Idempotency**: All operations check current state before making changes
