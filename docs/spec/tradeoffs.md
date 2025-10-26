# Architectural Tradeoffs and Alternatives

This document explores key architectural decisions, alternatives considered, and tradeoffs made in Task-Tactician's design.

## Tradeoff 1: Stateless vs. Stateful Architecture

### Decision: Stateless Architecture

**Chosen Approach**:

- No persistent state storage (database)
- GitHub serves as source of truth
- Configuration cached in memory with TTL
- Deduplication cache in memory (short TTL)

**Alternatives Considered**:

**Alternative A: Database for State Management**

- Store issue workflow state, event processing history, metrics
- Enables complex queries, historical analysis, auditing
- Provides consistent view of system state

**Alternative B: Event Sourcing**

- Store all events in event store
- Rebuild state by replaying events
- Complete audit trail, enables replay and debugging

### Tradeoffs

| Aspect | Stateless (Chosen) | Database | Event Sourcing |
|--------|-------------------|----------|----------------|
| **Complexity** | Low | Medium | High |
| **Operational Cost** | Low | Medium | High |
| **Data Consistency** | Eventual (GitHub) | Strong | Eventual |
| **Scalability** | Excellent | Good | Good |
| **Recovery** | Simple (no state to recover) | Complex (backup/restore) | Complex (replay) |
| **Debuggability** | Moderate (logs only) | Good (query history) | Excellent (full replay) |
| **Latency** | Low | Medium (DB queries) | Medium (event replay) |

**Pros of Stateless**:

- Simple deployment and operation
- No database to manage, backup, or scale
- Fast recovery (no state to restore)
- Lower operational cost
- Aligns with serverless model

**Cons of Stateless**:

- No historical data for analytics
- Limited auditability (log-based only)
- Cannot query past workflow state
- Deduplication window limited (memory only)

**Why Stateless Wins**:

- GitHub already provides persistent state (issues, PRs, labels)
- Simplicity reduces operational burden
- Aligns with serverless/cloud-native architecture
- Cost-effective for expected workload
- Logging provides sufficient audit trail for current requirements

**Future Consideration**: Add database for analytics if needed (non-blocking enhancement)

---

## Tradeoff 2: Event-Driven vs. Polling Architecture

### Decision: Event-Driven Architecture

**Chosen Approach**:

- Consume events from message queue (Azure Service Bus)
- Events pushed by Queue-Keeper (webhook processor)
- Processing triggered by event arrival

**Alternatives Considered**:

**Alternative A: Polling GitHub API**

- Periodically query GitHub for changes (every N seconds)
- Check for new issues, label changes, PR creations
- Process changes detected

**Alternative B: Hybrid (Events + Polling)**

- Use events for real-time processing
- Polling as backup to catch missed events
- Reconciliation process to ensure consistency

### Tradeoffs

| Aspect | Event-Driven (Chosen) | Polling | Hybrid |
|--------|----------------------|---------|--------|
| **Latency** | Low (<5s) | High (poll interval) | Low (events) + High (reconciliation) |
| **API Usage** | Low | Very High | High |
| **Complexity** | Low | Low | High |
| **Reliability** | Depends on webhooks | Guaranteed consistency | High |
| **Cost** | Low | High (rate limits) | High |
| **Real-time** | Yes | No | Yes |

**Pros of Event-Driven**:

- Near real-time processing (low latency)
- Efficient API usage (only process changes)
- Aligns with GitHub webhook model
- Scalable (events drive scaling)

**Cons of Event-Driven**:

- Dependency on webhook delivery (Queue-Keeper)
- Potential for missed events if webhooks fail
- Requires message queue infrastructure

**Why Event-Driven Wins**:

- Real-time processing requirement (users expect immediate branch creation)
- GitHub API rate limits make polling impractical (5000 req/hour)
- Queue-Keeper provides reliable webhook processing
- Event-driven aligns with serverless architecture

**Future Consideration**: Add periodic reconciliation job if event loss becomes issue

---

## Tradeoff 3: Configuration Storage: GitHub Files vs. Centralized Store

### Decision: GitHub Repository Files

**Chosen Approach**:

- Configuration stored in `.github/task-tactician.yml` (repository)
- Organization defaults in metadata repository
- System defaults embedded in code

**Alternatives Considered**:

**Alternative A: Azure App Configuration**

- Centralized configuration service
- Key-value store with versioning
- Hot reload without deployment

**Alternative B: Database (e.g., Cosmos DB)**

- Configuration stored in database
- UI for configuration management
- Query and reporting capabilities

### Tradeoffs

| Aspect | GitHub Files (Chosen) | Azure App Config | Database |
|--------|----------------------|------------------|----------|
| **Discoverability** | Excellent (in repo) | Poor (external) | Poor (external) |
| **Version Control** | Yes (Git) | Limited | No |
| **Access Control** | GitHub permissions | Azure RBAC | Custom |
| **Developer Experience** | Excellent | Moderate | Poor |
| **Change Process** | PR review | Portal/API | UI/API |
| **Cost** | Free | Low | Medium |
| **Complexity** | Low | Medium | High |

