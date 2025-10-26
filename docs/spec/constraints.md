# Implementation Constraints

This document defines explicit constraints that must be enforced during implementation. These rules ensure architectural principles are maintained and guide concrete design decisions.

## Type System Constraints

### TS-1: Branded Types for Domain Identifiers

**Constraint**: All domain identifiers must use branded types (newtypes), not primitive types.

**Rationale**: Prevents accidental mixing of different ID types (issue number vs. PR number)

**Examples**:

- `IssueNumber` (not `u32`)
- `PRNumber` (not `u32`)
- `RepositoryId` (not `String`)
- `BranchName` (not `String`)
- `CorrelationId` (not `String`)

**Enforcement**: Type checker rejects primitive types in domain operations

**Test Coverage**: Compilation failure if primitives used

---

### TS-2: Result Type for All Fallible Operations

**Constraint**: All domain operations that can fail must return `Result<T, E>`, never throw exceptions.

**Rationale**: Makes errors explicit and type-checked

**Examples**:

```rust
fn generate_branch_name(issue: Issue) -> Result<BranchName, ValidationError>
fn process_issue_event(event: WorkflowEvent) -> Result<Vec<WorkflowAction>, WorkflowError>
```

**Exceptions Allowed**: Only for unrecoverable errors (out of memory, panic)

**Enforcement**: Code review, linter rules

**Test Coverage**: All error paths tested

---

### TS-3: No `any` Type in Domain Code

**Constraint**: Domain logic must never use `any` type or equivalent (dynamic typing)

**Rationale**: Type safety is primary benefit; dynamic typing defeats this

**Allowed Exceptions**: Serialization/deserialization boundaries only

**Enforcement**: Linter rule, code review

**Test Coverage**: N/A (compile-time enforcement)

---

### TS-4: Discriminated Unions for Error Types

**Constraint**: All error enums must be discriminated unions (tagged enums)

**Rationale**: Enables exhaustive matching, type-safe error handling

**Examples**:

```rust
enum WorkflowError {
    ValidationError(ValidationError),
    BusinessRuleViolation(RuleViolation),
    ConfigurationError(ConfigError),
    ExternalSystemError(ExternalError),
}
```

**Enforcement**: Rust's enum system (language-native)

**Test Coverage**: Pattern matching exhaustiveness

---

## Module Boundary Constraints

### MB-1: Business Logic Organized by Domain Concepts

**Constraint**: Business logic modules organized by domain concepts, not architectural layers

**Examples of Good Organization**:

- `workflows/` (issue workflows, PR workflows)
- `branching/` (branch name generation, validation)
- `configuration/` (config loading, merging)
- `events/` (event processing, routing)

**Examples of Poor Organization** (Forbidden):

- `core/` (too vague)
- `domain/` (architectural layer, not business concept)
- `usecases/` (architectural pattern, not domain)
- `services/` (technical term, not domain)

**Rationale**: Business-meaningful names aid understanding and navigation

**Enforcement**: Code review, module structure validation

---

### MB-2: External System Interfaces as Abstractions

**Constraint**: External system interfaces defined in business logic layer, implemented in infrastructure layer

**Pattern**:

```
workflows/          (business logic, defines IGitHubOperations trait)
infrastructure/     (implements IGitHubOperations with concrete GitHub client)
```

**Forbidden**: Business logic importing infrastructure implementations directly

**Rationale**: Dependency inversion, testability

**Enforcement**: Dependency graph analysis, code review

**Test Coverage**: Unit tests mock interfaces, integration tests use real implementations

---

### MB-3: Infrastructure Never Imports Business Logic

**Constraint**: Infrastructure layer may implement business-defined interfaces, but never import business logic modules

**Allowed**: Infrastructure implements `trait IGitHubOperations`

**Forbidden**: Infrastructure calls business logic functions directly

**Rationale**: Clean architecture dependency flow (dependencies point inward)

**Enforcement**: Dependency graph analysis (build system can enforce)

---

### MB-4: No Circular Dependencies

**Constraint**: Modules must form directed acyclic graph (DAG), no circular dependencies

**Enforcement**: Build system detects circular dependencies

**Resolution**: Extract shared code to separate module, refactor dependencies

---

## Error Handling Constraints

### EH-1: Expected Errors Are Values

**Constraint**: Expected errors (validation, business rules, API failures) must be returned as `Result::Err`, not thrown

**Examples of Expected Errors**:

- Validation failure (invalid input)
- Business rule violation (branch exists)
- GitHub API 404 (resource not found)
- Configuration parse error

**Examples of Unexpected Errors** (Panic OK):

- Out of memory
- Integer overflow in debug mode
- Assertion failures

**Rationale**: Expected errors are part of business logic flow, must be handled explicitly

**Enforcement**: Code review, error type definitions

---

### EH-2: All Error Types Include Context

**Constraint**: Every error type must include context for debugging

**Required Context**:

- Correlation ID (if available)
- Resource identifier (issue number, repository, etc.)
- Operation being performed
- Original cause (if wrapping another error)

**Example**:

```rust
struct BranchCreationError {
    correlation_id: CorrelationId,
    repository: RepositoryId,
    issue_number: IssueNumber,
    branch_name: BranchName,
    cause: GitHubApiError,
}
```

**Rationale**: Enables effective debugging and troubleshooting

**Enforcement**: Error type definitions, code review

---

### EH-3: Error Classification for Retry Logic

**Constraint**: All error types must implement classification method indicating if retryable

**Required Method**:

```rust
impl WorkflowError {
    fn is_retryable(&self) -> bool { ... }
}
```

**Classification**:

- Transient errors (5xx, timeouts): Retryable
- Configuration errors: Non-retryable (require manual fix)
- Business logic errors: Non-retryable (invalid operation)
- Client errors (4xx): Non-retryable (except 429 rate limit)

**Rationale**: Consistent retry behavior across system

**Enforcement**: Trait implementation, code review

---

## Testing Constraints

### TC-1: Business Logic ≥90% Unit Test Coverage

**Constraint**: Business logic code must achieve at least 90% line coverage

**Measurement**: Code coverage tool (e.g., tarpaulin for Rust)

**Exemptions**: Panic handlers, unreachable code paths

**Enforcement**: CI pipeline fails if coverage below threshold

---

### TC-2: All External System Interfaces Have Contract Tests

**Constraint**: Every interface (trait) must have contract tests validating implementations

**Contract Test Requirements**:

- Test all methods in interface
- Test error scenarios
- Test boundary conditions
- Verify postconditions

**Enforcement**: CI pipeline, code review

---

### TC-3: Integration Tests Use Test Doubles

**Constraint**: Integration tests must use test doubles (mocks, stubs, fakes) for external dependencies

**Allowed**: Real in-memory implementations (in-memory queue, mock HTTP server)

**Forbidden**: Real external services (production GitHub API, Azure Service Bus)

**Exceptions**: End-to-end tests may use real services in isolated test environment

**Rationale**: Fast, reliable, isolated tests

**Enforcement**: Test infrastructure guidelines, code review

---

### TC-4: All Behavioral Assertions Have Tests

**Constraint**: Every assertion in `assertions.md` must have corresponding automated test

**Coverage**: All EP-, WF-, CF-, EH-, GH-, OB-, PF-, SE-, EC- assertions

**Test Types**:

- Unit test for pure logic
- Integration test for component interactions
- E2E test for complete workflows

**Enforcement**: Test coverage report cross-referenced with assertions

---

## Performance Constraints

### PC-1: Event Processing Latency P95 <5 Seconds

**Constraint**: 95th percentile event processing latency must be under 5 seconds

**Measurement**: End-to-end from event receipt to acknowledgment

**Enforcement**: Performance tests in CI, monitoring in production

**Acceptance**: PR blocked if performance regression detected

---

### PC-2: GitHub API Call Latency P95 <2 Seconds

**Constraint**: 95th percentile GitHub API call latency must be under 2 seconds

**Measurement**: Time from request send to response received

**Enforcement**: Monitoring, alerting on threshold breach

---

### PC-3: Configuration Cache Hit Rate >90%

**Constraint**: Configuration cache must hit (not reload) >90% of requests

**Measurement**: Cache hits / total config requests

**Enforcement**: Monitoring, alert if hit rate drops

**Rationale**: Reduces GitHub API calls, improves latency

---

### PC-4: Support 100 Events/Second

**Constraint**: System must process 100 events per second under load without degradation

**Measurement**: Load test with synthetic events

**Acceptance Criteria**: P95 latency <10s, error rate <1%, no queue backlog after load

**Enforcement**: Load tests in staging environment

---

## Security Constraints

### SC-1: No Secrets in Code or Configuration Files

**Constraint**: Secrets (private keys, connection strings) must never appear in code, config, or logs

**Storage**: Azure Key Vault or equivalent secure vault