**Pros of GitHub Files**:

- Configuration lives with code (discoverability)
- Git version control (history, blame, rollback)
- PR review process ensures quality
- No additional infrastructure
- Developers already familiar with workflow

**Cons of GitHub Files**:

- Requires GitHub API call to load
- Cache needed to reduce API calls
- Changes require commit (no instant updates)
- Schema validation not enforced (YAML parsing only)

**Why GitHub Files Win**:

- Developer-friendly (configuration where code lives)
- Version control built-in (Git)
- No additional infrastructure cost
- Aligns with GitOps principles
- Easy to review configuration changes via PRs

**Future Consideration**: Add schema validation service or pre-commit hooks

---

## Tradeoff 4: Result Type vs. Exceptions for Error Handling

### Decision: Result Type for Expected Errors

**Chosen Approach**:

- Business logic returns `Result<T, E>` for operations
- Expected errors (validation, business rules, API failures) as values
- Exceptions only for unexpected/unrecoverable errors

**Alternatives Considered**:

**Alternative A: Exceptions for All Errors**

- Throw exceptions for all error conditions
- Catch and handle at boundaries
- Traditional error handling model

**Alternative B: Error Codes**

- Return error codes (integers) for failures
- Caller checks codes and handles
- C-style error handling

### Tradeoffs

| Aspect | Result Type (Chosen) | Exceptions | Error Codes |
|--------|---------------------|------------|-------------|
| **Type Safety** | Excellent (compiler-enforced) | Moderate | Poor |
| **Explicitness** | Explicit (must handle Err) | Implicit (try-catch) | Explicit (check code) |
| **Performance** | Good | Poor (stack unwinding) | Excellent |
| **Ergonomics** | Good (match, ?, operators) | Good (try-catch) | Poor (manual checks) |
| **Composability** | Excellent (map, and_then) | Moderate | Poor |

**Pros of Result Type**:

- Type system enforces error handling (no forgotten errors)
- Clear distinction: expected vs. unexpected errors
- Composable (map, flat_map, chaining)
- No hidden control flow (exceptions can jump anywhere)
- Performance (no stack unwinding)

**Cons of Result Type**:

- More verbose than exceptions
- Requires error type definitions
- Learning curve for developers unfamiliar with pattern

**Why Result Type Wins**:

- Rust's idiomatic error handling (language-native)
- Type safety prevents forgotten error handling
- Clear separation: business errors (Result) vs. panics (unrecoverable)
- Composable error handling (operators, combinators)

**Implementation Note**: Use `anyhow` or `thiserror` crate for error types

---

## Tradeoff 5: Multi-Cloud vs. Azure-Only

### Decision: Azure-First with Multi-Cloud Abstraction

**Chosen Approach**:

- Primary deployment on Azure
- Infrastructure interfaces abstract cloud services
- Future AWS support possible without business logic changes

**Alternatives Considered**:

**Alternative A: Azure-Only**

- Hardcode Azure SDK calls
- No abstraction layer
- Fastest initial development

**Alternative B: Multi-Cloud from Day 1**

- Abstract all cloud services from start
- Maintain Azure and AWS implementations
- Test both continuously

### Tradeoffs

| Aspect | Azure-First (Chosen) | Azure-Only | Multi-Cloud Day 1 |
|--------|---------------------|------------|-------------------|
| **Initial Development Speed** | Medium | Fast | Slow |
| **Maintenance Cost** | Low | Lowest | High |
| **Flexibility** | Good | Poor | Excellent |
| **Testing Complexity** | Low | Lowest | High |
| **Vendor Lock-in Risk** | Low | High | None |

**Pros of Azure-First with Abstraction**:

- Interfaces enable future multi-cloud without rewrite
- Focus initial development on Azure (customer requirement)
- Abstraction aligns with Clean Architecture principles
- No premature optimization for unsupported platforms

**Cons of Azure-First with Abstraction**:

- Slightly more code than Azure-only
- Abstraction overhead without immediate benefit
- Risk of leaky abstractions

**Why Azure-First with Abstraction Wins**:

- Current requirement is Azure deployment
- Abstraction enables future AWS support if needed
- Interfaces improve testability (mock cloud services)
- Aligns with Clean Architecture dependency inversion

**Abstracted Services**:

- IMessageQueue: Azure Service Bus / AWS SQS
- ISecretStore: Azure Key Vault / AWS Secrets Manager
- ILogger: Application Insights / CloudWatch

---

## Tradeoff 6: Branch Prefix: Label-Based vs. Issue Type

### Decision: Label-Based Configurable Prefix

**Chosen Approach**:

- Branch prefix determined by issue labels
- Configuration maps labels to prefixes (bug → fix/, feature → feat/)
- Default prefix if no label matches

**Alternatives Considered**:

**Alternative A: Fixed Prefix**

- All branches use same prefix (e.g., "feat/")
- Simple, consistent

**Alternative B: GitHub Issue Type**

- Use GitHub issue type field (if available)
- More structured

### Tradeoffs

| Aspect | Label-Based (Chosen) | Fixed Prefix | Issue Type |
|--------|---------------------|--------------|------------|
| **Flexibility** | High | Low | Medium |
| **Consistency** | Depends on labels | Perfect | Good |
| **Configuration** | Required | None | None |
| **GitHub Support** | Universal (all repos) | Universal | Limited (Projects) |

**Pros of Label-Based**:

- Flexible (repositories can customize)
- Works with any repository (labels universal)
- Supports common convention (feat/, fix/, docs/)

**Cons of Label-Based**:

- Requires configuration
- Depends on users applying correct labels
- Inconsistent if labels misapplied

**Why Label-Based Wins**:

- Conventional commit types widely adopted (feat, fix, docs, etc.)
- Labels already used for issue categorization
- Configuration allows per-repository customization
- Falls back to default prefix if no label matches

**Assumption**: Repositories follow conventional commit label conventions

---

## Tradeoff 7: Ordered Processing vs. Parallel Processing

### Decision: Ordered per Repository, Parallel Across Repositories

**Chosen Approach**:

- Message queue sessions keyed by repository
- Events for same repository process sequentially
- Events for different repositories process in parallel

**Alternatives Considered**:

**Alternative A: Fully Parallel**

- All events process in parallel
- No ordering guarantees
- Maximum throughput

**Alternative B: Fully Sequential**

- All events process one at a time
- Global ordering
- Lowest throughput

### Tradeoffs

| Aspect | Repository-Ordered (Chosen) | Fully Parallel | Fully Sequential |
|--------|----------------------------|----------------|------------------|
| **Throughput** | High | Highest | Lowest |
| **Consistency** | Strong (per repo) | Weak | Strong |
| **Complexity** | Medium | Low | Low |
| **Scalability** | Excellent | Excellent | Poor |

**Pros of Repository-Ordered**:

- Prevents race conditions for same issue (e.g., concurrent label + assign)
- Enables parallel processing across repositories (scalability)
- Balances consistency and performance

**Cons of Repository-Ordered**:

- Requires session-based queuing (Service Bus feature)
- More complex than fully parallel

**Why Repository-Ordered Wins**:

- Race conditions possible without ordering (same issue events)
- Repositories independent (parallel OK)
- Service Bus sessions provide ordering mechanism
- Balances correctness (ordering) and performance (parallelism)

---

## Tradeoff 8: Deduplication: In-Memory vs. Persistent

### Decision: In-Memory with TTL

**Chosen Approach**:

- Deduplication cache in memory (10-minute TTL)
- Tracks processed message IDs
- No persistent storage

**Alternatives Considered**:

**Alternative A: Database Deduplication**

- Store processed message IDs in database
- Persistent, no TTL needed
- Query before processing

**Alternative B: No Deduplication**

- Rely on idempotent operations only
- No explicit duplicate detection

### Tradeoffs

| Aspect | In-Memory (Chosen) | Database | No Deduplication |
|--------|-------------------|----------|------------------|
| **Latency** | Low (memory lookup) | Medium (DB query) | Lowest (no check) |
| **Cost** | Low | Medium | Lowest |
| **Reliability** | Good (10-min window) | Excellent | Poor |
| **Complexity** | Low | Medium | Lowest |

**Pros of In-Memory**:

- Fast (no database query)
- Simple implementation
- Sufficient for at-least-once delivery
- Idempotent operations provide safety net

**Cons of In-Memory**:

- Limited window (10 minutes)
- Lost on service restart
- Duplicates possible outside window

**Why In-Memory Wins**:

- At-least-once delivery typical retry window <10 minutes
- Idempotent operations catch duplicates outside window
- No database infrastructure needed
- Fast duplicate detection

**Assumption**: Message queue retries occur within 10-minute window

---

## Summary of Key Decisions

| Decision Area | Chosen Approach | Primary Reason |
|---------------|----------------|----------------|
| State Management | Stateless | Simplicity, cost, scalability |
| Event Processing | Event-Driven | Real-time, API efficiency |
| Configuration | GitHub Files | Developer experience, version control |
| Error Handling | Result Type | Type safety, explicit errors |
| Cloud Platform | Azure-First + Abstraction | Current requirement + future flexibility |
| Branch Prefix | Label-Based | Flexibility, conventional commits |
| Processing Order | Repository-Ordered | Consistency + scalability |
| Deduplication | In-Memory | Performance, sufficient window |

---

## Future Reevaluation Triggers

Revisit these decisions if:

1. **State Management**: Analytics requirement emerges (add database)
2. **Event Processing**: Webhook reliability issues (add polling reconciliation)
3. **Configuration**: Developer complaints about file-based config (add UI/API)
4. **Cloud Platform**: Customer requires AWS deployment (implement AWS adapters)
5. **Processing Order**: Throughput insufficient (optimize session partitioning)
6. **Deduplication**: Duplicates outside 10-min window (extend window or persist)