**Access**: Via managed identity (no credentials)

**Enforcement**: Secret scanning in CI, code review

**Test Coverage**: Automated secret scanning on commits

---

### SC-2: All GitHub API Calls Authenticated

**Constraint**: Every GitHub API call must include valid installation token

**Authentication Flow**: JWT → Installation Token → API Call

**Token Lifespan**: Max 1 hour (automatic renewal)

**Enforcement**: GitHub client implementation, integration tests

---

### SC-3: Least Privilege Permissions

**Constraint**: GitHub App and Azure managed identity must have minimal required permissions

**GitHub App Permissions**:

- Issues: Read/Write (required)
- Pull Requests: Read/Write (required)
- Contents: Read (config files only)

**Forbidden Permissions**:

- Code: Write (Task-Tactician does not write code)
- Actions: Write
- Administration: Any

**Enforcement**: Permission review during setup, periodic audits

---

### SC-4: All External Communication Over TLS 1.2+

**Constraint**: All network communication must use TLS 1.2 or higher

**Enforcement**: HTTP client configuration, Azure service defaults

---

## Enhanced Workflow Constraints (MVP)

### EW-1: Stale Detection Must Be Non-Blocking

**Constraint**: Stale issue detection runs asynchronously and cannot block event processing

**Rationale**: Staleness is informational; must not impact core workflows

**Trigger**: Scheduled job (e.g., daily) or periodic check

**Failure Handling**: Stale detection failures logged but do not alert

**Enforcement**: Separate execution context, monitoring strategy

**Test Coverage**: Verify event processing continues if stale detection fails

---

### EW-2: Milestone Prompting Is Advisory Only

**Constraint**: Milestone prompts are suggestions, not requirements

**Rationale**: Teams may not use milestones; must not block issue creation

**Behavior**: Add comment with suggestion, do not prevent issue workflows

**Enforcement**: Design review, functional tests

**Test Coverage**: Verify workflow continues without milestone

---

### EW-3: Dependency Detection Limited to Same Repository

**Constraint**: Dependency tracking only validates references within the same repository

**Rationale**: Cross-repository dependencies add complexity and API calls

**Validation**: Issue number existence check limited to current repository

**Future Enhancement**: Cross-repo support can be added later

**Enforcement**: Implementation design, integration tests

**Test Coverage**: Verify cross-repo references logged as warnings

---

### EW-4: Cycle Time Recording Is Best-Effort

**Constraint**: Cycle time metrics recorded on best-effort basis

**Rationale**: Metric recording failures should not block issue closure

**Failure Handling**: Log metric recording errors, continue workflow

**Data Quality**: Accept potential gaps in analytics data

**Enforcement**: Error handling design, monitoring

**Test Coverage**: Verify issue closes even if metric recording fails

---

### EW-5: Enhanced Features Respect Configuration Flags

**Constraint**: All enhanced features (stale detection, milestone prompting, dependency tracking, cycle time) must be configurable per repository

**Configuration Options**:

- `stale_detection.enabled`: Boolean (default: true)
- `stale_detection.threshold_days`: Integer (default: 30)
- `stale_detection.exclusion_labels`: Array (default: ["blocked", "on-hold"])
- `milestone_prompting.enabled`: Boolean (default: true)
- `dependency_tracking.enabled`: Boolean (default: true)
- `cycle_time.enabled`: Boolean (default: true)

**Enforcement**: Configuration validation, unit tests

**Test Coverage**: Verify each feature can be disabled independently

---

## Operational Constraints

### OC-1: Structured JSON Logging

**Constraint**: All log entries must be structured JSON with required fields

**Required Fields**:

- `timestamp` (ISO 8601)
- `level` (debug, info, warn, error)
- `message` (human-readable)
- `correlation_id` (request tracking)

**Optional Fields** (context-dependent):

- `repository`, `issue_number`, `pr_number`, `operation`, `error_type`

**Enforcement**: Logging framework configuration

---

### OC-2: All Operations Include Correlation ID

**Constraint**: Every operation must propagate correlation ID from event through all logs and external calls

**Source**: Event envelope or generated UUID if missing

**Propagation**: Passed as parameter or context to all operations

**Enforcement**: Code review, log validation

---

### OC-3: Metrics Emission for Key Operations

**Constraint**: Key operations must emit metrics for monitoring

**Required Metrics**:

- `events_processed_total` (counter)
- `events_failed_total` (counter, labeled by error_type)
- `event_processing_duration_seconds` (histogram)
- `github_api_calls_total` (counter, labeled by endpoint)
- `github_api_rate_limit_remaining` (gauge)

**Enforcement**: Instrumentation in workflow engine, monitoring dashboard validation

---

### OC-4: Circuit Breaker for GitHub API

**Constraint**: GitHub API calls must implement circuit breaker pattern

**Trigger**: Rate limit <20% remaining (1000 of 5000 requests)

**Behavior**: Open circuit, queue non-critical operations, close after reset

**Enforcement**: Rate limiter implementation, integration tests

---

## Idempotency Constraints

### IC-1: All GitHub API Operations Idempotent

**Constraint**: All GitHub operations must check current state before modifying

**Pattern**: Check existence → Compare with desired state → Apply change if different

**Examples**:

- Branch creation: Check if exists, create only if missing
- Label application: Check if already applied, apply only if missing

**Rationale**: Safe to retry, prevents duplicate operations

**Enforcement**: Operation implementation, integration tests

---

### IC-2: Event Processing Idempotent

**Constraint**: Processing same event multiple times must produce identical results

**Mechanism**: Deduplication cache + idempotent operations

**Enforcement**: Idempotency tests (process event twice, assert same outcome)

---

## Configuration Constraints

### CC-1: Configuration Schema Validation

**Constraint**: All configuration files must validate against defined schema

**Schema Format**: JSON Schema or equivalent

**Validation Points**:

- On load (reject invalid config)
- On save (pre-commit hook, CI)

**Error Handling**: Invalid config logs error, falls back to defaults

**Enforcement**: Validation library, CI pipeline

---

### CC-2: Configuration Hierarchy Precedence

**Constraint**: Configuration merge must follow strict precedence: Repository > Organization > System Default

**Deep Merge**: Nested objects merged recursively

**Enforcement**: Configuration manager unit tests

---

### CC-3: Configuration Cache TTL = 5 Minutes

**Constraint**: Configuration cache must expire after 5 minutes

**Rationale**: Balance between API call reduction and staleness

**Enforcement**: Cache implementation, TTL tests

---

## Language-Specific Constraints (Rust)

### RS-1: No `unwrap()` in Production Code

**Constraint**: Production code must not use `.unwrap()` or `.expect()` on `Result` or `Option`

**Allowed**: Test code only

**Alternative**: Use `?` operator, `match`, `if let`, `unwrap_or`, etc.

**Rationale**: Prevents panics in production

**Enforcement**: Clippy lint, code review

---

### RS-2: All Public APIs Documented

**Constraint**: All public functions, structs, traits must have documentation comments

**Format**: Rustdoc comments (`///`)

**Required Sections**: Description, examples (for complex functions), errors (for fallible ops)

**Enforcement**: Clippy lint (`missing_docs`), CI pipeline

---

### RS-3: Clippy Lints Must Pass

**Constraint**: Code must pass Clippy lints with no warnings

**Allowed Exceptions**: Specific lints disabled with justification (`#[allow(clippy::...)]`)

**Enforcement**: CI pipeline fails on warnings

---

## Summary Table

| Category | Count | Enforcement |
|----------|-------|-------------|
| Type System | 4 | Compiler, Linter |
| Module Boundaries | 4 | Code Review, Build System |
| Error Handling | 3 | Code Review, Tests |
| Testing | 4 | CI Pipeline, Coverage Tools |
| Performance | 4 | Load Tests, Monitoring |
| Security | 4 | Secret Scanning, Audits |
| Operational | 4 | Logging Framework, Monitoring |
| Idempotency | 2 | Tests |
| Configuration | 3 | Validation, Tests |
| Language-Specific | 3 | Linter, CI |

**Total Constraints**: 35

**Enforcement Mechanisms**:

- Compile-time: Type system, build system
- CI Pipeline: Tests, linters, secret scanning, coverage
- Runtime: Monitoring, alerting
- Process: Code review, audits

## Constraint Violation Handling

**During Development**: Code review rejects violations

**During CI**: Pipeline fails, PR blocked

**During Runtime**: Monitoring alerts, incident response

**Retrospective**: Post-incident review, constraint updates

## Future Constraint Evolution

Constraints may be added/modified based on:

- Operational learnings (post-incident reviews)
- Performance requirements changes
- Security threat landscape changes
- Team feedback (too restrictive/too lenient)

**Process**: Propose constraint change → Discuss → Document → Enforce
