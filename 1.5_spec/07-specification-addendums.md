# System Mechanics Specification - Required Addendums v1.5

## Overview

This document contains critical addendums identified during specification review. These fill semantic gaps that would cause incorrect implementation if omitted. All addendums are **normative** and **mandatory** for conformant implementations.

### Addendums Index

**v1.1 Addendums (A-N)**: Core specification gaps
**v1.3 Addendums (O-T)**: Authority, storage, and invariant semantics
**v1.4 Addendums (U-W)**: UI Composition, experience, and field permission semantics
**v1.5 Addendums (X-AY)**: Advanced features and comprehensive application development support

---

## ADDENDUM A: TRANSITION GRAMMAR (REQUIRED)

### Purpose

Transitions are referenced throughout the specification (InputIntent → DataState, SMS steps, Policy appliesTo: transition) but were never formally defined as a first-class grammar object.

### Formal Definition

```yaml
transition:
  name: <string>
  from: <DataState>
  to: <DataState>
  triggeredBy:
    inputIntent | smsFlow
  guarantees: []
  idempotency:
    key: <expression>
    scope: global | per_subject
  authorization:
    policySet: <PolicySet>
```

### Properties

- **Typed edges**: Transitions are strongly-typed connections between DataStates
- **Triggerable**: Transitions have explicit triggers (never implicit)
- **Idempotent**: All transitions MUST declare idempotency semantics
- **Authorizable**: Policies may apply to transitions specifically

### Example

```yaml
transition:
  name: ApplyPayment
  from: OrderRecord
  to: OrderRecord
  triggeredBy:
    inputIntent: SubmitPayment
  guarantees:
    - payment_validity
    - idempotent_application
  idempotency:
    key:
      - payment.transaction_id
      - order.id
    scope: global
  authorization:
    policySet: PaymentAuthPolicies
```

### Validation Rules

1. `from` and `to` DataStates MUST exist
2. `triggeredBy` MUST reference existing InputIntent or SMS Flow
3. Idempotency key expressions MUST be deterministic
4. Referenced policies MUST exist

### Implementation Notes

- Transitions are not executed directly - they describe state changes
- SMS flows or work units implement the actual transition logic
- Idempotency keys prevent duplicate transitions
- Transitions can be audited independently

---

## ADDENDUM B: IDEMPOTENCY & REPLAY SEMANTICS

### Normative Rules

These rules MUST be enforced by all conformant implementations:

1. **All transitions MUST be idempotent**
   - Executing the same transition multiple times produces the same result
   - Side effects MUST be idempotent

2. **Workers MUST tolerate duplicate delivery**
   - At-least-once delivery is assumed
   - Exactly-once is not guaranteed
   - Deduplication is mandatory

3. **SMS MAY replay steps arbitrarily**
   - Workers may see the same request multiple times
   - Results MUST be identical
   - State transitions MUST be idempotent

4. **Deduplication keys MUST be deterministic**
   - Same input always produces same key
   - Keys MUST NOT depend on timing or randomness
   - Keys SHOULD include causal identifiers

### Idempotency Configuration

```yaml
idempotency:
  strategy: deterministic_key | content_hash | timestamp
  key:
    - <field_expression>
    - <field_expression>
  scope: global | per_subject | per_realm
  ttl: <duration>
```

### Strategy Types

#### Deterministic Key
```yaml
idempotency:
  strategy: deterministic_key
  key:
    - inputIntent.id
    - dataState.primary_key
  scope: global
  ttl: 24h
```

**Use Case**: Most common, explicitly defined keys

#### Content Hash
```yaml
idempotency:
  strategy: content_hash
  key:
    - entire_payload
  scope: per_subject
  ttl: 1h
```

**Use Case**: When logical key not available, use content hash

#### Timestamp-Based
```yaml
idempotency:
  strategy: timestamp
  key:
    - dataState.id
    - created_at
  scope: global
  ttl: 7d
```

**Use Case**: Time-series data or append-only logs

### Scope Semantics

- **global**: Key is unique across entire system
- **per_subject**: Key scoped to actor/tenant
- **per_realm**: Key scoped to realm isolation boundary

### Implementation Requirements

```go
type IdempotencyChecker interface {
    // Check if key already processed
    Check(key string) (bool, error)
    
    // Mark key as processed with result
    Mark(key string, result Data, ttl time.Duration) error
    
    // Get cached result if exists
    GetResult(key string) (Data, bool, error)
}
```

**Storage**:
- Redis, Memcached, or in-memory (with limits)
- TTL-based expiry
- Result caching optional but recommended

---

## ADDENDUM C: CORRELATION & TRACE CONTEXT

### Purpose

End-to-end tracing, policy evaluation, and debugging require correlation across components. This was implicitly relied upon but never formalized.

### Execution Context Grammar

```yaml
executionContext:
  trace_id: <uuid>
  causation_id: <uuid>
  correlation_id: <uuid>
  actor:
    subject: <subject>
    roles: []
    attributes: {}
  metadata:
    origin: <string>
    session_id: <string>
    client_ip: <string>
```

### Propagation Rules

1. **trace_id**: Root of entire execution tree
   - Created when flow starts
   - Propagated to all steps
   - Never changes during flow

2. **causation_id**: Parent of current operation
   - Set to trace_id initially
   - Updated to previous step's trace_id
   - Creates causal chain

3. **correlation_id**: Business correlation
   - e.g., order_id, user_id
   - Enables business-level grouping
   - Optional but recommended

### Propagation Mechanisms

#### NATS Headers
```
X-Trace-ID: 550e8400-e29b-41d4-a716-446655440000
X-Causation-ID: 660e8400-e29b-41d4-a716-446655440001
X-Correlation-ID: order-12345
X-Actor-Subject: user-789
X-Actor-Roles: customer,verified
```

#### HTTP Headers
```
Traceparent: 00-550e8400e29b41d4a716446655440000-660e8400e29b41d4-01
X-Correlation-ID: order-12345
X-Actor-Subject: user-789
```

#### In-Process
Context object propagated through function calls.

### Usage in Signals

```yaml
signal:
  type: latency
  trace_id: 550e8400-e29b-41d4-a716-446655440000
  correlation_id: order-12345
  metrics:
    duration_ms: 450
```

**Benefit**: Can trace signal back to originating flow

### Usage in Policy Evaluation

```go
func EvaluatePolicy(policy Policy, ctx ExecutionContext) Decision {
    decision := evaluate(policy.When, ctx)
    
    // Record decision with full context
    auditLog := AuditEntry{
        TraceID: ctx.TraceID,
        CorrelationID: ctx.CorrelationID,
        Policy: policy.Name,
        Version: policy.Version,
        Actor: ctx.Actor,
        Decision: decision,
        Timestamp: time.Now(),
    }
    
    emitAuditLog(auditLog)
    
    return decision
}
```

---

## ADDENDUM D: TIME SEMANTICS

### Purpose

Time constraints appear throughout the spec (maxStaleness, deferred constraints, drain periods, TTL) but were never unified into a coherent model.

### Time Constraint Grammar

```yaml
timeConstraint:
  type: ttl | deadline | staleness | window
  value: <duration>
  enforcement: strict | advisory
```

### Constraint Types

#### TTL (Time To Live)
```yaml
timeConstraint:
  type: ttl
  value: 24h
  enforcement: strict
```

**Use Cases**:
- Cache expiry
- Session timeout
- Registration lifetime
- Temporary data

**Semantics**: Data/registration invalid after TTL

#### Deadline
```yaml
timeConstraint:
  type: deadline
  value: 5s
  enforcement: strict
```

**Use Cases**:
- Request timeout
- SLA enforcement
- Step timeout in flows

**Semantics**: Operation must complete before absolute time

#### Staleness
```yaml
timeConstraint:
  type: staleness
  value: 30m
  enforcement: advisory
```

**Use Cases**:
- Materialized view freshness
- Cache invalidation hints
- Read-your-write consistency

**Semantics**: Data is "fresh enough" within time window

#### Window
```yaml
timeConstraint:
  type: window
  value: 1h
  enforcement: advisory
```

**Use Cases**:
- Metric aggregation
- Rate limiting
- Cooldown periods

**Semantics**: Operations grouped within time window

### Integration with Other Concepts

**Materialization**:
```yaml
materialization:
  name: CustomerOrderSummary
  freshness:
    maxStaleness: 5m  # Time constraint
```

**Policy Shadow Period**:
```yaml
policyArtifact:
  lifecycle:
    state: shadow
    shadow_period: 24h  # Time constraint
```

**Worker Drain**:
```yaml
worker:
  shutdown:
    drain_timeout: 5m  # Time constraint
```

---

## ADDENDUM E: REALM / TENANT ISOLATION (MINIMAL)

### Purpose

Policies and routing already reference "domain" and "realm", but isolation semantics were never formalized.

### Realm Grammar

```yaml
realm:
  name: <string>
  isolation:
    data: strict | shared
    policy: strict | shared
    signals: isolated | shared
  quota:
    workers: <integer>
    storage: <bytes>
```

### Isolation Semantics

#### Data Isolation (strict)
- DataStates NEVER cross realm boundaries
- No queries span realms
- Storage physically separated (recommended)

#### Policy Isolation (strict)
- Policies scoped per realm
- No cross-realm policy evaluation
- Different realms can have conflicting policies

#### Signal Isolation
- **isolated**: Each realm has separate signal streams
- **shared**: Realms share signal infrastructure (different namespaces)

### Multi-Tenant Routing

```yaml
wts:
  routing:
    strategy: realm_aware
    rules:
      - if: realm == "customer_acme"
        routeTo: acme_worker_pool
      - if: realm == "customer_contoso"
        routeTo: contoso_worker_pool
```

### Implementation Pattern

```go
type RealmRouter struct {
    // Worker pools by realm
    pools map[string]WorkerPool
}

func (r *RealmRouter) Route(intent Intent, realm string) (Worker, error) {
    pool, ok := r.pools[realm]
    if !ok {
        return nil, ErrRealmNotFound
    }
    
    // Route within realm only
    return pool.SelectWorker(intent)
}
```

### Quota Enforcement

```go
func (s *Scheduler) enforceQuota(realm Realm, desired int) int {
    current := s.getCurrentWorkerCount(realm)
    quota := realm.Quota.Workers
    
    if current + desired > quota {
        log.Warn("Quota exceeded, clamping",
            "realm", realm.Name,
            "desired", desired,
            "quota", quota)
        return quota - current
    }
    
    return desired
}
```

---

## ADDENDUM F: ERROR CLASSIFICATION GRAMMAR

### Purpose

Different error types require different handling (retry vs fail, log level, user visibility). This was implicitly relied upon but never formalized.

### Execution Outcome Grammar

```yaml
executionOutcome:
  type: success | rejection | failure
  retryable: true | false
  reason: <string>
  context: {}
```

### Outcome Types

#### Success
```yaml
executionOutcome:
  type: success
  retryable: false
  reason: "Operation completed successfully"
  context:
    result_id: <string>
```

**Semantics**: Transition committed, no retry needed

#### Rejection
```yaml
executionOutcome:
  type: rejection
  retryable: false
  reason: "Policy denied: insufficient permissions"
  context:
    policy: <policy_name>
    version: <version>
```

**Semantics**: Prevented by policy or constraint, don't retry (would fail again)

**Causes**:
- Policy denial
- Constraint violation
- Invalid input
- Precondition not met

#### Failure
```yaml
executionOutcome:
  type: failure
  retryable: true
  reason: "Database timeout"
  context:
    error_code: <code>
    retry_after: <duration>
```

**Semantics**: Infrastructure or transient error, retry may succeed

**Causes**:
- Network timeout
- Service unavailable
- Resource exhaustion
- Transient failures

### Retry Decision Matrix

| Outcome Type | Retryable | Retry Strategy |
|--------------|-----------|----------------|
| Success | No | N/A |
| Rejection (Policy) | No | Never retry |
| Rejection (Validation) | No | Fix input first |
| Failure (Timeout) | Yes | Exponential backoff |
| Failure (Unavailable) | Yes | Retry with delay |
| Failure (Internal) | Maybe | Investigate first |

### Error Hierarchy

```
ExecutionError
├── ClientError (rejection)
│   ├── ValidationError
│   ├── AuthorizationError
│   ├── ConflictError
│   └── NotFoundError
├── InfrastructureError (failure, retryable)
│   ├── TimeoutError
│   ├── NetworkError
│   ├── ServiceUnavailableError
│   └── ResourceExhaustedError
└── SystemError (failure, investigate)
    ├── InternalError
    ├── DataCorruptionError
    └── InvariantViolationError
```

### Implementation

```go
type ExecutionOutcome struct {
    Type       OutcomeType
    Retryable  bool
    Reason     string
    Context    map[string]interface{}
    Error      error  // Original error for logging
}

func ClassifyError(err error) ExecutionOutcome {
    switch {
    case errors.Is(err, ErrValidation):
        return ExecutionOutcome{
            Type: Rejection,
            Retryable: false,
            Reason: "validation failed: " + err.Error(),
        }
    case errors.Is(err, ErrAuthorization):
        return ExecutionOutcome{
            Type: Rejection,
            Retryable: false,
            Reason: "not authorized",
        }
    case errors.Is(err, ErrTimeout):
        return ExecutionOutcome{
            Type: Failure,
            Retryable: true,
            Reason: "operation timed out",
            Context: map[string]interface{}{
                "retry_after": "5s",
            },
        }
    default:
        return ExecutionOutcome{
            Type: Failure,
            Retryable: true,
            Reason: "unknown error",
        }
    }
}
```

---

## ADDENDUM G: MACHINE-INGESTABLE SPECIFICATION CONTRACT

### Purpose

Specification must be unambiguous about what is normative vs illustrative to enable correct code generation and tooling.

### Specification Metadata

```yaml
specMetadata:
  version: v1.1
  format: yaml
  ingestible: true
  exampleSemantics: illustrative
  machineReadable: true
  generationTargets:
    - go
    - typescript
    - rust
```

### Normative vs Illustrative

#### Normative Elements (MUST implement)
- Grammar structure
- Validation rules
- Semantic constraints
- Execution semantics
- Conformance requirements

#### Illustrative Elements (examples only)
- Specific field names in examples
- Example data values
- Narrative descriptions
- Suggested tooling

### Code Generation Contract

**Generator Requirements**:

1. **Type Safety**: Generated code MUST be type-safe in target language
2. **Version Preservation**: Generated code MUST include version information
3. **Validation**: Generated code MUST include constraint validation
4. **Idempotency**: Generated code MUST include idempotency checks
5. **Correlation**: Generated code MUST propagate execution context

**Generator Guarantees**:

1. **Determinism**: Same spec always generates same code
2. **Completeness**: All grammar elements generate code
3. **Correctness**: Generated code passes all validation rules
4. **Documentation**: Generated code includes spec references

### Example Generator Template

**Input** (from spec):
```yaml
dataType:
  name: Order
  version: v3
  fields:
    id: uuid
    total: decimal
```

**Output** (generated):
```go
// Code generated from System Mechanics Specification v1.1
// Source: order.yaml
// DataType: Order v3
// Generated: 2024-01-15T14:23:45Z
// DO NOT EDIT

package generated

// OrderV3 represents Order at version v3
// Spec reference: order.yaml#dataType.Order.v3
type OrderV3 struct {
    ID    uuid.UUID       `json:"id" spec:"order.yaml#Order.v3.id"`
    Total decimal.Decimal `json:"total" spec:"order.yaml#Order.v3.total"`
}

// Version returns the schema version (normative)
func (o OrderV3) Version() string {
    return "v3"
}

// Validate checks all constraints (generated from constraint declarations)
func (o OrderV3) Validate() error {
    // Validation logic generated from constraints
    return nil
}
```

### Tooling Requirements

**Parser/Validator**:
- MUST reject invalid specifications
- MUST report errors with locations
- MUST validate cross-references
- MUST check semantic rules

**Code Generator**:
- MUST preserve intent
- MUST generate deterministically
- MUST include version information
- MUST validate generated code

**Runtime**:
- MUST enforce specifications exactly
- MUST NOT infer missing semantics
- MUST fail fast on ambiguity
- MUST log specification violations

---

## ADDENDUM H: BOUNDARY SEMANTICS (CLARIFICATION)

### Purpose

Boundaries are referenced throughout (atomic groups, work units, topology) but their precise meaning in different contexts was ambiguous.

### Boundary Types

#### Execution Domain Boundary
**Definition**: Shared fate/locality boundary

**Example**: All work units in same container/process

**Implications**:
- Atomic groups can span
- Direct invocation possible
- Shared memory accessible
- Fast communication

#### Trust Boundary
**Definition**: Security context changes

**Example**: Crossing from trusted to untrusted zone

**Implications**:
- Authentication required
- Authorization enforced
- Audit logged
- Data filtered

#### Consistency Boundary
**Definition**: Different consistency guarantees

**Example**: Strong consistency → eventual consistency

**Implications**:
- Atomicity not guaranteed across
- Compensation may be required
- Ordering may be lost

#### Geographic Boundary
**Definition**: Physical location changes

**Example**: Region us-east → region eu-west

**Implications**:
- Latency increases
- Data residency rules apply
- Failure domains separate

### Boundary Declaration

```yaml
boundary:
  name: <string>
  type: execution | trust | consistency | geographic
  properties:
    consistency: strong | eventual
    trusted: true | false
    region: <string>
```

### Atomic Group Boundary Rules (Clarified)

**Rule**: Atomic groups MUST NOT cross consistency boundaries or trust boundaries.

**Allowed**: Atomic groups MAY cross execution domain boundaries IF consistency and trust are preserved.

**Example - Valid**:
```yaml
atomic_group:
  boundary: inventory  # Same consistency boundary
  steps:
    - work_unit: validate_stock  # May run in different container
    - work_unit: reserve_stock   # As long as same consistency domain
```

**Example - Invalid**:
```yaml
atomic_group:
  boundary: invalid
  steps:
    - work_unit: reserve_stock     # Strong consistency, trusted
    - work_unit: send_notification # Eventual consistency, untrusted
      # ❌ Crosses consistency boundary
```

---

## ADDENDUM I: FAILURE INVARIANTS (NORMATIVE)

### Purpose

Multi-region failure scenarios rely on invariants that were discussed but never codified formally.

### Invariant 1: Authority Scope

**Statement**: Only one controller may be authoritative for a worker within a topology scope.

**Enforcement**: Linter MUST reject specifications with multiple authoritative schedulers in same scope.

**Validation**:
```yaml
# Valid
scheduler regional_us_east:
  scope: region
  authority: true

scheduler regional_eu_west:
  scope: region
  authority: true  # Different scope, OK

# Invalid
scheduler scheduler_a:
  scope: region
  authority: true

scheduler scheduler_b:
  scope: region
  authority: true  # ❌ Same scope, conflict
```

### Invariant 2: Signal Absence

**Statement**: Absence of signal MUST NOT be interpreted as failure or success.

**Enforcement**: Schedulers MUST NOT scale down solely due to missing signals.

**Implementation**:
```go
func (s *Scheduler) evaluateScaling(signals Signals) {
    if signals.IsEmpty() {
        // Use last known state
        signals = s.lastKnownSignals
        log.Warn("No signals, using last known state")
    }
    
    // Never scale down due to missing signals
    if !signals.IsRecent(30 * time.Second) {
        log.Warn("Stale signals, holding current state")
        return s.currentState
    }
}
```

### Invariant 3: Monotonic Existence

**Statement**: Workers MUST NOT be removed solely due to controller unavailability.

**Enforcement**: Infrastructure adapter MUST continue running workers when scheduler unreachable.

**Implementation**:
```go
func (k *KubernetesAdapter) Reconcile() {
    desiredState, err := k.scheduler.GetDesiredState()
    
    if err != nil {
        // Scheduler unavailable
        log.Warn("Scheduler unavailable, maintaining current state")
        return  // Don't scale down
    }
    
    // Apply desired state only if scheduler healthy
    k.applyState(desiredState)
}
```

### Invariant 4: No Cross-Scope Coupling

**Statement**: Failure in one topology scope MUST NOT force scaling decisions in another.

**Enforcement**: Schedulers MUST NOT observe signals outside their scope.

**Validation**:
```go
func (s *Scheduler) validateScope(signal Signal) bool {
    if signal.Source.Region != s.region {
        log.Warn("Ignoring out-of-scope signal",
            "signal_region", signal.Source.Region,
            "scheduler_region", s.region)
        return false
    }
    return true
}
```

### Invariant 5: Conservative Degradation

**Statement**: Under uncertainty, controllers MUST favor stability over optimization.

**Enforcement**: Scaling decisions MUST be conservative when signals are missing, delayed, or conflicting.

**Implementation**:
```go
func (s *Scheduler) calculateDesiredCount(signals Signals) int {
    if !signals.IsConfident() {
        // Don't make aggressive changes
        return s.currentCount
    }
    
    // Multiple conflicting signals
    if signals.HasConflict() {
        // Choose conservative option
        return max(s.currentCount, s.placement.Min)
    }
    
    // Normal evaluation
    return s.evaluateNormally(signals)
}
```

---

## ADDENDUM J: TRANSPORT SELECTION DETAILS

### Purpose

Clarify how runtime selects between in-process, local socket, HTTP, NATS, etc. This is crucial for performance and correctness.

### Selection Algorithm (Normative)

```
For work unit invocation:

1. CHECK: Are constraints satisfied by local execution?
   - Boundary matches
   - Policy allows
   - Version compatible
   
2. IF YES and local worker exists:
   → TRY: In-process call
   → IF FAILS: Continue to next option
   
3. TRY: Unix socket (same node)
   → IF AVAILABLE: Use socket
   → IF FAILS: Continue
   
4. TRY: Loopback HTTP (same node)
   → IF AVAILABLE: Use HTTP
   → IF FAILS: Continue
   
5. TRY: Direct TCP (same network)
   → IF AVAILABLE: Use TCP
   → IF FAILS: Continue
   
6. TRY: NATS subject
   → PUBLISH to work.boundary.workunit
   → TIMEOUT: 5s default
   → IF FAILS: Return error
   
7. NO OPTIONS: Return ErrNoWorkerAvailable
```

### Priority Matrix

| Transport | Latency | Reliability | Use When |
|-----------|---------|-------------|----------|
| In-process | 5-20μs | Highest | Same runtime, constraints allow |
| Unix socket | 50-200μs | High | Same node, different process |
| Loopback HTTP | 100-500μs | High | Same node, HTTP available |
| Direct TCP | 1-5ms | Medium | Same network, reachable |
| NATS | 1-10ms | High | Cross-boundary, discovery needed |

### Constraint Impact on Selection

**Example 1**: Strong consistency requirement
```yaml
work_unit:
  name: reserve_inventory
  requires:
    state: stock
    consistency: strong
```

**Selection**: MUST use endpoint with local state access, even if remote endpoint available

**Example 2**: Untrusted boundary crossing
```yaml
work_unit:
  name: external_api_call
  boundary: external
  trust: untrusted
```

**Selection**: MUST NOT use shared memory or unix socket (isolation required)

---

## ADDENDUM K: COMPLETION SEMANTICS (FLOWS)

### Purpose

Flows may have different completion requirements (all steps, any step, quorum). This was mentioned but never fully specified.

### Completion Grammar

```yaml
flow:
  name: <string>
  steps: []
  completion:
    policy: all | any | quorum
    quorum: <integer>  # Required if policy = quorum
```

### Completion Policies

#### All (Default)
```yaml
completion:
  policy: all
```

**Semantics**: Flow completes when all steps complete successfully

**Use Case**: Sequential workflows

#### Any
```yaml
completion:
  policy: any
```

**Semantics**: Flow completes when first step completes

**Use Case**: Racing alternatives, fallback strategies

**Example**:
```yaml
flow:
  name: GetUserData
  steps:
    - work_unit: read_from_cache
    - work_unit: read_from_primary_db
    - work_unit: read_from_replica
  completion:
    policy: any  # Use whichever responds first
```

#### Quorum
```yaml
completion:
  policy: quorum
  quorum: 2
```

**Semantics**: Flow completes when N steps complete

**Use Case**: Consensus, redundant operations

**Example**:
```yaml
flow:
  name: ReplicatedWrite
  steps:
    - work_unit: write_replica_1
    - work_unit: write_replica_2
    - work_unit: write_replica_3
  completion:
    policy: quorum
    quorum: 2  # Succeed when 2 of 3 complete
```

### Validation Rules

1. `quorum` MUST be ≤ number of steps
2. `quorum` MUST be > 0
3. `any` completion MUST NOT have compensations (ambiguous)
4. `quorum` with ordered steps is illegal (contradictory)

---

## ADDENDUM L: WORKER HEARTBEAT PROTOCOL

### Purpose

Heartbeats are critical for discovery and health, but the exact protocol was never formalized.

### Heartbeat Grammar

```yaml
heartbeat:
  frequency: <duration>
  timeout: <duration>
  payload:
    health: <health_status>
    load: <load_metrics>
    capabilities: []
    endpoints: []
```

### Health Status Values

```yaml
health_status: healthy | degraded | unhealthy | draining | inactive
```

**Semantics**:
- `healthy`: Accepting work at full capacity
- `degraded`: Accepting work but impaired
- `unhealthy`: Not accepting work, should be avoided
- `draining`: Finishing existing work, no new work
- `inactive`: Shutting down, remove from routing

### Load Metrics

```yaml
load_metrics:
  inflight: <integer>
  max_capacity: <integer>
  queue_depth: <integer>
  cpu_usage: <float>  # 0.0 - 1.0
  memory_usage: <float>  # 0.0 - 1.0
```

### Heartbeat Timing

**Recommended Values**:
- Frequency: 5 seconds
- Timeout (TTL): 15 seconds (3x frequency)
- Grace period: 30 seconds before removal

**Rationale**: Balances responsiveness with network overhead

### Implementation Protocol

**Worker Side**:
```go
func (w *Worker) HeartbeatLoop(ctx context.Context) {
    ticker := time.NewTicker(5 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-ctx.Done():
            // Send final heartbeat
            w.sendHeartbeat(Inactive)
            return
            
        case <-ticker.C:
            w.sendHeartbeat(w.currentHealth())
        }
    }
}

func (w *Worker) sendHeartbeat(health HealthStatus) error {
    hb := Heartbeat{
        WorkerID: w.instanceID,
        Health: health,
        Load: w.currentLoad(),
        Capabilities: w.capabilities,
        Endpoints: w.endpoints,
        Timestamp: time.Now(),
    }
    
    // Update KV (refreshes TTL)
    key := fmt.Sprintf("worker.%s.%s", w.name, w.instanceID)
    err := w.kv.Put(key, hb)
    
    // Also emit as signal for schedulers
    w.nats.Publish("signal.worker."+w.name+".heartbeat", hb)
    
    return err
}
```

**Consumer Side**:
```go
func (r *RoutingCache) WatchHeartbeats(ctx context.Context) {
    // Watch KV for changes
    watcher, _ := r.kv.WatchAll()
    
    for {
        select {
        case <-ctx.Done():
            return
            
        case update := <-watcher.Updates():
            switch update.Operation() {
            
            case nats.KeyValuePut:
                hb := ParseHeartbeat(update.Value())
                r.updateEndpoint(hb)
                
            case nats.KeyValueDelete:
                // TTL expired
                workerID := extractWorkerID(update.Key())
                r.removeEndpoint(workerID)
                log.Info("Worker expired", "worker", workerID)
            }
        }
    }
}
```

---

## ADDENDUM M: FLOW FAILURE SEMANTICS

### Purpose

Failure handling strategies were mentioned but never fully formalized with all edge cases.

### Failure Policy Grammar (Extended)

```yaml
failure:
  policy: abort | retry | compensate | continue
  max_attempts: <integer>
  backoff:
    strategy: exponential | linear | fixed
    initial: <duration>
    max: <duration>
  compensations: []
  on_exhaustion: abort | continue | escalate
```

### Failure Policies

#### Abort (Fail Fast)
```yaml
failure:
  policy: abort
```

**Semantics**: Stop immediately on first failure, no retries

**Use Case**: Non-recoverable errors, validation failures

#### Retry
```yaml
failure:
  policy: retry
  max_attempts: 3
  backoff:
    strategy: exponential
    initial: 1s
    max: 30s
  on_exhaustion: abort
```

**Semantics**: Retry failed step with backoff

**Requirements**: Work unit MUST be idempotent

**Use Case**: Transient failures, network timeouts

#### Compensate (Saga Pattern)
```yaml
failure:
  policy: compensate
  compensations:
    - on: charge_card
      do: release_inventory
    - on: reserve_inventory
      do: release_reservation
```

**Semantics**: Execute reverse operations on failure

**Use Case**: Cross-boundary operations, distributed workflows

**Requirements**: Compensation work units MUST exist and be idempotent

#### Continue (Best Effort)
```yaml
failure:
  policy: continue
  on_exhaustion: continue
```

**Semantics**: Mark step as failed but continue flow

**Use Case**: Non-critical operations, audit logs, notifications

### Backoff Strategies

#### Exponential
```
Attempt 1: wait initial (1s)
Attempt 2: wait initial * 2 (2s)
Attempt 3: wait initial * 4 (4s)
Max: 30s
```

#### Linear
```
Attempt 1: wait initial (1s)
Attempt 2: wait initial + increment (2s)
Attempt 3: wait initial + 2*increment (3s)
```

#### Fixed
```
Every attempt: wait fixed duration (5s)
```

### Exhaustion Handling

**abort**: Return failure outcome, stop flow

**continue**: Mark step failed, continue to next step

**escalate**: Emit critical signal, wait for intervention

---

## ADDENDUM N: DATA REFERENCE PATTERN (LARGE PAYLOADS)

### Purpose

Large data must not flow through NATS or control channels. Reference pattern was mentioned but never formalized.

### Data Reference Grammar

```yaml
dataReference:
  type: inline | reference
  reference:
    uri: <uri>
    format: <format>
    size: <bytes>
    checksum: <hash>
```

### Reference Types

#### Inline (Small Data)
```yaml
data:
  type: inline
  value:
    order_id: "order-123"
    amount: 99.99
```

**Threshold**: < 1MB (configurable)

#### Object Storage Reference
```yaml
data:
  type: reference
  reference:
    uri: "s3://bucket/order-12345.json"
    format: "json"
    size: 5242880
    checksum: "sha256:abcd..."
```

**Use Case**: Large payloads, batch data

#### Stream Reference
```yaml
data:
  type: reference
  reference:
    uri: "nats://stream/ORDER_EVENTS"
    format: "cloudevents"
    offset: 12345
```

**Use Case**: Event sourcing, append-only logs

### Implementation Pattern

**Worker Receives Reference**:
```go
func (w *Worker) HandleIntent(intent Intent) (Data, error) {
    var inputData Data
    
    switch intent.Data.Type {
    
    case Inline:
        // Use directly
        inputData = intent.Data.Value
        
    case Reference:
        // Fetch from storage
        ref := intent.Data.Reference
        inputData, err = w.storage.Get(ref.URI)
        if err != nil {
            return nil, fmt.Errorf("failed to fetch reference: %w", err)
        }
        
        // Validate checksum
        if !validateChecksum(inputData, ref.Checksum) {
            return nil, ErrChecksumMismatch
        }
    }
    
    return w.execute(inputData)
}
```

**Worker Returns Large Data**:
```go
func (w *Worker) ReturnLargeResult(result Data) (DataReference, error) {
    if len(result) < w.inlineThreshold {
        // Return inline
        return DataReference{
            Type: Inline,
            Value: result,
        }, nil
    }
    
    // Store in object storage
    uri, err := w.storage.Put(result)
    if err != nil {
        return nil, err
    }
    
    checksum := calculateChecksum(result)
    
    return DataReference{
        Type: Reference,
        Reference: Reference{
            URI: uri,
            Format: "json",
            Size: len(result),
            Checksum: checksum,
        },
    }, nil
}
```

---

## ADDENDUM O: MUTABILITY AND AUTHORITY (NEW in v1.3)

### Purpose

SMS defines mutability as an explicit, governed property of data models. This addendum formalizes exclusive write authority at the entity level.

### Mutability Grammar

```yaml
mutability:
  scope: entity | append-only
  exclusive: true | false
  authority: <authority_ref>
```

### Core Rules

1. A model MAY declare entity-scoped mutability, in which exactly one authority is permitted to accept write operations for a given entity at any point in time.

2. When entity-scoped mutability is declared:
   - Write authority SHALL be exclusive
   - Write authority SHALL be explicit and observable
   - Write authority MAY change over time via a governed transition protocol

3. SMS SHALL NOT infer authority implicitly from topology, placement, or traffic.

### Why Entity-Scoped Authority

**Model-scoped authority** collapses availability when models are interdependent.

**Entity-scoped authority** preserves isolation, enabling:
- Partial failure tolerance
- Global scalability
- Per-entity write availability

### Validation Rules

1. Entities with `exclusive: true` MUST reference an authority declaration
2. Entities SHOULD NOT require multi-entity CAS operations
3. Authority must be resolvable for every entity

---

## ADDENDUM P: AUTHORITY STATE AND TRANSITIONS (NEW in v1.3)

### Purpose

Formalizes the runtime state machine for entity authority and the protocol for safely transferring authority between regions.

### Authority State Grammar

```yaml
authorityState:
  entity_id: <uuid>
  authority_region: <string>
  authority_epoch: <integer>
  authority_lease: <uuid>
  authority_status: ACTIVE | TRANSITIONING
  entity_version: <integer>
```

### State Definitions

| State | Description |
|-------|-------------|
| `ACTIVE(region, epoch)` | Region is accepting writes for entity |
| `TRANSITIONING(from, to, epoch)` | Authority is being transferred |

### Authority Transition Protocol

**Phase 0 - Steady State**:
- Authority is `ACTIVE(R1, epoch=N)`
- R1 accepts writes
- Other regions reject writes

**Phase 1 - Transition Initiation**:
1. New authority epoch allocated: `epoch = N + 1`
2. Entity enters `TRANSITIONING(R1 → R2, epoch=N+1)`
3. R1 stops accepting new writes
4. R1 completes in-flight writes

**Phase 2 - Authority Cutover**:
1. All prior writes committed
2. Authority lease granted to R2
3. State becomes `ACTIVE(R2, epoch=N+1)`
4. R2 begins accepting writes

**Phase 3 - Post-Cutover**:
- Views continue uninterrupted
- R1 transitions to read-only for entity

### Failure Safety

**Failure During Phase 1**:
- Authority remains with R1
- Epoch does not advance
- Writes temporarily rejected

**Failure During Phase 2**:
- Authority state recoverable from epoch + lease
- Either R1 (epoch N) or R2 (epoch N+1) authoritative
- Never both

---

## ADDENDUM Q: STORAGE ROLES (NEW in v1.3)

### Purpose

SMS distinguishes between data storage and control storage. This addendum formalizes the distinction.

### Storage Role Grammar

```yaml
storage:
  role: data | control
  rebuildable: true | false
```

### Role Definitions

| Role | Characteristics | Examples |
|------|-----------------|----------|
| `data` | Append-only, rebuildable, partitionable | Event streams, entity state |
| `control` | Strongly consistent, small, coordination | Authority KV, leases |

### Control Storage Rules

1. Control storage MUST be strongly consistent
2. Control storage SHALL be modified only by designated control components
3. Implementations SHALL NOT treat control storage as a source of domain data
4. Control storage includes:
   - Authority state (`SMS.AUTHORITY`)
   - Worker registrations
   - Policy artifacts
   - Lease management

### Data Storage Rules

1. Data storage MAY be partitioned, replayed, or materialized
2. Data storage is the source of truth for domain events
3. All derived state MUST be reconstructible from data storage

---

## ADDENDUM R: VIEW AUTHORITY INDEPENDENCE (NEW in v1.3)

### Purpose

Views derived from entity events must tolerate events from multiple regions and authority epochs.

### View Extension Grammar

```yaml
materialization:
  authority_agnostic: true | false
  epoch_tolerant: true | false
```

### Rules

1. Views derived from entity events SHALL be authority-agnostic unless explicitly stated otherwise

2. Views MUST tolerate:
   - Events originating from multiple regions
   - Events written under different authority epochs

3. Views SHALL NOT assume a fixed authoritative region for their inputs

### View Processing Requirements

1. Views must process events in epoch order, not arrival order
2. Views must include freshness metadata:
   ```yaml
   view_metadata:
     computed_at: <timestamp>
     source_epochs: [<epoch>, ...]
     freshness: current | stale
   ```

3. Views may lag during authority transitions but must eventually converge

---

## ADDENDUM S: ENTITY-SCOPED AVAILABILITY (NEW in v1.3)

### Purpose

SMS permits temporary write unavailability at the entity scope as part of governed authority transitions.

### Availability Grammar

```yaml
availability:
  write_scope: entity
  pause_allowed: true | false
  read_continuity: best-effort | guaranteed
```

### Rules

1. Write unavailability SHALL be bounded in duration
2. Write unavailability SHALL NOT affect unrelated entities
3. Write unavailability SHALL NOT interrupt read access to existing views
4. This behavior SHALL be considered compliant with SMS availability guarantees

### Write Failure Strategies

```yaml
write_failure:
  strategy: reject | client_queue | regional_buffer
```

| Strategy | Description |
|----------|-------------|
| `reject` | Immediately fail writes (recommended) |
| `client_queue` | Buffer at client (non-authoritative) |
| `regional_buffer` | Buffer in adjacent region (expires) |

### Buffering Rules

1. Buffered writes SHALL be non-authoritative
2. Buffered writes SHALL NOT replicate
3. Buffered writes SHALL expire if authority is not restored

---

## ADDENDUM T: GOVERNANCE-BOUND AUTHORITY MOBILITY (NEW in v1.3)

### Purpose

Authority migration for a model must be explicitly governed by policy.

### Migration Grammar

```yaml
authority:
  migration:
    allowed: true | false
    strategy: pause-and-cutover
    triggers:
      - type: load | latency | manual | time
        condition: <expression>
    constraints:
      - type: region | governance
        allowed: [<region>, ...]
        rule: <expression>
    governed_by: <policy_ref>
```

### Governance Rules

1. Models enabling authority migration MUST declare:
   - Whether migration is allowed
   - The conditions under which it may occur
   - The governance rules constraining region selection

2. Authority migration SHALL NOT violate:
   - Data residency constraints
   - Compliance requirements
   - Policy constraints

### Migration Suitability

| Use Case | Migration Recommendation |
|----------|-------------------------|
| High write contention | Disable migration |
| User-local data | Allow migration |
| Financial invariants | Prefer stable authority |
| Follow-the-sun workloads | Allow with triggers |

---

## FORMAL INVARIANTS (NEW in v1.3)

### Purpose

These invariants MUST be enforced by all conformant implementations to guarantee authority correctness.

### I1. Authority Singularity

At any point in time, an entity SHALL have at most one write-authoritative region.

### I2. Epoch Monotonicity

Authority epochs SHALL increase monotonically and SHALL NOT be reused.

### I3. No Overlapping Authority

No two regions SHALL simultaneously accept writes for the same entity.

### I4. CAS Safety

A write SHALL be accepted if and only if:
- The authority region is current
- The authority epoch matches
- The entity version matches

### I5. View Continuity

Derived views SHALL remain readable across authority transitions.

### I6. Rebuildability

All derived state SHALL be reconstructible from authoritative event history.

### I7. Governance Compliance

Authority transitions SHALL NOT violate declared governance constraints.

### I8. Failure Determinism

On failure during authority transition, the system SHALL resolve to a single authoritative region without ambiguity.

### Implementation Validation

```go
// Validate authority singularity
func (s *System) ValidateAuthoritySingularity(entityID string) error {
    authorities := s.getActiveAuthorities(entityID)
    if len(authorities) > 1 {
        return ErrMultipleAuthorities
    }
    return nil
}

// Validate epoch monotonicity
func (s *System) ValidateEpochMonotonicity(entityID string, newEpoch int) error {
    currentEpoch := s.getCurrentEpoch(entityID)
    if newEpoch <= currentEpoch {
        return ErrEpochNotMonotonic
    }
    return nil
}
```

---

## CAS TOKEN SEMANTICS (NEW in v1.3)

### Purpose

CAS tokens preserve linearizability across authority migrations.

### Token Structure

```yaml
cas_token:
  entity_id: <uuid>
  entity_version: <integer>
  authority_epoch: <integer>
  authority_region: <string>
  lease_id: <uuid>
```

### Validation Rules

A CAS write is accepted only if:
1. Request is sent to `authority_region`
2. `authority_epoch` matches current epoch
3. `entity_version` matches current version
4. `lease_id` is valid

### CAS Across Migration

Before migration:
```
epoch = 3, version = 120, region = us-east
```

After migration:
```
epoch = 4, version = 120, region = eu-west
```

A CAS token issued before migration **fails** due to epoch mismatch, even if data content is unchanged.

### Token Encoding

```go
type CASToken struct {
    EntityID        uuid.UUID
    EntityVersion   int64
    AuthorityEpoch  int64
    AuthorityRegion string
    LeaseID         uuid.UUID
}

func (t CASToken) Encode() string {
    payload := cbor.Marshal(t)
    signature := hmac.Sign(payload, secretKey)
    return base64.URLEncode(append(payload, signature...))
}

func (t *CASToken) Validate(expected CASToken) error {
    if t.AuthorityEpoch != expected.AuthorityEpoch {
        return ErrEpochMismatch
    }
    if t.EntityVersion != expected.EntityVersion {
        return ErrVersionMismatch
    }
    if t.AuthorityRegion != expected.AuthorityRegion {
        return ErrWrongRegion
    }
    return nil
}
```

---

## REVIEWER OBJECTION/RESPONSE (NEW in v1.3)

### Objection 1: "Moving authority risks split-brain writes"

**Response**: SMS explicitly forbids implicit authority. Write authority is granted via a single, observable lease tied to a monotonic epoch. During transitions, no region accepts new writes. There is no dual-write window.

### Objection 2: "CAS semantics break if authority changes"

**Response**: CAS tokens include the authority epoch. Any token minted before migration becomes invalid after transition, preserving linearizability and preventing stale writes.

### Objection 3: "This introduces downtime"

**Response**: SMS allows entity-scoped write pauses as a deliberate tradeoff. Reads remain uninterrupted via derived views. This is explicitly declared behavior, not failure.

### Objection 4: "Why not just pin authority to a model?"

**Response**: Model-scoped authority collapses availability when models are interdependent. Entity-scoped authority preserves isolation, enabling partial failure tolerance and global scalability.

### Objection 5: "Why is this not just an implementation detail?"

**Response**: Authority determines correctness, not performance. Any system permitting writes must define who may write, when, and under what guarantees. SMS captures this intent explicitly to prevent unsafe implementations.

---

## CRITICAL IMPLEMENTATION WARNINGS

### Warning 1: Do Not Embed Control Logic in Grammar

❌ **Wrong**: Adding scheduler algorithms to grammar

✅ **Right**: Keep grammar declarative, algorithms in runtime

### Warning 2: Do Not Cache Policies Indefinitely

❌ **Wrong**: Load policies once at startup

✅ **Right**: Subscribe to policy stream, update on changes

### Warning 3: Do Not Skip Shadow Phase

❌ **Wrong**: Deploy new version directly to production

✅ **Right**: Deploy in shadow, monitor, then promote

### Warning 4: Do Not Ignore Signals During Scale-Down

❌ **Wrong**: Scale down immediately when load drops

✅ **Right**: Wait for sustained low load, respect cooldown

### Warning 5: Do Not Bypass Idempotency Checks

❌ **Wrong**: Trust "exactly-once" delivery guarantees

✅ **Right**: Always check idempotency, at-least-once assumed

### Warning 6: Do Not Block on NATS in Hot Path

❌ **Wrong**: Synchronous NATS request for every invocation

✅ **Right**: Use cached routing, NATS for updates only

### Warning 7: Do Not Couple Workers to Each Other

❌ **Wrong**: Worker A directly calls Worker B

✅ **Right**: Flow coordinator routes through intent subjects

### Warning 8: Do Not Make Scheduler Authoritative for Data

❌ **Wrong**: Scheduler decides what data to process

✅ **Right**: Scheduler decides worker count, runtime routes work

---

## IMPLEMENTATION SUCCESS CRITERIA

A conformant implementation MUST demonstrate:

1. ✅ **Correctness**: All validation rules enforced
2. ✅ **Performance**: Local invocation < 50μs, remote < 10ms
3. ✅ **Resilience**: Survives chaos testing (20% failure rate)
4. ✅ **Consistency**: Idempotency prevents duplicates
5. ✅ **Evolution**: Zero-downtime version upgrades
6. ✅ **Observability**: End-to-end tracing works
7. ✅ **Scale**: Linear scaling to 10,000 workers
8. ✅ **Availability**: Operates during control plane outage
9. ✅ **Security**: Fail-closed policy enforcement
10. ✅ **Multi-Region**: Autonomous operation during partition

**v1.3 Additions**:

11. ✅ **Authority Singularity**: Only one region writes per entity
12. ✅ **Epoch Monotonicity**: Authority epochs never decrease
13. ✅ **CAS Correctness**: CAS tokens validated across migrations
14. ✅ **View Continuity**: Views readable during authority transitions
15. ✅ **Governance Compliance**: Authority transitions respect policies

**v1.4 Additions**:

16. ✅ **Composition Resolution**: Correct views resolved from compositions
17. ✅ **Experience Routing**: Entry points and includes correctly enforced
18. ✅ **Field Permission Enforcement**: Visibility and masking applied
19. ✅ **Navigation Hierarchy**: Parent relationships form acyclic tree
20. ✅ **Indicator Evaluation**: Dynamic navigation indicators update

---

## ADDENDUM U: PRESENTATION COMPOSITION SEMANTICS (NEW in v1.4)

### Purpose

Defines how compositions group views into workspaces and the semantic meaning of view relationships.

### Composition Resolution

When a composition is resolved:

1. **Primary View Resolution**
   - The primary view MUST exist and be resolvable
   - The primary view defines the composition's identity
   - Policy evaluation starts with the primary view's `policy_set`

2. **Related View Resolution**
   - Related views are resolved in dependency order
   - Required views (`required: true`) MUST succeed or the composition fails
   - Optional views MAY fail gracefully

3. **Data Scope Application**
   - `data_scope` filters are applied to related view queries
   - Template expressions (e.g., `{{entity.id}}`) are resolved from composition context
   - Invalid scopes result in empty result sets, not errors

### View Relationship Semantics

| Relationship | Resolution Timing | Failure Behavior | Caching |
|--------------|-------------------|------------------|---------|
| `derived` | With primary | Fail composition | With primary |
| `detail` | On navigation | Independent | Separate |
| `contextual` | On navigation | Independent | Separate |
| `action` | On trigger | Fail action | None |
| `supplementary` | Lazy | Silent omit | Separate |
| `historical` | On request | Independent | Separate |

### Basis Resolution

The `basis` property links related views to entity relationships:

```yaml
related:
  - view: TransactionListView
    basis: transaction_debits_account  # Uses relationship for query
```

**Resolution**:
1. Find the named relationship in the specification
2. Use relationship cardinality to determine query type
3. Apply relationship's causality rules to ordering

### Multiple View Instances

When the same view appears multiple times with different bases:

```yaml
related:
  - view: AccountDetailsView
    basis: transaction_debits_account
    alias: "From Account"
  - view: AccountDetailsView
    basis: transaction_credits_account
    alias: "To Account"
```

**Rules**:
- Each instance gets unique context from its `basis`
- `alias` distinguishes instances in navigation
- Policy applies to each instance independently

---

## ADDENDUM V: EXPERIENCE AND NAVIGATION SEMANTICS (NEW in v1.4)

### Purpose

Defines how experiences group compositions and how navigation hierarchy is constructed.

### Experience vs Composition

| Aspect | Composition | Experience |
|--------|-------------|------------|
| Contains | Views | Compositions |
| Scope | Single workspace | Complete application |
| Policy | Per-workspace | Default for all |
| Entry | Via navigation | Application entry |
| Lifecycle | Navigable | Session-bound |

### Experience Resolution

1. **Entry Point**
   - Experience MUST have valid `entry_point` composition
   - Entry point MUST be in `includes` list
   - Unauthenticated users redirected per `unauthenticated.redirect_to`

2. **Includes Validation**
   - All compositions in `includes` MUST exist
   - Compositions not in `includes` are inaccessible from this experience
   - Cross-experience navigation requires explicit handling

3. **Policy Application**
   - Experience `policy_set` applies as default
   - Composition `policy_set` overrides experience default
   - Related view policies override composition default

### On_unauthorized Behavior

| Behavior | Navigation | Content | Use Case |
|----------|------------|---------|----------|
| `conceal` | Hidden | N/A | Customer UI |
| `indicate` | Visible, disabled | Access denied hint | Admin UI |
| `redirect` | Visible, redirects | Target content | SSO flows |
| `deny` | Visible | Error message | Debug/audit |

### Navigation Hierarchy Construction

1. **Root Identification**
   - Compositions without `parent` are roots
   - Experience `entry_point` is typically a root

2. **Tree Building**
   - Follow `parent` references to build tree
   - Cycle detection MUST reject specification
   - Maximum depth implementation-defined (recommend: 8)

3. **Label Resolution**
   - Template expressions resolved at render time
   - Unknown variables render as empty string
   - Example: `"{{account.name}}"` → "Checking Account"

### Purpose Grouping

Purposes enable semantic grouping without prescribing layout:

```yaml
# Specification declares intent
navigation:
  purpose: accounts

# Runtime decides rendering
- Primary nav: home, accounts, transactions
- Utility nav: settings, help
- Hidden: authentication
```

### Indicator Evaluation

Indicators update dynamically based on expression evaluation:

```yaml
indicator:
  when: open_alerts > 0
  severity: warning
```

**Evaluation**:
- Expressions evaluated against current data context
- Polling interval implementation-defined (recommend: 30s)
- WebSocket updates MAY provide real-time updates
- Stale indicators MUST NOT block navigation

---

## ADDENDUM W: FIELD-LEVEL PERMISSIONS (NEW in v1.4)

### Purpose

Defines how field visibility and masking are enforced within views.

### Field Permission Evaluation

For each field in a view:

1. **Visibility Check**
   ```
   visible: true       → Show field
   visible: false      → Hide field
   visible: { policy } → Evaluate policy, then show/hide
   ```

2. **Mask Check** (if visible)
   ```
   IF mask.unless expression is true:
     Show full value
   ELSE:
     Apply mask.type with mask.reveal
   ```

### Mask Type Semantics

| Type | Description | Example Input | Example Output |
|------|-------------|---------------|----------------|
| `full` | Complete redaction | `123-45-6789` | `***-**-****` |
| `partial` | Reveal specified portion | `123-45-6789` | `***-**-6789` |
| `hash` | One-way hash | `123-45-6789` | `a1b2c3d4` |

### Reveal Types

| Reveal | Meaning | Example |
|--------|---------|---------|
| `last4` | Last 4 characters | `****6789` |
| `first3` | First 3 characters | `123*****` |
| `none` | No reveal | `********` |

### Policy-Based Visibility

When visibility references a policy:

```yaml
fieldPermissions:
  - field: ssn
    visible:
      policy: ViewSensitiveData
```

**Resolution**:
1. Evaluate `ViewSensitiveData` policy against current subject
2. If `allow`, proceed to mask evaluation
3. If `deny`, treat as `visible: false`

### Inheritance Rules

1. **View-level policy** applies to all fields by default
2. **Field-level policy** overrides view-level
3. **Mask evaluation** only occurs if visible
4. **Absent permission** means `visible: true, mask: none`

### Performance Considerations

- Field permission evaluation SHOULD be cached per (subject, view, version)
- Cache invalidation on policy change
- Mask application MUST NOT reveal original value in logs
- Masked values MUST NOT be reversible (except by authorized access)

### Audit Requirements

Field permission decisions SHOULD be auditable:

```yaml
# Audit log entry
field_access:
  field: ssn
  view: CustomerDetailsView
  subject: admin@example.com
  visible: true
  masked: true
  mask_type: partial
  timestamp: 2026-01-11T10:30:00Z
```

---

## CONCLUSION

These addendums complete the specification by formalizing concepts that were:
- Implicitly relied upon
- Discussed but not codified
- Critical for correctness
- Required for implementation

With these addendums, the specification is:
- **Complete**: No semantic gaps
- **Unambiguous**: One correct interpretation
- **Implementable**: Sufficient detail for building
- **Testable**: Clear success criteria

All addendums are normative and must be included in conformance testing.

---

# v1.5 Addendums: Advanced Features and Complete Application Development

## Overview

The v1.5 addendums extend SMS with advanced features that enable complete end-to-end application development while preserving the intent-first philosophy. These addendums add capabilities for:

- **User Experience**: Form binding, presentation hints, accessibility, conversational interfaces
- **Data Management**: Assets, search, structured content, spatial types, graph queries
- **Collaboration**: Sessions, notifications, collaborative editing, delegation
- **Integration**: External systems, scheduled triggers, edge devices
- **Governance**: Data governance, compliance tracking, audit trails
- **Orchestration**: Durable workflows, multi-step processes
- **Advanced Types**: Inference-derived fields, spatial semantics, structured content

### v1.5 Addendums Index

| ID | Name | Priority | Category |
|----|------|----------|----------|
| X | Form Intent Binding | CRITICAL | User Experience |
| Y | View Materialization Contract | HIGH | Data Management |
| Z | Intent Delivery Contract | HIGH | Integration |
| AA | Session Contract | MEDIUM | Collaboration |
| AB | Indicator Source Binding | MEDIUM | User Experience |
| AC | Composition Fetch Semantics | HIGH | Data Management |
| AD | Mask Expression Grammar | MEDIUM | Security |
| AE | Navigation Semantics | LOW | User Experience |
| AF | View Availability Policy | LOW | Data Management |
| AG | Presentation Hints | MEDIUM | User Experience |
| AH | Surface Container Semantics | MEDIUM | User Experience |
| AI | Accessibility Preferences | HIGH | User Experience |
| AJ | End-to-End Reference Flow | REFERENCE | Documentation |
| AK | Asset Semantics | HIGH | Data Management |
| AL | Search Semantics | HIGH | Data Management |
| AM | Scheduled Trigger Semantics | MEDIUM | Orchestration |
| AN | Notification Channel Semantics | MEDIUM | Collaboration |
| AO | Collaborative Session Semantics | MEDIUM | Collaboration |
| AP | Structured Content Type | MEDIUM | Advanced Types |
| AQ | Inference Derived Fields | LOW | Advanced Types |
| AR | Spatial Type Semantics | LOW | Advanced Types |
| AS | Graph Query Semantics | MEDIUM | Data Management |
| AT | Durable Workflow Semantics | MEDIUM | Orchestration |
| AU | Delegation Consent Semantics | HIGH | Governance |
| AV | Conversational Experience Semantics | MEDIUM | User Experience |
| AW | External Integration Semantics | HIGH | Integration |
| AX | Data Governance Semantics | HIGH | Governance |
| AY | Edge Device Semantics | LOW | Integration |

### Integration Philosophy

v1.5 addendums maintain SMS core principles:

1. **Intent-First**: User actions remain declarative, runtime decides how
2. **Runtime-Decided**: Implementations choose optimal strategies
3. **Backward Compatible**: v1.5 is purely additive, no breaking changes
4. **Governance-Bound**: All features respect policy and compliance
5. **Observable**: Complete tracing and audit support
6. **Rebuildable**: All state derives from authoritative events

---

## v1.5 ADDENDUMS

<!-- Addendums X-AY will be integrated below this line -->

---

## ADDENDUM X: FORM INTENT BINDING (NEW in v1.5)

**Purpose**: Defines how presentation views bind to input intents as forms, enabling user interaction through the UI layer  
**Extends**: `presentationView`  
**Priority**: CRITICAL

---

## Problem Statement

The current specification defines:
- `inputIntent` with `supplies` (required/optional fields)
- `presentationView` with `consumes` (data bindings)
- `interactionModel.actions` with `intent` references

However, there is no grammar for expressing:
1. How a view renders as a form that submits to an intent
2. How view fields map to intent `supplies` requirements
3. What validation constraints apply at the UI layer
4. What happens on successful or failed submission

**Impact**: Users cannot interact with the system through forms in the UI. The demo required manual curl commands to submit intents.

---

## Design Principle Alignment

### Intent-First

This addendum expresses **what** the binding means, not **how** to render it:

| Intent-Level (This Spec) | Runtime Decides |
|--------------------------|-----------------|
| `binding: form` | HTML `<form>`, React component, voice prompt |
| `field_mapping.supplies: amount` | Input field type, placeholder text |
| `validation.constraint: amount > 0` | Inline validation, submit-time check |
| `on_success: navigate_back` | Browser history, mobile nav stack |
| `on_error: display_inline` | Toast, inline errors, modal |

### Transport-Agnostic

The grammar does not specify:
- HTTP endpoints or NATS subjects
- Request/response wire formats
- UI framework or component library

---

## Intent-Level Grammar

### Triggers Block (New)

Add `triggers` block to `presentationView` for intent binding:

```yaml
presentationView:
  name: <string>
  version: <vN>
  consumes:
    dataState: <DataState>
  triggers:                           # NEW: Intent binding
    intent: <InputIntent>             # Which intent this view submits to
    binding: form | action | link     # Semantic binding type
    
    field_mapping:                    # How view fields map to intent supplies
      - view_field: <field_name>      # Field in this view's data
        supplies: <intent_field>       # Field in intent's supplies
        source: context | input | computed  # Where value comes from
        required: true | false        # Is this mapping required
        default: <value>              # Default if not provided
        validation:                   # UI-level validation
          constraint: <expression>    # Constraint expression
          message: <string>           # Human-readable error
    
    confirmation:                     # Optional confirmation before submit
      required: true | false
      message: <string_template>      # Supports {{field}} interpolation
    
    on_success:                       # What happens on successful submission
      action: navigate_back | navigate_to | stay | refresh | close
      target: <composition_ref>       # If navigate_to
      params: { <key>: <expression> } # Parameters for navigation
      notification:                   # Optional success notification
        type: toast | inline | none
        message: <string_template>
    
    on_error:                         # What happens on failed submission
      action: display_inline | display_modal | retry
      show_field_errors: true | false # Map server errors to fields
      notification:
        type: toast | inline | modal
        message: <string_template>
```

### Binding Types

| Binding | Description | Example |
|---------|-------------|---------|
| `form` | Multi-field input form | Transfer form with amount, accounts |
| `action` | Single-click action button | "Acknowledge Alert" button |
| `link` | Navigation that triggers intent | "View Details" with entity_id |

### Field Source Types

| Source | Description | Example |
|--------|-------------|---------|
| `context` | From composition/session context | `session.subject_id` |
| `input` | User provides value | Text input, dropdown |
| `computed` | Derived from other fields | `balance - amount` |

### Success/Error Actions

| Action | Description |
|--------|-------------|
| `navigate_back` | Return to previous composition |
| `navigate_to` | Go to specific composition |
| `stay` | Remain on current view |
| `refresh` | Reload current composition |
| `close` | Close modal/overlay |
| `display_inline` | Show error in form |
| `display_modal` | Show error in modal |
| `retry` | Enable retry with same data |

---

## Semantic Guarantees

When a `presentationView` declares `triggers`:

1. **Field Mapping Completeness**: All `required` fields in the intent's `supplies.required` MUST have a mapping
2. **Validation Alignment**: View validation constraints MUST NOT contradict intent constraints
3. **Action Determinism**: `on_success` and `on_error` MUST specify unambiguous behavior
4. **Context Availability**: Fields sourced from `context` MUST be available in composition context

---

## Runtime Responsibilities

The runtime MUST:

1. **Render Appropriately**: Translate `binding: form` to platform-appropriate input mechanism
2. **Validate Locally**: Evaluate `validation.constraint` before submission
3. **Submit to Intent**: Route to intent via configured transport binding
4. **Handle Response**: Execute `on_success` or `on_error` based on intent result
5. **Map Field Errors**: If `show_field_errors: true`, map server field errors to view fields

The runtime MAY:

1. Choose any transport (HTTP, NATS, WebSocket, etc.)
2. Add loading states during submission
3. Debounce rapid submissions
4. Cache validation results

---

## Examples

### Example 1: Transfer Form

```yaml
presentationView:
  name: TransferFormView
  version: v1
  consumes:
    dataState: AccountState
  
  triggers:
    intent: InitiateTransfer
    binding: form
    
    field_mapping:
      - view_field: source_account
        supplies: from_account_id
        source: context
        required: true
        
      - view_field: target_account
        supplies: to_account_id
        source: input
        required: true
        validation:
          constraint: target_account != source_account
          message: "Cannot transfer to same account"
          
      - view_field: transfer_amount
        supplies: amount
        source: input
        required: true
        validation:
          constraint: transfer_amount > 0 AND transfer_amount <= source_account.balance
          message: "Amount must be between $0.01 and available balance"
          
      - view_field: memo
        supplies: description
        source: input
        required: false
        default: ""
    
    confirmation:
      required: true
      message: "Transfer {{transfer_amount}} from {{source_account.number}} to {{target_account.number}}?"
    
    on_success:
      action: navigate_to
      target: AccountDetailWorkspace
      params:
        account_id: "{{source_account.id}}"
      notification:
        type: toast
        message: "Transfer initiated successfully"
    
    on_error:
      action: display_inline
      show_field_errors: true
      notification:
        type: inline
        message: "{{error.message}}"
```

### Example 2: Acknowledge Alert (Action Binding)

```yaml
presentationView:
  name: AlertItemView
  version: v1
  consumes:
    dataState: AlertState
  
  triggers:
    intent: AcknowledgeAlert
    binding: action
    
    field_mapping:
      - view_field: alert.id
        supplies: alert_id
        source: context
        required: true
    
    on_success:
      action: refresh
      notification:
        type: toast
        message: "Alert acknowledged"
    
    on_error:
      action: display_inline
      notification:
        type: toast
        message: "Failed to acknowledge: {{error.message}}"
```

### Example 3: View Details Link

```yaml
presentationView:
  name: AccountSummaryView
  version: v1
  consumes:
    dataState: AccountState
  
  triggers:
    intent: ViewAccountDetails
    binding: link
    
    field_mapping:
      - view_field: account.id
        supplies: account_id
        source: context
        required: true
    
    on_success:
      action: navigate_to
      target: AccountDetailWorkspace
      params:
        account_id: "{{account.id}}"
```

---

## Validation Criteria

A conformant implementation MUST:

1. **Parse** the `triggers` block without error
2. **Reject** views where required intent supplies lack mappings
3. **Evaluate** validation constraints before submission
4. **Execute** appropriate `on_success` or `on_error` action
5. **Support** all three binding types: `form`, `action`, `link`

A conformant implementation SHOULD:

1. Display confirmation dialog when `confirmation.required: true`
2. Show field-level errors when `show_field_errors: true`
3. Support template interpolation in messages

---

## Integration with Existing Grammar

### Extends

- `presentationView` - Adds `triggers` block

### References

- `inputIntent` - Target of intent binding
- `inputIntent.supplies` - Fields that must be mapped
- `presentationComposition` - Navigation targets

### Supersedes

- `interactionModel.actions.intent` - More expressive replacement for form bindings
- Views MAY use both `interactionModel.actions` (for simple actions) and `triggers` (for forms)

---

## EBNF Grammar Addition

```ebnf
triggers        ::= "triggers:" NEWLINE INDENT
                    "intent:" intent_ref NEWLINE
                    "binding:" binding_type NEWLINE
                    field_mapping?
                    confirmation?
                    on_success
                    on_error
                    DEDENT

binding_type    ::= "form" | "action" | "link"

field_mapping   ::= "field_mapping:" NEWLINE INDENT
                    ( field_map NEWLINE )+
                    DEDENT

field_map       ::= "- view_field:" identifier NEWLINE
                    "supplies:" identifier NEWLINE
                    "source:" source_type NEWLINE
                    "required:" boolean NEWLINE
                    ( "default:" value NEWLINE )?
                    validation?

source_type     ::= "context" | "input" | "computed"

validation      ::= "validation:" NEWLINE INDENT
                    "constraint:" expression NEWLINE
                    "message:" string NEWLINE
                    DEDENT

confirmation    ::= "confirmation:" NEWLINE INDENT
                    "required:" boolean NEWLINE
                    "message:" string_template NEWLINE
                    DEDENT

on_success      ::= "on_success:" NEWLINE INDENT
                    "action:" success_action NEWLINE
                    ( "target:" composition_ref NEWLINE )?
                    ( "params:" params_map NEWLINE )?
                    notification?
                    DEDENT

on_error        ::= "on_error:" NEWLINE INDENT
                    "action:" error_action NEWLINE
                    ( "show_field_errors:" boolean NEWLINE )?
                    notification?
                    DEDENT

success_action  ::= "navigate_back" | "navigate_to" | "stay" | "refresh" | "close"

error_action    ::= "display_inline" | "display_modal" | "retry"

notification    ::= "notification:" NEWLINE INDENT
                    "type:" notification_type NEWLINE
                    "message:" string_template NEWLINE
                    DEDENT

notification_type ::= "toast" | "inline" | "modal" | "none"
```

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial proposal |

---

## ADDENDUM Y: VIEW MATERIALIZATION CONTRACT (NEW in v1.5)

**Purpose**: Defines retrieval contracts, access patterns, and temporal streaming semantics for materialized views  
**Extends**: `materialization`  
**Priority**: HIGH

---

## Problem Statement

The current specification defines:
- `materialization` with `source`, `targetState`, and `freshness`
- Views are implicitly "fetchable" by UI components

However, there is no grammar for expressing:
1. How views are keyed and retrieved (by entity, by query, etc.)
2. What field serves as the entity key
3. What happens when a view is unavailable
4. How list/collection views are bounded
5. Temporal windowing for streaming/time-series data
6. Backpressure and overflow handling for real-time streams

**Impact**: Implementations must invent key formats (`{viewName}.{entityId}`) and storage conventions without specification guidance. Real-time streaming applications lack declarative window semantics.

---

## Design Principle Alignment

### Intent-First

This addendum expresses **what** the retrieval contract is, not **how** to implement it:

| Intent-Level (This Spec) | Runtime Decides |
|--------------------------|-----------------|
| `by_entity: true` | KV bucket, key format, serialization |
| `entity_key: account_id` | Actual key construction |
| `on_unavailable: use_cached` | Cache implementation, TTL |
| `list.max_items: 100` | Pagination strategy |

### Transport-Agnostic

The grammar does not specify:
- NATS KV bucket names
- Key format patterns
- Serialization format (JSON, Protobuf, etc.)
- Storage engine (NATS, Redis, PostgreSQL)

---

## Intent-Level Grammar

### Retrieval Block (New)

Add `retrieval` block to `materialization` for access contracts:

```yaml
materialization:
  name: <string>
  version: <vN>
  source: <DataState | list>
  targetState: <DataState>
  
  retrieval:                          # NEW: Access contract
    mode: by_entity | by_query | singleton  # How views are accessed
    
    # For by_entity mode
    entity_key: <field_name>          # Which field is the entity key
    
    # For by_query mode  
    query_fields: [<field_name>, ...] # Fields used for querying
    
    # For list views
    list:
      max_items: <integer>            # Maximum items per fetch
      default_order: <field_name>     # Default sort field
      order_direction: asc | desc     # Default sort direction
    
    freshness:                        # Already in spec, clarified here
      max_staleness: <duration>       # Maximum acceptable age
      
    on_unavailable:                   # What to do when view is not available
      strategy: use_cached | fail | degrade | wait
      cache_ttl: <duration>           # How long cached data remains valid
      degrade_to: <view_ref>          # Fallback view if degrading
      wait_timeout: <duration>        # How long to wait before failing
      
  temporal:                           # NEW: Time-series/streaming semantics
    window:
      type: tumbling | sliding | session | hopping
      size: <duration>                # Window size
      slide: <duration>               # For sliding/hopping: slide interval
      gap: <duration>                 # For session: inactivity gap
      
    aggregation:                      # Windowed aggregations
      - field: <field_name>
        function: sum | avg | min | max | count | first | last | stddev
        as: <result_field>
        
    timestamp_field: <field_name>     # Event time field
    watermark: <duration>             # Late event tolerance
    
    overflow:                         # Backpressure handling
      strategy: drop_oldest | pause_producer | sample | buffer
      buffer_size: <integer>          # For buffer strategy
      sample_rate: <decimal>          # For sample strategy (0.0-1.0)
      
    emit:
      trigger: on_window_close | on_each_event | periodic
      periodic_interval: <duration>   # For periodic trigger
      include_partial: true | false   # Emit partial windows
```

### Retrieval Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| `by_entity` | One view instance per entity | Account details, user profile |
| `by_query` | Views filtered/searched by fields | Transaction history, search results |
| `singleton` | Single global view instance | Dashboard aggregates, system status |

### Unavailability Strategies

| Strategy | Description | When to Use |
|----------|-------------|-------------|
| `use_cached` | Return stale cached data | Read-heavy, staleness acceptable |
| `fail` | Return error immediately | Consistency critical |
| `degrade` | Return simplified view | Graceful degradation |
| `wait` | Wait for availability | Short transient failures |

---

## Semantic Guarantees

When a `materialization` declares `retrieval`:

1. **Key Uniqueness**: For `by_entity`, `entity_key` MUST uniquely identify view instances
2. **Freshness Bound**: Views returned MUST be within `max_staleness` or strategy MUST be applied
3. **List Bounds**: For list views, `max_items` MUST be respected
4. **Degradation Safety**: `degrade_to` view MUST be a valid, simpler view

When a `materialization` declares `temporal`:

5. **Window Semantics**: Events MUST be windowed according to declared type and size
6. **Aggregation Application**: Declared aggregation functions MUST be computed per window
7. **Watermark Enforcement**: Late events beyond watermark MAY be dropped
8. **Overflow Handling**: Overflow strategy MUST be applied when buffer limits reached
9. **Emit Timing**: Results MUST be emitted per declared trigger

---

## Runtime Responsibilities

The runtime MUST:

1. **Key Construction**: Build storage keys using `entity_key` for entity-scoped views
2. **Freshness Tracking**: Track view age and apply strategy when stale
3. **List Pagination**: Respect `max_items` and provide pagination mechanism
4. **Strategy Execution**: Implement configured `on_unavailable` strategy

The runtime MAY:

1. Choose storage engine (NATS KV, Redis, PostgreSQL, etc.)
2. Define key format conventions
3. Implement additional caching layers
4. Provide query optimization
5. Choose stream processing engine (Flink, Kafka Streams, custom)
6. Implement windowing with any algorithm
7. Optimize aggregation computation

---

## Examples

### Example 1: Entity-Scoped View

```yaml
materialization:
  name: AccountBalanceView
  version: v1
  source: AccountState
  targetState: AccountBalanceMaterialized
  
  retrieval:
    mode: by_entity
    entity_key: account_id
    
    freshness:
      max_staleness: 5s
    
    on_unavailable:
      strategy: use_cached
      cache_ttl: 60s
```

### Example 2: Query-Based List View

```yaml
materialization:
  name: TransactionHistoryView
  version: v1
  source: [TransactionState, AccountState]
  targetState: TransactionHistoryMaterialized
  
  retrieval:
    mode: by_query
    query_fields: [account_id, date_range, transaction_type]
    
    list:
      max_items: 50
      default_order: created_at
      order_direction: desc
    
    freshness:
      max_staleness: 10s
    
    on_unavailable:
      strategy: degrade
      degrade_to: TransactionSummaryView
```

### Example 3: Singleton Aggregate View

```yaml
materialization:
  name: DashboardMetricsView
  version: v1
  source: [AccountState, TransactionState, AlertState]
  targetState: DashboardMetricsMaterialized
  
  retrieval:
    mode: singleton
    
    freshness:
      max_staleness: 30s
    
    on_unavailable:
      strategy: use_cached
      cache_ttl: 300s
```

### Example 4: Strict Consistency View

```yaml
materialization:
  name: AccountLockStatusView
  version: v1
  source: AccountState
  targetState: AccountLockStatusMaterialized
  
  retrieval:
    mode: by_entity
    entity_key: account_id
    
    freshness:
      max_staleness: 0s  # Must be current
    
    on_unavailable:
      strategy: fail  # Do not serve stale data
```

### Example 5: Real-Time Price Ticker (Streaming)

```yaml
materialization:
  name: PriceTickerView
  version: v1
  source: PriceEventState
  targetState: PriceTickerMaterialized
  
  retrieval:
    mode: by_entity
    entity_key: symbol
    
    temporal:
      window:
        type: sliding
        size: 5m
        slide: 10s
        
      aggregation:
        - field: price
          function: avg
          as: avg_price
        - field: price
          function: min
          as: low_price
        - field: price
          function: max
          as: high_price
        - field: volume
          function: sum
          as: total_volume
          
      timestamp_field: event_time
      watermark: 5s
      
      overflow:
        strategy: drop_oldest
        
      emit:
        trigger: periodic
        periodic_interval: 1s
        include_partial: true
        
    freshness:
      max_staleness: 100ms
```

### Example 6: User Session Analytics

```yaml
materialization:
  name: UserSessionMetricsView
  version: v1
  source: UserActivityEventState
  targetState: UserSessionMetricsMaterialized
  
  retrieval:
    mode: by_entity
    entity_key: user_id
    
    temporal:
      window:
        type: session
        gap: 30m                    # 30 min inactivity = new session
        
      aggregation:
        - field: page_view
          function: count
          as: pages_viewed
        - field: event_time
          function: first
          as: session_start
        - field: event_time
          function: last
          as: session_end
          
      timestamp_field: event_time
      
      emit:
        trigger: on_window_close
        include_partial: false
```

### Example 7: IoT Sensor Aggregation

```yaml
materialization:
  name: SensorReadingAggregateView
  version: v1
  source: SensorTelemetryState
  targetState: SensorAggregatedMaterialized
  
  retrieval:
    mode: by_entity
    entity_key: sensor_id
    
    temporal:
      window:
        type: tumbling
        size: 1m
        
      aggregation:
        - field: temperature
          function: avg
          as: avg_temperature
        - field: temperature
          function: stddev
          as: temperature_variance
        - field: reading
          function: count
          as: sample_count
          
      timestamp_field: reading_time
      watermark: 10s
      
      overflow:
        strategy: sample
        sample_rate: 0.1           # Keep 10% if overwhelmed
        
      emit:
        trigger: on_window_close
```

---

## Validation Criteria

A conformant implementation MUST:

1. **Parse** the `retrieval` block without error
2. **Reject** `by_entity` views without `entity_key`
3. **Reject** `by_query` views without `query_fields`
4. **Track** view freshness and compare to `max_staleness`
5. **Execute** configured `on_unavailable` strategy

A conformant implementation SHOULD:

1. Provide efficient key lookup for `by_entity` views
2. Support indexed queries for `query_fields`
3. Implement pagination for list views

---

## Integration with Existing Grammar

### Extends

- `materialization` - Adds `retrieval` block

### References

- `DataState` - Source and target state definitions
- `presentationView.consumes` - Views consume materializations
- `presentationComposition` - Compositions fetch materializations

### Enhances

- `materialization.freshness` - More detailed freshness and unavailability handling

---

## EBNF Grammar Addition

```ebnf
retrieval       ::= "retrieval:" NEWLINE INDENT
                    "mode:" retrieval_mode NEWLINE
                    ( entity_key | query_fields )?
                    list_config?
                    freshness?
                    on_unavailable?
                    DEDENT

retrieval_mode  ::= "by_entity" | "by_query" | "singleton"

entity_key      ::= "entity_key:" identifier NEWLINE

query_fields    ::= "query_fields:" "[" identifier_list "]" NEWLINE

list_config     ::= "list:" NEWLINE INDENT
                    "max_items:" integer NEWLINE
                    ( "default_order:" identifier NEWLINE )?
                    ( "order_direction:" order_dir NEWLINE )?
                    DEDENT

order_dir       ::= "asc" | "desc"

freshness       ::= "freshness:" NEWLINE INDENT
                    "max_staleness:" duration NEWLINE
                    DEDENT

on_unavailable  ::= "on_unavailable:" NEWLINE INDENT
                    "strategy:" unavail_strategy NEWLINE
                    ( "cache_ttl:" duration NEWLINE )?
                    ( "degrade_to:" view_ref NEWLINE )?
                    ( "wait_timeout:" duration NEWLINE )?
                    DEDENT

unavail_strategy ::= "use_cached" | "fail" | "degrade" | "wait"

temporal        ::= "temporal:" NEWLINE INDENT
                    temporal_window
                    temporal_aggregation?
                    ( "timestamp_field:" identifier NEWLINE )?
                    ( "watermark:" duration NEWLINE )?
                    temporal_overflow?
                    temporal_emit?
                    DEDENT

temporal_window ::= "window:" NEWLINE INDENT
                    "type:" window_type NEWLINE
                    "size:" duration NEWLINE
                    ( "slide:" duration NEWLINE )?
                    ( "gap:" duration NEWLINE )?
                    DEDENT

window_type     ::= "tumbling" | "sliding" | "session" | "hopping"

temporal_aggregation ::= "aggregation:" NEWLINE INDENT
                         aggregation_def+
                         DEDENT

aggregation_def ::= "-" "field:" identifier NEWLINE INDENT
                    "function:" agg_function NEWLINE
                    "as:" identifier NEWLINE
                    DEDENT

agg_function    ::= "sum" | "avg" | "min" | "max" | "count" | 
                    "first" | "last" | "stddev"

temporal_overflow ::= "overflow:" NEWLINE INDENT
                      "strategy:" overflow_strategy NEWLINE
                      ( "buffer_size:" integer NEWLINE )?
                      ( "sample_rate:" decimal NEWLINE )?
                      DEDENT

overflow_strategy ::= "drop_oldest" | "pause_producer" | "sample" | "buffer"

temporal_emit   ::= "emit:" NEWLINE INDENT
                    "trigger:" emit_trigger NEWLINE
                    ( "periodic_interval:" duration NEWLINE )?
                    ( "include_partial:" boolean NEWLINE )?
                    DEDENT

emit_trigger    ::= "on_window_close" | "on_each_event" | "periodic"
```

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial proposal |
| Draft | 2026-01 | Added temporal window semantics for streaming |

---

## ADDENDUM Z: INTENT DELIVERY CONTRACT (NEW in v1.5)

**Purpose**: Defines delivery guarantees, acknowledgment modes, timeout semantics, and response schemas for intent-to-worker routing  
**Extends**: `inputIntent`  
**Priority**: HIGH

---

## Problem Statement

The current specification defines:
- `inputIntent` with `supplies` (required/optional fields)
- `inputIntent` with `proposes` (target DataState)
- Workers with `acceptsInputs` (which intents they handle)

However, there is no grammar for expressing:
1. Delivery guarantees (at-least-once, at-most-once, exactly-once)
2. Whether acknowledgment is required
3. Timeout semantics for intent processing
4. Response schema (what success/error responses contain)

**Impact**: Implementations must invent routing conventions (NATS subjects, HTTP endpoints) and response formats without specification guidance.

---

## Design Principle Alignment

### Intent-First

This addendum expresses **what** delivery guarantees are needed, not **how** to achieve them:

| Intent-Level (This Spec) | Runtime Decides |
|--------------------------|-----------------|
| `guarantee: at_least_once` | NATS JetStream, Kafka, or custom queue |
| `acknowledgment: required` | Sync request-reply, async with callback |
| `timeout: 30s` | How to implement timeout (context, deadline) |
| `response.on_success.includes` | Wire format (JSON, Protobuf) |

### Transport-Agnostic

The grammar does not specify:
- NATS subject patterns
- HTTP endpoints or methods
- Message encoding format
- Queue implementation

---

## Intent-Level Grammar

### Delivery Block (New)

Add `delivery` block to `inputIntent` for delivery contracts:

```yaml
inputIntent:
  name: <string>
  version: <vN>
  proposes:
    dataState: <DataState>
  supplies:
    required: [<field>, ...]
    optional: [<field>, ...]
  
  delivery:                           # NEW: Delivery contract
    guarantee: at_least_once | at_most_once | exactly_once
    acknowledgment: required | optional | fire_and_forget
    timeout: <duration>               # How long caller waits for response
    retry:                            # If acknowledgment fails
      allowed: true | false
      max_attempts: <integer>
      backoff: linear | exponential | fixed
  
  response:                           # NEW: Response schema
    on_success:
      includes:                       # Fields returned on success
        - entity_id                   # Created/affected entity
        - <field_name>                # Additional fields
      guarantees:                     # What success means
        - entity_persisted            # Entity is durably stored
        - <guarantee>                 # Additional guarantees
    
    on_error:
      includes:                       # Fields returned on error
        - error_code                  # Machine-readable code
        - message                     # Human-readable message
      field_errors:                   # Per-field validation errors
        enabled: true | false
        format: map | list            # { field: error } or [{ field, error }]
      categories:                     # Error classification
        - validation                  # Input validation failure
        - authorization               # Permission denied
        - conflict                    # Concurrent modification
        - unavailable                 # Service unavailable
```

### Delivery Guarantees

| Guarantee | Description | Use Case |
|-----------|-------------|----------|
| `at_least_once` | May be delivered multiple times, never lost | Most intents (with idempotency) |
| `at_most_once` | Delivered zero or one time, may be lost | Non-critical notifications |
| `exactly_once` | Delivered exactly one time | Financial transactions |

### Acknowledgment Modes

| Mode | Description | Response |
|------|-------------|----------|
| `required` | Caller waits for response | Success or error |
| `optional` | Caller may or may not wait | Best-effort response |
| `fire_and_forget` | Caller does not wait | No response |

### Error Categories

| Category | Description | Retryable |
|----------|-------------|-----------|
| `validation` | Input failed constraints | No |
| `authorization` | Permission denied | No |
| `conflict` | Concurrent modification | Maybe |
| `unavailable` | Service temporarily unavailable | Yes |
| `internal` | Internal processing error | Maybe |

---

## Semantic Guarantees

When an `inputIntent` declares `delivery`:

1. **Delivery Bound**: Intent MUST be delivered according to `guarantee`
2. **Timeout Enforcement**: Response MUST arrive within `timeout` or error
3. **Retry Safety**: If `retry.allowed`, intent MUST be idempotent
4. **Response Schema**: Response MUST include declared `includes` fields

---

## Runtime Responsibilities

The runtime MUST:

1. **Guarantee Delivery**: Implement delivery guarantee via chosen transport
2. **Timeout Handling**: Cancel/fail if response not received within timeout
3. **Response Formatting**: Include all declared `includes` fields
4. **Error Categorization**: Classify errors using declared `categories`

The runtime MAY:

1. Choose transport (NATS, HTTP, gRPC, WebSocket)
2. Implement retry with any compliant backoff strategy
3. Add additional response fields beyond `includes`
4. Implement circuit breakers for unavailable workers

---

## Examples

### Example 1: Transfer Intent (Synchronous, Acknowledged)

```yaml
inputIntent:
  name: InitiateTransfer
  version: v1
  proposes:
    dataState: TransactionState
  supplies:
    required: [from_account_id, to_account_id, amount]
    optional: [description]
  
  delivery:
    guarantee: exactly_once
    acknowledgment: required
    timeout: 30s
    retry:
      allowed: true
      max_attempts: 3
      backoff: exponential
  
  response:
    on_success:
      includes:
        - entity_id
        - confirmation_number
        - estimated_completion
      guarantees:
        - entity_persisted
        - funds_reserved
    
    on_error:
      includes:
        - error_code
        - message
      field_errors:
        enabled: true
        format: map
      categories:
        - validation
        - authorization
        - conflict
        - unavailable
```

### Example 2: Acknowledge Alert (Simple Action)

```yaml
inputIntent:
  name: AcknowledgeAlert
  version: v1
  proposes:
    dataState: AlertState
  supplies:
    required: [alert_id]
    optional: []
  
  delivery:
    guarantee: at_least_once
    acknowledgment: required
    timeout: 10s
    retry:
      allowed: true
      max_attempts: 2
      backoff: fixed
  
  response:
    on_success:
      includes:
        - entity_id
      guarantees:
        - entity_updated
    
    on_error:
      includes:
        - error_code
        - message
      field_errors:
        enabled: false
      categories:
        - validation
        - authorization
```

### Example 3: Analytics Event (Fire-and-Forget)

```yaml
inputIntent:
  name: TrackUserAction
  version: v1
  proposes:
    dataState: AnalyticsEvent
  supplies:
    required: [action_type, timestamp]
    optional: [metadata]
  
  delivery:
    guarantee: at_most_once
    acknowledgment: fire_and_forget
    timeout: 0s  # Not applicable
    retry:
      allowed: false
  
  response:
    on_success:
      includes: []
      guarantees: []
    on_error:
      includes: []
      field_errors:
        enabled: false
      categories: []
```

---

## Validation Criteria

A conformant implementation MUST:

1. **Parse** the `delivery` and `response` blocks without error
2. **Enforce** timeout semantics for acknowledged intents
3. **Include** all declared `response.includes` fields
4. **Categorize** errors using declared categories
5. **Respect** retry configuration for acknowledged intents

A conformant implementation SHOULD:

1. Implement idempotency for `at_least_once` intents
2. Provide field-level errors when `field_errors.enabled`
3. Support exponential backoff for retries

---

## Integration with Existing Grammar

### Extends

- `inputIntent` - Adds `delivery` and `response` blocks

### References

- `wts.workers.acceptsInputs` - Workers that handle this intent
- `sms.flows.triggeredBy` - Flows triggered by intent
- `idempotency` - Required for `at_least_once` with retry

### Enhances

- `inputIntent.supplies` - Response includes complement supplies

---

## EBNF Grammar Addition

```ebnf
delivery        ::= "delivery:" NEWLINE INDENT
                    "guarantee:" delivery_guarantee NEWLINE
                    "acknowledgment:" ack_mode NEWLINE
                    "timeout:" duration NEWLINE
                    retry_config?
                    DEDENT

delivery_guarantee ::= "at_least_once" | "at_most_once" | "exactly_once"

ack_mode        ::= "required" | "optional" | "fire_and_forget"

retry_config    ::= "retry:" NEWLINE INDENT
                    "allowed:" boolean NEWLINE
                    ( "max_attempts:" integer NEWLINE )?
                    ( "backoff:" backoff_strategy NEWLINE )?
                    DEDENT

backoff_strategy ::= "linear" | "exponential" | "fixed"

response        ::= "response:" NEWLINE INDENT
                    on_success_response
                    on_error_response
                    DEDENT

on_success_response ::= "on_success:" NEWLINE INDENT
                        "includes:" "[" identifier_list "]" NEWLINE
                        ( "guarantees:" "[" guarantee_list "]" NEWLINE )?
                        DEDENT

on_error_response ::= "on_error:" NEWLINE INDENT
                      "includes:" "[" identifier_list "]" NEWLINE
                      field_errors_config?
                      ( "categories:" "[" error_category_list "]" NEWLINE )?
                      DEDENT

field_errors_config ::= "field_errors:" NEWLINE INDENT
                        "enabled:" boolean NEWLINE
                        ( "format:" field_error_format NEWLINE )?
                        DEDENT

field_error_format ::= "map" | "list"

error_category_list ::= error_category ( "," error_category )*

error_category  ::= "validation" | "authorization" | "conflict" 
                  | "unavailable" | "internal"
```

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial proposal |

---

## ADDENDUM AA: SESSION CONTRACT (NEW in v1.5)

**Purpose**: Defines session management binding, lifetime semantics, multi-device sync, and offline operation for experiences  
**Extends**: `experience`  
**Priority**: MEDIUM

---

## Problem Statement

The current specification defines:
- `experience` with `policy_set` for authorization
- `experience` with `unauthenticated.redirect_to` for auth redirects
- Subject, role, and attribute primitives for RBAC/ABAC

However, there is no grammar for expressing:
1. Whether an experience requires a session
2. What the session must contain (subject_id, roles, realm, etc.)
3. Session lifetime semantics (bounded, sliding, etc.)
4. What happens when session expires or is invalid
5. Multi-device support (phone, tablet, desktop per subject)
6. Offline operation and sync semantics
7. Cross-device handoff and continuity

**Impact**: Implementations must hardcode session storage (NATS KV), bucket names, and TTL values without specification guidance. Multi-device applications lack declarative sync semantics.

---

## Design Principle Alignment

### Intent-First

This addendum expresses **what** session semantics are needed, not **how** to implement them:

| Intent-Level (This Spec) | Runtime Decides |
|--------------------------|-----------------|
| `required: true` | Cookie, JWT, session ID |
| `contains: [subject_id, roles]` | Storage format, serialization |
| `lifetime: bounded` | Actual TTL, refresh mechanism |
| `on_expired: redirect_to_auth` | Login URL, OAuth flow |

### Transport-Agnostic

The grammar does not specify:
- Session storage mechanism (KV, cookies, JWT, database)
- Cookie names or JWT claims
- OAuth/OIDC configuration
- Session ID generation

---

## Intent-Level Grammar

### Session Block (New)

Add `session` block to `experience` for session contracts:

```yaml
experience:
  name: <string>
  version: <vN>
  entry_point: <composition_ref>
  includes: [<composition_ref>, ...]
  
  session:                            # NEW: Session contract
    required: true | false            # Must have valid session
    
    subject_binding:                  # How subject is bound
      mode: authenticated | anonymous | either
      
    contains:                         # What session must include
      required:                       # Required fields
        - subject_id                  # Unique subject identifier
        - roles                       # Subject's roles
      optional:                       # Optional fields
        - realm                       # Multi-tenancy realm
        - attributes                  # Additional attributes
        - preferences                 # User preferences
    
    lifetime:                         # Session duration semantics
      mode: bounded | sliding | permanent
      idle_timeout: <duration>        # For sliding: timeout after inactivity
      max_duration: <duration>        # For bounded: absolute maximum
    
    on_expired:                       # What happens when session expires
      action: redirect_to_auth | prompt_reauth | degrade_to_anonymous
      preserve_location: true | false # Remember where user was
    
    on_invalid:                       # What happens when session is invalid
      action: redirect_to_auth | clear_and_retry | error
      
  devices:                            # NEW: Multi-device semantics
    enabled: true | false
    max_active: <integer>             # Max concurrent devices
    
    identification:
      by: device_id | fingerprint | user_agent
      
    state_scope:                      # What state is per-device vs shared
      per_device:                     # Device-local only
        - <field_path>
      shared:                         # Synced across devices
        - <field_path>
        
    handoff:                          # Cross-device continuity
      enabled: true | false
      state_transfer: immediate | on_request | manual
      transferable: [<field_path>, ...]
      
    concurrent_limits:                # Per-device behavior
      on_new_device: allow | logout_oldest | prompt | reject
      
  offline:                            # NEW: Offline operation semantics
    enabled: true | false
    
    capabilities:                     # What works offline
      read: true | false
      write: queue | reject
      
    sync:
      on_reconnect: immediate | background | manual
      conflict_resolution: client_wins | server_wins | merge | prompt
      
      field_strategies:               # Per-field conflict handling
        - field: <field_path>
          strategy: client_wins | server_wins | merge | union | max | min
          
    queue:
      max_size: <integer>
      max_age: <duration>
      on_overflow: drop_oldest | reject_new
      
    cache:
      views: [<view_ref>, ...]        # Views to cache for offline
      max_size: <bytes>
      ttl: <duration>
```

### Subject Binding Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| `authenticated` | Must have verified subject | Protected experiences |
| `anonymous` | Must NOT have subject | Public experiences |
| `either` | Subject optional | Mixed experiences |

### Lifetime Modes

| Mode | Description | Behavior |
|------|-------------|----------|
| `bounded` | Fixed duration | Expires at `max_duration` regardless of activity |
| `sliding` | Activity-based | Expires after `idle_timeout` of inactivity |
| `permanent` | No expiration | Never expires (requires explicit logout) |

### Expiration Actions

| Action | Description |
|--------|-------------|
| `redirect_to_auth` | Send user to authentication flow |
| `prompt_reauth` | Show inline reauthentication prompt |
| `degrade_to_anonymous` | Continue as anonymous user |

---

## Semantic Guarantees

When an `experience` declares `session`:

1. **Binding Enforcement**: Subject binding mode MUST be enforced
2. **Content Availability**: All `required` fields MUST be present in session context
3. **Lifetime Enforcement**: Session MUST expire according to `lifetime` mode
4. **Action Execution**: Expired/invalid sessions MUST trigger declared action

When `session.devices` is declared:

5. **Device Limit**: `max_active` concurrent devices MUST be enforced
6. **State Scoping**: `per_device` state MUST NOT sync across devices
7. **Shared Sync**: `shared` state MUST sync across devices
8. **Handoff**: State transfer MUST occur per declared policy

When `session.offline` is declared:

9. **Queue Persistence**: Offline writes MUST be queued per configuration
10. **Sync Execution**: Reconnect sync MUST follow declared strategy
11. **Conflict Resolution**: Conflicts MUST be resolved per field strategies
12. **Cache Bounds**: Cached data MUST respect size and TTL limits

---

## Runtime Responsibilities

The runtime MUST:

1. **Validate Session**: Check session validity before serving experience
2. **Enforce Binding**: Reject requests that don't match `subject_binding` mode
3. **Populate Context**: Make `contains` fields available to compositions
4. **Track Lifetime**: Implement chosen lifetime mode
5. **Execute Actions**: Perform `on_expired` and `on_invalid` actions

The runtime MAY:

1. Choose session storage (KV, cookies, JWT, database)
2. Implement session refresh/renewal mechanisms
3. Add additional session fields beyond `contains`
4. Implement session revocation
5. Choose sync mechanism (WebSocket, polling, background sync)
6. Implement any CRDT or merge algorithm
7. Choose offline storage (IndexedDB, SQLite, etc.)
8. Implement device fingerprinting

---

## Examples

### Example 1: Customer Banking Experience

```yaml
experience:
  name: CustomerBanking
  version: v1
  entry_point: CustomerDashboard
  includes:
    - CustomerDashboard
    - AccountListWorkspace
    - AccountDetailWorkspace
    - TransferWorkspace
  
  session:
    required: true
    
    subject_binding:
      mode: authenticated
    
    contains:
      required:
        - subject_id
        - roles
        - realm
      optional:
        - attributes
        - preferences
    
    lifetime:
      mode: sliding
      idle_timeout: 30m
      max_duration: 24h
    
    on_expired:
      action: redirect_to_auth
      preserve_location: true
    
    on_invalid:
      action: redirect_to_auth
```

### Example 2: Public Marketing Experience

```yaml
experience:
  name: PublicMarketing
  version: v1
  entry_point: MarketingHome
  includes:
    - MarketingHome
    - ProductInfo
    - ContactForm
  
  session:
    required: false
    
    subject_binding:
      mode: either  # Works for both anonymous and authenticated
    
    contains:
      required: []  # No required fields
      optional:
        - subject_id  # If authenticated
        - preferences
    
    lifetime:
      mode: permanent  # Anonymous sessions don't expire
    
    on_expired:
      action: degrade_to_anonymous
      preserve_location: true
    
    on_invalid:
      action: clear_and_retry
```

### Example 3: Admin Console Experience

```yaml
experience:
  name: AdminConsole
  version: v1
  entry_point: AdminDashboard
  includes:
    - AdminDashboard
    - UserManagement
    - SystemSettings
    - AuditLogs
  
  session:
    required: true
    
    subject_binding:
      mode: authenticated
    
    contains:
      required:
        - subject_id
        - roles
        - realm
        - attributes
      optional:
        - mfa_verified  # Multi-factor auth status
        - ip_address    # For audit
    
    lifetime:
      mode: bounded
      idle_timeout: 15m  # Shorter for security
      max_duration: 8h   # Work day maximum
    
    on_expired:
      action: prompt_reauth
      preserve_location: true
    
    on_invalid:
      action: redirect_to_auth
```

### Example 4: Multi-Device Mobile App

```yaml
experience:
  name: MobileShoppingApp
  version: v1
  entry_point: HomeScreen
  includes:
    - HomeScreen
    - ProductBrowse
    - ShoppingCart
    - Checkout
    - OrderHistory
  
  session:
    required: true
    
    subject_binding:
      mode: authenticated
    
    contains:
      required:
        - subject_id
        - roles
      optional:
        - preferences
        - cart_id
    
    lifetime:
      mode: sliding
      idle_timeout: 30d
      max_duration: 90d
    
    on_expired:
      action: degrade_to_anonymous
      preserve_location: true
    
    devices:
      enabled: true
      max_active: 5
      
      identification:
        by: device_id
        
      state_scope:
        per_device:
          - ui.theme
          - ui.last_viewed_product
          - ui.scroll_positions
        shared:
          - cart.items
          - favorites
          - recently_viewed
          
      handoff:
        enabled: true
        state_transfer: immediate
        transferable:
          - cart.items
          - checkout.step
          
      concurrent_limits:
        on_new_device: allow
        
    offline:
      enabled: true
      
      capabilities:
        read: true
        write: queue
        
      sync:
        on_reconnect: immediate
        conflict_resolution: merge
        
        field_strategies:
          - field: cart.items
            strategy: union
          - field: favorites
            strategy: union
          - field: preferences.notifications
            strategy: client_wins
            
      queue:
        max_size: 100
        max_age: 7d
        on_overflow: drop_oldest
        
      cache:
        views: [ProductCatalogView, CategoryView]
        max_size: 50MB
        ttl: 24h
```

### Example 5: Cross-Platform Note-Taking App

```yaml
experience:
  name: NotesApp
  version: v1
  entry_point: NotesList
  
  session:
    required: true
    
    subject_binding:
      mode: authenticated
    
    lifetime:
      mode: permanent
    
    devices:
      enabled: true
      max_active: 10
      
      identification:
        by: device_id
        
      state_scope:
        per_device:
          - ui.sidebar_collapsed
          - ui.editor_settings
          - ui.recent_notes
        shared:
          - notes
          - folders
          - tags
          
      handoff:
        enabled: true
        state_transfer: immediate
        transferable:
          - current_note_id
          - cursor_position
          
      concurrent_limits:
        on_new_device: allow
        
    offline:
      enabled: true
      
      capabilities:
        read: true
        write: queue
        
      sync:
        on_reconnect: background
        conflict_resolution: merge
        
        field_strategies:
          - field: note.content
            strategy: merge              # Use text merge
          - field: note.title
            strategy: server_wins
          - field: tags
            strategy: union
            
      queue:
        max_size: 500
        max_age: 30d
        on_overflow: reject_new
        
      cache:
        views: [NotesListView, NoteDetailView, FolderView]
        max_size: 100MB
        ttl: 168h                        # 7 days
```

---

## Validation Criteria

A conformant implementation MUST:

1. **Parse** the `session` block without error
2. **Enforce** `subject_binding` mode for all experience requests
3. **Provide** all `required` session fields to composition context
4. **Implement** chosen `lifetime` mode
5. **Execute** `on_expired` and `on_invalid` actions

A conformant implementation SHOULD:

1. Support all three lifetime modes
2. Preserve user location when `preserve_location: true`
3. Provide session refresh mechanism for `sliding` mode

---

## Integration with Existing Grammar

### Extends

- `experience` - Adds `session` block

### References

- `subject` - Subject bound to session
- `role` - Roles stored in session
- `realm` - Realm for multi-tenancy
- `policy` - Policies evaluated using session context

### Enhances

- `experience.unauthenticated` - More detailed authentication handling
- `experience.policy_set` - Session provides evaluation context

---

## EBNF Grammar Addition

```ebnf
session         ::= "session:" NEWLINE INDENT
                    "required:" boolean NEWLINE
                    subject_binding
                    session_contains
                    session_lifetime
                    on_expired
                    on_invalid
                    DEDENT

subject_binding ::= "subject_binding:" NEWLINE INDENT
                    "mode:" binding_mode NEWLINE
                    DEDENT

binding_mode    ::= "authenticated" | "anonymous" | "either"

session_contains ::= "contains:" NEWLINE INDENT
                     "required:" "[" identifier_list "]" NEWLINE
                     ( "optional:" "[" identifier_list "]" NEWLINE )?
                     DEDENT

session_lifetime ::= "lifetime:" NEWLINE INDENT
                     "mode:" lifetime_mode NEWLINE
                     ( "idle_timeout:" duration NEWLINE )?
                     ( "max_duration:" duration NEWLINE )?
                     DEDENT

lifetime_mode   ::= "bounded" | "sliding" | "permanent"

on_expired      ::= "on_expired:" NEWLINE INDENT
                    "action:" expired_action NEWLINE
                    ( "preserve_location:" boolean NEWLINE )?
                    DEDENT

expired_action  ::= "redirect_to_auth" | "prompt_reauth" | "degrade_to_anonymous"

on_invalid      ::= "on_invalid:" NEWLINE INDENT
                    "action:" invalid_action NEWLINE
                    DEDENT

invalid_action  ::= "redirect_to_auth" | "clear_and_retry" | "error"

devices         ::= "devices:" NEWLINE INDENT
                    "enabled:" boolean NEWLINE
                    ( "max_active:" integer NEWLINE )?
                    device_identification?
                    device_state_scope?
                    device_handoff?
                    device_concurrent_limits?
                    DEDENT

device_identification ::= "identification:" NEWLINE INDENT
                          "by:" identification_method NEWLINE
                          DEDENT

identification_method ::= "device_id" | "fingerprint" | "user_agent"

device_state_scope ::= "state_scope:" NEWLINE INDENT
                       ( "per_device:" "[" field_path_list "]" NEWLINE )?
                       ( "shared:" "[" field_path_list "]" NEWLINE )?
                       DEDENT

device_handoff     ::= "handoff:" NEWLINE INDENT
                       "enabled:" boolean NEWLINE
                       ( "state_transfer:" transfer_mode NEWLINE )?
                       ( "transferable:" "[" field_path_list "]" NEWLINE )?
                       DEDENT

transfer_mode      ::= "immediate" | "on_request" | "manual"

device_concurrent_limits ::= "concurrent_limits:" NEWLINE INDENT
                             "on_new_device:" new_device_action NEWLINE
                             DEDENT

new_device_action  ::= "allow" | "logout_oldest" | "prompt" | "reject"

offline            ::= "offline:" NEWLINE INDENT
                       "enabled:" boolean NEWLINE
                       offline_capabilities?
                       offline_sync?
                       offline_queue?
                       offline_cache?
                       DEDENT

offline_capabilities ::= "capabilities:" NEWLINE INDENT
                         ( "read:" boolean NEWLINE )?
                         ( "write:" write_capability NEWLINE )?
                         DEDENT

write_capability   ::= "queue" | "reject"

offline_sync       ::= "sync:" NEWLINE INDENT
                       ( "on_reconnect:" reconnect_strategy NEWLINE )?
                       ( "conflict_resolution:" conflict_strategy NEWLINE )?
                       field_strategies?
                       DEDENT

reconnect_strategy ::= "immediate" | "background" | "manual"

conflict_strategy  ::= "client_wins" | "server_wins" | "merge" | "prompt"

field_strategies   ::= "field_strategies:" NEWLINE INDENT
                       field_strategy_def+
                       DEDENT

field_strategy_def ::= "-" "field:" field_path NEWLINE INDENT
                       "strategy:" field_conflict_strategy NEWLINE
                       DEDENT

field_conflict_strategy ::= "client_wins" | "server_wins" | "merge" | 
                            "union" | "max" | "min"

offline_queue      ::= "queue:" NEWLINE INDENT
                       ( "max_size:" integer NEWLINE )?
                       ( "max_age:" duration NEWLINE )?
                       ( "on_overflow:" overflow_action NEWLINE )?
                       DEDENT

overflow_action    ::= "drop_oldest" | "reject_new"

offline_cache      ::= "cache:" NEWLINE INDENT
                       ( "views:" "[" view_ref_list "]" NEWLINE )?
                       ( "max_size:" bytes NEWLINE )?
                       ( "ttl:" duration NEWLINE )?
                       DEDENT
```

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial proposal |
| Draft | 2026-01 | Added multi-device and offline semantics |

---

## ADDENDUM AB: INDICATOR SOURCE BINDING (NEW in v1.5)

**Purpose**: Defines how navigation indicators bind to data sources and update dynamically  
**Extends**: `indicator` (within navigation)  
**Priority**: MEDIUM

---

## Problem Statement

The current specification defines:
- `navigation.indicator` with `when` expression and `severity`
- Indicators can show dynamic state on navigation items

However, there is no grammar for expressing:
1. Where indicator data comes from (which view, which fields)
2. How the indicator is scoped (current user, global, entity-specific)
3. When the indicator value updates (on view change, interval, manual)
4. How to throttle/debounce updates

**Impact**: Implementations must hardcode indicator data fetching, scoping, and update strategies.

---

## Design Principle Alignment

### Intent-First

This addendum expresses **what** data the indicator needs, not **how** to fetch it:

| Intent-Level (This Spec) | Runtime Decides |
|--------------------------|-----------------|
| `source.view: AlertState` | NATS subject, query method |
| `scope: session.subject_id` | How to filter data |
| `update.trigger: view_change` | Subscription mechanism |
| `update.throttle: 500ms` | Debounce implementation |

### Transport-Agnostic

The grammar does not specify:
- NATS subjects for signals
- Polling intervals vs WebSocket push
- Cache invalidation mechanism
- Signal stream configuration

---

## Intent-Level Grammar

### Source and Update Blocks (New)

Extend `indicator` with source binding and update semantics:

```yaml
navigation:
  indicator:
    name: <string>                    # Identifier for the indicator
    type: count | boolean | value     # What kind of indicator
    
    source:                           # NEW: Data source binding
      view: <view_ref>                # Which materialized view
      field: <field_name>             # Which field to evaluate (for value type)
      scope:                          # How to scope the query
        expression: <expression>      # e.g., session.subject_id
        type: user | entity | global  # Scope category
      filter: <expression>            # Filter expression on view data
      aggregation: count | sum | min | max | avg  # For aggregate indicators
    
    when: <expression>                # Existing: when to show indicator
    severity: info | warning | error | critical  # Existing
    
    update:                           # NEW: Update semantics
      trigger: view_change | interval | manual | event
      interval: <duration>            # For interval trigger
      throttle: <duration>            # Minimum time between updates
      debounce: <duration>            # Wait for activity to stop
      
    display:                          # NEW: Display hints
      format: number | badge | dot    # How to show the value
      max_value: <integer>            # Show "99+" for values above this
      hide_when_zero: true | false    # Hide indicator when value is 0
```

### Indicator Types

| Type | Description | Example |
|------|-------------|---------|
| `count` | Numeric count of items | Unread messages: 5 |
| `boolean` | True/false state | Has alerts: red dot |
| `value` | Specific field value | Balance: $1,234 |

### Scope Types

| Type | Description | Filter Applied |
|------|-------------|----------------|
| `user` | Scoped to current user | `entity.user_id == session.subject_id` |
| `entity` | Scoped to current entity | `entity.id == context.entity_id` |
| `global` | System-wide, no scope | No filter |

### Update Triggers

| Trigger | Description | Use Case |
|---------|-------------|----------|
| `view_change` | When source view updates | Real-time indicators |
| `interval` | Periodic refresh | Less critical data |
| `manual` | Only on explicit request | User-triggered refresh |
| `event` | On specific event | External system events |

---

## Semantic Guarantees

When an `indicator` declares `source`:

1. **Source Binding**: Indicator MUST read from declared view
2. **Scope Application**: Filter MUST apply scope expression
3. **Update Timing**: Updates MUST respect trigger and throttle settings
4. **Type Consistency**: Aggregation MUST produce type-appropriate value

---

## Runtime Responsibilities

The runtime MUST:

1. **Bind Source**: Connect indicator to declared view
2. **Apply Scope**: Filter data using scope expression
3. **Trigger Updates**: Implement declared trigger mechanism
4. **Throttle Updates**: Respect throttle/debounce settings
5. **Evaluate Expression**: Evaluate `when` expression for visibility

The runtime MAY:

1. Choose subscription mechanism (WebSocket, polling, NATS)
2. Implement caching for indicator values
3. Batch indicator updates for efficiency
4. Provide optimistic updates

---

## Examples

### Example 1: Unread Alerts Counter

```yaml
navigation:
  label: "Alerts"
  indicator:
    name: unread_alerts
    type: count
    
    source:
      view: AlertState
      scope:
        expression: session.subject_id
        type: user
      filter: acknowledged == false
      aggregation: count
    
    when: unread_alerts > 0
    severity: warning
    
    update:
      trigger: view_change
      throttle: 500ms
    
    display:
      format: number
      max_value: 99
      hide_when_zero: true
```

### Example 2: Pending Approvals Badge

```yaml
navigation:
  label: "Approvals"
  indicator:
    name: pending_approvals
    type: count
    
    source:
      view: ApprovalRequestState
      scope:
        expression: session.roles contains "approver"
        type: user
      filter: status == "pending"
      aggregation: count
    
    when: pending_approvals > 0
    severity: info
    
    update:
      trigger: view_change
      throttle: 1s
    
    display:
      format: badge
      max_value: 50
      hide_when_zero: true
```

### Example 3: Account Balance Indicator

```yaml
navigation:
  label: "Account {{account.number}}"
  indicator:
    name: low_balance
    type: boolean
    
    source:
      view: AccountBalanceView
      field: balance
      scope:
        expression: context.account_id
        type: entity
      filter: balance < 100  # Low balance threshold
    
    when: low_balance == true
    severity: warning
    
    update:
      trigger: view_change
      debounce: 2s
    
    display:
      format: dot
      hide_when_zero: false
```

### Example 4: System Health Indicator

```yaml
navigation:
  label: "System Status"
  indicator:
    name: system_errors
    type: count
    
    source:
      view: SystemHealthView
      scope:
        type: global
      filter: severity == "error" AND timestamp > now() - 1h
      aggregation: count
    
    when: system_errors > 0
    severity: critical
    
    update:
      trigger: interval
      interval: 30s
      throttle: 10s
    
    display:
      format: number
      max_value: 999
      hide_when_zero: false
```

---

## Validation Criteria

A conformant implementation MUST:

1. **Parse** the `source` and `update` blocks without error
2. **Bind** indicator to declared view
3. **Apply** scope expression as filter
4. **Evaluate** `when` expression for visibility
5. **Respect** throttle/debounce settings

A conformant implementation SHOULD:

1. Support all trigger types
2. Implement efficient view subscriptions
3. Batch updates when multiple indicators change

---

## Integration with Existing Grammar

### Extends

- `navigation.indicator` - Adds `source`, `update`, and `display` blocks

### References

- `materialization` - Source views are materializations
- `expression` - Filter and scope use expression grammar
- `session` - User scope references session context

### Enhances

- `navigation.indicator.when` - Source binding makes expressions more powerful
- `navigation.indicator.severity` - Display hints complement severity

---

## EBNF Grammar Addition

```ebnf
indicator       ::= "indicator:" NEWLINE INDENT
                    "name:" identifier NEWLINE
                    "type:" indicator_type NEWLINE
                    indicator_source
                    "when:" expression NEWLINE
                    "severity:" severity_level NEWLINE
                    indicator_update
                    indicator_display?
                    DEDENT

indicator_type  ::= "count" | "boolean" | "value"

indicator_source ::= "source:" NEWLINE INDENT
                     "view:" view_ref NEWLINE
                     ( "field:" identifier NEWLINE )?
                     indicator_scope
                     ( "filter:" expression NEWLINE )?
                     ( "aggregation:" aggregation_type NEWLINE )?
                     DEDENT

indicator_scope ::= "scope:" NEWLINE INDENT
                    ( "expression:" expression NEWLINE )?
                    "type:" scope_type NEWLINE
                    DEDENT

scope_type      ::= "user" | "entity" | "global"

aggregation_type ::= "count" | "sum" | "min" | "max" | "avg"

indicator_update ::= "update:" NEWLINE INDENT
                     "trigger:" update_trigger NEWLINE
                     ( "interval:" duration NEWLINE )?
                     ( "throttle:" duration NEWLINE )?
                     ( "debounce:" duration NEWLINE )?
                     DEDENT

update_trigger  ::= "view_change" | "interval" | "manual" | "event"

indicator_display ::= "display:" NEWLINE INDENT
                      "format:" display_format NEWLINE
                      ( "max_value:" integer NEWLINE )?
                      ( "hide_when_zero:" boolean NEWLINE )?
                      DEDENT

display_format  ::= "number" | "badge" | "dot"
```

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial proposal |

---

## ADDENDUM AC: COMPOSITION FETCH SEMANTICS (NEW in v1.5)

**Purpose**: Defines fetch ordering, parallel/sequential strategies, failure handling, and caching behavior for composition data dependencies  
**Extends**: `presentationComposition`  
**Priority**: HIGH

---

## Problem Statement

The current specification defines:
- `presentationComposition` with `primary` view reference
- `presentationComposition` with `related` view list
- Views have `consumes.dataState` binding

However, there is no grammar for expressing:
1. Fetch ordering (primary before related, all parallel, etc.)
2. Parallel vs sequential fetch for related views
3. What happens when primary or related views fail to load
4. Caching and refresh behavior

**Impact**: Implementations must make arbitrary decisions about fetch ordering and error handling without specification guidance.

---

## Design Principle Alignment

### Intent-First

This addendum expresses **what** the fetch behavior should be, not **how** to implement it:

| Intent-Level (This Spec) | Runtime Decides |
|--------------------------|-----------------|
| `order: primary_first` | HTTP calls, NATS fetches |
| `related.strategy: parallel` | goroutines, Promise.all |
| `on_partial_failure.primary: fail` | Error handling logic |
| `cache.strategy: stale_while_revalidate` | Cache implementation |

### Transport-Agnostic

The grammar does not specify:
- HTTP endpoints or NATS subjects
- Concurrency primitives
- Cache storage mechanism
- Network timeout handling

---

## Intent-Level Grammar

### Fetch Block (New)

Add `fetch` block to `presentationComposition` for fetch semantics:

```yaml
presentationComposition:
  name: <string>
  version: <vN>
  primary: <view_ref>
  related:
    - view: <view_ref>
      # ... existing related view properties
  
  fetch:                              # NEW: Fetch semantics
    order: primary_first | parallel_all | sequential
    
    primary:                          # Primary view fetch config
      required: true                  # If false, composition can render without
      timeout: <duration>             # How long to wait
    
    related:                          # Related views fetch config
      strategy: parallel | sequential | lazy
      timeout: <duration>             # Per-view timeout
      max_concurrent: <integer>       # For parallel: limit concurrency
    
    on_partial_failure:               # What to do when some views fail
      primary: fail_composition | degrade
      related: degrade_gracefully | omit | fail_composition
    
    retry:                            # Retry configuration
      enabled: true | false
      max_attempts: <integer>
      scope: failed_only | all        # Retry just failed or all views
    
    cache:                            # Caching behavior
      strategy: none | cache_first | stale_while_revalidate | network_first
      ttl: <duration>                 # How long cached data is valid
      scope: composition | view       # Cache at composition or view level
```

### Fetch Orders

| Order | Description | When to Use |
|-------|-------------|-------------|
| `primary_first` | Fetch primary, then related | When related depends on primary |
| `parallel_all` | Fetch all views simultaneously | Maximum parallelism |
| `sequential` | Fetch one at a time in order | When order matters or rate-limited |

### Related View Strategies

| Strategy | Description | When to Use |
|----------|-------------|-------------|
| `parallel` | Fetch all related views at once | Performance-critical |
| `sequential` | Fetch related views in order | When order matters |
| `lazy` | Fetch only when view becomes visible | Large compositions |

### Failure Handling

| Action | Description |
|--------|-------------|
| `fail_composition` | Entire composition fails |
| `degrade_gracefully` | Show what loaded, placeholder for failed |
| `degrade` | Show degraded version of failed view |
| `omit` | Simply don't show the failed view |

### Cache Strategies

| Strategy | Description |
|----------|-------------|
| `none` | Never cache, always fetch |
| `cache_first` | Use cache if available, fallback to network |
| `stale_while_revalidate` | Return cache immediately, refresh in background |
| `network_first` | Try network first, fallback to cache |

---

## Semantic Guarantees

When a `presentationComposition` declares `fetch`:

1. **Order Enforcement**: Fetch MUST respect declared order
2. **Timeout Enforcement**: Fetches MUST timeout as configured
3. **Failure Handling**: Partial failures MUST be handled as declared
4. **Cache Behavior**: Caching MUST follow declared strategy

---

## Runtime Responsibilities

The runtime MUST:

1. **Implement Order**: Fetch views in declared order
2. **Respect Timeouts**: Cancel/fail fetches that exceed timeout
3. **Handle Failures**: Execute declared failure strategy
4. **Manage Cache**: Implement declared cache strategy

The runtime MAY:

1. Choose fetch mechanism (HTTP, NATS, in-memory)
2. Optimize parallel fetches beyond `max_concurrent`
3. Pre-warm caches based on navigation patterns
4. Implement smarter retry backoff

---

## Examples

### Example 1: Account Detail with Parallel Related Views

```yaml
presentationComposition:
  name: AccountDetailWorkspace
  version: v1
  primary: AccountDetailsView
  related:
    - view: TransactionHistoryView
      relationship: detail
      cardinality: many
    - view: AccountAlertsView
      relationship: contextual
      cardinality: many
    - view: TransferFormView
      relationship: action
      trigger: InitiateTransfer
  
  fetch:
    order: primary_first
    
    primary:
      required: true
      timeout: 5s
    
    related:
      strategy: parallel
      timeout: 3s
      max_concurrent: 3
    
    on_partial_failure:
      primary: fail_composition
      related: degrade_gracefully
    
    retry:
      enabled: true
      max_attempts: 2
      scope: failed_only
    
    cache:
      strategy: stale_while_revalidate
      ttl: 30s
      scope: view
```

### Example 2: Dashboard with All Parallel

```yaml
presentationComposition:
  name: CustomerDashboard
  version: v1
  primary: DashboardSummaryView
  related:
    - view: RecentTransactionsView
      relationship: detail
    - view: AccountOverviewView
      relationship: derived
    - view: AlertsSummaryView
      relationship: contextual
    - view: QuickActionsView
      relationship: action
  
  fetch:
    order: parallel_all
    
    primary:
      required: true
      timeout: 3s
    
    related:
      strategy: parallel
      timeout: 3s
      max_concurrent: 5
    
    on_partial_failure:
      primary: fail_composition
      related: omit
    
    retry:
      enabled: false
    
    cache:
      strategy: cache_first
      ttl: 60s
      scope: composition
```

### Example 3: Lazy-Loading Large Composition

```yaml
presentationComposition:
  name: CustomerProfileWorkspace
  version: v1
  primary: CustomerBasicInfoView
  related:
    - view: CustomerContactsView
      relationship: detail
    - view: CustomerDocumentsView
      relationship: detail
      cardinality: many
    - view: CustomerActivityLogView
      relationship: historical
      cardinality: many
    - view: CustomerNotesView
      relationship: supplementary
  
  fetch:
    order: primary_first
    
    primary:
      required: true
      timeout: 5s
    
    related:
      strategy: lazy  # Only fetch when scrolled into view
      timeout: 5s
      max_concurrent: 2
    
    on_partial_failure:
      primary: fail_composition
      related: degrade_gracefully
    
    retry:
      enabled: true
      max_attempts: 3
      scope: failed_only
    
    cache:
      strategy: network_first
      ttl: 120s
      scope: view
```

### Example 4: Strict Sequential Fetch

```yaml
presentationComposition:
  name: ComplianceReviewWorkspace
  version: v1
  primary: ComplianceHeaderView
  related:
    - view: CustomerKYCView
      relationship: derived
    - view: RiskAssessmentView
      relationship: derived
    - view: ComplianceDecisionView
      relationship: action
  
  fetch:
    order: sequential  # Each view depends on previous
    
    primary:
      required: true
      timeout: 10s
    
    related:
      strategy: sequential
      timeout: 10s
    
    on_partial_failure:
      primary: fail_composition
      related: fail_composition  # All required for compliance
    
    retry:
      enabled: true
      max_attempts: 3
      scope: all
    
    cache:
      strategy: none  # Compliance data must be fresh
      ttl: 0s
      scope: composition
```

---

## Validation Criteria

A conformant implementation MUST:

1. **Parse** the `fetch` block without error
2. **Respect** declared fetch order
3. **Enforce** timeouts as configured
4. **Execute** declared failure handling strategy
5. **Implement** at least one cache strategy

A conformant implementation SHOULD:

1. Support all three fetch orders
2. Support all related view strategies
3. Implement all cache strategies
4. Provide visibility into fetch status during loading

---

## Integration with Existing Grammar

### Extends

- `presentationComposition` - Adds `fetch` block

### References

- `materialization.retrieval` - View retrieval contracts
- `related.required` - Whether related view is required
- `timeConstraint` - Timeout semantics

### Enhances

- `presentationComposition.primary` - Adds fetch configuration
- `presentationComposition.related` - Adds fetch strategy

---

## EBNF Grammar Addition

```ebnf
fetch           ::= "fetch:" NEWLINE INDENT
                    "order:" fetch_order NEWLINE
                    primary_fetch
                    related_fetch
                    on_partial_failure
                    retry_config?
                    cache_config?
                    DEDENT

fetch_order     ::= "primary_first" | "parallel_all" | "sequential"

primary_fetch   ::= "primary:" NEWLINE INDENT
                    "required:" boolean NEWLINE
                    "timeout:" duration NEWLINE
                    DEDENT

related_fetch   ::= "related:" NEWLINE INDENT
                    "strategy:" related_strategy NEWLINE
                    "timeout:" duration NEWLINE
                    ( "max_concurrent:" integer NEWLINE )?
                    DEDENT

related_strategy ::= "parallel" | "sequential" | "lazy"

on_partial_failure ::= "on_partial_failure:" NEWLINE INDENT
                       "primary:" failure_action NEWLINE
                       "related:" failure_action NEWLINE
                       DEDENT

failure_action  ::= "fail_composition" | "degrade_gracefully" | "degrade" | "omit"

retry_config    ::= "retry:" NEWLINE INDENT
                    "enabled:" boolean NEWLINE
                    ( "max_attempts:" integer NEWLINE )?
                    ( "scope:" retry_scope NEWLINE )?
                    DEDENT

retry_scope     ::= "failed_only" | "all"

cache_config    ::= "cache:" NEWLINE INDENT
                    "strategy:" cache_strategy NEWLINE
                    ( "ttl:" duration NEWLINE )?
                    ( "scope:" cache_scope NEWLINE )?
                    DEDENT

cache_strategy  ::= "none" | "cache_first" | "stale_while_revalidate" | "network_first"

cache_scope     ::= "composition" | "view"
```

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial proposal |

---

## ADDENDUM AD: MASK EXPRESSION GRAMMAR (NEW in v1.5)

**Purpose**: Defines formal transform grammar, contextual masking, and custom mask patterns for field masking runtime  
**Extends**: `fieldPermissions.mask`  
**Priority**: MEDIUM

---

## Problem Statement

The current specification defines:
- `fieldPermissions` with `mask.type` (full, partial, hash)
- `fieldPermissions` with `mask.reveal` (last4, first3, none)
- `fieldPermissions` with `mask.unless` expression

However, there is no grammar for expressing:
1. Custom mask patterns beyond built-in types
2. Formal definition of mask transform functions
3. Placeholder characters and format preservation
4. Contextual masking (different masks for different contexts)

**Impact**: Implementations hardcode mask patterns and transformations without specification guidance.

---

## Design Principle Alignment

### Intent-First

This addendum expresses **what** transformation to apply, not **how** to implement it:

| Intent-Level (This Spec) | Runtime Decides |
|--------------------------|-----------------|
| `transform: reveal_last(4)` | String manipulation code |
| `placeholder: "***-**-"` | Character rendering |
| `preserve_format: true` | Regex or format parsing |
| `context: print` | Context detection |

### Transport-Agnostic

The grammar does not specify:
- String manipulation implementation
- Regex patterns
- Character encoding handling
- Rendering technology

---

## Intent-Level Grammar

### Extended Mask Block

Extend `fieldPermissions.mask` with formal transform grammar:

```yaml
fieldPermissions:
  - field: <field_name>
    visible: true | false | { policy: <policy_ref> }
    
    mask:                             # Existing, extended
      unless: <expression>            # Existing: skip mask when true
      
      transform: <transform_expr>     # NEW: Formal transform
      placeholder: <string>           # NEW: What to show for masked part
      preserve_format: true | false   # NEW: Keep original formatting
      
      context:                        # NEW: Context-specific masking
        default: <transform_expr>
        display: <transform_expr>     # For screen display
        print: <transform_expr>       # For printed documents
        export: <transform_expr>      # For data export
        log: <transform_expr>         # For audit logs
```

### Standard Mask Transforms (Normative)

These transforms MUST be supported by all conformant implementations:

```yaml
# Built-in transforms
mask_transforms:
  # Reveal transforms - show portion of value
  - reveal_last(n)                # Show last n characters
  - reveal_first(n)               # Show first n characters
  - reveal_middle(start, end)     # Show from position start to end
  - reveal_pattern(regex)         # Show parts matching regex
  
  # Hide transforms - hide portion of value
  - hide_last(n)                  # Hide last n characters
  - hide_first(n)                 # Hide first n characters
  - hide_middle(start, end)       # Hide from position start to end
  - hide_pattern(regex)           # Hide parts matching regex
  
  # Replace transforms - replace with different representation
  - redact                        # Replace entirely with placeholder
  - hash                          # Replace with consistent hash
  - hash_partial(n)               # Hash but show last n characters
  - truncate(n)                   # Show first n characters only
  
  # Format-aware transforms
  - mask_email                    # j***@example.com
  - mask_phone                    # (***) ***-1234
  - mask_credit_card              # ****-****-****-1234
  - mask_ssn                      # ***-**-1234
  - mask_account                  # ****1234
```

### Transform Expression Syntax

```yaml
# Simple transform
transform: reveal_last(4)

# Chained transforms (apply left to right)
transform: reveal_last(4) | uppercase

# Conditional transform
transform: 
  when: field.length > 10
  then: reveal_last(4)
  else: redact

# Composite placeholder
transform: reveal_last(4)
placeholder: "●●●●●●"  # Unicode bullets instead of *
```

### Context Types

| Context | Description | Use Case |
|---------|-------------|----------|
| `default` | Fallback for unspecified contexts | General display |
| `display` | Screen/UI display | Web, mobile apps |
| `print` | Printed documents | Reports, statements |
| `export` | Data export (CSV, Excel) | Downloads |
| `log` | Audit/debug logs | System logs |

---

## Semantic Guarantees

When a `fieldPermission` declares `mask`:

1. **Transform Application**: Transform MUST be applied unless `unless` expression is true
2. **Transform Availability**: All standard transforms MUST be available
3. **Placeholder Consistency**: Placeholder MUST be used for masked portions
4. **Context Respect**: Context-specific transforms MUST be applied when context is known

---

## Runtime Responsibilities

The runtime MUST:

1. **Implement Standard Transforms**: All transforms in the standard list
2. **Evaluate Unless**: Skip masking when `unless` expression is true
3. **Apply Placeholder**: Use declared placeholder character(s)
4. **Detect Context**: Determine current context and apply appropriate transform

The runtime MAY:

1. Implement additional custom transforms
2. Optimize transform evaluation
3. Cache masked values
4. Provide unmasking for authorized users

---

## Examples

### Example 1: Social Security Number

```yaml
fieldPermissions:
  - field: ssn
    visible:
      policy: PII_Access_Policy
    
    mask:
      unless: subject.roles contains "compliance"
      transform: mask_ssn
      placeholder: "●"
      preserve_format: true
      
      context:
        default: mask_ssn           # ●●●-●●-1234
        display: mask_ssn           # ●●●-●●-1234
        print: reveal_last(4)       # ***-**-1234
        export: redact              # [REDACTED]
        log: hash                   # sha256:a1b2c3...
```

### Example 2: Credit Card Number

```yaml
fieldPermissions:
  - field: card_number
    visible: true
    
    mask:
      unless: subject.roles contains "fraud_analyst"
      transform: mask_credit_card
      placeholder: "●"
      preserve_format: true
      
      context:
        default: mask_credit_card   # ●●●●-●●●●-●●●●-1234
        display: mask_credit_card
        print: reveal_last(4)       # ****-****-****-1234
        export: hash_partial(4)     # hash:1234
        log: redact                 # [REDACTED]
```

### Example 3: Email Address

```yaml
fieldPermissions:
  - field: email
    visible: true
    
    mask:
      unless: subject.id == entity.owner_id
      transform: mask_email
      placeholder: "●"
      preserve_format: true
      
      context:
        default: mask_email         # j●●●@example.com
        display: mask_email
        print: mask_email
        export: mask_email
        log: hash                   # sha256:...
```

### Example 4: Phone Number with Conditional

```yaml
fieldPermissions:
  - field: phone
    visible: true
    
    mask:
      unless: subject.roles contains "support" OR subject.id == entity.owner_id
      transform:
        when: phone.country_code == "US"
        then: mask_phone
        else: reveal_last(4)
      placeholder: "●"
      preserve_format: true
```

### Example 5: Account Number with Custom Pattern

```yaml
fieldPermissions:
  - field: account_number
    visible: true
    
    mask:
      unless: subject.roles contains "teller"
      transform: reveal_last(4)
      placeholder: "●"
      preserve_format: false  # Show as ●●●●1234 not ●●●●-●●●●-1234
```

### Example 6: Balance with Redaction

```yaml
fieldPermissions:
  - field: balance
    visible: true
    
    mask:
      unless: subject.id == entity.owner_id OR subject.roles contains "banker"
      transform: redact
      placeholder: "—"
      
      context:
        default: redact             # —
        display: redact
        print: redact
        export: redact
        log: redact
```

---

## Validation Criteria

A conformant implementation MUST:

1. **Parse** extended mask syntax without error
2. **Implement** all standard transforms
3. **Apply** placeholder as declared
4. **Respect** `unless` expression
5. **Support** context-specific transforms

A conformant implementation SHOULD:

1. Preserve format when `preserve_format: true`
2. Support transform chaining
3. Support conditional transforms
4. Provide consistent hashing for `hash` transform

---

## Integration with Existing Grammar

### Extends

- `fieldPermissions.mask` - Adds `transform`, `placeholder`, `preserve_format`, `context`

### References

- `policy` - Referenced in `visible.policy`
- `expression` - Used in `unless` and conditional transforms
- `subject` - Available in `unless` expressions

### Supersedes

- `mask.type` and `mask.reveal` - Replaced by `transform` expression
- Legacy syntax SHOULD still be supported for compatibility

---

## EBNF Grammar Addition

```ebnf
mask            ::= "mask:" NEWLINE INDENT
                    ( "unless:" expression NEWLINE )?
                    "transform:" transform_expr NEWLINE
                    ( "placeholder:" string NEWLINE )?
                    ( "preserve_format:" boolean NEWLINE )?
                    mask_context?
                    DEDENT

transform_expr  ::= simple_transform
                  | chained_transform
                  | conditional_transform

simple_transform ::= transform_name ( "(" transform_args ")" )?

chained_transform ::= simple_transform ( "|" simple_transform )*

conditional_transform ::= NEWLINE INDENT
                          "when:" expression NEWLINE
                          "then:" transform_expr NEWLINE
                          "else:" transform_expr NEWLINE
                          DEDENT

transform_name  ::= "reveal_last" | "reveal_first" | "reveal_middle"
                  | "reveal_pattern" | "hide_last" | "hide_first"
                  | "hide_middle" | "hide_pattern" | "redact" | "hash"
                  | "hash_partial" | "truncate" | "mask_email"
                  | "mask_phone" | "mask_credit_card" | "mask_ssn"
                  | "mask_account" | "uppercase" | "lowercase"

transform_args  ::= integer
                  | integer "," integer
                  | string

mask_context    ::= "context:" NEWLINE INDENT
                    ( "default:" transform_expr NEWLINE )?
                    ( "display:" transform_expr NEWLINE )?
                    ( "print:" transform_expr NEWLINE )?
                    ( "export:" transform_expr NEWLINE )?
                    ( "log:" transform_expr NEWLINE )?
                    DEDENT
```

---

## Standard Transforms Reference

### Reveal Transforms

| Transform | Input | Output |
|-----------|-------|--------|
| `reveal_last(4)` | `123-45-6789` | `●●●-●●-6789` |
| `reveal_first(3)` | `123-45-6789` | `123-●●-●●●●` |
| `reveal_middle(4,6)` | `123-45-6789` | `●●●-45-●●●●` |

### Hide Transforms

| Transform | Input | Output |
|-----------|-------|--------|
| `hide_last(4)` | `123-45-6789` | `123-45-●●●●` |
| `hide_first(3)` | `123-45-6789` | `●●●-45-6789` |
| `hide_middle(4,6)` | `123-45-6789` | `123-●●-6789` |

### Format-Aware Transforms

| Transform | Input | Output |
|-----------|-------|--------|
| `mask_email` | `john.doe@example.com` | `j●●●●●●@example.com` |
| `mask_phone` | `(555) 123-4567` | `(●●●) ●●●-4567` |
| `mask_credit_card` | `4111-1111-1111-1234` | `●●●●-●●●●-●●●●-1234` |
| `mask_ssn` | `123-45-6789` | `●●●-●●-6789` |
| `mask_account` | `9876543210` | `●●●●●●3210` |

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial proposal |

---

## ADDENDUM AE: Navigation Semantics (NEW in v1.5)

**Purpose**: Express navigation identity, active state, breadcrumb configuration, and visibility without specifying URL structure or browser implementation  
**Extends**: `navigation` (within `presentationComposition`)  
**Priority**: LOW

---

## Problem Statement

The current specification defines:
- `navigation.label` for display text
- `navigation.purpose` for semantic grouping
- `navigation.parent` for hierarchy
- `navigation.indicator` for dynamic badges

However, there is no grammar for expressing:
1. What parameters uniquely identify a navigation destination
2. How to construct dynamic breadcrumb labels
3. When a navigation item should be marked as "active"
4. How navigation items relate to each other for highlighting

**Impact**: Implementations must infer URL patterns and active states without specification guidance.

---

## Design Principle Alignment

### Intent-First

This addendum expresses **what** navigation identity and state mean, not **how** to render:

| Intent-Level (This Spec) | Runtime Decides |
|--------------------------|-----------------|
| `distinguishes_by: [account_id]` | URL path, query params, hash |
| `breadcrumb.template` | HTML rendering, separator chars |
| `active_when: current_composition` | CSS classes, aria attributes |
| `highlight_ancestors: true` | Visual styling |

### Transport-Agnostic

The grammar does not specify:
- URL structure (path, query, hash)
- Browser history API usage
- Mobile navigation stack behavior
- Breadcrumb separator characters

---

## Intent-Level Grammar

### Extended Navigation Block

Extend `navigation` with identity, breadcrumb, and active state semantics:

```yaml
presentationComposition:
  name: <string>
  navigation:
    label: <string>                   # Existing
    purpose: <purpose_ref>            # Existing
    parent: <composition_ref>         # Existing
    
    identity:                         # NEW: What makes this nav unique
      distinguishes_by: [<field>, ...]  # Parameters that identify instance
      canonical: true | false         # Is this the canonical route
      aliases: [<composition_ref>, ...]  # Other compositions that activate this
    
    breadcrumb:                       # NEW: Breadcrumb configuration
      template: <string_template>     # Dynamic label with {{field}} interpolation
      show: always | when_active | never
      separator: <string>             # Separator character (if runtime uses)
      truncate_at: <integer>          # Max characters before truncation
    
    active:                           # NEW: Active state configuration
      when: <active_expression>       # When to mark as active
      highlight_ancestors: true | false  # Also highlight parent items
      exact_match: true | false       # Require exact parameter match
    
    visibility:                       # NEW: When to show in navigation
      show_when: <expression>         # Conditional visibility
      hide_when_empty: true | false   # Hide if no accessible children
      collapse_single_child: true | false  # Skip if only one child
```

### Distinguishing Parameters

Parameters that make each navigation instance unique:

```yaml
# Single parameter
distinguishes_by: [account_id]

# Multiple parameters  
distinguishes_by: [customer_id, account_id]

# No parameters (singleton)
distinguishes_by: []
```

### Active Expressions

When a navigation item should be marked active:

```yaml
# Active when this composition is current
active_when: current_composition

# Active when this or descendants are current
active_when: current_composition OR descendant_active

# Active when specific condition is true
active_when: context.account_type == "premium"

# Active when any aliased composition is active
active_when: current_composition OR alias_active
```

### Breadcrumb Templates

Dynamic breadcrumb labels with field interpolation:

```yaml
# Static label
breadcrumb:
  template: "Accounts"

# Dynamic with entity field
breadcrumb:
  template: "Account {{account.number}}"

# Conditional template
breadcrumb:
  template: "{{account.type | capitalize}} Account - {{account.number}}"
```

---

## Semantic Guarantees

When a `navigation` declares these extended properties:

1. **Identity Uniqueness**: Navigation instances with different `distinguishes_by` values MUST be distinct
2. **Active Consistency**: Active state MUST be determined by `active.when` expression
3. **Breadcrumb Availability**: Template fields MUST be available in composition context
4. **Visibility Enforcement**: Navigation items MUST respect `visibility.show_when`

---

## Runtime Responsibilities

The runtime MUST:

1. **Track Identity**: Use `distinguishes_by` to create unique navigation entries
2. **Evaluate Active**: Compute active state using `active.when` expression
3. **Render Breadcrumb**: Interpolate template with available context
4. **Apply Visibility**: Show/hide based on `visibility` rules

The runtime MAY:

1. Choose URL scheme (path, query, hash, none)
2. Implement navigation history (browser, in-memory)
3. Animate breadcrumb transitions
4. Provide keyboard navigation

---

## Examples

### Example 1: Account Details with Dynamic Breadcrumb

```yaml
presentationComposition:
  name: AccountDetailWorkspace
  primary: AccountDetailsView
  
  navigation:
    label: "Account Details"
    purpose: accounts
    parent: AccountListWorkspace
    
    identity:
      distinguishes_by: [account_id]
      canonical: true
    
    breadcrumb:
      template: "Account {{account.number}} ({{account.type}})"
      show: always
      truncate_at: 30
    
    active:
      when: current_composition
      highlight_ancestors: true
      exact_match: true
    
    visibility:
      show_when: true
```

### Example 2: Dashboard with Conditional Visibility

```yaml
presentationComposition:
  name: PremiumDashboard
  primary: PremiumDashboardView
  
  navigation:
    label: "Premium Dashboard"
    purpose: home
    
    identity:
      distinguishes_by: []
      canonical: true
    
    breadcrumb:
      template: "Premium Dashboard"
      show: always
    
    active:
      when: current_composition
      highlight_ancestors: false
    
    visibility:
      show_when: session.account_tier == "premium"
      hide_when_empty: false
```

### Example 3: Transaction List with Ancestor Highlighting

```yaml
presentationComposition:
  name: TransactionDetailWorkspace
  primary: TransactionDetailView
  
  navigation:
    label: "Transaction"
    purpose: transactions
    parent: AccountDetailWorkspace
    
    identity:
      distinguishes_by: [account_id, transaction_id]
      canonical: true
    
    breadcrumb:
      template: "{{transaction.type | capitalize}} - {{transaction.amount | currency}}"
      show: when_active
      truncate_at: 25
    
    active:
      when: current_composition
      highlight_ancestors: true
      exact_match: true
```

### Example 4: Settings with Aliases

```yaml
presentationComposition:
  name: AccountSettingsWorkspace
  primary: AccountSettingsView
  
  navigation:
    label: "Settings"
    purpose: settings
    parent: AccountDetailWorkspace
    
    identity:
      distinguishes_by: [account_id]
      canonical: true
      aliases:
        - NotificationSettingsWorkspace
        - SecuritySettingsWorkspace
    
    breadcrumb:
      template: "Settings"
      show: always
    
    active:
      when: current_composition OR alias_active
      highlight_ancestors: true
      exact_match: false
```

### Example 5: Collapsible Section

```yaml
presentationComposition:
  name: ReportsSection
  primary: ReportsOverviewView
  
  navigation:
    label: "Reports"
    purpose: reports
    
    identity:
      distinguishes_by: []
      canonical: true
    
    breadcrumb:
      template: "Reports"
      show: always
    
    active:
      when: current_composition OR descendant_active
      highlight_ancestors: true
    
    visibility:
      show_when: subject.roles contains "analyst"
      hide_when_empty: true
      collapse_single_child: true
```

---

## Validation Criteria

A conformant implementation MUST:

1. **Parse** extended navigation syntax without error
2. **Track** navigation identity using `distinguishes_by`
3. **Evaluate** `active.when` expressions
4. **Interpolate** breadcrumb templates
5. **Apply** visibility rules

A conformant implementation SHOULD:

1. Highlight ancestors when `highlight_ancestors: true`
2. Support template functions in breadcrumbs (e.g., `| capitalize`)
3. Implement `collapse_single_child` optimization

---

## Integration with Existing Grammar

### Extends

- `presentationComposition.navigation` - Adds `identity`, `breadcrumb`, `active`, `visibility`

### References

- `expression` - Used in `active.when` and `visibility.show_when`
- `presentationComposition` - Navigation hierarchy via `parent`
- `session` - Available in visibility expressions

### Enhances

- `navigation.label` - Breadcrumb template provides dynamic labels
- `navigation.parent` - Ancestor highlighting uses parent hierarchy

---

## EBNF Grammar Addition

```ebnf
navigation      ::= "navigation:" NEWLINE INDENT
                    "label:" string NEWLINE
                    ( "purpose:" identifier NEWLINE )?
                    ( "parent:" composition_ref NEWLINE )?
                    nav_identity?
                    nav_breadcrumb?
                    nav_active?
                    nav_visibility?
                    indicator?
                    DEDENT

nav_identity    ::= "identity:" NEWLINE INDENT
                    "distinguishes_by:" "[" identifier_list "]" NEWLINE
                    ( "canonical:" boolean NEWLINE )?
                    ( "aliases:" "[" composition_ref_list "]" NEWLINE )?
                    DEDENT

nav_breadcrumb  ::= "breadcrumb:" NEWLINE INDENT
                    "template:" string_template NEWLINE
                    ( "show:" show_mode NEWLINE )?
                    ( "separator:" string NEWLINE )?
                    ( "truncate_at:" integer NEWLINE )?
                    DEDENT

show_mode       ::= "always" | "when_active" | "never"

nav_active      ::= "active:" NEWLINE INDENT
                    "when:" active_expression NEWLINE
                    ( "highlight_ancestors:" boolean NEWLINE )?
                    ( "exact_match:" boolean NEWLINE )?
                    DEDENT

active_expression ::= "current_composition"
                    | "descendant_active"
                    | "alias_active"
                    | expression
                    | active_expression "OR" active_expression
                    | active_expression "AND" active_expression

nav_visibility  ::= "visibility:" NEWLINE INDENT
                    ( "show_when:" expression NEWLINE )?
                    ( "hide_when_empty:" boolean NEWLINE )?
                    ( "collapse_single_child:" boolean NEWLINE )?
                    DEDENT
```

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial proposal |

---

## ADDENDUM AF: View Availability Policy (NEW in v1.5)

**Purpose**: Express read availability semantics, staleness tolerance, and multi-region fallback strategies without specifying infrastructure or replication technology  
**Extends**: `materialization`  
**Priority**: LOW

---

## Problem Statement

The current specification defines:
- `materialization` with `freshness.maxStaleness`
- `materialization` with `authority_agnostic` for multi-region views
- Authority semantics for write operations

However, there is no grammar for expressing:
1. Read region preferences (prefer local, any, specific)
2. What to do when preferred region is unavailable
3. Staleness tolerance beyond simple max duration
4. Fallback strategies for multi-region deployments

**Impact**: Implementations make arbitrary read routing decisions without specification guidance for multi-region scenarios.

---

## Design Principle Alignment

### Intent-First

This addendum expresses **what** availability semantics are needed, not **how** to implement them:

| Intent-Level (This Spec) | Runtime Decides |
|--------------------------|-----------------|
| `read_preference: local_preferred` | DNS routing, load balancer config |
| `staleness_budget: 5s` | Cache TTL, replication lag check |
| `fallback: remote` | Region failover logic |
| `degrade_to: cached` | Cache storage, expiry |

### Transport-Agnostic

The grammar does not specify:
- Region names or identifiers
- DNS or load balancer configuration
- Replication technology
- Network topology

---

## Intent-Level Grammar

### Availability Block (New)

Add `availability` block to `materialization` for read availability policies:

```yaml
materialization:
  name: <string>
  version: <vN>
  source: <DataState | list>
  targetState: <DataState>
  
  authority_agnostic: true | false    # Existing v1.3
  epoch_tolerant: true | false        # Existing v1.3
  
  availability:                       # NEW: Read availability policy
    read_preference:                  # Where to prefer reading from
      mode: local_preferred | any | specific | nearest
      specific_region: <region_ref>   # For specific mode
      
    staleness:                        # Staleness tolerance
      budget: <duration>              # Maximum acceptable age
      check: strict | advisory        # Fail vs warn on stale
      
    on_region_unavailable:            # What to do when region fails
      strategy: fallback_remote | use_cached | fail | wait
      fallback_order: [<region_ref>, ...]  # For fallback_remote
      wait_timeout: <duration>        # For wait strategy
      
    degradation:                      # Graceful degradation options
      enabled: true | false
      degrade_to: cached | partial | placeholder
      cache_max_age: <duration>       # How old cached data can be
      
    replication:                      # Replication expectations
      lag_tolerance: <duration>       # Expected replication delay
      consistency: eventual | strong   # Read consistency requirement
```

### Read Preference Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| `local_preferred` | Prefer local region, fallback to remote | Most views |
| `any` | Read from any available region | Highly replicated data |
| `specific` | Always read from specific region | Compliance requirements |
| `nearest` | Read from lowest latency region | Latency-sensitive |

### Unavailability Strategies

| Strategy | Description | Tradeoff |
|----------|-------------|----------|
| `fallback_remote` | Try other regions in order | Higher latency |
| `use_cached` | Return cached/stale data | Data may be old |
| `fail` | Return error immediately | No degradation |
| `wait` | Wait for recovery | Blocking |

### Degradation Modes

| Mode | Description |
|------|-------------|
| `cached` | Return cached data regardless of age |
| `partial` | Return subset of fields that are available |
| `placeholder` | Return placeholder/skeleton data |

---

## Semantic Guarantees

When a `materialization` declares `availability`:

1. **Preference Enforcement**: Read SHOULD come from preferred region when available
2. **Staleness Bound**: Data returned MUST be within `staleness.budget` or strategy applied
3. **Strategy Execution**: Unavailability MUST trigger declared strategy
4. **Degradation Safety**: Degraded responses MUST be clearly marked

---

## Runtime Responsibilities

The runtime MUST:

1. **Route Reads**: Route to preferred region when possible
2. **Check Staleness**: Validate data age against budget
3. **Handle Unavailability**: Execute declared strategy on failure
4. **Mark Degraded**: Indicate when response is degraded

The runtime MAY:

1. Implement region discovery/health checking
2. Maintain region latency metrics for `nearest` mode
3. Pre-warm caches for faster degradation
4. Implement adaptive routing based on conditions

---

## Examples

### Example 1: Standard Local-Preferred View

```yaml
materialization:
  name: AccountBalanceView
  version: v1
  source: AccountState
  targetState: AccountBalanceMaterialized
  authority_agnostic: true
  epoch_tolerant: true
  
  availability:
    read_preference:
      mode: local_preferred
    
    staleness:
      budget: 5s
      check: strict
    
    on_region_unavailable:
      strategy: fallback_remote
      fallback_order: [us-west, us-east, eu-west]
    
    degradation:
      enabled: true
      degrade_to: cached
      cache_max_age: 60s
    
    replication:
      lag_tolerance: 1s
      consistency: eventual
```

### Example 2: Compliance-Required Specific Region

```yaml
materialization:
  name: CustomerPIIView
  version: v1
  source: CustomerState
  targetState: CustomerPIIMaterialized
  authority_agnostic: false  # Must respect authority
  epoch_tolerant: false
  
  availability:
    read_preference:
      mode: specific
      specific_region: eu-west  # GDPR compliance
    
    staleness:
      budget: 0s  # Must be current
      check: strict
    
    on_region_unavailable:
      strategy: fail  # Do not fallback for compliance
    
    degradation:
      enabled: false
    
    replication:
      lag_tolerance: 0s
      consistency: strong
```

### Example 3: High-Availability Dashboard Metrics

```yaml
materialization:
  name: DashboardMetricsView
  version: v1
  source: [AccountState, TransactionState]
  targetState: DashboardMetricsMaterialized
  authority_agnostic: true
  epoch_tolerant: true
  
  availability:
    read_preference:
      mode: any  # Read from wherever is fastest
    
    staleness:
      budget: 30s
      check: advisory  # Warn but don't fail
    
    on_region_unavailable:
      strategy: use_cached
    
    degradation:
      enabled: true
      degrade_to: partial
      cache_max_age: 300s  # 5 minutes
    
    replication:
      lag_tolerance: 10s
      consistency: eventual
```

### Example 4: Latency-Optimized Search Results

```yaml
materialization:
  name: SearchResultsView
  version: v1
  source: SearchIndexState
  targetState: SearchResultsMaterialized
  authority_agnostic: true
  epoch_tolerant: true
  
  availability:
    read_preference:
      mode: nearest  # Lowest latency
    
    staleness:
      budget: 60s  # Search index can be somewhat stale
      check: advisory
    
    on_region_unavailable:
      strategy: fallback_remote
      fallback_order: []  # Any available region
    
    degradation:
      enabled: true
      degrade_to: placeholder
      cache_max_age: 120s
    
    replication:
      lag_tolerance: 30s
      consistency: eventual
```

### Example 5: Wait-for-Consistency View

```yaml
materialization:
  name: TransactionConfirmationView
  version: v1
  source: TransactionState
  targetState: TransactionConfirmationMaterialized
  authority_agnostic: false
  epoch_tolerant: false
  
  availability:
    read_preference:
      mode: local_preferred
    
    staleness:
      budget: 0s
      check: strict
    
    on_region_unavailable:
      strategy: wait
      wait_timeout: 10s
    
    degradation:
      enabled: false
    
    replication:
      lag_tolerance: 0s
      consistency: strong
```

---

## Validation Criteria

A conformant implementation MUST:

1. **Parse** the `availability` block without error
2. **Route** reads according to preference when possible
3. **Check** staleness against budget
4. **Execute** unavailability strategy
5. **Mark** degraded responses appropriately

A conformant implementation SHOULD:

1. Support all read preference modes
2. Implement all unavailability strategies
3. Track replication lag for `lag_tolerance`
4. Provide observability into read routing decisions

---

## Integration with Existing Grammar

### Extends

- `materialization` - Adds `availability` block

### References

- `materialization.authority_agnostic` - Multi-region read capability
- `materialization.epoch_tolerant` - Cross-epoch read capability
- `freshness.maxStaleness` - Existing staleness (complemented by this)

### Enhances

- `materialization.freshness` - More granular staleness control
- Authority semantics - Read-side policies for writes

---

## EBNF Grammar Addition

```ebnf
availability    ::= "availability:" NEWLINE INDENT
                    read_preference
                    staleness_config
                    on_region_unavailable
                    degradation_config?
                    replication_config?
                    DEDENT

read_preference ::= "read_preference:" NEWLINE INDENT
                    "mode:" read_mode NEWLINE
                    ( "specific_region:" region_ref NEWLINE )?
                    DEDENT

read_mode       ::= "local_preferred" | "any" | "specific" | "nearest"

staleness_config ::= "staleness:" NEWLINE INDENT
                     "budget:" duration NEWLINE
                     ( "check:" staleness_check NEWLINE )?
                     DEDENT

staleness_check ::= "strict" | "advisory"

on_region_unavailable ::= "on_region_unavailable:" NEWLINE INDENT
                          "strategy:" unavail_strategy NEWLINE
                          ( "fallback_order:" "[" region_ref_list "]" NEWLINE )?
                          ( "wait_timeout:" duration NEWLINE )?
                          DEDENT

unavail_strategy ::= "fallback_remote" | "use_cached" | "fail" | "wait"

degradation_config ::= "degradation:" NEWLINE INDENT
                       "enabled:" boolean NEWLINE
                       ( "degrade_to:" degrade_mode NEWLINE )?
                       ( "cache_max_age:" duration NEWLINE )?
                       DEDENT

degrade_mode    ::= "cached" | "partial" | "placeholder"

replication_config ::= "replication:" NEWLINE INDENT
                       ( "lag_tolerance:" duration NEWLINE )?
                       ( "consistency:" consistency_mode NEWLINE )?
                       DEDENT

consistency_mode ::= "eventual" | "strong"
```

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial proposal |

---

## ADDENDUM AG: Presentation Hints (NEW in v1.5)

**Purpose**: Express field-level display semantics, emphasis levels, and visualization intent without specifying CSS, fonts, or UI component libraries  
**Extends**: `dataType.fields`  
**Priority**: MEDIUM

---

## Problem Statement

The current specification defines:
- `dataType` with field types (string, decimal, boolean, etc.)
- `presentationView` with `consumes.dataState` binding
- Field types are structural, not presentational

However, there is no grammar for expressing:
1. How fields should be displayed (currency, badge, date format, etc.)
2. Relative importance/emphasis of fields
3. Semantic meaning of field values (positive/negative, active/inactive)
4. Display variants based on field values

**Impact**: UI implementations hardcode domain-specific display logic (CSS classes, formatters) without specification guidance.

---

## Design Principle Alignment

### Intent-First

This addendum expresses **what** semantic presentation means, not **how** to render:

| Intent-Level (This Spec) | Runtime Decides |
|--------------------------|-----------------|
| `display_as: currency` | Number formatting, symbol placement |
| `emphasis: primary` | Font size, weight, color |
| `semantic_variants.active: positive` | Green color, checkmark icon |
| `locale_aware: true` | ICU formatting, locale detection |

### Transport-Agnostic

The grammar does not specify:
- CSS classes or styles
- Font families or sizes
- Color values
- Icon libraries
- UI component libraries

---

## Intent-Level Grammar

### Presentation Block (New)

Add `presentation` block to `dataType.fields` for display hints:

```yaml
dataType:
  name: <string>
  version: <vN>
  fields:
    <field_name>:
      type: <type>
      required: <boolean>
      
      presentation:                   # NEW: Display hints
        display_as: <display_type>    # How to format the value
        emphasis: primary | secondary | tertiary | muted
        locale_aware: true | false    # Use locale formatting
        
        # For numeric types
        format:
          precision: <integer>        # Decimal places
          grouping: true | false      # Thousand separators
          currency_code: <string>     # For currency display
          unit: <string>              # For unit display (%, km, etc.)
          
        # For enum/status fields
        semantic_variants:            # Meaning of each value
          <value>: positive | negative | neutral | warning | info
          
        # For temporal types
        temporal:
          format: relative | absolute | both
          precision: year | month | day | hour | minute | second
          
        # For identifiers
        identifier:
          style: full | abbreviated | masked
          copyable: true | false
          
        # For boolean types
        boolean:
          true_label: <string>
          false_label: <string>
          style: text | toggle | badge
```

### Display Types

| Type | Description | Example |
|------|-------------|---------|
| `text` | Plain text display | Names, descriptions |
| `currency` | Formatted currency | $1,234.56 |
| `number` | Formatted number | 1,234.56 |
| `percentage` | Percentage format | 12.5% |
| `date` | Date display | Jan 15, 2026 |
| `datetime` | Date and time | Jan 15, 2026 2:30 PM |
| `time` | Time only | 2:30 PM |
| `duration` | Time duration | 2h 30m |
| `badge` | Badge/chip display | Status badges |
| `identifier` | ID/code display | Monospace, copyable |
| `email` | Email with mailto | user@example.com |
| `phone` | Phone with tel | (555) 123-4567 |
| `url` | Clickable link | https://example.com |
| `boolean` | True/false display | Yes/No, ✓/✗ |
| `rating` | Star/score display | ★★★★☆ |
| `progress` | Progress indicator | 75% bar |
| `image` | Image display | Avatar, thumbnail |

### Emphasis Levels

| Level | Description | Typical Treatment |
|-------|-------------|-------------------|
| `primary` | Most important | Larger, bolder, prominent |
| `secondary` | Supporting info | Normal weight, standard size |
| `tertiary` | Additional detail | Smaller, lighter |
| `muted` | De-emphasized | Grayed, smallest |

### Semantic Variants

| Variant | Description | Typical Treatment |
|---------|-------------|-------------------|
| `positive` | Good state | Green, success icon |
| `negative` | Bad state | Red, error icon |
| `neutral` | Normal state | Gray, no icon |
| `warning` | Caution state | Yellow/orange, warning icon |
| `info` | Informational | Blue, info icon |

---

## Semantic Guarantees

When a field declares `presentation`:

1. **Display Consistency**: Same `display_as` type MUST render consistently
2. **Emphasis Ordering**: Emphasis levels MUST follow visual hierarchy
3. **Variant Mapping**: Semantic variants MUST map to distinguishable treatments
4. **Locale Respect**: `locale_aware: true` MUST use appropriate locale

---

## Runtime Responsibilities

The runtime MUST:

1. **Recognize Display Types**: Support all standard display types
2. **Apply Emphasis**: Implement visual hierarchy for emphasis levels
3. **Map Variants**: Translate semantic variants to visual treatment
4. **Handle Locales**: Apply locale formatting when `locale_aware: true`

The runtime MAY:

1. Choose specific colors, fonts, icons for variants
2. Provide theme customization for variant colors
3. Implement additional display types beyond standard
4. Optimize rendering for specific platforms

---

## Examples

### Example 1: Account Entity

```yaml
dataType:
  name: Account
  version: v1
  fields:
    id:
      type: uuid
      presentation:
        display_as: identifier
        emphasis: muted
        identifier:
          style: abbreviated
          copyable: true
    
    account_number:
      type: string
      presentation:
        display_as: identifier
        emphasis: secondary
        identifier:
          style: full
          copyable: true
    
    account_type:
      type: string
      presentation:
        display_as: badge
        emphasis: secondary
        semantic_variants:
          checking: info
          savings: positive
          money_market: info
          cd: neutral
    
    balance:
      type: decimal
      presentation:
        display_as: currency
        emphasis: primary
        locale_aware: true
        format:
          precision: 2
          grouping: true
          currency_code: USD
    
    status:
      type: string
      presentation:
        display_as: badge
        emphasis: secondary
        semantic_variants:
          active: positive
          inactive: neutral
          frozen: negative
          pending: warning
    
    created_at:
      type: timestamp
      presentation:
        display_as: date
        emphasis: muted
        temporal:
          format: relative
          precision: day
```

### Example 2: Transaction Entity

```yaml
dataType:
  name: Transaction
  version: v1
  fields:
    id:
      type: uuid
      presentation:
        display_as: identifier
        emphasis: muted
        identifier:
          style: abbreviated
          copyable: true
    
    type:
      type: string
      presentation:
        display_as: badge
        emphasis: secondary
        semantic_variants:
          credit: positive
          debit: negative
          transfer: neutral
          fee: warning
    
    amount:
      type: decimal
      presentation:
        display_as: currency
        emphasis: primary
        locale_aware: true
        format:
          precision: 2
          currency_code: USD
    
    description:
      type: string
      presentation:
        display_as: text
        emphasis: secondary
    
    status:
      type: string
      presentation:
        display_as: badge
        emphasis: tertiary
        semantic_variants:
          completed: positive
          pending: warning
          failed: negative
          cancelled: neutral
    
    timestamp:
      type: timestamp
      presentation:
        display_as: datetime
        emphasis: tertiary
        temporal:
          format: both
          precision: minute
```

### Example 3: Alert Entity

```yaml
dataType:
  name: Alert
  version: v1
  fields:
    id:
      type: uuid
      presentation:
        display_as: identifier
        emphasis: muted
    
    severity:
      type: string
      presentation:
        display_as: badge
        emphasis: primary
        semantic_variants:
          critical: negative
          high: warning
          medium: info
          low: neutral
    
    title:
      type: string
      presentation:
        display_as: text
        emphasis: primary
    
    message:
      type: string
      presentation:
        display_as: text
        emphasis: secondary
    
    acknowledged:
      type: boolean
      presentation:
        display_as: boolean
        emphasis: tertiary
        boolean:
          true_label: "Acknowledged"
          false_label: "Pending"
          style: badge
    
    created_at:
      type: timestamp
      presentation:
        display_as: datetime
        emphasis: muted
        temporal:
          format: relative
          precision: minute
```

### Example 4: Customer Entity with Contact Info

```yaml
dataType:
  name: Customer
  version: v1
  fields:
    name:
      type: string
      presentation:
        display_as: text
        emphasis: primary
    
    email:
      type: string
      presentation:
        display_as: email
        emphasis: secondary
    
    phone:
      type: string
      presentation:
        display_as: phone
        emphasis: secondary
    
    tier:
      type: string
      presentation:
        display_as: badge
        emphasis: secondary
        semantic_variants:
          platinum: positive
          gold: info
          silver: neutral
          bronze: muted
    
    satisfaction_score:
      type: decimal
      presentation:
        display_as: rating
        emphasis: tertiary
        format:
          precision: 1
    
    onboarding_progress:
      type: decimal
      presentation:
        display_as: progress
        emphasis: tertiary
        format:
          precision: 0
          unit: "%"
```

---

## Visualization Grammar

For fields containing aggregate or time-series data, visualization hints enable chart and graph rendering.

### Visualization Block

Add `visualization` block to fields for chart/graph display:

```yaml
dataType:
  name: <string>
  fields:
    <field_name>:
      type: array | object
      
      visualization:                  # NEW: Visualization intent
        type: chart | sparkline | gauge | indicator
        
        # For chart type
        chart:
          type: bar | line | area | pie | donut | scatter | heatmap
          orientation: horizontal | vertical  # For bar charts
          stacked: true | false               # For bar/area charts
          
          dimensions:
            x: <field_path>           # Field for x-axis
            y: [<field_path>, ...]    # Field(s) for y-axis (multi-series)
            group_by: <field_path>    # Optional grouping field
            
          time_series:
            enabled: true | false
            granularity: hour | day | week | month | quarter | year
            range: last_7_days | last_30_days | last_12_months | all | custom
            
          series_treatment:           # Semantic meaning of series
            <series_name>: positive | negative | neutral | comparison
            
          axis:
            x:
              label: <string>
              format: <display_type>  # References display types above
            y:
              label: <string>
              format: <display_type>
              min: <number>
              max: <number>
              
          legend:
            position: top | bottom | left | right | none
            
        # For sparkline type
        sparkline:
          trend_direction: rising | falling | neutral | auto
          show_endpoint: true | false
          fill: true | false
          
        # For gauge type
        gauge:
          min: <number>
          max: <number>
          thresholds:
            <threshold_value>: positive | warning | negative
          display_value: true | false
          
        # For indicator type
        indicator:
          comparison_to: previous | target | baseline
          direction_semantics:
            up: positive | negative | neutral
            down: positive | negative | neutral
```

### Visualization Types

| Type | Description | Use Case |
|------|-------------|----------|
| `chart` | Full chart visualization | Dashboards, reports |
| `sparkline` | Inline mini chart | Tables, lists, compact views |
| `gauge` | Progress/meter display | KPIs, thresholds |
| `indicator` | Trend with comparison | Metrics, deltas |

### Chart Types

| Chart | Description | Best For |
|-------|-------------|----------|
| `bar` | Categorical comparison | Comparing values |
| `line` | Trend over time | Time series |
| `area` | Cumulative trends | Stacked totals |
| `pie` | Part-to-whole | Distributions |
| `donut` | Part-to-whole with center | Distributions with metric |
| `scatter` | Correlation | Relationships |
| `heatmap` | Density/intensity | Patterns |

### Visualization Examples

#### Example: Monthly Transaction Summary

```yaml
dataType:
  name: TransactionSummary
  version: v1
  fields:
    monthly_totals:
      type: array
      items:
        type: object
        fields:
          month: { type: string }
          income: { type: decimal }
          expenses: { type: decimal }
          net: { type: decimal }
      
      visualization:
        type: chart
        chart:
          type: bar
          orientation: vertical
          stacked: false
          
          dimensions:
            x: month
            y: [income, expenses]
            
          time_series:
            enabled: true
            granularity: month
            range: last_12_months
            
          series_treatment:
            income: positive
            expenses: negative
            
          axis:
            x:
              label: "Month"
              format: date
            y:
              label: "Amount"
              format: currency
              
          legend:
            position: top
```

#### Example: Balance Trend Sparkline

```yaml
dataType:
  name: Account
  version: v1
  fields:
    balance_history:
      type: array
      items:
        type: object
        fields:
          date: { type: timestamp }
          balance: { type: decimal }
      
      visualization:
        type: sparkline
        sparkline:
          trend_direction: auto
          show_endpoint: true
          fill: true
```

#### Example: Credit Utilization Gauge

```yaml
dataType:
  name: CreditCard
  version: v1
  fields:
    utilization_percent:
      type: decimal
      
      visualization:
        type: gauge
        gauge:
          min: 0
          max: 100
          thresholds:
            30: positive     # 0-30% is good
            70: warning      # 30-70% is caution
            100: negative    # 70-100% is high
          display_value: true
```

#### Example: Balance Change Indicator

```yaml
dataType:
  name: Account
  version: v1
  fields:
    balance_change:
      type: object
      fields:
        current: { type: decimal }
        previous: { type: decimal }
        delta: { type: decimal }
        delta_percent: { type: decimal }
      
      visualization:
        type: indicator
        indicator:
          comparison_to: previous
          direction_semantics:
            up: positive      # Balance increasing is good
            down: negative    # Balance decreasing is bad
```

### Visualization Runtime Responsibilities

The runtime MUST:

1. **Recognize** all visualization types
2. **Map** chart types to appropriate rendering
3. **Apply** series treatment semantics
4. **Honor** axis formatting from display types

The runtime MAY:

1. Choose specific charting libraries
2. Implement interactivity (hover, zoom, pan)
3. Provide responsive resizing
4. Add animation to chart transitions

---

## Validation Criteria

A conformant implementation MUST:

1. **Parse** the `presentation` block without error
2. **Support** all standard display types
3. **Implement** emphasis visual hierarchy
4. **Map** semantic variants to distinguishable treatments
5. **Apply** locale formatting when requested

A conformant implementation SHOULD:

1. Provide consistent treatment for each semantic variant
2. Support theme customization for variant colors
3. Implement all display type options
4. Provide accessibility (aria-labels) for semantic content

---

## Integration with Existing Grammar

### Extends

- `dataType.fields` - Adds `presentation` block

### References

- `presentationView.consumes` - Views display fields with presentation hints
- `fieldPermissions` - Masking may interact with presentation
- `materialization` - Views materialize fields with presentation

### Complements

- `fieldPermissions.mask` - Masking transforms apply before presentation
- `presentationView.interactionModel` - Badges and actions complement presentation

---

## EBNF Grammar Addition

```ebnf
presentation    ::= "presentation:" NEWLINE INDENT
                    "display_as:" display_type NEWLINE
                    ( "emphasis:" emphasis_level NEWLINE )?
                    ( "locale_aware:" boolean NEWLINE )?
                    format_config?
                    semantic_variants?
                    temporal_config?
                    identifier_config?
                    boolean_config?
                    DEDENT

display_type    ::= "text" | "currency" | "number" | "percentage"
                  | "date" | "datetime" | "time" | "duration"
                  | "badge" | "identifier" | "email" | "phone"
                  | "url" | "boolean" | "rating" | "progress" | "image"

emphasis_level  ::= "primary" | "secondary" | "tertiary" | "muted"

format_config   ::= "format:" NEWLINE INDENT
                    ( "precision:" integer NEWLINE )?
                    ( "grouping:" boolean NEWLINE )?
                    ( "currency_code:" string NEWLINE )?
                    ( "unit:" string NEWLINE )?
                    DEDENT

semantic_variants ::= "semantic_variants:" NEWLINE INDENT
                      ( identifier ":" semantic_value NEWLINE )+
                      DEDENT

semantic_value  ::= "positive" | "negative" | "neutral" | "warning" | "info"

temporal_config ::= "temporal:" NEWLINE INDENT
                    ( "format:" temporal_format NEWLINE )?
                    ( "precision:" temporal_precision NEWLINE )?
                    DEDENT

temporal_format ::= "relative" | "absolute" | "both"

temporal_precision ::= "year" | "month" | "day" | "hour" | "minute" | "second"

identifier_config ::= "identifier:" NEWLINE INDENT
                      ( "style:" identifier_style NEWLINE )?
                      ( "copyable:" boolean NEWLINE )?
                      DEDENT

identifier_style ::= "full" | "abbreviated" | "masked"

boolean_config  ::= "boolean:" NEWLINE INDENT
                    ( "true_label:" string NEWLINE )?
                    ( "false_label:" string NEWLINE )?
                    ( "style:" boolean_style NEWLINE )?
                    DEDENT

boolean_style   ::= "text" | "toggle" | "badge"

visualization   ::= "visualization:" NEWLINE INDENT
                    "type:" visualization_type NEWLINE
                    ( chart_config | sparkline_config | gauge_config | indicator_config )?
                    DEDENT

visualization_type ::= "chart" | "sparkline" | "gauge" | "indicator"

chart_config    ::= "chart:" NEWLINE INDENT
                    "type:" chart_type NEWLINE
                    ( "orientation:" orientation NEWLINE )?
                    ( "stacked:" boolean NEWLINE )?
                    ( chart_dimensions )?
                    ( time_series_config )?
                    ( series_treatment )?
                    ( axis_config )?
                    ( legend_config )?
                    DEDENT

chart_type      ::= "bar" | "line" | "area" | "pie" | "donut" | "scatter" | "heatmap"

orientation     ::= "horizontal" | "vertical"

chart_dimensions ::= "dimensions:" NEWLINE INDENT
                     "x:" field_path NEWLINE
                     "y:" "[" field_path_list "]" NEWLINE
                     ( "group_by:" field_path NEWLINE )?
                     DEDENT

time_series_config ::= "time_series:" NEWLINE INDENT
                       "enabled:" boolean NEWLINE
                       ( "granularity:" time_granularity NEWLINE )?
                       ( "range:" time_range NEWLINE )?
                       DEDENT

time_granularity ::= "hour" | "day" | "week" | "month" | "quarter" | "year"

time_range      ::= "last_7_days" | "last_30_days" | "last_12_months" | "all" | "custom"

series_treatment ::= "series_treatment:" NEWLINE INDENT
                     ( identifier ":" series_semantic NEWLINE )+
                     DEDENT

series_semantic ::= "positive" | "negative" | "neutral" | "comparison"

axis_config     ::= "axis:" NEWLINE INDENT
                    ( "x:" axis_definition NEWLINE )?
                    ( "y:" axis_definition NEWLINE )?
                    DEDENT

axis_definition ::= NEWLINE INDENT
                    ( "label:" string NEWLINE )?
                    ( "format:" display_type NEWLINE )?
                    ( "min:" number NEWLINE )?
                    ( "max:" number NEWLINE )?
                    DEDENT

legend_config   ::= "legend:" NEWLINE INDENT
                    "position:" legend_position NEWLINE
                    DEDENT

legend_position ::= "top" | "bottom" | "left" | "right" | "none"

sparkline_config ::= "sparkline:" NEWLINE INDENT
                     ( "trend_direction:" trend_direction NEWLINE )?
                     ( "show_endpoint:" boolean NEWLINE )?
                     ( "fill:" boolean NEWLINE )?
                     DEDENT

trend_direction ::= "rising" | "falling" | "neutral" | "auto"

gauge_config    ::= "gauge:" NEWLINE INDENT
                    "min:" number NEWLINE
                    "max:" number NEWLINE
                    ( gauge_thresholds )?
                    ( "display_value:" boolean NEWLINE )?
                    DEDENT

gauge_thresholds ::= "thresholds:" NEWLINE INDENT
                     ( number ":" semantic_value NEWLINE )+
                     DEDENT

indicator_config ::= "indicator:" NEWLINE INDENT
                     "comparison_to:" comparison_target NEWLINE
                     ( direction_semantics )?
                     DEDENT

comparison_target ::= "previous" | "target" | "baseline"

direction_semantics ::= "direction_semantics:" NEWLINE INDENT
                        "up:" semantic_value NEWLINE
                        "down:" semantic_value NEWLINE
                        DEDENT
```

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial proposal |

---

## ADDENDUM AH: Surface and Container Semantics (NEW in v1.5)

**Purpose**: Express UI structure intent for surfaces (modals, drawers) and layout regions without specifying CSS, JavaScript libraries, or platform-specific components  
**Extends**: `presentationComposition`, `navigation`  
**Priority**: MEDIUM

---

## Problem Statement

The current specification defines:
- `presentationComposition` with primary and related views
- `navigation` with purposes and scopes
- `layout` hints for view-level field arrangement

However, there is no grammar for expressing:
1. How compositions should be presented (modal, drawer, inline)
2. How views should be positioned within a composition (regions)
3. When and how content should collapse or expand
4. How navigation items should behave as menus

**Impact**: Implementations make arbitrary decisions about UI structure without specification guidance.

---

## Design Principle Alignment

### Intent-First

This addendum expresses **what** UI structure intent means, not **how** to render:

| Intent-Level (This Spec) | Runtime Decides |
|--------------------------|-----------------|
| `surface.type: modal` | Dialog component, overlay styling |
| `container.regions.position_hint: aside` | CSS sidebar, flexbox layout |
| `collapsibility.default_state: collapsed` | Accordion, disclosure widget |
| `menu_intent.behavior: on_demand` | Hamburger, dropdown, slide-out |

### Transport-Agnostic

The grammar does not specify:
- CSS classes or layout properties
- JavaScript interaction libraries
- Animation implementations
- Platform-specific UI components

---

## Intent-Level Grammar

### 1. Surface Types

Add `surface` block to `presentationComposition` for overlay/modal behavior:

```yaml
presentationComposition:
  name: <string>
  version: <vN>
  
  surface:                          # NEW: How this composition is presented
    type: inline | modal | drawer | sheet | popover | fullscreen
    
    # For overlay types (modal, drawer, sheet, popover)
    size: small | medium | large | auto
    position: center | top | bottom | start | end  # Semantic positions
    
    dismissible: true | false       # Can user dismiss without action
    dismissal_triggers:             # What dismisses the surface
      - escape                      # Escape key
      - backdrop                    # Click outside
      - explicit                    # Only via action button
    
    focus_management:
      trap: true | false            # Trap focus within surface
      return_on_close: true | false # Return focus when closed
      initial_focus: <field_ref>    # Where to focus on open
    
    transition:                     # Semantic transition intent
      enter: fade | slide | scale | none
      exit: fade | slide | scale | none
```

#### Surface Type Semantics

| Type | Description | Typical Use |
|------|-------------|-------------|
| `inline` | Renders within page flow | Standard compositions |
| `modal` | Centered overlay blocking interaction | Confirmations, critical actions |
| `drawer` | Side panel sliding in | Navigation, filters, details |
| `sheet` | Bottom panel sliding up | Mobile actions, quick forms |
| `popover` | Small contextual overlay | Tooltips, inline actions |
| `fullscreen` | Takes over entire viewport | Immersive experiences, wizards |

---

### 2. Container Regions

Add `container` block to `presentationComposition` for layout regions:

```yaml
presentationComposition:
  name: <string>
  
  container:                        # NEW: Layout region structure
    regions:
      - name: main                  # Semantic region name
        priority: primary           # Visual priority
        scrollable: true            # Content can scroll
        
      - name: sidebar
        priority: secondary
        position_hint: aside        # Semantic position
        width_hint: narrow | medium | wide  # Relative width
        collapsible: true
        default_collapsed: false
        collapse_breakpoint: tablet # When to auto-collapse
        
      - name: actions
        priority: tertiary
        position_hint: fixed        # Stays visible
        anchor: bottom              # Anchor point
        
      - name: header
        priority: primary
        position_hint: before_main  # Before main content
        sticky: true                # Sticks on scroll
    
    # Assign views to regions
    view_placement:
      primary: main                 # Primary view goes to main
      related:
        <view_ref>: <region_name>   # Map related views to regions
```

#### Position Hints

| Hint | Description | Typical Rendering |
|------|-------------|-------------------|
| `main` | Primary content area | Center column |
| `aside` | Secondary content beside main | Sidebar |
| `before_main` | Content before main | Header, breadcrumbs |
| `after_main` | Content after main | Footer |
| `fixed` | Persists during scroll | Floating actions |
| `overlay` | Above other content | Notifications |

#### Width Hints

| Hint | Description | Typical Treatment |
|------|-------------|-------------------|
| `narrow` | Minimal width | 200-300px sidebar |
| `medium` | Moderate width | 300-400px sidebar |
| `wide` | Significant width | 400-600px sidebar |

---

### 3. Collapsibility Semantics

Add `collapsibility` block to related views and regions:

```yaml
presentationComposition:
  name: AccountDetail
  related:
    - view: TransactionHistory
      collapsibility:               # NEW: Collapse behavior
        collapsible: true
        default_state: expanded | collapsed
        
        trigger:                    # What triggers collapse
          type: user | viewport | breakpoint | policy
          breakpoint: mobile | tablet | desktop  # For breakpoint type
          
        preserve_state:
          scope: session | view | never
          
        summary:                    # What to show when collapsed
          template: "{{count}} transactions"
          show_indicator: true      # Show expand indicator
        
        animation: true | false     # Animate expand/collapse
        
    - view: AccountSettings
      collapsibility:
        collapsible: true
        default_state: collapsed
        expandable_by:              # Policy integration
          - role: admin
          - role: owner
        mutually_exclusive: [TransactionHistory]  # Only one expanded
```

#### Trigger Types

| Trigger | Description | Use Case |
|---------|-------------|----------|
| `user` | User explicitly toggles | Optional content |
| `viewport` | Based on viewport size | Responsive design |
| `breakpoint` | At specific breakpoints | Mobile-first |
| `policy` | Based on authorization | Feature gating |

---

### 4. Menu Behavior Intent

Add `menu_intent` to navigation for menu behavior semantics:

```yaml
navigation:
  menu_intent:                      # NEW: Menu behavior intent
    purpose_groups:
      primary:                      # Groups certain purposes together
        purposes: [home, accounts, transactions]
        behavior: always_visible    # Always shown
        selection: single           # One item selected at a time
        orientation: horizontal | vertical  # Layout orientation
        
      utility:
        purposes: [settings, help, profile]
        behavior: on_demand         # User-triggered to reveal
        selection: none             # No persistent selection
        position_hint: end          # Positioned at end
        
      contextual:
        purposes: [action]
        behavior: context_sensitive # Based on current composition
        selection: multi            # Multiple can be active
        position_hint: inline       # With current content
    
    overflow:                       # When items don't fit
      strategy: collapse | scroll | wrap | prioritize
      priority_order: [home, accounts, transactions]
      collapse_label: "More"        # Label for overflow menu
    
    mobile_behavior:                # Mobile-specific behavior
      strategy: drawer | bottom_nav | hamburger
      drawer_position: start | end
```

#### Menu Behaviors

| Behavior | Description | Typical Rendering |
|----------|-------------|-------------------|
| `always_visible` | Always shown in nav | Tab bar, top nav |
| `on_demand` | Revealed by user action | Dropdown, hamburger |
| `context_sensitive` | Based on current context | Contextual actions |
| `hover_reveal` | Revealed on hover | Mega menu |

---

## Semantic Guarantees

When a composition declares surface/container semantics:

1. **Surface Consistency**: Same `surface.type` MUST provide consistent behavior
2. **Region Ordering**: `priority` MUST influence visual prominence
3. **Collapse State**: `default_state` MUST be respected on initial render
4. **Focus Management**: `focus_management` MUST be honored for accessibility
5. **Menu Grouping**: `purpose_groups` MUST group navigation items consistently

---

## Runtime Responsibilities

The runtime MUST:

1. **Render Surfaces**: Implement all surface types with appropriate behavior
2. **Position Regions**: Arrange regions according to `position_hint`
3. **Manage Collapse**: Implement collapse triggers and state preservation
4. **Focus Trap**: Implement focus management for overlay surfaces
5. **Group Navigation**: Group items by declared purpose groups

The runtime MAY:

1. Choose specific UI components for each surface type
2. Implement platform-specific surface behavior
3. Add animations beyond declared transition intent
4. Provide additional collapse/expand gestures

---

## Examples

### Example 1: Modal Confirmation Dialog

```yaml
presentationComposition:
  name: TransferConfirmation
  version: v1
  primary: TransferConfirmationView
  
  surface:
    type: modal
    size: medium
    position: center
    dismissible: false              # Must explicitly confirm or cancel
    dismissal_triggers:
      - explicit                    # Only via buttons
    focus_management:
      trap: true
      return_on_close: true
      initial_focus: confirm_button
    transition:
      enter: scale
      exit: fade
  
  triggers:                         # What opens this modal
    - intent: InitiateTransfer
      condition: requires_confirmation
```

### Example 2: Settings Drawer

```yaml
presentationComposition:
  name: SettingsDrawer
  version: v1
  primary: SettingsView
  
  surface:
    type: drawer
    size: medium
    position: end                   # Right side (LTR) / Left side (RTL)
    dismissible: true
    dismissal_triggers:
      - escape
      - backdrop
    focus_management:
      trap: true
      return_on_close: true
    transition:
      enter: slide
      exit: slide
  
  related:
    - view: NotificationPreferences
      collapsibility:
        collapsible: true
        default_state: collapsed
        trigger:
          type: user
```

### Example 3: Dashboard with Regions

```yaml
presentationComposition:
  name: CustomerDashboard
  version: v1
  primary: DashboardSummaryView
  
  container:
    regions:
      - name: main
        priority: primary
        scrollable: true
        
      - name: sidebar
        priority: secondary
        position_hint: aside
        width_hint: narrow
        collapsible: true
        default_collapsed: false
        collapse_breakpoint: tablet
        
      - name: quick_actions
        priority: tertiary
        position_hint: fixed
        anchor: bottom
    
    view_placement:
      primary: main
      related:
        AccountListView: sidebar
        AlertSummaryView: sidebar
        QuickActionBar: quick_actions
  
  related:
    - view: AccountListView
      relationship: contextual
      collapsibility:
        collapsible: true
        default_state: expanded
        summary:
          template: "{{count}} accounts"
          
    - view: AlertSummaryView
      relationship: contextual
      collapsibility:
        collapsible: true
        default_state: collapsed
        summary:
          template: "{{count}} alerts"
          show_indicator: true
          
    - view: QuickActionBar
      relationship: action
```

### Example 4: Mobile Bottom Sheet

```yaml
presentationComposition:
  name: AccountActionsSheet
  version: v1
  primary: AccountActionsView
  
  surface:
    type: sheet
    size: auto                      # Size based on content
    position: bottom
    dismissible: true
    dismissal_triggers:
      - escape
      - backdrop
      - swipe_down                  # Mobile gesture
    focus_management:
      trap: true
    transition:
      enter: slide
      exit: slide
```

### Example 5: Navigation Menu Intent

```yaml
navigation:
  menu_intent:
    purpose_groups:
      primary:
        purposes: [home, accounts, transactions]
        behavior: always_visible
        selection: single
        orientation: horizontal
        
      actions:
        purposes: [action]
        behavior: context_sensitive
        selection: none
        position_hint: inline
        
      settings:
        purposes: [settings, help, authentication]
        behavior: on_demand
        selection: none
        position_hint: end
    
    overflow:
      strategy: prioritize
      priority_order: [home, accounts, settings]
      collapse_label: "More"
    
    mobile_behavior:
      strategy: bottom_nav
      purposes_in_bottom: [home, accounts, transactions, settings]
```

---

## Validation Criteria

A conformant implementation MUST:

1. **Parse** all surface and container blocks without error
2. **Support** all surface types with appropriate behavior
3. **Implement** focus management for overlay surfaces
4. **Respect** collapse state and triggers
5. **Group** navigation by purpose groups

A conformant implementation SHOULD:

1. Provide responsive behavior for regions
2. Implement touch gestures for mobile surfaces
3. Support keyboard navigation within menus
4. Animate transitions according to intent

---

## Integration with Existing Grammar

### Extends

- `presentationComposition` - Adds `surface`, `container`, `triggers`
- `presentationComposition.related` - Adds `collapsibility`
- `navigation` - Adds `menu_intent`

### References

- `navigation.purpose` - Purpose categories for grouping
- `fieldPermissions` - Policy integration for `expandable_by`
- `experience` - Experience-level navigation behavior

### Complements

- `08-navigation-semantics.md` - Navigation identity and active state
- `10-presentation-hints.md` - Field-level presentation
- `01-form-intent-binding.md` - Form actions that trigger surfaces

---

## EBNF Grammar Addition

```ebnf
surface         ::= "surface:" NEWLINE INDENT
                    "type:" surface_type NEWLINE
                    ( "size:" surface_size NEWLINE )?
                    ( "position:" surface_position NEWLINE )?
                    ( "dismissible:" boolean NEWLINE )?
                    ( "dismissal_triggers:" "[" dismissal_trigger_list "]" NEWLINE )?
                    ( focus_management )?
                    ( transition )?
                    DEDENT

surface_type    ::= "inline" | "modal" | "drawer" | "sheet" | "popover" | "fullscreen"

surface_size    ::= "small" | "medium" | "large" | "auto"

surface_position ::= "center" | "top" | "bottom" | "start" | "end"

dismissal_trigger ::= "escape" | "backdrop" | "explicit" | "swipe_down"

focus_management ::= "focus_management:" NEWLINE INDENT
                     ( "trap:" boolean NEWLINE )?
                     ( "return_on_close:" boolean NEWLINE )?
                     ( "initial_focus:" field_ref NEWLINE )?
                     DEDENT

transition      ::= "transition:" NEWLINE INDENT
                    ( "enter:" transition_type NEWLINE )?
                    ( "exit:" transition_type NEWLINE )?
                    DEDENT

transition_type ::= "fade" | "slide" | "scale" | "none"

container       ::= "container:" NEWLINE INDENT
                    "regions:" NEWLINE INDENT
                    region_definition+
                    DEDENT
                    ( view_placement )?
                    DEDENT

region_definition ::= "-" "name:" identifier NEWLINE INDENT
                      ( "priority:" priority_level NEWLINE )?
                      ( "position_hint:" position_hint NEWLINE )?
                      ( "width_hint:" width_hint NEWLINE )?
                      ( "scrollable:" boolean NEWLINE )?
                      ( "collapsible:" boolean NEWLINE )?
                      ( "default_collapsed:" boolean NEWLINE )?
                      ( "collapse_breakpoint:" breakpoint NEWLINE )?
                      ( "sticky:" boolean NEWLINE )?
                      ( "anchor:" anchor_position NEWLINE )?
                      DEDENT

priority_level  ::= "primary" | "secondary" | "tertiary"

position_hint   ::= "main" | "aside" | "before_main" | "after_main" | "fixed" | "overlay"

width_hint      ::= "narrow" | "medium" | "wide"

breakpoint      ::= "mobile" | "tablet" | "desktop"

anchor_position ::= "top" | "bottom" | "start" | "end"

view_placement  ::= "view_placement:" NEWLINE INDENT
                    "primary:" identifier NEWLINE
                    ( "related:" NEWLINE INDENT
                      ( view_ref ":" identifier NEWLINE )+
                      DEDENT )?
                    DEDENT

collapsibility  ::= "collapsibility:" NEWLINE INDENT
                    "collapsible:" boolean NEWLINE
                    ( "default_state:" collapse_state NEWLINE )?
                    ( collapse_trigger )?
                    ( preserve_state )?
                    ( collapse_summary )?
                    ( "animation:" boolean NEWLINE )?
                    ( "expandable_by:" "[" role_list "]" NEWLINE )?
                    ( "mutually_exclusive:" "[" view_ref_list "]" NEWLINE )?
                    DEDENT

collapse_state  ::= "expanded" | "collapsed"

collapse_trigger ::= "trigger:" NEWLINE INDENT
                     "type:" trigger_type NEWLINE
                     ( "breakpoint:" breakpoint NEWLINE )?
                     DEDENT

trigger_type    ::= "user" | "viewport" | "breakpoint" | "policy"

preserve_state  ::= "preserve_state:" NEWLINE INDENT
                    "scope:" state_scope NEWLINE
                    DEDENT

state_scope     ::= "session" | "view" | "never"

collapse_summary ::= "summary:" NEWLINE INDENT
                     "template:" string NEWLINE
                     ( "show_indicator:" boolean NEWLINE )?
                     DEDENT

menu_intent     ::= "menu_intent:" NEWLINE INDENT
                    "purpose_groups:" NEWLINE INDENT
                    purpose_group_definition+
                    DEDENT
                    ( menu_overflow )?
                    ( mobile_behavior )?
                    DEDENT

purpose_group_definition ::= identifier ":" NEWLINE INDENT
                             "purposes:" "[" purpose_list "]" NEWLINE
                             "behavior:" menu_behavior NEWLINE
                             ( "selection:" selection_mode NEWLINE )?
                             ( "orientation:" orientation NEWLINE )?
                             ( "position_hint:" group_position NEWLINE )?
                             DEDENT

menu_behavior   ::= "always_visible" | "on_demand" | "context_sensitive" | "hover_reveal"

selection_mode  ::= "single" | "multi" | "none"

orientation     ::= "horizontal" | "vertical"

group_position  ::= "start" | "end" | "inline"

menu_overflow   ::= "overflow:" NEWLINE INDENT
                    "strategy:" overflow_strategy NEWLINE
                    ( "priority_order:" "[" purpose_list "]" NEWLINE )?
                    ( "collapse_label:" string NEWLINE )?
                    DEDENT

overflow_strategy ::= "collapse" | "scroll" | "wrap" | "prioritize"

mobile_behavior ::= "mobile_behavior:" NEWLINE INDENT
                    "strategy:" mobile_strategy NEWLINE
                    ( "drawer_position:" drawer_position NEWLINE )?
                    ( "purposes_in_bottom:" "[" purpose_list "]" NEWLINE )?
                    DEDENT

mobile_strategy ::= "drawer" | "bottom_nav" | "hamburger"

drawer_position ::= "start" | "end"
```

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial proposal |

---

## ADDENDUM AI: ACCESSIBILITY AND USER PREFERENCES (NEW in v1.5)

**Purpose**: Define semantic accessibility roles, labels, live region behavior, keyboard navigation intent, user preference schemas, and their effect on UI rendering.  
**Extends**: `dataType.fields`, `presentationView`, `presentationComposition`, `experience`  
**Priority**: HIGH

---

## Problem Statement

The current specification and proposed addendums have:
- A single mention of accessibility as a SHOULD in `10-presentation-hints.md`
- A reference to `preferences` as a session field in `04-session-contract.md`

However, there is no grammar for expressing:
1. Semantic accessibility roles and labels
2. Live region behavior for dynamic content
3. Keyboard navigation intent
4. User preference schemas and their effect on UI
5. Motion, contrast, and theme preferences

**Impact**: Implementations treat accessibility as an afterthought rather than a first-class specification concern.

---

## Design Principle Alignment

### Intent-First

This addendum expresses **what** accessibility and preferences mean, not **how** to implement:

| Intent-Level (This Spec) | Runtime Decides |
|--------------------------|-----------------|
| `accessibility.role: status` | `role="status"`, ARIA attributes |
| `live_region: polite` | `aria-live="polite"`, screen reader timing |
| `preferences.theme: dark` | CSS variables, theme classes |
| `motion: reduced` | `prefers-reduced-motion` handling |

### Transport-Agnostic

The grammar does not specify:
- Specific ARIA attributes
- Platform-specific accessibility APIs
- CSS implementation of preferences
- Assistive technology behavior

---

## Intent-Level Grammar

### 1. Field-Level Accessibility

Add `accessibility` block to `dataType.fields` for semantic accessibility:

```yaml
dataType:
  name: <string>
  version: <vN>
  fields:
    <field_name>:
      type: <type>
      
      accessibility:                  # NEW: Accessibility semantics
        role: <semantic_role>         # Semantic role for assistive tech
        
        label:                        # How to describe this field
          static: <string>            # Static label
          template: <string_template> # Dynamic label with {{field}} interpolation
          
        description: <string>         # Extended description for screen readers
        
        announce_changes: true | false  # Announce when value changes
        announcement_template: <string> # What to announce on change
        
        hidden_from_assistive: true | false  # Hide from screen readers (decorative)
```

#### Semantic Roles

| Role | Description | Typical Use |
|------|-------------|-------------|
| `text` | Plain text content | Descriptions, names |
| `heading` | Section heading | Titles, headers |
| `status` | Status information | Account status, alerts |
| `alert` | Important message | Error messages, warnings |
| `progress` | Progress indicator | Loading, completion |
| `meter` | Bounded value | Scores, ratings |
| `timer` | Time-based value | Countdowns, durations |
| `log` | Logged information | Activity history |
| `marquee` | Changing content | Tickers, live feeds |
| `img` | Image content | Avatars, icons |
| `button` | Interactive button | Actions |
| `link` | Navigation link | Links to other content |
| `listitem` | Item in a list | List entries |

---

### 2. View-Level Accessibility

Add `accessibility` block to `presentationView` for view-level semantics:

```yaml
presentationView:
  name: <string>
  version: <vN>
  
  accessibility:                      # NEW: View-level accessibility
    landmark: <landmark_type>         # What this view represents
    label: <string>                   # Accessible name for the view
    description: <string>             # Extended description
    
    heading:
      level: 1 | 2 | 3 | 4 | 5 | 6   # Semantic heading level
      text: <string>                  # Heading text (or use view name)
    
    live_region:                      # Dynamic content behavior
      type: polite | assertive | off
      atomic: true | false            # Announce entire region or just changes
      relevant: additions | removals | text | all
    
    skip_link:                        # For skip navigation
      label: <string>                 # "Skip to {{label}}"
      priority: <integer>             # Order in skip links
    
    keyboard:                         # Keyboard navigation intent
      focusable: true | false
      tab_order: <integer>            # Explicit tab order (use sparingly)
      shortcuts:                      # Keyboard shortcuts
        - key: <key_combination>
          action: <action_ref>
          description: <string>
```

#### Landmark Types

| Landmark | Description | Typical Use |
|----------|-------------|-------------|
| `main` | Main content | Primary view content |
| `navigation` | Navigation section | Navigation menus |
| `complementary` | Complementary content | Sidebars, related info |
| `banner` | Header content | Page/app header |
| `contentinfo` | Footer content | Copyright, links |
| `search` | Search functionality | Search forms |
| `form` | Form region | Input forms |
| `region` | Generic region | Named sections |
| `alert` | Alert content | Notifications, errors |

---

### 3. Composition-Level Accessibility

Add `accessibility` block to `presentationComposition`:

```yaml
presentationComposition:
  name: <string>
  
  accessibility:                      # NEW: Composition-level a11y
    announce_on_enter:                # Announce when composition loads
      enabled: true | false
      template: <string>              # "Now viewing {{composition.label}}"
    
    focus_on_enter:                   # Where to focus on load
      target: first_heading | first_interactive | specific
      specific_field: <field_ref>     # For specific target
    
    keyboard_navigation:              # Keyboard nav within composition
      arrow_navigation: true | false  # Arrow keys navigate items
      wrap: true | false              # Wrap at start/end
      home_end: true | false          # Home/End jump to start/end
    
    reduce_motion:                    # Motion reduction behavior
      respect_preference: true        # Honor user preference
      fallback: instant | fade        # What to use when reduced
```

---

### 4. User Preferences Schema

Extend `experience` with a formal preferences schema:

```yaml
experience:
  name: <string>
  
  preferences:                        # NEW: User preference schema
    schema:
      theme:
        type: enum
        values: [light, dark, system, high_contrast]
        default: system
        affects: [color_scheme, contrast]
      
      color_scheme:
        type: enum
        values: [default, protanopia, deuteranopia, tritanopia]
        default: default
        affects: [semantic_colors]
      
      density:
        type: enum
        values: [compact, comfortable, spacious]
        default: comfortable
        affects: [spacing, font_size]
      
      motion:
        type: enum
        values: [full, reduced, none]
        default: full
        affects: [animations, transitions]
      
      font_size:
        type: enum
        values: [small, medium, large, x_large]
        default: medium
        affects: [typography]
      
      language:
        type: string
        default: "en-US"
        affects: [localization, text_direction]
      
      timezone:
        type: string
        default: "UTC"
        affects: [datetime_display]
      
      date_format:
        type: enum
        values: [iso, us, eu, relative]
        default: relative
        affects: [date_display]
      
      number_format:
        type: enum
        values: [us, eu, indian, system]
        default: system
        affects: [number_display, currency_display]
    
    storage:                          # How preferences are persisted
      scope: session | persistent | both
      sync_across_devices: true | false
    
    system_detection:                 # Auto-detect from OS/browser
      enabled: true | false
      detectable: [theme, motion, language, timezone]
```

---

### 5. Preference Effects

Define how preferences affect UI:

```yaml
experience:
  preferences:
    effects:                          # NEW: How preferences affect rendering
      theme:
        light:
          applies:
            - color_scheme: light
            - background: light
        dark:
          applies:
            - color_scheme: dark
            - background: dark
        high_contrast:
          applies:
            - color_scheme: high_contrast
            - border_emphasis: increased
            - focus_indicator: enhanced
      
      motion:
        reduced:
          applies:
            - transitions: instant
            - animations: disabled
            - auto_play: disabled
        none:
          applies:
            - transitions: disabled
            - animations: disabled
            - auto_play: disabled
            - parallax: disabled
      
      density:
        compact:
          applies:
            - spacing: reduced
            - font_size: smaller
            - line_height: tighter
        spacious:
          applies:
            - spacing: increased
            - font_size: larger
            - line_height: relaxed
```

---

### 6. Accessibility Announcements

Define semantic announcements for UI events:

```yaml
presentationComposition:
  accessibility:
    announcements:                    # NEW: Event-based announcements
      on_load:
        template: "{{composition.label}} loaded"
        priority: polite
        
      on_update:
        template: "Content updated"
        priority: polite
        
      on_error:
        template: "Error: {{error.message}}"
        priority: assertive
        
      on_action_complete:
        template: "{{action.label}} completed successfully"
        priority: polite
```

---

## Semantic Guarantees

When accessibility semantics are declared:

1. **Role Consistency**: Declared `role` MUST map to assistive technology roles
2. **Label Availability**: Labels MUST be available to assistive technology
3. **Live Region Behavior**: `live_region` MUST announce changes appropriately
4. **Preference Respect**: User preferences MUST affect rendering
5. **Focus Management**: Focus MUST be managed according to declaration

---

## Runtime Responsibilities

The runtime MUST:

1. **Map Roles**: Translate semantic roles to platform accessibility APIs
2. **Provide Labels**: Expose labels to assistive technology
3. **Manage Announcements**: Announce changes based on live region settings
4. **Apply Preferences**: Render according to user preferences
5. **Detect System Preferences**: Detect OS/browser preferences when `system_detection.enabled`

The runtime MAY:

1. Provide additional accessibility features beyond specification
2. Implement platform-specific accessibility optimizations
3. Cache preference effects for performance
4. Provide accessibility testing tools

---

## Examples

### Example 1: Account Balance Field with Accessibility

```yaml
dataType:
  name: Account
  version: v1
  fields:
    balance:
      type: decimal
      presentation:
        display_as: currency
        emphasis: primary
      accessibility:
        role: text
        label:
          template: "Account balance: {{value | currency}}"
        announce_changes: true
        announcement_template: "Balance updated to {{value | currency}}"
    
    status:
      type: string
      presentation:
        display_as: badge
      accessibility:
        role: status
        label:
          template: "Account status: {{value}}"
        announce_changes: true
```

### Example 2: Alert List View with Live Region

```yaml
presentationView:
  name: AlertListView
  version: v1
  consumes:
    dataState: AlertState
  
  accessibility:
    landmark: region
    label: "Account Alerts"
    description: "List of alerts and notifications for your accounts"
    
    heading:
      level: 2
      text: "Alerts"
    
    live_region:
      type: polite
      atomic: false
      relevant: additions
    
    skip_link:
      label: "Skip to alerts"
      priority: 2
    
    keyboard:
      focusable: true
      shortcuts:
        - key: "a"
          action: acknowledge_selected
          description: "Acknowledge selected alert"
        - key: "d"
          action: dismiss_selected
          description: "Dismiss selected alert"
```

### Example 3: Dashboard Composition with Focus Management

```yaml
presentationComposition:
  name: CustomerDashboard
  version: v1
  primary: DashboardSummaryView
  
  accessibility:
    announce_on_enter:
      enabled: true
      template: "Customer dashboard loaded. You have {{alerts_count}} alerts."
    
    focus_on_enter:
      target: first_heading
    
    keyboard_navigation:
      arrow_navigation: false         # Not a list, standard tab navigation
      wrap: false
    
    reduce_motion:
      respect_preference: true
      fallback: fade
    
    announcements:
      on_load:
        template: "Dashboard loaded with {{accounts_count}} accounts"
        priority: polite
      on_error:
        template: "Failed to load dashboard: {{error.message}}"
        priority: assertive
```

### Example 4: Full Preferences Schema

```yaml
experience:
  name: CustomerBanking
  version: v1
  
  preferences:
    schema:
      theme:
        type: enum
        values: [light, dark, system, high_contrast]
        default: system
        affects: [color_scheme]
      
      motion:
        type: enum
        values: [full, reduced, none]
        default: full
        affects: [animations]
      
      density:
        type: enum
        values: [compact, comfortable, spacious]
        default: comfortable
        affects: [spacing]
      
      font_size:
        type: enum
        values: [small, medium, large, x_large]
        default: medium
        affects: [typography]
      
      language:
        type: string
        default: "en-US"
        affects: [localization]
      
      currency_display:
        type: enum
        values: [symbol, code, name]
        default: symbol
        affects: [currency_formatting]
      
      date_format:
        type: enum
        values: [iso, us, eu, relative]
        default: relative
        affects: [date_display]
    
    storage:
      scope: persistent
      sync_across_devices: true
    
    system_detection:
      enabled: true
      detectable: [theme, motion, language, timezone]
    
    effects:
      theme:
        high_contrast:
          applies:
            - border_emphasis: increased
            - focus_indicator: enhanced
            - minimum_contrast: "7:1"
      
      motion:
        reduced:
          applies:
            - transitions: instant
            - animations: disabled
        none:
          applies:
            - transitions: disabled
            - animations: disabled
      
      density:
        compact:
          applies:
            - base_spacing: "4px"
            - base_font_size: "14px"
        spacious:
          applies:
            - base_spacing: "12px"
            - base_font_size: "18px"
```

### Example 5: Accessible Form with Error Handling

```yaml
presentationView:
  name: TransferFormView
  version: v1
  
  accessibility:
    landmark: form
    label: "Transfer Money Form"
    description: "Form to transfer money between accounts"
    
    heading:
      level: 2
      text: "Transfer Money"
    
    keyboard:
      focusable: true
      shortcuts:
        - key: "Enter"
          action: submit
          description: "Submit transfer"
        - key: "Escape"
          action: cancel
          description: "Cancel transfer"
  
  form:
    intent: InitiateTransfer
    fields:
      - name: from_account_id
        accessibility:
          label:
            static: "From account"
          description: "Select the account to transfer from"
        
      - name: to_account_id
        accessibility:
          label:
            static: "To account"
          description: "Select the account to transfer to"
        
      - name: amount
        accessibility:
          label:
            static: "Transfer amount"
          description: "Enter the amount to transfer"
    
    validation:
      error_announcements:
        priority: assertive
        template: "Validation error: {{field.label}} - {{error.message}}"
    
    submit:
      accessibility:
        label: "Submit transfer"
        description: "Transfer {{amount | currency}} from {{from_account}} to {{to_account}}"
```

---

## Validation Criteria

A conformant implementation MUST:

1. **Parse** all accessibility blocks without error
2. **Map** semantic roles to platform accessibility APIs
3. **Expose** labels to assistive technology
4. **Implement** live region announcements
5. **Apply** user preferences to rendering
6. **Respect** system preference detection

A conformant implementation SHOULD:

1. Support keyboard navigation for all interactive elements
2. Provide visible focus indicators
3. Implement skip navigation links
4. Support high contrast mode
5. Respect reduced motion preferences

---

## Integration with Existing Grammar

### Extends

- `dataType.fields` - Adds `accessibility` block
- `presentationView` - Adds `accessibility` block
- `presentationComposition` - Adds `accessibility` block
- `experience` - Adds `preferences` block

### References

- `10-presentation-hints.md` - Presentation and accessibility work together
- `01-form-intent-binding.md` - Form field accessibility
- `11-surface-container-semantics.md` - Focus management in surfaces
- `04-session-contract.md` - Session stores preferences

### Complements

- `fieldPermissions.mask` - Masked content needs accessible labels
- `navigation.indicator` - Indicators need accessible announcements

---

## EBNF Grammar Addition

```ebnf
field_accessibility ::= "accessibility:" NEWLINE INDENT
                        ( "role:" semantic_role NEWLINE )?
                        ( accessibility_label )?
                        ( "description:" string NEWLINE )?
                        ( "announce_changes:" boolean NEWLINE )?
                        ( "announcement_template:" string NEWLINE )?
                        ( "hidden_from_assistive:" boolean NEWLINE )?
                        DEDENT

semantic_role       ::= "text" | "heading" | "status" | "alert"
                      | "progress" | "meter" | "timer" | "log"
                      | "marquee" | "img" | "button" | "link" | "listitem"

accessibility_label ::= "label:" NEWLINE INDENT
                        ( "static:" string NEWLINE )?
                        ( "template:" string NEWLINE )?
                        DEDENT

view_accessibility  ::= "accessibility:" NEWLINE INDENT
                        ( "landmark:" landmark_type NEWLINE )?
                        ( "label:" string NEWLINE )?
                        ( "description:" string NEWLINE )?
                        ( accessibility_heading )?
                        ( live_region )?
                        ( skip_link )?
                        ( keyboard_config )?
                        DEDENT

landmark_type       ::= "main" | "navigation" | "complementary"
                      | "banner" | "contentinfo" | "search"
                      | "form" | "region" | "alert"

accessibility_heading ::= "heading:" NEWLINE INDENT
                          "level:" heading_level NEWLINE
                          ( "text:" string NEWLINE )?
                          DEDENT

heading_level       ::= "1" | "2" | "3" | "4" | "5" | "6"

live_region         ::= "live_region:" NEWLINE INDENT
                        "type:" live_region_type NEWLINE
                        ( "atomic:" boolean NEWLINE )?
                        ( "relevant:" live_region_relevant NEWLINE )?
                        DEDENT

live_region_type    ::= "polite" | "assertive" | "off"

live_region_relevant ::= "additions" | "removals" | "text" | "all"

skip_link           ::= "skip_link:" NEWLINE INDENT
                        "label:" string NEWLINE
                        ( "priority:" integer NEWLINE )?
                        DEDENT

keyboard_config     ::= "keyboard:" NEWLINE INDENT
                        ( "focusable:" boolean NEWLINE )?
                        ( "tab_order:" integer NEWLINE )?
                        ( keyboard_shortcuts )?
                        DEDENT

keyboard_shortcuts  ::= "shortcuts:" NEWLINE INDENT
                        shortcut_definition+
                        DEDENT

shortcut_definition ::= "-" "key:" string NEWLINE INDENT
                        "action:" action_ref NEWLINE
                        ( "description:" string NEWLINE )?
                        DEDENT

composition_accessibility ::= "accessibility:" NEWLINE INDENT
                              ( announce_on_enter )?
                              ( focus_on_enter )?
                              ( keyboard_navigation )?
                              ( reduce_motion )?
                              ( announcements )?
                              DEDENT

announce_on_enter   ::= "announce_on_enter:" NEWLINE INDENT
                        "enabled:" boolean NEWLINE
                        ( "template:" string NEWLINE )?
                        DEDENT

focus_on_enter      ::= "focus_on_enter:" NEWLINE INDENT
                        "target:" focus_target NEWLINE
                        ( "specific_field:" field_ref NEWLINE )?
                        DEDENT

focus_target        ::= "first_heading" | "first_interactive" | "specific"

keyboard_navigation ::= "keyboard_navigation:" NEWLINE INDENT
                        ( "arrow_navigation:" boolean NEWLINE )?
                        ( "wrap:" boolean NEWLINE )?
                        ( "home_end:" boolean NEWLINE )?
                        DEDENT

reduce_motion       ::= "reduce_motion:" NEWLINE INDENT
                        "respect_preference:" boolean NEWLINE
                        ( "fallback:" motion_fallback NEWLINE )?
                        DEDENT

motion_fallback     ::= "instant" | "fade"

announcements       ::= "announcements:" NEWLINE INDENT
                        announcement_definition+
                        DEDENT

announcement_definition ::= announcement_event ":" NEWLINE INDENT
                            "template:" string NEWLINE
                            ( "priority:" announcement_priority NEWLINE )?
                            DEDENT

announcement_event  ::= "on_load" | "on_update" | "on_error" | "on_action_complete"

announcement_priority ::= "polite" | "assertive"

preferences         ::= "preferences:" NEWLINE INDENT
                        preference_schema
                        ( preference_storage )?
                        ( system_detection )?
                        ( preference_effects )?
                        DEDENT

preference_schema   ::= "schema:" NEWLINE INDENT
                        preference_definition+
                        DEDENT

preference_definition ::= identifier ":" NEWLINE INDENT
                          "type:" preference_type NEWLINE
                          ( "values:" "[" value_list "]" NEWLINE )?
                          ( "default:" value NEWLINE )?
                          ( "affects:" "[" identifier_list "]" NEWLINE )?
                          DEDENT

preference_type     ::= "enum" | "string" | "boolean" | "integer"

preference_storage  ::= "storage:" NEWLINE INDENT
                        "scope:" storage_scope NEWLINE
                        ( "sync_across_devices:" boolean NEWLINE )?
                        DEDENT

storage_scope       ::= "session" | "persistent" | "both"

system_detection    ::= "system_detection:" NEWLINE INDENT
                        "enabled:" boolean NEWLINE
                        ( "detectable:" "[" identifier_list "]" NEWLINE )?
                        DEDENT

preference_effects  ::= "effects:" NEWLINE INDENT
                        effect_group+
                        DEDENT

effect_group        ::= identifier ":" NEWLINE INDENT
                        effect_value+
                        DEDENT

effect_value        ::= identifier ":" NEWLINE INDENT
                        "applies:" NEWLINE INDENT
                        effect_application+
                        DEDENT
                        DEDENT

effect_application  ::= "-" identifier ":" value NEWLINE
```

---

## ADDENDUM AJ: END-TO-END REFERENCE FLOW (NEW in v1.5)

**Purpose**: Provide a comprehensive reference flow demonstrating how all SMS components work together from UI interaction through data persistence and back to UI update. This serves as a learning aid and integration reference for implementers.  
**Type**: Informative (Non-Normative)  
**Priority**: REFERENCE

---

## Purpose

This document provides a comprehensive reference flow demonstrating how all SMS components work together from UI interaction through data persistence and back to UI update. It serves as a learning aid and integration reference for implementers.

---

## Scenario: Money Transfer Form Submission

A customer (Alice) submits a transfer form to move $100 from her checking account to her savings account. This example traces the complete path through all system layers.

---

## System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              UI LAYER (v1.4)                                     │
├─────────────────────────────────────────────────────────────────────────────────┤
│  ┌──────────┐    ┌───────────┐    ┌──────────────┐    ┌────────────────┐       │
│  │ Browser  │───▶│SMS Client │───▶│ HTTP Server  │───▶│ Experience     │       │
│  │ Client   │    │ SDK       │    │              │    │ Router         │       │
│  └──────────┘    └───────────┘    └──────────────┘    └────────────────┘       │
│       ▲                                                       │                 │
│       │                                                       ▼                 │
│       │              ┌────────────────┐    ┌──────────────────────────┐        │
│       │              │ Field          │◀───│ Composition              │        │
│       │              │ Permission     │    │ Resolver                 │        │
│       │              │ Enforcer       │    └──────────────────────────┘        │
│       │              └────────────────┘              │                          │
│       │                     │                        │                          │
│       │              ┌──────▼───────┐    ┌──────────▼──────────┐               │
│       └──────────────│ Web Renderer │◀───│ Navigation Tree     │               │
│                      └──────────────┘    │ Builder             │               │
│                                          └─────────────────────┘               │
└─────────────────────────────────────────────────────────────────────────────────┘
                                    │
                          ┌─────────▼─────────┐
                          │   NATS JetStream  │
                          └─────────┬─────────┘
                                    │
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            CONTROL PLANE (v1.3)                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│  ┌────────────────┐    ┌─────────────────┐    ┌───────────────────┐            │
│  │ Flow Executor  │───▶│ Policy Evaluator│───▶│ Authority State   │            │
│  │ (SMS Runtime)  │    │                 │    │ Store             │            │
│  └────────────────┘    └─────────────────┘    └───────────────────┘            │
│          │                                              │                        │
│          │                                              ▼                        │
│          │                                    ┌───────────────────┐             │
│          │                                    │ CAS Token Service │             │
│          │                                    └───────────────────┘             │
│          ▼                                                                       │
└─────────────────────────────────────────────────────────────────────────────────┘
                                    │
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              DATA PLANE (v1.3)                                   │
├─────────────────────────────────────────────────────────────────────────────────┤
│  ┌────────────────┐    ┌─────────────────┐    ┌───────────────────┐            │
│  │ Work Unit      │───▶│ DataState       │───▶│ Event Stream      │            │
│  │ Executor       │    │ Engine          │    │ (NATS JetStream)  │            │
│  └────────────────┘    └─────────────────┘    └───────────────────┘            │
│                                                         │                        │
│                                                         ▼                        │
│                               ┌─────────────────────────────────────────┐       │
│                               │ View Materializer                        │       │
│                               │ (authority-agnostic, epoch-tolerant)    │       │
│                               └─────────────────────────────────────────┘       │
│                                                         │                        │
│                                                         ▼                        │
│                               ┌─────────────────────────────────────────┐       │
│                               │ KV Bucket: SMS_VIEWS                     │       │
│                               │ Key: AccountBalanceView.{account_id}     │       │
│                               └─────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Phase 1: UI Form Interaction

### Step 1: User Views Transfer Form

**Component**: `Browser Client`  
**Layer**: UI Layer

The form is rendered from the `presentationView` specification:

```yaml
presentationView:
  name: TransferFormView
  version: v1
  consumes:
    dataState: AccountState
  
  form:
    intent: InitiateTransfer
    fields:
      - name: from_account_id
        type: select
        label: "From Account"
        source: "{{context.accounts}}"
        display_expression: "{{account_number}} - {{account_type}}"
        value_expression: "{{id}}"
        
      - name: to_account_id
        type: select
        label: "To Account"
        source: "{{context.accounts}}"
        exclude_expression: "{{from_account_id}}"
        
      - name: amount
        type: currency
        label: "Amount"
        validation:
          min: 0.01
          max_expression: "{{from_account.balance}}"
          message: "Amount cannot exceed available balance"
    
    submit:
      label: "Transfer"
      confirm: "Transfer {{amount | currency}} from {{from_account.account_number}}?"
      loading_label: "Processing..."
    
    on_success:
      action: navigate_to
      target: AccountDetailComposition
      params:
        account_id: "{{from_account_id}}"
      toast: "Transfer of {{amount | currency}} completed successfully"
    
    on_error:
      action: display_inline
      field_errors: true
```

**User Action**: Fills in form fields and clicks "Transfer" button.

---

### Step 2: Client-Side Validation

**Component**: `Browser Client` (JavaScript)  
**Layer**: UI Layer

Before submission, the browser validates:

| Check | Expression | Result |
|-------|------------|--------|
| Amount > 0 | `amount >= 0.01` | ✓ Pass |
| Amount ≤ balance | `amount <= from_account.balance` | ✓ Pass (100 ≤ 1000) |
| From ≠ To | `from_account_id != to_account_id` | ✓ Pass |

If validation fails, inline errors are displayed without server round-trip.

---

### Step 3: SDK Submits Intent

**Component**: `@sms/client SDK`  
**Layer**: UI Layer

```typescript
// Browser calls SDK
const result = await smsClient.submitIntent("InitiateTransfer", {
  from_account_id: "acc-checking-123",
  to_account_id: "acc-savings-456",
  amount: 100.00,
  correlation_id: "ui-txn-789"
});
```

**SDK Internal Actions**:

1. **Discover Route**: Query `SMS_ROUTES` KV for `InitiateTransfer` endpoint
2. **Select Transport**: Choose optimal transport based on client capabilities
3. **Serialize Payload**: Convert to JSON with correlation context
4. **Submit**: Send via selected transport

---

## Phase 2: Transport & Routing

### Step 4: Route Discovery

**Component**: `Route Discoverer`  
**Layer**: UI Layer

```yaml
# KV Lookup: SMS_ROUTES.routes.InitiateTransfer
RouteRegistration:
  intent: InitiateTransfer
  endpoints:
    - transport: nats-ws
      subject: "intent.banking.InitiateTransfer"
      priority: 5
    - transport: http
      endpoint: "/api/intents/InitiateTransfer"
      priority: 20
```

### Step 5: Transport Selection

**Component**: `Transport Selector`  
**Layer**: UI Layer

| Transport | Client Support | Priority | Selected |
|-----------|---------------|----------|----------|
| nats-ws | ✓ (WebSocket) | 5 | **Yes** |
| http | ✓ (fetch) | 20 | Fallback |

**Decision**: Use NATS-WS (lowest priority number = highest preference)

### Step 6: Intent Delivery

**Component**: `NATS JetStream`  
**Layer**: Messaging

```yaml
# Message Published
Subject: intent.banking.InitiateTransfer
Headers:
  Nats-Msg-Id: "ui-txn-789"  # Idempotency key
  X-Correlation-Id: "ui-txn-789"
  X-Session-Id: "session-abc"
  X-Subject-Id: "alice"
Payload:
  from_account_id: "acc-checking-123"
  to_account_id: "acc-savings-456"
  amount: 100.00
```

**Delivery Contract** (from `03-intent-delivery-contract.md`):

```yaml
inputIntent:
  name: InitiateTransfer
  delivery_contract:
    reliability: at_least_once
    ordering: fifo
    response_expectation: synchronous
    timeout: 5s
    acknowledgment_semantics: all_outcomes
```

---

## Phase 3: Flow Execution

### Step 7: Flow Executor Receives Intent

**Component**: `Flow Executor (SMS Runtime)`  
**Layer**: Control Plane

The Flow Executor subscribes to intent subjects and begins execution:

```go
// FlowEngine receives intent
func (e *FlowEngine) HandleIntent(msg *nats.Msg) {
    intent := ParseIntent(msg)
    flow := e.registry.GetFlowFor(intent.Name)
    
    instance := NewFlowInstance(flow, intent)
    result := e.Execute(instance)
    
    msg.Respond(result.Serialize())
}
```

**Flow Specification**:

```yaml
smsFlow:
  name: ProcessTransfer
  version: v1
  triggeredBy:
    inputIntent: InitiateTransfer
  
  policySet: TransferPolicies
  
  steps:
    - name: validate_input
      type: validate
      schema: transfer_format
    
    - name: verify_accounts
      type: work_unit
      unit: verify_accounts_exist
      reads: [Account]
    
    - name: acquire_authority
      type: work_unit
      unit: get_authority_token
      for: "{{input.from_account_id}}"
    
    - name: execute_transfer
      type: work_unit
      unit: execute_transfer
      writes: [Transaction]
      atomicGroup: transfer_execution
    
    - name: emit_event
      type: emit
      event: transfer_completed
  
  failure:
    policy: abort
    compensation: none
```

### Step 8: Policy Evaluation

**Component**: `Policy Evaluator`  
**Layer**: Control Plane

```yaml
policySet:
  name: TransferPolicies
  policies:
    - name: CustomerCanTransfer
      condition: |
        subject.roles contains "customer" AND
        input.from_account_id in subject.owned_accounts AND
        input.amount <= subject.daily_transfer_limit
```

**Evaluation Context**:

| Attribute | Value | Check | Result |
|-----------|-------|-------|--------|
| subject.roles | ["customer"] | contains "customer" | ✓ |
| subject.owned_accounts | ["acc-checking-123", "acc-savings-456"] | contains from_account | ✓ |
| input.amount | 100.00 | ≤ daily_limit (10000) | ✓ |

**Decision**: `ALLOW`

### Step 9: Authority Verification

**Component**: `Authority State Store` + `CAS Token Service`  
**Layer**: Control Plane

```go
// Check authority for source account
authorityState := authorityStore.Get("acc-checking-123")
// Returns:
// {
//   entity_id: "acc-checking-123",
//   authority_region: "us-east",
//   authority_epoch: 42,
//   lease_holder: null,
//   state: "STABLE"
// }

// Acquire CAS token for atomic write
casToken := casService.AcquireToken(CASRequest{
    EntityID: "acc-checking-123",
    ExpectedEpoch: 42,
    Operation: "write",
    TTL: 5 * time.Second,
})
// Returns:
// {
//   token_id: "cas-token-xyz",
//   entity_id: "acc-checking-123",
//   epoch: 42,
//   expires_at: "2026-01-13T14:30:05Z"
// }
```

---

## Phase 4: Work Unit Execution

### Step 10: Locality-First Routing

**Component**: `Work Unit Router`  
**Layer**: Data Plane

```
Locality-First Decision Tree:
┌────────────────────────────────────────┐
│ 1. Check routing cache for work unit   │
│    → Found: execute_transfer           │
├────────────────────────────────────────┤
│ 2. Local endpoint available?           │
│    → Yes: worker running in-process    │
├────────────────────────────────────────┤
│ 3. Constraints satisfied?              │
│    → Yes: region=us-east matches       │
├────────────────────────────────────────┤
│ 4. Decision: LOCAL INVOCATION          │
└────────────────────────────────────────┘
```

### Step 11: Work Unit Execution

**Component**: `Work Unit Executor (Worker Runtime)`  
**Layer**: Data Plane

```go
func ExecuteTransfer(ctx ExecutionContext, input TransferInput) (TransferOutput, error) {
    // 1. Validate CAS token is still valid
    if !casService.Validate(input.CASToken) {
        return nil, &WorkUnitError{
            Code: "STALE_TOKEN",
            Message: "Authority token expired, retry required",
            Retryable: true,
        }
    }
    
    // 2. Load current account state
    fromAccount := dataState.Get[Account](input.FromAccountID)
    toAccount := dataState.Get[Account](input.ToAccountID)
    
    // 3. Business validation
    if fromAccount.Balance < input.Amount {
        return nil, &WorkUnitError{
            Code: "INSUFFICIENT_FUNDS",
            Message: fmt.Sprintf("Available balance: %v", fromAccount.Balance),
            Retryable: false,
            FieldErrors: map[string]string{
                "amount": "Exceeds available balance",
            },
        }
    }
    
    if fromAccount.Status != "active" {
        return nil, &WorkUnitError{
            Code: "ACCOUNT_INACTIVE",
            Message: "Source account is not active",
            Retryable: false,
        }
    }
    
    // 4. Create transaction record
    transaction := Transaction{
        ID:            uuid.New().String(),
        FromAccountID: input.FromAccountID,
        ToAccountID:   input.ToAccountID,
        Amount:        input.Amount,
        Type:          "transfer",
        Status:        "completed",
        Timestamp:     time.Now(),
        CorrelationID: ctx.CorrelationID,
    }
    
    // 5. Return for persistence
    return TransferOutput{
        Transaction: transaction,
        CASToken:    input.CASToken,
    }, nil
}
```

---

## Phase 5: Data Persistence

### Step 12: DataState Engine Persists

**Component**: `DataState Engine`  
**Layer**: Data Plane

```go
func (ds *DataStateEngine) Persist(output TransferOutput) error {
    // 1. Final CAS validation before write
    currentEpoch := ds.authority.CurrentEpoch(output.Transaction.FromAccountID)
    if output.CASToken.Epoch != currentEpoch {
        return ErrEpochMismatch{
            Expected: output.CASToken.Epoch,
            Actual:   currentEpoch,
        }
    }
    
    // 2. Create immutable event
    event := Event{
        ID:        uuid.New().String(),
        EntityID:  output.Transaction.ID,
        EntityType: "Transaction",
        EventType: "TransactionCreated",
        Data:      output.Transaction,
        Metadata: EventMetadata{
            Epoch:         output.CASToken.Epoch,
            Region:        "us-east",
            Timestamp:     time.Now(),
            CorrelationID: output.Transaction.CorrelationID,
            CausedBy:      "intent.InitiateTransfer",
        },
    }
    
    // 3. Append to event stream (immutable, append-only)
    return ds.eventStream.Publish(event)
}
```

### Step 13: Event Stream Append

**Component**: `NATS JetStream (Event Streams)`  
**Layer**: Messaging

```yaml
# Stream Configuration
Stream: SMS_EVENTS
Subjects: ["events.>"]
Storage: file
Retention: limits
MaxAge: 365d
DenyDelete: true   # Append-only
DenyPurge: true
Replicas: 3

# Published Event
Subject: events.us-east.Transaction.txn-789
Sequence: 1458293
Timestamp: 2026-01-13T14:30:00.123Z
Headers:
  Nats-Msg-Id: "evt-txn-789"
  X-Epoch: "42"
  X-Region: "us-east"
  X-Correlation-Id: "ui-txn-789"
Payload:
  id: "txn-789"
  entity_type: "Transaction"
  event_type: "TransactionCreated"
  data:
    id: "txn-789"
    from_account_id: "acc-checking-123"
    to_account_id: "acc-savings-456"
    amount: 100.00
    type: "transfer"
    status: "completed"
    timestamp: "2026-01-13T14:30:00.123Z"
```

---

## Phase 6: View Materialization

### Step 14: View Materializer Consumes Event

**Component**: `View Materializer`  
**Layer**: Data Plane

The View Materializer subscribes to event streams and updates materialized views:

```go
type ViewMaterializer struct {
    sourceStream nats.JetStreamContext
    viewKV       nats.KeyValue
    regions      []string
}

// Authority-agnostic: consumes events from ALL regions
func (v *ViewMaterializer) Start() error {
    for _, region := range v.regions {
        subject := fmt.Sprintf("events.%s.Transaction.>", region)
        
        _, err := v.sourceStream.Subscribe(subject, func(msg *nats.Msg) {
            event := ParseEvent(msg)
            
            // Process regardless of source region
            v.handleEvent(event)
            
            msg.Ack()
        }, nats.DeliverAll(), nats.AckExplicit())
        
        if err != nil {
            return err
        }
    }
    return nil
}

func (v *ViewMaterializer) handleEvent(event Event) error {
    switch event.EventType {
    case "TransactionCreated":
        return v.updateAccountBalanceViews(event)
    case "TransactionReversed":
        return v.reverseAccountBalanceViews(event)
    default:
        return nil // Unknown event type, skip
    }
}
```

### Step 15: Compute Derived Views

**Component**: `View Materializer`  
**Layer**: Data Plane

```go
func (v *ViewMaterializer) updateAccountBalanceViews(event Event) error {
    tx := event.Data.(Transaction)
    
    // Update FROM account (debit)
    if err := v.updateAccountView(tx.FromAccountID, func(view *AccountBalanceView) {
        view.Balance -= tx.Amount
        view.TransactionCount++
        view.LastTransactionAt = tx.Timestamp
        view.LastTransactionType = "debit"
    }); err != nil {
        return err
    }
    
    // Update TO account (credit)
    if err := v.updateAccountView(tx.ToAccountID, func(view *AccountBalanceView) {
        view.Balance += tx.Amount
        view.TransactionCount++
        view.LastTransactionAt = tx.Timestamp
        view.LastTransactionType = "credit"
    }); err != nil {
        return err
    }
    
    return nil
}

func (v *ViewMaterializer) updateAccountView(
    accountID string,
    mutate func(*AccountBalanceView),
) error {
    key := fmt.Sprintf("AccountBalanceView.%s", accountID)
    
    // Get current view (or create new)
    entry, err := v.viewKV.Get(key)
    var view AccountBalanceView
    if err == nats.ErrKeyNotFound {
        view = AccountBalanceView{AccountID: accountID}
    } else {
        json.Unmarshal(entry.Value(), &view)
    }
    
    // Apply mutation
    mutate(&view)
    
    // Update metadata
    view.ComputedAt = time.Now()
    view.SourceEpochs = append(view.SourceEpochs, currentEpoch)
    view.Freshness = "current"
    
    // Save
    data, _ := json.Marshal(view)
    _, err = v.viewKV.Put(key, data)
    return err
}
```

### Step 16: Write to View KV Bucket

**Component**: `NATS KV (SMS_VIEWS)`  
**Layer**: Messaging

```yaml
# KV Bucket Configuration
Bucket: SMS_VIEWS
Storage: file
Replicas: 3
TTL: 0  # No expiry
MaxValueSize: 1MB

# Updated Entry: FROM Account
Key: AccountBalanceView.acc-checking-123
Revision: 847
Value:
  account_id: "acc-checking-123"
  account_number: "****4567"
  account_type: "checking"
  balance: 900.00           # Was 1000.00, now 900.00
  available_balance: 900.00
  transaction_count: 156
  last_transaction_at: "2026-01-13T14:30:00.123Z"
  last_transaction_type: "debit"
  computed_at: "2026-01-13T14:30:00.234Z"
  source_epochs: [42]
  freshness: "current"

# Updated Entry: TO Account
Key: AccountBalanceView.acc-savings-456
Revision: 523
Value:
  account_id: "acc-savings-456"
  account_number: "****7890"
  account_type: "savings"
  balance: 5100.00          # Was 5000.00, now 5100.00
  available_balance: 5100.00
  transaction_count: 89
  last_transaction_at: "2026-01-13T14:30:00.123Z"
  last_transaction_type: "credit"
  computed_at: "2026-01-13T14:30:00.234Z"
  source_epochs: [42]
  freshness: "current"
```

---

## Phase 7: Response to Client

### Step 17: Flow Returns Result

**Component**: `Flow Executor`  
**Layer**: Control Plane

```go
// All steps completed successfully
result := FlowResult{
    Success: true,
    Data: map[string]interface{}{
        "transaction_id": "txn-789",
        "from_account":   "acc-checking-123",
        "to_account":     "acc-savings-456",
        "amount":         100.00,
        "status":         "completed",
    },
    CorrelationID: input.CorrelationID,
}

// Respond to original request
msg.Respond(result.Serialize())
```

### Step 18: Transport Returns Response

**Component**: `NATS-WS`  
**Layer**: Messaging

```yaml
# Reply Message
Subject: _INBOX.reply-abc123
Payload:
  success: true
  data:
    transaction_id: "txn-789"
    from_account: "acc-checking-123"
    to_account: "acc-savings-456"
    amount: 100.00
    status: "completed"
  correlation_id: "ui-txn-789"
```

### Step 19: SDK Processes Response

**Component**: `@sms/client SDK`  
**Layer**: UI Layer

```typescript
// SDK receives and parses response
const response: IntentResult<TransferResult> = {
  success: true,
  data: {
    transactionId: "txn-789",
    fromAccount: "acc-checking-123",
    toAccount: "acc-savings-456",
    amount: 100.00,
    status: "completed"
  },
  correlationId: "ui-txn-789"
};

// Return to caller
return response;
```

---

## Phase 8: UI Update

### Step 20: Handle Success Action

**Component**: `Browser Client`  
**Layer**: UI Layer

Based on the form's `on_success` specification:

```typescript
// Form success handler (generated from spec)
async function handleTransferSuccess(result: IntentResult<TransferResult>) {
  // 1. Show toast notification
  showToast({
    type: "success",
    message: `Transfer of ${formatCurrency(formData.amount)} completed successfully`
  });
  
  // 2. Navigate to account detail
  router.navigate("/compositions/AccountDetailComposition", {
    params: { account_id: formData.from_account_id }
  });
}
```

### Step 21: Composition Resolution

**Component**: `Experience Router` → `Composition Resolver`  
**Layer**: UI Layer

```
HTTP Request: GET /experience/CustomerBanking/AccountDetailComposition?account_id=acc-checking-123
Cookie: session_id=session-abc

Experience Router:
  1. GetSession("session-abc") → Session(subject=alice, roles=[customer])
  2. EvaluatePolicy(CustomerAccess, alice) → ALLOW
  3. SelectComposition(AccountDetailComposition, params)

Composition Resolver:
  1. LoadComposition("AccountDetailComposition")
  2. ResolvePrimaryView("AccountBalanceView", params)
  3. ResolveRelatedViews([TransactionListView, AlertListView])
```

### Step 22: Fetch Materialized Views

**Component**: `Composition Resolver` → `NATS KV`  
**Layer**: UI Layer

```go
func (r *CompositionResolver) Resolve(
    composition string,
    params map[string]string,
    session Session,
) (*ViewTree, error) {
    comp := r.registry.Get(composition)
    
    // Fetch primary view (already updated by materializer!)
    primaryKey := fmt.Sprintf("%s.%s", comp.Primary, params["account_id"])
    primaryData, _ := r.viewKV.Get(primaryKey)
    // → Returns balance = 900.00 (the NEW balance)
    
    // Fetch related views
    var related []RelatedView
    for _, rel := range comp.Related {
        relData := r.fetchRelated(rel, params, primaryData)
        related = append(related, relData)
    }
    
    return &ViewTree{
        Primary: primaryData,
        Related: related,
    }, nil
}
```

### Step 23: Apply Field Permissions

**Component**: `Field Permission Enforcer`  
**Layer**: UI Layer

```yaml
# Field permissions from spec
fieldPermissions:
  - field: account_number
    visible: true
    mask:
      type: partial
      reveal: last4
  
  - field: balance
    visible: true
    # No mask - show full value
  
  - field: ssn
    visible:
      policy: CanViewSSN
    mask:
      unless: "roles contains 'compliance_officer'"
      type: full
```

**Applied Masking**:

| Field | Original | Masked | Reason |
|-------|----------|--------|--------|
| account_number | "1234567890" | "****7890" | partial reveal last4 |
| balance | 900.00 | 900.00 | no mask |
| transaction_count | 156 | 156 | no mask |

### Step 24: Build Navigation Tree

**Component**: `Navigation Tree Builder` → `Indicator Evaluator`  
**Layer**: UI Layer

```go
func (b *TreeBuilder) BuildTree(experience string, session Session) *NavigationTree {
    // Build hierarchy from parent relationships
    tree := b.buildHierarchy(experience)
    
    // Evaluate indicators
    for _, node := range tree.AllNodes() {
        if node.Indicator != nil {
            state := b.indicatorEval.Evaluate(node.Indicator, session)
            node.IndicatorState = state
        }
    }
    
    // Apply visibility based on policies
    b.applyVisibility(tree, session)
    
    return tree
}
```

**Navigation with Indicators**:

```yaml
NavigationTree:
  - label: "Dashboard"
    purpose: home
    active: false
    
  - label: "My Accounts"
    purpose: accounts
    active: true  # Current section
    children:
      - label: "Checking ****4567"
        active: true  # Current item
        indicator:
          value: 156
          severity: info
          label: "transactions"
      - label: "Savings ****7890"
        active: false
        
  - label: "Transfers"
    purpose: transactions
    indicator:
      value: 1
      severity: info
      label: "pending"
```

### Step 25: Render Response

**Component**: `Web Renderer`  
**Layer**: UI Layer

```go
func (r *WebRenderer) Render(viewTree *ViewTree, nav *NavigationTree) ([]byte, error) {
    data := TemplateData{
        Composition: viewTree.CompositionName,
        Primary: viewTree.Primary,
        Related: viewTree.Related,
        Navigation: nav,
        Breadcrumbs: r.buildBreadcrumbs(viewTree),
    }
    
    return r.template.Execute("composition.html", data)
}
```

### Step 26: Browser Displays Updated UI

**Component**: `Browser Client`  
**Layer**: UI Layer

```html
<!-- Rendered HTML shows UPDATED balance -->
<main class="composition account-detail">
  <nav class="breadcrumbs">
    <a href="/dashboard">Dashboard</a> / 
    <a href="/accounts">My Accounts</a> / 
    <span>Checking ****4567</span>
  </nav>
  
  <section class="primary-view account-balance">
    <header>
      <h1>Checking Account</h1>
      <span class="account-number">****4567</span>
    </header>
    
    <div class="balance-display">
      <span class="label">Current Balance</span>
      <span class="amount currency emphasis-primary">$900.00</span>
    </div>
    
    <div class="stats">
      <span class="transaction-count">156 transactions</span>
      <span class="last-activity">Last activity: Just now</span>
    </div>
  </section>
  
  <aside class="related-views">
    <section class="transaction-list">
      <h2>Recent Transactions</h2>
      <ul>
        <li class="transaction debit">
          <span class="description">Transfer to Savings ****7890</span>
          <span class="amount negative">-$100.00</span>
          <span class="timestamp">Just now</span>
        </li>
        <!-- More transactions... -->
      </ul>
    </section>
  </aside>
</main>
```

---

## Real-Time Update Alternative (WebSocket)

If the user remains on a page that subscribes to updates, the **Indicator System** provides real-time notification:

### Step A: View Materializer Publishes Update Signal

**Component**: `View Materializer` → `NATS Pub/Sub`

```go
// After updating view, publish signal
func (v *ViewMaterializer) publishUpdateSignal(accountID string) {
    signal := ViewUpdateSignal{
        ViewName:  "AccountBalanceView",
        EntityID:  accountID,
        Timestamp: time.Now(),
    }
    
    subject := fmt.Sprintf("SMS.UI.VIEW.%s.%s", "AccountBalanceView", accountID)
    v.nats.Publish(subject, signal.Serialize())
}
```

### Step B: Indicator Evaluator Receives Signal

**Component**: `Indicator Evaluator`

```go
func (e *IndicatorEvaluator) handleViewUpdate(signal ViewUpdateSignal) {
    // Find compositions that display this view
    compositions := e.registry.CompositionsUsing(signal.ViewName)
    
    for _, comp := range compositions {
        // Re-evaluate indicator
        state := e.evaluate(comp.Navigation.Indicator)
        
        // Publish to subscribers
        subject := fmt.Sprintf("SMS.UI.INDICATOR.%s.%s", comp.Experience, comp.Name)
        e.nats.Publish(subject, IndicatorUpdate{
            Composition: comp.Name,
            State:       state,
        })
    }
}
```

### Step C: SMS Client SDK Receives Update

**Component**: `@sms/client SDK` (Browser)

```typescript
// Earlier subscription
smsClient.subscribeIndicators("CustomerBanking", (update) => {
  console.log("Indicator update:", update);
  // { composition: "AccountDetailComposition", state: { value: 156, severity: "info" } }
  
  // Update UI without full page reload
  updateIndicatorBadge(update.composition, update.state);
});
```

### Step D: Browser Updates UI In-Place

**Component**: `Browser Client`

```typescript
function updateIndicatorBadge(composition: string, state: IndicatorState) {
  const navItem = document.querySelector(`[data-composition="${composition}"] .indicator`);
  if (navItem) {
    navItem.textContent = state.value.toString();
    navItem.className = `indicator severity-${state.severity}`;
  }
}
```

---

## Complete Component Path Summary

| Step | Component | Layer | Responsibility |
|------|-----------|-------|----------------|
| 1 | Browser Client | UI | Display form from spec |
| 2 | Browser Client | UI | Client-side validation |
| 3 | @sms/client SDK | UI | Prepare and submit intent |
| 4 | Route Discoverer | UI | Find intent endpoint |
| 5 | Transport Selector | UI | Choose optimal transport |
| 6 | NATS JetStream | Messaging | Deliver intent message |
| 7 | Flow Executor | Control | Begin flow execution |
| 8 | Policy Evaluator | Control | Check authorization |
| 9 | Authority State Store | Control | Verify entity authority |
| 10 | CAS Token Service | Control | Acquire write token |
| 11 | Work Unit Router | Data | Route to worker |
| 12 | Work Unit Executor | Data | Execute business logic |
| 13 | DataState Engine | Data | Validate and persist |
| 14 | NATS JetStream (Streams) | Messaging | Store event durably |
| 15 | View Materializer | Data | Consume event |
| 16 | View Materializer | Data | Compute derived state |
| 17 | NATS KV (SMS_VIEWS) | Messaging | Write materialized view |
| 18 | Flow Executor | Control | Return success result |
| 19 | NATS-WS | Messaging | Transport response |
| 20 | @sms/client SDK | UI | Parse response |
| 21 | Browser Client | UI | Execute on_success action |
| 22 | Experience Router | UI | Route to composition |
| 23 | Composition Resolver | UI | Resolve views |
| 24 | Field Permission Enforcer | UI | Apply masking |
| 25 | Navigation Tree Builder | UI | Build nav + indicators |
| 26 | Web Renderer | UI | Render HTML |
| 27 | Browser Client | UI | Display updated UI |

---

## Key Invariants Maintained

Throughout this flow, the following SMS invariants are maintained:

| Invariant | How Maintained |
|-----------|----------------|
| **Single Authority** | CAS token validates epoch before write |
| **Append-Only Events** | Stream configured with DenyDelete, DenyPurge |
| **Authority-Agnostic Views** | Materializer consumes from all regions |
| **Epoch Ordering** | Events processed in epoch order, not arrival |
| **Policy Enforcement** | Checked at flow entry, work unit, and field level |
| **Idempotency** | Nats-Msg-Id prevents duplicate processing |
| **Eventual Consistency** | Views update asynchronously but converge |

---

## Error Handling Paths

### Validation Error (Step 11)

```
Work Unit returns: InsufficientFunds error
  → Flow Executor: Returns failure result
  → SDK: Receives error response
  → Browser: Displays inline error on amount field
  → User: Sees "Amount exceeds available balance"
```

### Authority Error (Step 9)

```
CAS Token acquisition fails: WrongRegion
  → Flow Executor: Returns redirect error
  → SDK: Receives error with redirect_to
  → Browser: Automatically retries to correct region
  → Flow continues from correct authority
```

### Timeout (Step 6)

```
NATS request times out after 5s
  → SDK: Catches timeout error
  → Browser: Shows "Request timed out, please try again"
  → User: Can retry submission
```

---

## Related Addendums

| Addendum | Relevance to This Flow |
|----------|----------------------|
| 01-form-intent-binding.md | Form specification, success/error handling |
| 02-view-materialization-contract.md | View storage contract |
| 03-intent-delivery-contract.md | Delivery guarantees, timeout |
| 04-session-contract.md | Session for subject binding |
| 06-composition-fetch-semantics.md | How views are fetched |
| 07-mask-expression-grammar.md | Field masking |
| 10-presentation-hints.md | Display formatting |
| 11-surface-container-semantics.md | UI structure |
| 12-accessibility-preferences.md | Accessible rendering |

---

## ADDENDUM AK: ASSET SEMANTICS (NEW in v1.5)

**Purpose**: Define binary or file asset fields, size constraints, mime type restrictions, asset lifecycle, reference semantics, and progressive loading hints for images, documents, and media handling.  
**Extends**: `dataType.fields`, `inputIntent`, `presentationView`  
**Priority**: HIGH

---

## Problem Statement

The current specification defines:
- `dataType.fields` with scalar types (string, decimal, boolean, timestamp, etc.)
- `materialization` for derived state
- `fieldPermissions` for visibility and masking

However, there is no grammar for expressing:
1. Binary or file asset fields (images, documents, media)
2. Asset size constraints and mime type restrictions
3. Asset lifecycle (temporary uploads, permanent storage, expiration)
4. Asset reference semantics (by-reference vs by-value)
5. Progressive loading hints for large assets

**Impact**: Applications requiring file uploads, document management, or media handling must implement these patterns entirely in runtime code without specification guidance.

---

## Design Principle Alignment

### Intent-First

This addendum expresses **what** asset semantics mean, not **how** to store or serve assets:

| Intent-Level (This Spec) | Runtime Decides |
|--------------------------|-----------------|
| `asset.type: image` | S3, GCS, Azure Blob, local filesystem |
| `max_size: 10MB` | Upload validation, chunking strategy |
| `mime_types: [image/png, image/jpeg]` | Content-type validation, magic byte checking |
| `lifecycle: permanent` | Storage tier, replication strategy |
| `access: signed_url` | CloudFront signed URLs, GCS signed URLs, SAS tokens |

### Transport-Agnostic

The grammar does not specify:
- Cloud storage providers or APIs
- CDN configuration
- Upload protocols (multipart, resumable, chunked)
- Thumbnail generation pipelines
- Virus scanning implementations

### Entity-Scoped

Assets are bound to entities and follow entity authority:
- Asset writes require entity write authority
- Asset reads respect field permissions
- Asset lifecycle follows entity lifecycle

---

## Intent-Level Grammar

### Asset Field Type

Add `asset` as a semantic field type in `dataType.fields`:

```yaml
dataType:
  name: <string>
  version: <vN>
  fields:
    <field_name>:
      type: asset                        # NEW: Asset field type
      
      asset:                             # Asset semantics
        category: image | document | media | archive | generic
        
        constraints:
          max_size: <size_expression>    # e.g., "10MB", "100KB"
          mime_types: [<mime_type>, ...] # Allowed content types
          extensions: [<extension>, ...] # Allowed file extensions (optional)
          
        lifecycle:
          type: permanent | temporary | expiring
          expires_after: <duration>      # For expiring type
          
        reference:
          style: by_reference | embedded # How asset is referenced
          
        access:
          read: public | signed | authenticated | policy
          policy: <policy_ref>           # For policy-based access
          signed_url_ttl: <duration>     # For signed access
          
        variants:                        # Generated variants (optional)
          - name: <string>
            transform: <transform_type>
            params: { ... }
```

### Asset Categories

| Category | Description | Typical Use |
|----------|-------------|-------------|
| `image` | Image files (raster, vector) | Avatars, photos, diagrams |
| `document` | Document files | PDFs, Word docs, spreadsheets |
| `media` | Audio/video files | Podcasts, videos, recordings |
| `archive` | Compressed archives | ZIP, TAR, backups |
| `generic` | Any binary content | Arbitrary files |

### Lifecycle Types

| Type | Description | Behavior |
|------|-------------|----------|
| `permanent` | Stored indefinitely | Follows entity lifecycle |
| `temporary` | Short-lived staging | Auto-deleted after upload completes or fails |
| `expiring` | Time-bounded | Auto-deleted after `expires_after` duration |

### Access Modes

| Mode | Description | Typical Rendering |
|------|-------------|-------------------|
| `public` | Publicly accessible URL | Direct CDN URL |
| `signed` | Time-limited signed URL | Signed URL with expiration |
| `authenticated` | Requires authentication | URL with session token |
| `policy` | Policy-evaluated access | URL generation guarded by policy |

### Transform Types (for Variants)

| Transform | Description | Example Params |
|-----------|-------------|----------------|
| `resize` | Resize image | `{ width: 200, height: 200, fit: cover }` |
| `thumbnail` | Generate thumbnail | `{ size: 128 }` |
| `compress` | Compress file | `{ quality: 80 }` |
| `convert` | Convert format | `{ target_format: "webp" }` |
| `extract_page` | Extract PDF page | `{ page: 1, format: "png" }` |

---

## Input Intent Integration

Extend `inputIntent` for asset upload semantics:

```yaml
inputIntent:
  name: <string>
  version: <vN>
  supplies:
    required:
      - <field_name>                    # Can reference asset fields
    optional:
      - <field_name>
  
  asset_upload:                          # NEW: Asset upload semantics
    <field_name>:
      staging: required | optional       # Whether upload must complete before submission
      resume: enabled | disabled         # Resumable upload support
      progress: report | silent          # Progress reporting
      validation:
        timing: immediate | deferred     # When to validate
        on_failure: reject | warn        # What to do on validation failure
```

---

## Presentation View Integration

Extend `presentationView` for asset display:

```yaml
presentationView:
  name: <string>
  version: <vN>
  
  fields:
    <field_name>:
      presentation:
        display_as: asset               # NEW: Asset display type
        asset:
          display: preview | link | download | embed
          variant: <variant_name>       # Which variant to display
          fallback: icon | placeholder | hide
          
          # For images
          image:
            fit: cover | contain | fill | none
            aspect_ratio: <ratio>       # e.g., "16:9", "1:1"
            lazy_load: true | false
            
          # For documents
          document:
            preview_pages: <integer>    # Pages to preview
            show_metadata: true | false
            
          # For media
          media:
            autoplay: false | true
            controls: visible | hidden | minimal
            poster: <variant_name>      # Poster image variant
```

---

## Semantic Guarantees

When a field declares `type: asset`:

1. **Size Enforcement**: Runtime MUST reject uploads exceeding `max_size`
2. **Type Validation**: Runtime MUST validate mime types against `mime_types` list
3. **Lifecycle Compliance**: Runtime MUST implement declared lifecycle behavior
4. **Access Enforcement**: Runtime MUST enforce access mode before serving
5. **Variant Availability**: If variants declared, runtime MUST generate before access
6. **Authority Binding**: Asset writes MUST respect entity authority

---

## Runtime Responsibilities

The runtime MUST:

1. **Validate Constraints**: Check size and mime type before accepting
2. **Manage Lifecycle**: Implement temporary staging, permanent storage, expiration
3. **Enforce Access**: Generate appropriate URLs based on access mode
4. **Handle Variants**: Generate declared variants on upload or on-demand
5. **Track References**: Maintain asset-to-entity relationships

The runtime MAY:

1. Choose any storage backend (S3, GCS, Azure Blob, local)
2. Implement chunked or resumable uploads
3. Use CDN for asset delivery
4. Apply virus scanning or content analysis
5. Optimize storage (deduplication, compression)

---

## Examples

### Example 1: User Avatar

```yaml
dataType:
  name: UserProfile
  version: v1
  fields:
    id:
      type: uuid
    name:
      type: string
    avatar:
      type: asset
      asset:
        category: image
        constraints:
          max_size: 5MB
          mime_types: [image/png, image/jpeg, image/gif, image/webp]
        lifecycle:
          type: permanent
        reference:
          style: by_reference
        access:
          read: public
        variants:
          - name: thumbnail
            transform: resize
            params:
              width: 128
              height: 128
              fit: cover
          - name: small
            transform: resize
            params:
              width: 256
              height: 256
              fit: cover
```

### Example 2: Document Attachment

```yaml
dataType:
  name: SupportTicket
  version: v1
  fields:
    id:
      type: uuid
    description:
      type: string
    attachment:
      type: asset
      asset:
        category: document
        constraints:
          max_size: 25MB
          mime_types:
            - application/pdf
            - application/msword
            - application/vnd.openxmlformats-officedocument.wordprocessingml.document
            - image/png
            - image/jpeg
        lifecycle:
          type: permanent
        reference:
          style: by_reference
        access:
          read: authenticated
          policy: CanViewTicketAttachments
        variants:
          - name: preview
            transform: extract_page
            params:
              page: 1
              format: png
              width: 600
```

### Example 3: Temporary Upload

```yaml
dataType:
  name: ImportBatch
  version: v1
  fields:
    id:
      type: uuid
    source_file:
      type: asset
      asset:
        category: generic
        constraints:
          max_size: 100MB
          mime_types:
            - text/csv
            - application/json
            - application/vnd.ms-excel
        lifecycle:
          type: expiring
          expires_after: 24h
        reference:
          style: by_reference
        access:
          read: authenticated
```

### Example 4: Media with Streaming

```yaml
dataType:
  name: TrainingVideo
  version: v1
  fields:
    id:
      type: uuid
    title:
      type: string
    video:
      type: asset
      asset:
        category: media
        constraints:
          max_size: 500MB
          mime_types:
            - video/mp4
            - video/webm
        lifecycle:
          type: permanent
        reference:
          style: by_reference
        access:
          read: signed
          signed_url_ttl: 4h
        variants:
          - name: poster
            transform: extract_page
            params:
              time: 5s
              format: jpeg
              width: 1280
          - name: preview
            transform: compress
            params:
              quality: 50
              max_duration: 30s
```

---

## Presentation View Example

```yaml
presentationView:
  name: UserProfileView
  version: v1
  consumes:
    dataState: UserProfileState
  
  fields:
    avatar:
      presentation:
        display_as: asset
        asset:
          display: preview
          variant: thumbnail
          fallback: icon
          image:
            fit: cover
            aspect_ratio: "1:1"
            lazy_load: true

    attachment:
      presentation:
        display_as: asset
        asset:
          display: link
          fallback: icon
```

---

## Integration with Existing Grammar

### Extends

- `dataType.fields` - Adds `asset` type and configuration
- `inputIntent.supplies` - Asset fields can be supplied
- `presentationView` - Adds asset display presentation

### References

- `fieldPermissions` - Asset visibility can be policy-controlled
- `policy` - Asset access can be policy-evaluated
- `materialization` - Views can include asset references

### Complements

- `10-presentation-hints.md` - Asset display is a presentation type
- `01-form-intent-binding.md` - Forms can include asset uploads
- `11-surface-container-semantics.md` - Asset preview in modals/drawers

---

## Validation Criteria

A conformant implementation MUST:

1. **Parse** asset type and configuration without error
2. **Enforce** size and mime type constraints
3. **Implement** lifecycle behaviors (permanent, temporary, expiring)
4. **Generate** declared variants (or on-demand)
5. **Enforce** access modes correctly

A conformant implementation SHOULD:

1. Support resumable uploads for large files
2. Provide progress reporting during upload
3. Implement efficient variant generation
4. Support CDN integration for public/signed access

---

## EBNF Grammar Addition

```ebnf
asset_field     ::= "type:" "asset" NEWLINE
                    "asset:" NEWLINE INDENT
                    "category:" asset_category NEWLINE
                    asset_constraints?
                    asset_lifecycle?
                    asset_reference?
                    asset_access?
                    asset_variants?
                    DEDENT

asset_category  ::= "image" | "document" | "media" | "archive" | "generic"

asset_constraints ::= "constraints:" NEWLINE INDENT
                      ( "max_size:" size_expression NEWLINE )?
                      ( "mime_types:" "[" mime_type_list "]" NEWLINE )?
                      ( "extensions:" "[" extension_list "]" NEWLINE )?
                      DEDENT

size_expression ::= number size_unit
size_unit       ::= "B" | "KB" | "MB" | "GB"

asset_lifecycle ::= "lifecycle:" NEWLINE INDENT
                    "type:" lifecycle_type NEWLINE
                    ( "expires_after:" duration NEWLINE )?
                    DEDENT

lifecycle_type  ::= "permanent" | "temporary" | "expiring"

asset_reference ::= "reference:" NEWLINE INDENT
                    "style:" reference_style NEWLINE
                    DEDENT

reference_style ::= "by_reference" | "embedded"

asset_access    ::= "access:" NEWLINE INDENT
                    "read:" access_mode NEWLINE
                    ( "policy:" policy_ref NEWLINE )?
                    ( "signed_url_ttl:" duration NEWLINE )?
                    DEDENT

access_mode     ::= "public" | "signed" | "authenticated" | "policy"

asset_variants  ::= "variants:" NEWLINE INDENT
                    variant_definition+
                    DEDENT

variant_definition ::= "-" "name:" identifier NEWLINE INDENT
                       "transform:" transform_type NEWLINE
                       ( "params:" params_map NEWLINE )?
                       DEDENT

transform_type  ::= "resize" | "thumbnail" | "compress" | "convert" | "extract_page"
```

---

## ADDENDUM AL: SEARCH SEMANTICS (NEW in v1.5)

**Purpose**: Define searchable field annotations, search index characteristics, faceting and aggregation semantics, relevance and ranking hints, and search result presentation.  
**Extends**: `dataType.fields`, `materialization`, `presentationView`  
**Priority**: HIGH

---

## Problem Statement

The current specification defines:
- `dataType.fields` with various scalar and complex types
- `materialization` for derived views
- `presentationView` with filters in `interactionModel`

However, there is no grammar for expressing:
1. Which fields are searchable and how
2. Search index characteristics (full-text, exact, fuzzy)
3. Faceting and aggregation semantics
4. Relevance and ranking hints
5. Search result presentation

**Impact**: Applications requiring search and discovery must implement indexing strategies, relevance tuning, and faceted navigation entirely in runtime code without specification guidance.

---

## Design Principle Alignment

### Intent-First

This addendum expresses **what** search semantics mean, not **how** to implement search:

| Intent-Level (This Spec) | Runtime Decides |
|--------------------------|-----------------|
| `searchable: full_text` | Elasticsearch, Algolia, Postgres FTS, Typesense |
| `weight: 2.0` | Boost in search query, ranking algorithm |
| `facet: enabled` | Aggregation query, facet extraction |
| `suggest: enabled` | Autocomplete implementation |

### Transport-Agnostic

The grammar does not specify:
- Search engine or database
- Index configuration or mapping
- Query DSL or syntax
- Ranking algorithms
- Index synchronization strategy

### Materialization-Compatible

Search indexes are a form of materialized state:
- Derived from source events
- Eventually consistent with source
- Rebuildable from events
- Authority-agnostic

---

## Intent-Level Grammar

### Searchable Field Annotation

Add `search` block to `dataType.fields` for search semantics:

```yaml
dataType:
  name: <string>
  version: <vN>
  fields:
    <field_name>:
      type: <type>
      
      search:                            # NEW: Search semantics
        indexed: true | false            # Whether field is indexed
        
        strategy: full_text | exact | prefix | fuzzy | semantic
        
        # For full_text strategy
        full_text:
          analyzer: standard | language | custom
          language: <language_code>      # e.g., "en", "es", "de"
          stemming: enabled | disabled
          stop_words: enabled | disabled
          synonyms: <synonym_set_ref>    # Optional synonym mapping
          
        # For fuzzy strategy
        fuzzy:
          max_edits: 1 | 2               # Levenshtein distance
          prefix_length: <integer>       # Exact prefix before fuzzy
          
        # Relevance hints
        weight: <decimal>                # Relative importance (default 1.0)
        boost_recent: true | false       # Boost recently updated
        
        # Suggest/autocomplete
        suggest:
          enabled: true | false
          type: completion | phrase | term
          
        # Faceting
        facet:
          enabled: true | false
          type: terms | range | date_histogram | nested
          ranges: [<range_definition>, ...]  # For range facets
          
        # Highlighting
        highlight:
          enabled: true | false
          fragment_size: <integer>
          pre_tag: <string>
          post_tag: <string>
```

### Search Strategies

| Strategy | Description | Use Case |
|----------|-------------|----------|
| `full_text` | Full-text search with analysis | Natural language content |
| `exact` | Exact match only | IDs, codes, enums |
| `prefix` | Prefix matching | Autocomplete, type-ahead |
| `fuzzy` | Approximate matching | Typo tolerance |
| `semantic` | Vector/embedding similarity | AI-powered search |

### Facet Types

| Type | Description | Example |
|------|-------------|---------|
| `terms` | Unique value counts | Category filter |
| `range` | Numeric ranges | Price ranges |
| `date_histogram` | Date buckets | Monthly breakdown |
| `nested` | Nested object facets | Multi-value attributes |

---

## Search Index Materialization

Extend `materialization` for search index semantics:

```yaml
materialization:
  name: <string>
  source: <DataState | list>
  targetState: <DataState>
  
  search_index:                          # NEW: Search index configuration
    enabled: true | false
    
    document:                            # How to structure search documents
      root: <entity_type>                # Root entity for documents
      include_related:                   # Related entities to denormalize
        - entity: <entity_type>
          relationship: <relationship_ref>
          fields: [<field>, ...]
          
    freshness:
      strategy: real_time | near_real_time | batch
      max_lag: <duration>                # For near_real_time
      batch_interval: <duration>         # For batch
      
    rebuild:
      trigger: manual | schema_change | daily
      zero_downtime: true | false        # Alias swapping
```

### Freshness Strategies

| Strategy | Description | Typical Latency |
|----------|-------------|-----------------|
| `real_time` | Index on every event | <100ms |
| `near_real_time` | Batched micro-updates | <1s |
| `batch` | Periodic bulk updates | Minutes to hours |

---

## Search Intent

Add `searchIntent` for search query semantics:

```yaml
searchIntent:
  name: <string>
  version: <vN>
  
  searches: <DataState>                  # What data is being searched
  
  query:
    fields: [<field>, ...]               # Fields to search
    default_operator: and | or           # How terms combine
    phrase_slop: <integer>               # Phrase proximity tolerance
    
  filters:
    required: [<field>, ...]             # Filters that must be provided
    optional: [<field>, ...]             # Optional filter fields
    
  facets:
    include: [<field>, ...]              # Facets to return
    
  pagination:
    default_size: <integer>
    max_size: <integer>
    cursor_based: true | false           # Use cursor vs offset
    
  sort:
    default: <field> | _score            # Default sort
    allowed: [<field>, ...]              # Allowed sort fields
    
  result:
    includes: [<field>, ...]             # Fields to return
    highlights: [<field>, ...]           # Fields to highlight
    
  authorization:
    policy_set: <policy_set_ref>         # Search access policy
    filter_by: <field>                   # Per-result filtering field
```

---

## Presentation View Integration

Extend `presentationView` for search UI:

```yaml
presentationView:
  name: <string>
  version: <vN>
  
  search:                                # NEW: Search UI semantics
    intent: <searchIntent>               # Which search intent
    
    input:
      placeholder: <string>
      debounce: <duration>               # Input debounce
      min_chars: <integer>               # Minimum chars before search
      
    suggestions:
      enabled: true | false
      max: <integer>                     # Max suggestions to show
      
    results:
      empty_state:
        message: <string>
        actions: [<action>, ...]
      loading_indicator: spinner | skeleton | none
      
    facets:
      position: sidebar | inline | above
      collapsible: true | false
      show_counts: true | false
      multi_select: true | false
      
    filters:
      clear_all: true | false
      active_display: chips | list | inline
```

---

## Semantic Guarantees

When a field declares `search`:

1. **Index Inclusion**: Fields with `indexed: true` MUST be included in search index
2. **Strategy Compliance**: Search behavior MUST match declared strategy
3. **Weight Application**: Relevance scoring MUST respect declared weights
4. **Facet Availability**: Declared facets MUST be available in search results
5. **Highlight Accuracy**: Highlights MUST match actual query terms

---

## Runtime Responsibilities

The runtime MUST:

1. **Build Index**: Create search index from declared searchable fields
2. **Maintain Freshness**: Keep index within declared freshness guarantees
3. **Apply Strategies**: Implement search strategies correctly
4. **Compute Facets**: Generate facets as declared
5. **Enforce Authorization**: Respect search and result-level policies

The runtime MAY:

1. Choose any search engine (Elasticsearch, Algolia, Typesense, etc.)
2. Implement custom analyzers for full_text
3. Use vector databases for semantic search
4. Optimize index structure for performance
5. Cache frequent queries

---

## Examples

### Example 1: Product Search

```yaml
dataType:
  name: Product
  version: v1
  fields:
    id:
      type: uuid
    name:
      type: string
      search:
        indexed: true
        strategy: full_text
        full_text:
          analyzer: standard
          stemming: enabled
        weight: 2.0
        suggest:
          enabled: true
          type: completion
        highlight:
          enabled: true
          
    description:
      type: string
      search:
        indexed: true
        strategy: full_text
        weight: 1.0
        highlight:
          enabled: true
          fragment_size: 150
          
    sku:
      type: string
      search:
        indexed: true
        strategy: exact
        suggest:
          enabled: true
          type: term
          
    category:
      type: string
      search:
        indexed: true
        strategy: exact
        facet:
          enabled: true
          type: terms
          
    price:
      type: decimal
      search:
        indexed: true
        strategy: exact
        facet:
          enabled: true
          type: range
          ranges:
            - { to: 25, label: "Under $25" }
            - { from: 25, to: 50, label: "$25 - $50" }
            - { from: 50, to: 100, label: "$50 - $100" }
            - { from: 100, label: "Over $100" }
            
    brand:
      type: string
      search:
        indexed: true
        strategy: exact
        facet:
          enabled: true
          type: terms
          
    rating:
      type: decimal
      search:
        indexed: true
        strategy: exact
        facet:
          enabled: true
          type: range
          ranges:
            - { from: 4, label: "4+ Stars" }
            - { from: 3, label: "3+ Stars" }
            
    created_at:
      type: timestamp
      search:
        indexed: true
        boost_recent: true
```

### Example 2: Search Intent for Products

```yaml
searchIntent:
  name: ProductSearch
  version: v1
  
  searches: ProductState
  
  query:
    fields: [name, description, sku, brand]
    default_operator: and
    
  filters:
    required: []
    optional: [category, brand, price, rating]
    
  facets:
    include: [category, brand, price, rating]
    
  pagination:
    default_size: 20
    max_size: 100
    cursor_based: false
    
  sort:
    default: _score
    allowed: [price, rating, created_at, name]
    
  result:
    includes: [id, name, description, price, rating, image]
    highlights: [name, description]
```

### Example 3: Customer Search (with Authorization)

```yaml
dataType:
  name: Customer
  version: v1
  fields:
    id:
      type: uuid
    name:
      type: string
      search:
        indexed: true
        strategy: full_text
        weight: 2.0
        
    email:
      type: string
      search:
        indexed: true
        strategy: prefix
        weight: 1.5
        
    phone:
      type: string
      search:
        indexed: true
        strategy: prefix
        
    organization_id:
      type: string
      search:
        indexed: true
        strategy: exact
        # Used for authorization filtering

searchIntent:
  name: CustomerSearch
  version: v1
  
  searches: CustomerState
  
  query:
    fields: [name, email, phone]
    
  authorization:
    policy_set: CustomerSearchAccess
    filter_by: organization_id    # Filter results by org
```

### Example 4: Search Materialization

```yaml
materialization:
  name: ProductSearchIndex
  version: v1
  source: [ProductState, CategoryState, BrandState]
  targetState: ProductSearchViewState
  
  search_index:
    enabled: true
    
    document:
      root: Product
      include_related:
        - entity: Category
          relationship: product_in_category
          fields: [name, path]
        - entity: Brand
          relationship: product_of_brand
          fields: [name, logo]
          
    freshness:
      strategy: near_real_time
      max_lag: 5s
      
    rebuild:
      trigger: schema_change
      zero_downtime: true
```

### Example 5: Search Presentation View

```yaml
presentationView:
  name: ProductSearchView
  version: v1
  consumes:
    dataState: ProductSearchViewState
    
  search:
    intent: ProductSearch
    
    input:
      placeholder: "Search products..."
      debounce: 300ms
      min_chars: 2
      
    suggestions:
      enabled: true
      max: 5
      
    results:
      empty_state:
        message: "No products found"
        actions:
          - label: "Clear filters"
            action: clear_filters
          - label: "Browse all"
            action: browse_all
      loading_indicator: skeleton
      
    facets:
      position: sidebar
      collapsible: true
      show_counts: true
      multi_select: true
      
    filters:
      clear_all: true
      active_display: chips
```

---

## Integration with Existing Grammar

### Extends

- `dataType.fields` - Adds `search` configuration
- `materialization` - Adds `search_index` block
- `presentationView` - Adds `search` UI semantics

### References

- `relationship` - For denormalization in search documents
- `policy` - For search authorization
- `interactionModel.filters` - Complements search facets

### Complements

- `10-presentation-hints.md` - Search results use presentation hints
- `01-form-intent-binding.md` - Search forms bind to search intents
- `06-composition-fetch-semantics.md` - Search results in compositions

---

## Validation Criteria

A conformant implementation MUST:

1. **Parse** search configuration without error
2. **Index** fields with `indexed: true`
3. **Apply** declared search strategies
4. **Generate** declared facets
5. **Enforce** authorization policies

A conformant implementation SHOULD:

1. Support multiple search strategies
2. Implement facet caching
3. Provide query suggestions
4. Support result highlighting
5. Maintain index freshness within declared limits

---

## EBNF Grammar Addition

```ebnf
search_config   ::= "search:" NEWLINE INDENT
                    "indexed:" boolean NEWLINE
                    ( "strategy:" search_strategy NEWLINE )?
                    full_text_config?
                    fuzzy_config?
                    ( "weight:" decimal NEWLINE )?
                    ( "boost_recent:" boolean NEWLINE )?
                    suggest_config?
                    facet_config?
                    highlight_config?
                    DEDENT

search_strategy ::= "full_text" | "exact" | "prefix" | "fuzzy" | "semantic"

full_text_config ::= "full_text:" NEWLINE INDENT
                     ( "analyzer:" analyzer_type NEWLINE )?
                     ( "language:" string NEWLINE )?
                     ( "stemming:" enabled_disabled NEWLINE )?
                     ( "stop_words:" enabled_disabled NEWLINE )?
                     ( "synonyms:" identifier NEWLINE )?
                     DEDENT

analyzer_type   ::= "standard" | "language" | "custom"
enabled_disabled ::= "enabled" | "disabled"

fuzzy_config    ::= "fuzzy:" NEWLINE INDENT
                    ( "max_edits:" integer NEWLINE )?
                    ( "prefix_length:" integer NEWLINE )?
                    DEDENT

suggest_config  ::= "suggest:" NEWLINE INDENT
                    "enabled:" boolean NEWLINE
                    ( "type:" suggest_type NEWLINE )?
                    DEDENT

suggest_type    ::= "completion" | "phrase" | "term"

facet_config    ::= "facet:" NEWLINE INDENT
                    "enabled:" boolean NEWLINE
                    ( "type:" facet_type NEWLINE )?
                    ( "ranges:" "[" range_list "]" NEWLINE )?
                    DEDENT

facet_type      ::= "terms" | "range" | "date_histogram" | "nested"

highlight_config ::= "highlight:" NEWLINE INDENT
                     "enabled:" boolean NEWLINE
                     ( "fragment_size:" integer NEWLINE )?
                     ( "pre_tag:" string NEWLINE )?
                     ( "post_tag:" string NEWLINE )?
                     DEDENT

search_intent   ::= "searchIntent:" NEWLINE INDENT
                    "name:" identifier NEWLINE
                    "version:" version NEWLINE
                    "searches:" data_state_ref NEWLINE
                    query_config?
                    filter_config?
                    facet_include?
                    pagination_config?
                    sort_config?
                    result_config?
                    search_authorization?
                    DEDENT

search_index    ::= "search_index:" NEWLINE INDENT
                    "enabled:" boolean NEWLINE
                    document_config?
                    freshness_config?
                    rebuild_config?
                    DEDENT

freshness_config ::= "freshness:" NEWLINE INDENT
                     "strategy:" freshness_strategy NEWLINE
                     ( "max_lag:" duration NEWLINE )?
                     ( "batch_interval:" duration NEWLINE )?
                     DEDENT

freshness_strategy ::= "real_time" | "near_real_time" | "batch"
```

---

## ADDENDUM AM: SCHEDULED TRIGGER SEMANTICS (NEW in v1.5)

**Purpose**: Time-Based Intent and Flow Triggers  
**Extends**: `inputIntent`, `smsFlow`, `signal`  
**Priority**: MEDIUM

---

## Problem Statement

The current specification defines:
- `inputIntent` triggered by user/system proposals
- `smsFlow` triggered by intents
- `signal` for operational observations
- `scheduler` for infrastructure scheduling

However, there is no grammar for expressing:
1. Time-based triggers for intents (cron schedules)
2. Recurring flow execution
3. Delayed intent submission
4. Batch processing windows
5. Time-zone aware scheduling

**Impact**: Applications requiring scheduled jobs, recurring reports, or time-based processing must implement these patterns entirely outside the specification.

---

## Design Principle Alignment

### Intent-First

This addendum expresses **what** schedule means, not **how** to implement scheduling:

| Intent-Level (This Spec) | Runtime Decides |
|--------------------------|-----------------|
| `schedule: "0 8 * * MON"` | Kubernetes CronJob, Temporal, custom scheduler |
| `timezone: America/New_York` | Timezone library, system locale |
| `delay: 5m` | Timer implementation, queue delay |
| `batch.window: 1h` | Batch aggregation strategy |

### Transport-Agnostic

The grammar does not specify:
- Scheduler implementation (cron, Temporal, Airflow, etc.)
- Queue or timer mechanisms
- Distributed lock strategy for exactly-once
- Failure recovery implementation

### Signals Integration

Scheduled triggers integrate with the existing signal pattern:
- Emit signals about schedule execution
- Observable by monitoring systems
- Can influence scheduling decisions

---

## Intent-Level Grammar

### Scheduled Intent Trigger

Add `schedule` block to `inputIntent` for time-based triggering:

```yaml
inputIntent:
  name: <string>
  version: <vN>
  
  schedule:                              # NEW: Scheduled triggering
    enabled: true | false
    
    trigger:
      type: cron | interval | delay | event_window
      
      # For cron type
      cron:
        expression: <cron_expression>    # Standard cron syntax
        timezone: <timezone>             # IANA timezone
        
      # For interval type
      interval:
        every: <duration>                # e.g., "1h", "30m", "1d"
        jitter: <duration>               # Random offset to avoid thundering herd
        
      # For delay type
      delay:
        after: <duration>                # Delay from creation
        
      # For event_window type
      event_window:
        collect_for: <duration>          # Window duration
        trigger_on: count | time | both  # What triggers processing
        min_count: <integer>             # Minimum events before trigger
        
    scope:
      realm: all | specific              # Which realms
      realms: [<realm>, ...]             # For specific
      
    concurrency:
      policy: allow | skip | queue       # If previous still running
      max_instances: <integer>           # Maximum concurrent executions
      
    failure:
      retry: enabled | disabled
      max_retries: <integer>
      backoff: linear | exponential | fixed
      dead_letter: <intent_ref>          # Intent to call on final failure
      
    observability:
      emit_start: true | false           # Emit signal on start
      emit_complete: true | false        # Emit signal on complete
      emit_skip: true | false            # Emit signal when skipped
      
    idempotency:
      key_includes: [timestamp, realm, ...]  # What makes execution unique
```

### Trigger Types

| Type | Description | Use Case |
|------|-------------|----------|
| `cron` | Standard cron schedule | Daily reports, weekly cleanups |
| `interval` | Fixed interval | Polling, health checks |
| `delay` | One-time delayed | Reminders, follow-ups |
| `event_window` | Aggregate events over time | Batch processing |

### Concurrency Policies

| Policy | Description | Behavior |
|--------|-------------|----------|
| `allow` | Allow concurrent | Multiple instances may run |
| `skip` | Skip if running | New instance not started |
| `queue` | Queue for later | New instance waits for previous |

---

## Scheduled Flow Trigger

Extend `smsFlow` for scheduled execution:

```yaml
smsFlow:
  name: <string>
  version: <vN>
  
  triggeredBy:
    schedule:                            # NEW: Schedule trigger
      intent: <inputIntent>              # Which scheduled intent triggers this
      
  # ... rest of flow definition
```

---

## Batch Processing Semantics

Add `batch` block for batch processing patterns:

```yaml
inputIntent:
  name: <string>
  version: <vN>
  
  batch:                                 # NEW: Batch processing semantics
    enabled: true | false
    
    source:
      query: <expression>                # What entities to process
      order_by: <field>                  # Processing order
      direction: asc | desc
      
    chunking:
      size: <integer>                    # Entities per chunk
      parallel: <integer>                # Parallel chunks
      
    checkpoint:
      enabled: true | false
      frequency: every_chunk | every_n   # When to checkpoint
      every_n: <integer>                 # For every_n frequency
      
    progress:
      emit_interval: <duration>          # Progress signal interval
      
    completion:
      on_success: <intent_ref>           # Intent on batch complete
      on_partial: <intent_ref>           # Intent on partial completion
      on_failure: <intent_ref>           # Intent on batch failure
```

---

## Schedule Constraint Expression

Add schedule constraints for governance:

```yaml
schedule:
  constraints:                           # NEW: Schedule governance
    allowed_hours:
      start: <time>                      # e.g., "06:00"
      end: <time>                        # e.g., "22:00"
      timezone: <timezone>
      
    allowed_days: [<day>, ...]           # e.g., [MON, TUE, WED, THU, FRI]
    
    blackout_dates:
      - date: <date>                     # Specific dates
        reason: <string>
      - range:
          start: <date>
          end: <date>
          reason: <string>
          
    rate_limit:
      max_per_hour: <integer>
      max_per_day: <integer>
```

---

## Semantic Guarantees

When an intent declares `schedule`:

1. **Trigger Accuracy**: Schedule MUST trigger within acceptable tolerance of declared time
2. **Timezone Correctness**: Timezone MUST be respected including DST transitions
3. **Concurrency Compliance**: Concurrency policy MUST be enforced
4. **Idempotency**: Scheduled executions MUST be idempotent
5. **Observability**: Declared signals MUST be emitted
6. **Constraint Enforcement**: Schedule constraints MUST be respected

---

## Runtime Responsibilities

The runtime MUST:

1. **Parse Schedules**: Correctly parse cron expressions and intervals
2. **Handle Timezones**: Respect timezone including DST
3. **Enforce Concurrency**: Implement concurrency policies
4. **Emit Signals**: Emit observability signals as configured
5. **Checkpoint Progress**: For batch, maintain checkpoint state

The runtime MAY:

1. Choose any scheduler implementation
2. Distribute scheduled execution across workers
3. Implement distributed locking for exactly-once
4. Provide schedule UI for monitoring
5. Support dynamic schedule modification

---

## Examples

### Example 1: Daily Report Generation

```yaml
inputIntent:
  name: GenerateDailyReport
  version: v1
  
  proposes:
    dataState: ReportState
    
  supplies:
    required: []
    optional:
      - report_date
      - format
      
  schedule:
    enabled: true
    trigger:
      type: cron
      cron:
        expression: "0 6 * * *"          # 6 AM daily
        timezone: America/New_York
        
    scope:
      realm: all
      
    concurrency:
      policy: skip                        # Skip if previous still running
      
    failure:
      retry: enabled
      max_retries: 3
      backoff: exponential
      
    observability:
      emit_start: true
      emit_complete: true
      
    idempotency:
      key_includes: [timestamp, realm]
```

### Example 2: Interval Health Check

```yaml
inputIntent:
  name: SystemHealthCheck
  version: v1
  
  proposes:
    dataState: HealthCheckState
    
  schedule:
    enabled: true
    trigger:
      type: interval
      interval:
        every: 5m
        jitter: 30s                       # Random 0-30s offset
        
    concurrency:
      policy: skip
      
    observability:
      emit_complete: true
```

### Example 3: Delayed Reminder

```yaml
inputIntent:
  name: SendFollowUpReminder
  version: v1
  
  proposes:
    dataState: ReminderState
    
  supplies:
    required:
      - entity_id
      - reminder_type
      
  schedule:
    enabled: true
    trigger:
      type: delay
      delay:
        after: 24h                        # 24 hours after creation
        
    concurrency:
      policy: allow
      
    failure:
      retry: enabled
      max_retries: 5
      backoff: exponential
```

### Example 4: Event Window Batching

```yaml
inputIntent:
  name: ProcessEventBatch
  version: v1
  
  proposes:
    dataState: BatchResultState
    
  schedule:
    enabled: true
    trigger:
      type: event_window
      event_window:
        collect_for: 1h                   # Collect events for 1 hour
        trigger_on: both
        min_count: 10                     # At least 10 events
        
    concurrency:
      policy: queue
      max_instances: 3
```

### Example 5: Batch Processing with Checkpoints

```yaml
inputIntent:
  name: RecalculateAllBalances
  version: v1
  
  proposes:
    dataState: AccountState
    
  schedule:
    enabled: true
    trigger:
      type: cron
      cron:
        expression: "0 2 * * SUN"         # 2 AM every Sunday
        timezone: UTC
        
    concurrency:
      policy: skip
      
    constraints:
      allowed_hours:
        start: "00:00"
        end: "06:00"
        timezone: UTC
      blackout_dates:
        - range:
            start: "2026-12-24"
            end: "2026-12-26"
            reason: "Holiday freeze"
            
  batch:
    enabled: true
    source:
      query: "account.status == 'active'"
      order_by: id
      direction: asc
      
    chunking:
      size: 1000
      parallel: 4
      
    checkpoint:
      enabled: true
      frequency: every_chunk
      
    progress:
      emit_interval: 1m
      
    completion:
      on_success: NotifyBatchComplete
      on_failure: NotifyBatchFailure
```

### Example 6: Schedule with Governance

```yaml
inputIntent:
  name: SendMarketingEmails
  version: v1
  
  proposes:
    dataState: EmailCampaignState
    
  schedule:
    enabled: true
    trigger:
      type: cron
      cron:
        expression: "0 10 * * *"
        timezone: America/Los_Angeles
        
    constraints:
      allowed_hours:
        start: "09:00"
        end: "18:00"
        timezone: America/Los_Angeles
        
      allowed_days: [MON, TUE, WED, THU, FRI]
      
      rate_limit:
        max_per_hour: 10000
        max_per_day: 50000
```

---

## Integration with Existing Grammar

### Extends

- `inputIntent` - Adds `schedule` and `batch` blocks
- `smsFlow.triggeredBy` - Adds schedule trigger type

### References

- `signal` - Schedule execution emits signals
- `scheduler` - Integrates with existing scheduler concepts
- `idempotency` - Schedule executions use idempotency keys
- `policy` - Schedule can be policy-controlled

### Complements

- `03-intent-delivery-contract.md` - Scheduled intents use delivery contracts
- `05-indicator-source-binding.md` - Indicators can show schedule status
- `signal` - Schedule signals integrate with observability

---

## Validation Criteria

A conformant implementation MUST:

1. **Parse** schedule expressions correctly
2. **Handle** timezone and DST transitions
3. **Enforce** concurrency policies
4. **Emit** observability signals
5. **Respect** schedule constraints

A conformant implementation SHOULD:

1. Provide schedule monitoring UI
2. Support schedule pause/resume
3. Provide execution history
4. Support dynamic schedule updates
5. Implement distributed execution with locks

---

## EBNF Grammar Addition

```ebnf
schedule        ::= "schedule:" NEWLINE INDENT
                    "enabled:" boolean NEWLINE
                    schedule_trigger
                    schedule_scope?
                    schedule_concurrency?
                    schedule_failure?
                    schedule_observability?
                    schedule_idempotency?
                    schedule_constraints?
                    DEDENT

schedule_trigger ::= "trigger:" NEWLINE INDENT
                     "type:" trigger_type NEWLINE
                     ( cron_config | interval_config | delay_config | event_window_config )
                     DEDENT

trigger_type    ::= "cron" | "interval" | "delay" | "event_window"

cron_config     ::= "cron:" NEWLINE INDENT
                    "expression:" string NEWLINE
                    ( "timezone:" string NEWLINE )?
                    DEDENT

interval_config ::= "interval:" NEWLINE INDENT
                    "every:" duration NEWLINE
                    ( "jitter:" duration NEWLINE )?
                    DEDENT

delay_config    ::= "delay:" NEWLINE INDENT
                    "after:" duration NEWLINE
                    DEDENT

event_window_config ::= "event_window:" NEWLINE INDENT
                        "collect_for:" duration NEWLINE
                        ( "trigger_on:" window_trigger NEWLINE )?
                        ( "min_count:" integer NEWLINE )?
                        DEDENT

window_trigger  ::= "count" | "time" | "both"

schedule_scope  ::= "scope:" NEWLINE INDENT
                    "realm:" realm_scope NEWLINE
                    ( "realms:" "[" realm_list "]" NEWLINE )?
                    DEDENT

realm_scope     ::= "all" | "specific"

schedule_concurrency ::= "concurrency:" NEWLINE INDENT
                         "policy:" concurrency_policy NEWLINE
                         ( "max_instances:" integer NEWLINE )?
                         DEDENT

concurrency_policy ::= "allow" | "skip" | "queue"

schedule_failure ::= "failure:" NEWLINE INDENT
                     ( "retry:" enabled_disabled NEWLINE )?
                     ( "max_retries:" integer NEWLINE )?
                     ( "backoff:" backoff_strategy NEWLINE )?
                     ( "dead_letter:" intent_ref NEWLINE )?
                     DEDENT

backoff_strategy ::= "linear" | "exponential" | "fixed"

schedule_observability ::= "observability:" NEWLINE INDENT
                           ( "emit_start:" boolean NEWLINE )?
                           ( "emit_complete:" boolean NEWLINE )?
                           ( "emit_skip:" boolean NEWLINE )?
                           DEDENT

schedule_constraints ::= "constraints:" NEWLINE INDENT
                         allowed_hours?
                         ( "allowed_days:" "[" day_list "]" NEWLINE )?
                         blackout_dates?
                         rate_limit?
                         DEDENT

allowed_hours   ::= "allowed_hours:" NEWLINE INDENT
                    "start:" time NEWLINE
                    "end:" time NEWLINE
                    ( "timezone:" string NEWLINE )?
                    DEDENT

blackout_dates  ::= "blackout_dates:" NEWLINE INDENT
                    blackout_entry+
                    DEDENT

rate_limit      ::= "rate_limit:" NEWLINE INDENT
                    ( "max_per_hour:" integer NEWLINE )?
                    ( "max_per_day:" integer NEWLINE )?
                    DEDENT

batch           ::= "batch:" NEWLINE INDENT
                    "enabled:" boolean NEWLINE
                    batch_source?
                    batch_chunking?
                    batch_checkpoint?
                    batch_progress?
                    batch_completion?
                    DEDENT
```

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial proposal |

---

## ADDENDUM AN: NOTIFICATION CHANNEL SEMANTICS (NEW in v1.5)

**Purpose**: Multi-Channel Notification Binding  
**Extends**: `signal`, `inputIntent`, `experience`  
**Priority**: MEDIUM

---

## Problem Statement

The current specification defines:
- `indicator` for navigation badges and alerts
- `signal` for operational observations
- `presentationView` for UI display

However, there is no grammar for expressing:
1. Multi-channel notification delivery (email, SMS, push, in-app)
2. Notification templates and personalization
3. User notification preferences
4. Delivery guarantees and acknowledgment
5. Notification grouping and throttling

**Impact**: Applications requiring notifications must implement channel routing, template rendering, and preference management entirely in runtime code without specification guidance.

---

## Design Principle Alignment

### Intent-First

This addendum expresses **what** notification semantics mean, not **how** to deliver:

| Intent-Level (This Spec) | Runtime Decides |
|--------------------------|-----------------|
| `channel: email` | SendGrid, SES, SMTP, Mailgun |
| `channel: sms` | Twilio, SNS, Vonage |
| `channel: push` | FCM, APNs, OneSignal |
| `priority: high` | Delivery queue priority, retry strategy |
| `template: password_reset` | Template engine, rendering pipeline |

### Transport-Agnostic

The grammar does not specify:
- Notification service providers
- Template rendering engines
- Delivery infrastructure
- Push notification protocols

### Subject-Bound

Notifications are delivered to subjects:
- Respect subject notification preferences
- Observe privacy policies
- Track delivery per subject

---

## Intent-Level Grammar

### Notification Intent

Add `notification` block to `inputIntent` for notification semantics:

```yaml
inputIntent:
  name: <string>
  version: <vN>
  
  notification:                          # NEW: Notification semantics
    enabled: true | false
    
    trigger:
      on: intent_success | intent_failure | always | condition
      condition: <expression>            # For condition trigger
      
    channels:
      - type: email | sms | push | in_app | webhook
        priority: <integer>              # Channel priority (1 = highest)
        fallback: true | false           # Try next if this fails
        
        # Channel-specific config
        email:
          template: <template_ref>
          from: <sender_ref>
          subject: <string_template>
          reply_to: <string_template>
          
        sms:
          template: <template_ref>
          from: <sender_ref>
          
        push:
          template: <template_ref>
          title: <string_template>
          body: <string_template>
          action: <action_ref>
          badge_increment: <integer>
          
        in_app:
          template: <template_ref>
          severity: info | warning | error | success
          duration: <duration>           # Auto-dismiss time
          dismissible: true | false
          action: <action_ref>
          
        webhook:
          template: <template_ref>       # Payload template
          
    recipient:
      type: subject | role | query | explicit
      subject_field: <field>             # For subject type
      role: <role_ref>                   # For role type
      query: <expression>                # For query type
      explicit: [<recipient>, ...]       # For explicit type
      
    personalization:
      include: [<field>, ...]            # Fields to include in template
      
    delivery:
      guarantee: best_effort | at_least_once | exactly_once
      timeout: <duration>
      
    grouping:
      enabled: true | false
      key: <expression>                  # How to group notifications
      window: <duration>                 # Grouping window
      max_group_size: <integer>
      
    throttling:
      enabled: true | false
      max_per_hour: <integer>
      max_per_day: <integer>
      per: subject | channel | global
      
    tracking:
      delivery: true | false             # Track delivery status
      open: true | false                 # Track opens (email)
      click: true | false                # Track clicks
      dismiss: true | false              # Track dismissals
```

### Channel Types

| Type | Description | Typical Use |
|------|-------------|-------------|
| `email` | Email delivery | Transactional, marketing, reports |
| `sms` | SMS/text message | Alerts, 2FA, critical notifications |
| `push` | Mobile push notification | Real-time alerts, reminders |
| `in_app` | In-application notification | Toast, banner, notification center |
| `webhook` | HTTP webhook delivery | Integrations, automations |

### Recipient Types

| Type | Description | Example |
|------|-------------|---------|
| `subject` | Specific subject field | `customer_id` from intent |
| `role` | All subjects with role | All `admin` users |
| `query` | Query-based recipients | Active users in region |
| `explicit` | Explicit list | Specific email addresses |

---

## Notification Template

Add `notificationTemplate` for template definitions:

```yaml
notificationTemplate:
  name: <string>
  version: <vN>
  
  channels:
    email:
      subject: <string_template>
      body:
        html: <template_ref>
        text: <template_ref>
      attachments:
        - name: <string>
          asset: <field_ref>
          
    sms:
      body: <string_template>
      
    push:
      title: <string_template>
      body: <string_template>
      image: <string_template>           # URL or asset reference
      data: { <key>: <expression>, ... } # Custom data payload
      
    in_app:
      title: <string_template>
      body: <string_template>
      icon: <icon_ref>
      actions:
        - label: <string>
          action: <action_ref>
          style: primary | secondary | danger
          
  variables:
    required: [<field>, ...]
    optional: [<field>, ...]
    
  localization:
    supported: [<locale>, ...]
    default: <locale>
```

---

## User Notification Preferences

Extend `experience` for notification preference schema:

```yaml
experience:
  name: <string>
  version: <vN>
  
  notification_preferences:              # NEW: User notification preferences
    schema:
      channels:
        email:
          enabled: { type: boolean, default: true }
          address: { type: string, source: subject.email }
          
        sms:
          enabled: { type: boolean, default: false }
          number: { type: string, source: subject.phone }
          
        push:
          enabled: { type: boolean, default: true }
          
        in_app:
          enabled: { type: boolean, default: true }
          
      categories:                        # User can opt in/out per category
        - name: marketing
          label: "Marketing & Promotions"
          default: false
          channels: [email]
          
        - name: security
          label: "Security Alerts"
          default: true
          required: true                 # Cannot be disabled
          channels: [email, sms, push]
          
        - name: transactions
          label: "Transaction Notifications"
          default: true
          channels: [email, push, in_app]
          
        - name: reports
          label: "Reports & Summaries"
          default: true
          channels: [email]
          
      quiet_hours:
        enabled: { type: boolean, default: false }
        start: { type: time }
        end: { type: time }
        timezone: { type: string, source: subject.timezone }
        exceptions: [security]           # Categories that ignore quiet hours
```

---

## Notification Signal Integration

Notifications emit signals for observability:

```yaml
signal:
  type: notification
  source:
    component: notification
    intent: <intent_name>
  subject:
    notification_id: <uuid>
    recipient: <subject_id>
    channel: <channel_type>
  metrics:
    status: pending | delivered | failed | bounced | opened | clicked
    attempt: <integer>
    latency_ms: <integer>
  severity: info
  timestamp: <iso8601>
```

---

## Semantic Guarantees

When an intent declares `notification`:

1. **Channel Priority**: Channels MUST be attempted in priority order
2. **Fallback Behavior**: If `fallback: true`, next channel MUST be tried on failure
3. **Preference Compliance**: User preferences MUST be respected
4. **Quiet Hours**: Quiet hours MUST be respected (except exceptions)
5. **Throttling**: Throttle limits MUST be enforced
6. **Tracking Accuracy**: Tracking events MUST reflect actual delivery state

---

## Runtime Responsibilities

The runtime MUST:

1. **Route Channels**: Deliver to configured channels
2. **Respect Preferences**: Honor user notification preferences
3. **Enforce Throttling**: Apply throttle limits
4. **Track Delivery**: Track and emit delivery signals
5. **Apply Templates**: Render templates with personalization

The runtime MAY:

1. Choose any delivery provider per channel
2. Implement custom retry strategies
3. Batch notifications for efficiency
4. Provide notification management UI
5. Implement A/B testing for templates

---

## Examples

### Example 1: Password Reset Email

```yaml
inputIntent:
  name: RequestPasswordReset
  version: v1
  
  proposes:
    dataState: PasswordResetState
    
  notification:
    enabled: true
    trigger:
      on: intent_success
      
    channels:
      - type: email
        priority: 1
        email:
          template: password_reset_email
          from: security
          subject: "Reset your password"
          
    recipient:
      type: subject
      subject_field: user_id
      
    personalization:
      include: [reset_link, expiry_time, user_name]
      
    delivery:
      guarantee: at_least_once
      timeout: 5m
      
    tracking:
      delivery: true
      click: true

notificationTemplate:
  name: password_reset_email
  version: v1
  
  channels:
    email:
      subject: "Reset your password for {{app_name}}"
      body:
        html: templates/email/password_reset.html
        text: templates/email/password_reset.txt
        
  variables:
    required: [reset_link, expiry_time, user_name]
    
  localization:
    supported: [en, es, fr, de]
    default: en
```

### Example 2: Transaction Alert (Multi-Channel)

```yaml
inputIntent:
  name: ProcessTransaction
  version: v1
  
  proposes:
    dataState: TransactionState
    
  notification:
    enabled: true
    trigger:
      on: condition
      condition: "transaction.amount > 1000"
      
    channels:
      - type: push
        priority: 1
        fallback: true
        push:
          template: large_transaction_push
          title: "Large Transaction Alert"
          body: "{{amount | currency}} {{type}} on {{account.name}}"
          badge_increment: 1
          
      - type: in_app
        priority: 2
        fallback: true
        in_app:
          template: large_transaction_in_app
          severity: warning
          duration: 10s
          dismissible: true
          
      - type: sms
        priority: 3
        fallback: false
        sms:
          template: large_transaction_sms
          from: alerts
          
    recipient:
      type: subject
      subject_field: account.owner_id
      
    personalization:
      include: [amount, type, account, timestamp, location]
      
    throttling:
      enabled: true
      max_per_hour: 10
      per: subject
```

### Example 3: Marketing Email with Preferences

```yaml
inputIntent:
  name: SendWeeklyNewsletter
  version: v1
  
  notification:
    enabled: true
    trigger:
      on: always
      
    channels:
      - type: email
        priority: 1
        email:
          template: weekly_newsletter
          from: marketing
          subject: "Your weekly update from {{app_name}}"
          
    recipient:
      type: query
      query: "user.newsletter_subscribed == true AND user.status == 'active'"
      
    grouping:
      enabled: false
      
    throttling:
      enabled: true
      max_per_day: 1
      per: subject
      
    tracking:
      delivery: true
      open: true
      click: true
```

### Example 4: Security Alert (Required Channel)

```yaml
inputIntent:
  name: DetectSuspiciousLogin
  version: v1
  
  notification:
    enabled: true
    trigger:
      on: intent_success
      
    channels:
      - type: email
        priority: 1
        fallback: true
        email:
          template: suspicious_login_email
          from: security
          subject: "Security Alert: New sign-in detected"
          
      - type: sms
        priority: 2
        fallback: false
        sms:
          template: suspicious_login_sms
          
      - type: push
        priority: 3
        push:
          template: suspicious_login_push
          title: "Security Alert"
          body: "New sign-in from {{location}}"
          action: review_activity
          
    recipient:
      type: subject
      subject_field: user_id
      
    personalization:
      include: [ip_address, location, device, timestamp]
      
    delivery:
      guarantee: at_least_once
      timeout: 30s
      
    # Security notifications ignore quiet hours
```

### Example 5: Notification Preferences UI

```yaml
experience:
  name: CustomerBanking
  version: v1
  
  notification_preferences:
    schema:
      channels:
        email:
          enabled: { type: boolean, default: true }
          address: { type: string, source: subject.email }
          
        sms:
          enabled: { type: boolean, default: false }
          number: { type: string, source: subject.phone }
          verified: { type: boolean, default: false }
          
        push:
          enabled: { type: boolean, default: true }
          
      categories:
        - name: security
          label: "Security Alerts"
          description: "Login attempts, password changes, suspicious activity"
          default: true
          required: true
          channels: [email, sms, push]
          
        - name: transactions
          label: "Transaction Notifications"
          description: "Deposits, withdrawals, transfers"
          default: true
          channels: [email, push]
          subcategories:
            - name: large_transactions
              label: "Large transactions only"
              default: false
              threshold: 1000
              
        - name: account
          label: "Account Updates"
          description: "Statements, balance alerts, fee notifications"
          default: true
          channels: [email]
          
        - name: marketing
          label: "Offers & Promotions"
          description: "Special offers, new features, surveys"
          default: false
          channels: [email]
          
      quiet_hours:
        enabled: { type: boolean, default: false }
        start: { type: time, default: "22:00" }
        end: { type: time, default: "08:00" }
        timezone: { type: string, source: subject.timezone }
        exceptions: [security]
```

---

## Integration with Existing Grammar

### Extends

- `inputIntent` - Adds `notification` block
- `experience` - Adds `notification_preferences`
- `signal` - Adds notification signal type

### References

- `policy` - Notification delivery can be policy-controlled
- `indicator` - In-app notifications complement indicators
- `subject` - Notifications delivered to subjects

### Complements

- `08-indicator-system.md` - In-app notifications complement indicators
- `12-accessibility-preferences.md` - Notification preferences extend user preferences
- `16-scheduled-trigger-semantics.md` - Scheduled notifications

---

## Validation Criteria

A conformant implementation MUST:

1. **Parse** notification configuration without error
2. **Respect** channel priorities and fallback
3. **Honor** user notification preferences
4. **Enforce** throttling limits
5. **Track** delivery status

A conformant implementation SHOULD:

1. Support multiple channels per notification
2. Provide notification template preview
3. Track delivery metrics
4. Support notification grouping
5. Implement quiet hours correctly

---

## EBNF Grammar Addition

```ebnf
notification    ::= "notification:" NEWLINE INDENT
                    "enabled:" boolean NEWLINE
                    notification_trigger
                    notification_channels
                    notification_recipient
                    notification_personalization?
                    notification_delivery?
                    notification_grouping?
                    notification_throttling?
                    notification_tracking?
                    DEDENT

notification_trigger ::= "trigger:" NEWLINE INDENT
                         "on:" trigger_event NEWLINE
                         ( "condition:" expression NEWLINE )?
                         DEDENT

trigger_event   ::= "intent_success" | "intent_failure" | "always" | "condition"

notification_channels ::= "channels:" NEWLINE INDENT
                          channel_config+
                          DEDENT

channel_config  ::= "-" "type:" channel_type NEWLINE INDENT
                    ( "priority:" integer NEWLINE )?
                    ( "fallback:" boolean NEWLINE )?
                    channel_specific_config?
                    DEDENT

channel_type    ::= "email" | "sms" | "push" | "in_app" | "webhook"

notification_recipient ::= "recipient:" NEWLINE INDENT
                           "type:" recipient_type NEWLINE
                           ( "subject_field:" identifier NEWLINE )?
                           ( "role:" identifier NEWLINE )?
                           ( "query:" expression NEWLINE )?
                           ( "explicit:" "[" recipient_list "]" NEWLINE )?
                           DEDENT

recipient_type  ::= "subject" | "role" | "query" | "explicit"

notification_throttling ::= "throttling:" NEWLINE INDENT
                            "enabled:" boolean NEWLINE
                            ( "max_per_hour:" integer NEWLINE )?
                            ( "max_per_day:" integer NEWLINE )?
                            ( "per:" throttle_scope NEWLINE )?
                            DEDENT

throttle_scope  ::= "subject" | "channel" | "global"

notification_template ::= "notificationTemplate:" NEWLINE INDENT
                          "name:" identifier NEWLINE
                          "version:" version NEWLINE
                          template_channels
                          template_variables?
                          template_localization?
                          DEDENT

notification_preferences ::= "notification_preferences:" NEWLINE INDENT
                             "schema:" NEWLINE INDENT
                             channels_schema
                             categories_schema?
                             quiet_hours_schema?
                             DEDENT
                             DEDENT
```

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial proposal |

---

## ADDENDUM AO: COLLABORATIVE SESSION SEMANTICS (NEW in v1.5)

**Purpose**: Real-Time Collaboration Without Violating Entity Authority  
**Extends**: `dataState`, `experience`, `presentationView`  
**Priority**: MEDIUM

---

## Problem Statement

The current specification defines:
- Entity-scoped authority with single writer semantics
- CAS tokens for optimistic concurrency
- Pause-and-cutover for authority transitions

Real-time collaboration (like Google Docs) appears to require multi-writer semantics, which would violate the core authority principles.

However, **collaborative editing can be decomposed** into:
1. **Single-authority entity** (the document) - unchanged
2. **Ephemeral presence stream** (cursors, selections) - non-authoritative
3. **Operation proposals** (edits) - submitted to authority

This addendum provides grammar for expressing collaborative session intent while preserving entity-scoped authority.

---

## Design Principle Alignment

### Intent-First

This addendum expresses **what** collaboration means, not **how** to implement:

| Intent-Level (This Spec) | Runtime Decides |
|--------------------------|-----------------|
| `presence.ttl: 30s` | WebSocket heartbeat, NATS ephemeral, polling |
| `operations.ordering: causal` | Vector clocks, Lamport timestamps, sequence numbers |
| `conflict_hint: last_writer_wins` | OT, CRDT, server-side merge, or simple LWW |
| `subscription.to: operations` | WebSocket, Server-Sent Events, NATS pub/sub |

### Transport-Agnostic

The grammar does not specify:
- Conflict resolution algorithm (OT vs CRDT vs custom)
- Real-time protocol (WebSocket, SSE, long-poll, NATS)
- Presence propagation mechanism
- Operation serialization format

### Authority-Preserving

The key insight: **Presence is ephemeral, not authoritative. Operations are proposals.**

- The entity retains single-writer authority
- Clients propose operations to the authority
- Authority resolves conflicts and broadcasts canonical state
- Presence is side-channel, non-persistent data

---

## Intent-Level Grammar

### Collaborative Session Definition

Add `collaborativeSession` as a top-level specification element:

```yaml
collaborativeSession:
  name: <string>
  version: <vN>
  description: <string>
  
  binds_to:
    dataState: <dataState_ref>           # Single-authority entity
    entity_key: <field>                  # What identifies each session instance
    
  presence:                              # Ephemeral, non-authoritative state
    enabled: true | false
    
    fields:                              # What presence data to track
      <field_name>:
        type: <type>
        scope: per_subject | per_client  # Subject can have multiple clients
        
    ttl: <duration>                      # Auto-expire after inactivity
    
    broadcast:
      to: all | same_entity | custom
      custom_filter: <expression>        # For custom routing
      
    throttle: <duration>                 # Limit presence update frequency
    
  operations:                            # Edit proposals to authority
    enabled: true | false
    
    mode: stream | batch                 # Continuous or periodic
    
    ordering:
      guarantee: none | causal | total   # Ordering semantics
      
    conflict:
      hint: last_writer_wins | merge | reject | queue
      # Hint to runtime - actual strategy is runtime-decided
      
    batching:
      enabled: true | false
      max_delay: <duration>              # Max time before flush
      max_operations: <integer>          # Max ops before flush
      
    acknowledgment:
      required: true | false
      timeout: <duration>
      
  subscription:                          # What clients receive
    to: state | operations | presence | all
    
    state:
      mode: full | delta                 # Full state or incremental
      debounce: <duration>               # Limit update frequency
      
    operations:
      include_own: true | false          # Receive own operations back
      
  authorization:
    policy_set: <policy_set_ref>
    
    roles:
      - name: editor
        can: [read, write, presence]
      - name: viewer
        can: [read, presence]
      - name: commenter
        can: [read, comment, presence]
        
  limits:
    max_participants: <integer>
    max_operations_per_second: <integer>
    max_presence_updates_per_second: <integer>
```

### Presence Field Types

| Type | Description | Example |
|------|-------------|---------|
| `position` | Cursor position in content | `{ line: 5, column: 12 }` |
| `range` | Selection range | `{ start: {...}, end: {...} }` |
| `viewport` | Visible area | `{ top: 100, bottom: 500 }` |
| `status` | User status | `active`, `idle`, `away` |
| `pointer` | Mouse/touch position | `{ x: 150, y: 200 }` |

### Conflict Hints

| Hint | Description | Use Case |
|------|-------------|----------|
| `last_writer_wins` | Most recent operation wins | Simple documents |
| `merge` | Attempt to merge concurrent edits | Rich text, structured content |
| `reject` | Reject conflicting operations | Strict consistency |
| `queue` | Queue for manual resolution | Critical data |

---

## Presentation View Integration

Extend `presentationView` for collaborative awareness:

```yaml
presentationView:
  name: CollaborativeDocumentView
  version: v1
  
  consumes:
    dataState: DocumentState
    
  collaborative:                         # NEW: Collaborative UI semantics
    session: DocumentCollabSession       # Which session
    
    presence_display:
      cursors:
        show: true
        style: colored_caret | avatar | both
        label: subject.name
        
      selections:
        show: true
        style: highlight | underline | border
        
      participants:
        show: true
        position: sidebar | header | floating
        max_visible: 5
        show_count: true
        
    status_indicators:
      connection: true                   # Show connection status
      sync_status: true                  # Show sync status
      last_saved: true                   # Show last save time
      
    conflict_ui:
      display: inline | modal | toast
      allow_manual_resolution: true
```

---

## Experience Integration

Extend `experience` for collaborative session management:

```yaml
experience:
  name: CollaborativeEditing
  version: v1
  
  collaborative_sessions:                # NEW: Session configuration
    - session: DocumentCollabSession
      auto_join: on_view                 # When to join session
      auto_leave: on_navigate            # When to leave session
      
      reconnect:
        enabled: true
        max_attempts: 5
        backoff: exponential
        
      offline:
        enabled: true
        queue_operations: true
        max_queue_size: 100
        sync_on_reconnect: true
```

---

## Semantic Guarantees

When a `collaborativeSession` is declared:

1. **Authority Preservation**: The bound `dataState` retains single-writer authority
2. **Presence Ephemeral**: Presence data MUST NOT be persisted as authoritative state
3. **TTL Enforcement**: Presence MUST expire after declared TTL
4. **Ordering Compliance**: Operations MUST be ordered per declared guarantee
5. **Acknowledgment**: If `acknowledgment.required`, operations MUST be confirmed

---

## Runtime Responsibilities

The runtime MUST:

1. **Maintain Authority**: Ensure single-writer semantics for the entity
2. **Broadcast Presence**: Distribute presence updates to subscribers
3. **Order Operations**: Apply ordering guarantees as declared
4. **Enforce TTL**: Expire presence data per TTL
5. **Handle Reconnection**: Support reconnection semantics

The runtime MAY:

1. Choose any conflict resolution algorithm
2. Implement any real-time transport
3. Optimize presence updates (batching, compression)
4. Provide offline support
5. Implement operation transformation or CRDT

---

## Examples

### Example 1: Document Collaboration

```yaml
collaborativeSession:
  name: DocumentCollabSession
  version: v1
  description: "Real-time document editing session"
  
  binds_to:
    dataState: DocumentState
    entity_key: document_id
    
  presence:
    enabled: true
    fields:
      cursor:
        type: position
        scope: per_client
      selection:
        type: range
        scope: per_client
      status:
        type: enum
        values: [active, idle, away]
        scope: per_subject
    ttl: 30s
    broadcast:
      to: same_entity
    throttle: 50ms
    
  operations:
    enabled: true
    mode: stream
    ordering:
      guarantee: causal
    conflict:
      hint: merge
    acknowledgment:
      required: true
      timeout: 5s
      
  subscription:
    to: all
    state:
      mode: delta
      debounce: 100ms
    operations:
      include_own: false
      
  authorization:
    policy_set: DocumentAccessPolicy
    roles:
      - name: owner
        can: [read, write, presence, manage]
      - name: editor
        can: [read, write, presence]
      - name: viewer
        can: [read, presence]
        
  limits:
    max_participants: 50
    max_operations_per_second: 100
```

### Example 2: Whiteboard Collaboration

```yaml
collaborativeSession:
  name: WhiteboardSession
  version: v1
  description: "Real-time whiteboard collaboration"
  
  binds_to:
    dataState: WhiteboardState
    entity_key: board_id
    
  presence:
    enabled: true
    fields:
      pointer:
        type: point
        scope: per_client
      tool:
        type: enum
        values: [pen, eraser, select, text]
        scope: per_client
      color:
        type: string
        scope: per_subject
    ttl: 10s
    broadcast:
      to: same_entity
    throttle: 16ms                       # ~60fps
    
  operations:
    enabled: true
    mode: stream
    ordering:
      guarantee: causal
    conflict:
      hint: last_writer_wins             # Shapes don't merge
    batching:
      enabled: true
      max_delay: 50ms
      max_operations: 10
      
  subscription:
    to: all
    state:
      mode: delta
      
  limits:
    max_participants: 20
```

### Example 3: Spreadsheet Collaboration

```yaml
collaborativeSession:
  name: SpreadsheetSession
  version: v1
  
  binds_to:
    dataState: SpreadsheetState
    entity_key: spreadsheet_id
    
  presence:
    enabled: true
    fields:
      active_cell:
        type: object
        schema:
          row: integer
          column: integer
        scope: per_client
      selection:
        type: object
        schema:
          start_row: integer
          start_col: integer
          end_row: integer
          end_col: integer
        scope: per_client
    ttl: 30s
    broadcast:
      to: same_entity
      
  operations:
    enabled: true
    mode: stream
    ordering:
      guarantee: total                   # Cell edits need total order
    conflict:
      hint: reject                       # Reject concurrent cell edits
    acknowledgment:
      required: true
      timeout: 3s
      
  subscription:
    to: all
    state:
      mode: delta
```

---

## Integration with Existing Grammar

### Extends

- `dataState` - Collaborative sessions bind to data states
- `experience` - Session lifecycle management
- `presentationView` - Collaborative UI elements

### References

- `authority` - Sessions respect entity authority
- `policy` - Session access is policy-controlled
- `signal` - Session events emit signals

### Complements

- `05-indicator-source-binding.md` - Participant counts as indicators
- `04-session-contract.md` - Collaborative sessions within user sessions
- `17-notification-channel-semantics.md` - Notify on session events

---

## Validation Criteria

A conformant implementation MUST:

1. **Parse** collaborative session definitions without error
2. **Preserve** entity single-writer authority
3. **Expire** presence data per TTL
4. **Order** operations per declared guarantee
5. **Enforce** participant limits

A conformant implementation SHOULD:

1. Support reconnection with state recovery
2. Provide conflict resolution UI
3. Support offline operation queuing
4. Optimize presence updates

---

## EBNF Grammar Addition

```ebnf
collaborative_session ::= "collaborativeSession:" NEWLINE INDENT
                          "name:" identifier NEWLINE
                          "version:" version NEWLINE
                          ( "description:" string NEWLINE )?
                          session_binds_to
                          session_presence?
                          session_operations?
                          session_subscription?
                          session_authorization?
                          session_limits?
                          DEDENT

session_binds_to ::= "binds_to:" NEWLINE INDENT
                     "dataState:" data_state_ref NEWLINE
                     "entity_key:" identifier NEWLINE
                     DEDENT

session_presence ::= "presence:" NEWLINE INDENT
                     "enabled:" boolean NEWLINE
                     presence_fields?
                     ( "ttl:" duration NEWLINE )?
                     presence_broadcast?
                     ( "throttle:" duration NEWLINE )?
                     DEDENT

presence_fields ::= "fields:" NEWLINE INDENT
                    ( presence_field_def NEWLINE )+
                    DEDENT

presence_field_def ::= identifier ":" NEWLINE INDENT
                       "type:" type_ref NEWLINE
                       ( "scope:" presence_scope NEWLINE )?
                       DEDENT

presence_scope ::= "per_subject" | "per_client"

session_operations ::= "operations:" NEWLINE INDENT
                       "enabled:" boolean NEWLINE
                       ( "mode:" operation_mode NEWLINE )?
                       operation_ordering?
                       operation_conflict?
                       operation_batching?
                       operation_acknowledgment?
                       DEDENT

operation_mode ::= "stream" | "batch"

operation_ordering ::= "ordering:" NEWLINE INDENT
                       "guarantee:" ordering_guarantee NEWLINE
                       DEDENT

ordering_guarantee ::= "none" | "causal" | "total"

operation_conflict ::= "conflict:" NEWLINE INDENT
                       "hint:" conflict_hint NEWLINE
                       DEDENT

conflict_hint ::= "last_writer_wins" | "merge" | "reject" | "queue"

session_subscription ::= "subscription:" NEWLINE INDENT
                         "to:" subscription_target NEWLINE
                         state_subscription?
                         operations_subscription?
                         DEDENT

subscription_target ::= "state" | "operations" | "presence" | "all"
```

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial proposal |

---

## ADDENDUM AP: STRUCTURED CONTENT TYPE (NEW in v1.5)

**Purpose**: Rich Text and Structured Content Without Prescribing Editor  
**Extends**: `dataType.fields`, `presentationView`, `constraint`  
**Priority**: MEDIUM

---

## Problem Statement

The current specification defines:
- Basic field types (`string`, `decimal`, `boolean`, `timestamp`, etc.)
- `asset` type for binary files (Addendum 14)
- Presentation hints for display formatting (Addendum 10)

However, there is no grammar for expressing:
1. Structured/rich content with semantic blocks (paragraphs, headings, lists)
2. Inline formatting (bold, italic, links)
3. Content schema constraints (allowed blocks, nesting limits)
4. Content structure validation

**Impact**: Applications requiring rich text editing (CMS, documentation, notes) must implement content structure entirely in runtime code without specification guidance.

---

## Design Principle Alignment

### Intent-First

This addendum expresses **what** content structure means, not **how** to edit:

| Intent-Level (This Spec) | Runtime Decides |
|--------------------------|-----------------|
| `blocks.allowed: [paragraph, heading]` | Slate, ProseMirror, TipTap, Lexical, Quill |
| `marks.allowed: [bold, italic, link]` | Toolbar implementation, keyboard shortcuts |
| `structure.outline: true` | Heading hierarchy validation |
| `constraints.max_length: 50000` | Character counting, truncation |

### Transport-Agnostic

The grammar does not specify:
- Editor framework or library
- Content serialization format (though JSON is implied)
- Rendering implementation
- Toolbar or input mechanisms

### Schema-Driven

Structured content is validated against a schema:
- Explicit allowed blocks and marks
- Nesting and depth constraints
- Content-type-specific rules
- Version-aware evolution

---

## Intent-Level Grammar

### Structured Content Field Type

Add `structured_content` as a semantic field type:

```yaml
dataType:
  name: <string>
  version: <vN>
  fields:
    <field_name>:
      type: structured_content           # NEW: Rich content type
      
      structured_content:
        schema: <schema_name>            # Named schema reference
        
        blocks:
          allowed: [<block_type>, ...]   # Which blocks are valid
          required: [<block_type>, ...]  # Which must appear at least once
          forbidden: [<block_type>, ...] # Explicitly disallowed
          
          # Block-specific constraints
          <block_type>:
            max_count: <integer>         # Maximum occurrences
            min_count: <integer>         # Minimum occurrences
            
        marks:
          allowed: [<mark_type>, ...]    # Which inline marks are valid
          forbidden: [<mark_type>, ...]  # Explicitly disallowed
          
        nesting:
          max_depth: <integer>           # Maximum nesting level
          rules:                         # Parent-child rules
            <parent_block>:
              can_contain: [<child_block>, ...]
              cannot_contain: [<block>, ...]
              
        constraints:
          max_length: <integer>          # Total character limit
          max_blocks: <integer>          # Maximum block count
          max_words: <integer>           # Word count limit
          
        structure:
          outline: true | false          # Enforce heading hierarchy
          sections: true | false         # Content divided into sections
          
        links:
          internal:
            enabled: true | false
            targets: [<entity_type>, ...] # What can be linked
          external:
            enabled: true | false
            validate: true | false       # Validate URL format
            allow_embed: true | false    # Allow embedded content
            
        embeds:
          allowed: [<embed_type>, ...]   # What can be embedded
          <embed_type>:
            max_size: <size>
            mime_types: [<type>, ...]
            
        localization:
          enabled: true | false
          languages: [<locale>, ...]
          fallback: <locale>
```

### Standard Block Types

| Block | Description | Typical Rendering |
|-------|-------------|-------------------|
| `paragraph` | Standard text paragraph | `<p>` |
| `heading` | Section heading (1-6) | `<h1>` - `<h6>` |
| `list` | Ordered or unordered list | `<ul>`, `<ol>` |
| `list_item` | Item within a list | `<li>` |
| `blockquote` | Block quotation | `<blockquote>` |
| `code_block` | Code with syntax highlighting | `<pre><code>` |
| `image` | Embedded image | `<figure><img>` |
| `video` | Embedded video | `<video>` |
| `table` | Data table | `<table>` |
| `divider` | Horizontal rule | `<hr>` |
| `callout` | Highlighted callout box | Custom box |
| `toggle` | Collapsible content | `<details>` |
| `embed` | External embed (YouTube, etc.) | `<iframe>` |

### Standard Mark Types

| Mark | Description | Typical Rendering |
|------|-------------|-------------------|
| `bold` | Bold text | `<strong>` |
| `italic` | Italic text | `<em>` |
| `underline` | Underlined text | `<u>` |
| `strikethrough` | Struck-through text | `<s>` |
| `code` | Inline code | `<code>` |
| `link` | Hyperlink | `<a>` |
| `highlight` | Highlighted/marked | `<mark>` |
| `superscript` | Superscript | `<sup>` |
| `subscript` | Subscript | `<sub>` |
| `mention` | User/entity mention | Custom span |
| `comment` | Inline comment | Annotation |

---

## Content Schema Definition

Add `contentSchema` for reusable schema definitions:

```yaml
contentSchema:
  name: <string>
  version: <vN>
  description: <string>
  
  extends: <schema_ref>                  # Inherit from another schema
  
  blocks:
    - type: <block_type>
      attributes:                        # Block-level attributes
        <attr_name>:
          type: <type>
          required: true | false
          values: [<value>, ...]         # For enums
      children:
        allowed: [<block_type>, ...]
        text: true | false               # Can contain text
        
  marks:
    - type: <mark_type>
      attributes:
        <attr_name>:
          type: <type>
          required: true | false
          
  # Custom blocks
  custom_blocks:
    - type: <custom_name>
      attributes: { ... }
      children: { ... }
      presentation:
        display_as: <hint>
```

---

## Predefined Schemas

```yaml
# Minimal - basic text formatting
contentSchema:
  name: minimal
  version: v1
  blocks:
    - type: paragraph
  marks:
    - type: bold
    - type: italic
    - type: link

# Article - blog/article content
contentSchema:
  name: article
  version: v1
  blocks:
    - type: paragraph
    - type: heading
      attributes:
        level: { type: integer, values: [1, 2, 3, 4] }
    - type: list
    - type: blockquote
    - type: code_block
    - type: image
    - type: divider
  marks:
    - type: bold
    - type: italic
    - type: link
    - type: code

# Documentation - technical docs
contentSchema:
  name: documentation
  version: v1
  extends: article
  blocks:
    - type: callout
      attributes:
        variant: { type: enum, values: [info, warning, danger, tip] }
    - type: table
    - type: toggle
  marks:
    - type: highlight
    - type: mention

# Comment - simple comments
contentSchema:
  name: comment
  version: v1
  blocks:
    - type: paragraph
  marks:
    - type: bold
    - type: italic
    - type: link
    - type: mention
  constraints:
    max_length: 5000
    max_blocks: 20
```

---

## Presentation View Integration

Extend `presentationView` for structured content display:

```yaml
presentationView:
  name: ArticleView
  version: v1
  
  consumes:
    dataState: ArticleState
    
  fields:
    content:
      presentation:
        display_as: structured_content
        
        structured_content:
          mode: view | edit              # Display mode
          
          # View mode options
          view:
            heading_anchors: true        # Add anchor links to headings
            code_highlighting: true      # Syntax highlighting
            link_previews: true          # Show link previews
            table_of_contents: true      # Generate TOC
            lazy_images: true            # Lazy load images
            
          # Edit mode options
          edit:
            toolbar: full | minimal | none
            placeholder: <string>
            autosave:
              enabled: true
              interval: 30s
            word_count: true
            character_count: true
            
          # Rendering customization
          rendering:
            paragraph:
              spacing: normal | compact | spacious
            heading:
              numbering: none | auto | custom
            code_block:
              line_numbers: true
              copy_button: true
            image:
              lightbox: true
              captions: true
```

---

## Semantic Guarantees

When a field declares `type: structured_content`:

1. **Schema Compliance**: Content MUST validate against declared schema
2. **Block Validity**: Only allowed blocks MUST be accepted
3. **Mark Validity**: Only allowed marks MUST be accepted
4. **Nesting Compliance**: Nesting rules MUST be enforced
5. **Constraint Enforcement**: Length/count limits MUST be validated
6. **Outline Integrity**: If `outline: true`, heading hierarchy MUST be valid

---

## Runtime Responsibilities

The runtime MUST:

1. **Validate Content**: Reject content not matching schema
2. **Enforce Constraints**: Apply length, block count limits
3. **Serialize Correctly**: Store in schema-compliant format
4. **Render Appropriately**: Display blocks and marks correctly

The runtime MAY:

1. Choose any editor framework (Slate, ProseMirror, TipTap, etc.)
2. Implement custom toolbar and input mechanisms
3. Provide collaborative editing support
4. Optimize rendering for performance
5. Support markdown import/export

---

## Examples

### Example 1: Blog Article

```yaml
dataType:
  name: BlogPost
  version: v1
  fields:
    title:
      type: string
      constraints:
        max_length: 200
        
    excerpt:
      type: structured_content
      structured_content:
        schema: minimal
        constraints:
          max_length: 500
          max_blocks: 3
          
    content:
      type: structured_content
      structured_content:
        schema: article
        
        blocks:
          allowed: [paragraph, heading, list, blockquote, code_block, image, divider]
          heading:
            max_count: 20
            
        marks:
          allowed: [bold, italic, link, code, highlight]
          
        nesting:
          max_depth: 3
          rules:
            list:
              can_contain: [list_item]
            list_item:
              can_contain: [paragraph, list]
              
        constraints:
          max_length: 50000
          max_blocks: 500
          
        structure:
          outline: true
          
        links:
          internal:
            enabled: true
            targets: [BlogPost, Author]
          external:
            enabled: true
            validate: true
            
        embeds:
          allowed: [image, video]
          image:
            max_size: 5MB
            mime_types: [image/png, image/jpeg, image/webp]
```

### Example 2: Documentation Page

```yaml
dataType:
  name: DocumentationPage
  version: v1
  fields:
    title:
      type: string
      
    content:
      type: structured_content
      structured_content:
        schema: documentation
        
        blocks:
          allowed: [paragraph, heading, list, code_block, table, callout, toggle, image]
          
        marks:
          allowed: [bold, italic, code, link, highlight]
          
        nesting:
          max_depth: 4
          rules:
            toggle:
              can_contain: [paragraph, list, code_block, callout]
              
        structure:
          outline: true
          sections: true
          
        embeds:
          allowed: [image, code_sandbox]
```

### Example 3: User Comment

```yaml
dataType:
  name: Comment
  version: v1
  fields:
    author_id:
      type: uuid
      
    content:
      type: structured_content
      structured_content:
        schema: comment
        
        blocks:
          allowed: [paragraph]
          forbidden: [heading, image, video, embed]
          
        marks:
          allowed: [bold, italic, link, mention]
          
        constraints:
          max_length: 2000
          max_blocks: 10
          
        links:
          internal:
            enabled: true
            targets: [User]
          external:
            enabled: true
```

### Example 4: Product Description

```yaml
dataType:
  name: Product
  version: v1
  fields:
    name:
      type: string
      
    short_description:
      type: structured_content
      structured_content:
        blocks:
          allowed: [paragraph]
        marks:
          allowed: [bold, italic]
        constraints:
          max_length: 500
          max_blocks: 2
          
    full_description:
      type: structured_content
      structured_content:
        blocks:
          allowed: [paragraph, heading, list, table, image]
          heading:
            max_count: 5
        marks:
          allowed: [bold, italic, link]
        constraints:
          max_length: 10000
        embeds:
          allowed: [image]
          image:
            max_size: 2MB
```

---

## Integration with Existing Grammar

### Extends

- `dataType.fields` - Adds `structured_content` type
- `presentationView` - Adds structured content presentation
- `constraint` - Schema acts as content constraint

### References

- `14-asset-semantics.md` - Embedded assets use asset semantics
- `15-search-semantics.md` - Content can be full-text indexed
- `18-collaborative-session-semantics.md` - Content can be collaboratively edited

### Complements

- `10-presentation-hints.md` - Content blocks can have presentation hints
- `12-accessibility-preferences.md` - Content accessibility requirements
- `01-form-intent-binding.md` - Content fields in forms

---

## Validation Criteria

A conformant implementation MUST:

1. **Parse** structured content schema without error
2. **Validate** content against declared schema
3. **Enforce** block and mark restrictions
4. **Apply** nesting rules correctly
5. **Enforce** length and count constraints

A conformant implementation SHOULD:

1. Provide schema-aware editor
2. Support content migration between schema versions
3. Provide markdown import/export
4. Support content search

---

## EBNF Grammar Addition

```ebnf
structured_content ::= "type:" "structured_content" NEWLINE
                       "structured_content:" NEWLINE INDENT
                       ( "schema:" identifier NEWLINE )?
                       content_blocks?
                       content_marks?
                       content_nesting?
                       content_constraints?
                       content_structure?
                       content_links?
                       content_embeds?
                       content_localization?
                       DEDENT

content_blocks ::= "blocks:" NEWLINE INDENT
                   ( "allowed:" "[" block_type_list "]" NEWLINE )?
                   ( "required:" "[" block_type_list "]" NEWLINE )?
                   ( "forbidden:" "[" block_type_list "]" NEWLINE )?
                   block_constraints*
                   DEDENT

block_type ::= "paragraph" | "heading" | "list" | "list_item"
             | "blockquote" | "code_block" | "image" | "video"
             | "table" | "divider" | "callout" | "toggle" | "embed"
             | identifier

content_marks ::= "marks:" NEWLINE INDENT
                  ( "allowed:" "[" mark_type_list "]" NEWLINE )?
                  ( "forbidden:" "[" mark_type_list "]" NEWLINE )?
                  DEDENT

mark_type ::= "bold" | "italic" | "underline" | "strikethrough"
            | "code" | "link" | "highlight" | "superscript"
            | "subscript" | "mention" | "comment" | identifier

content_nesting ::= "nesting:" NEWLINE INDENT
                    ( "max_depth:" integer NEWLINE )?
                    nesting_rules?
                    DEDENT

content_constraints ::= "constraints:" NEWLINE INDENT
                        ( "max_length:" integer NEWLINE )?
                        ( "max_blocks:" integer NEWLINE )?
                        ( "max_words:" integer NEWLINE )?
                        DEDENT

content_schema ::= "contentSchema:" NEWLINE INDENT
                   "name:" identifier NEWLINE
                   "version:" version NEWLINE
                   ( "description:" string NEWLINE )?
                   ( "extends:" identifier NEWLINE )?
                   schema_blocks?
                   schema_marks?
                   schema_custom_blocks?
                   DEDENT
```

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial proposal |

---

## ADDENDUM AQ: INFERENCE-DERIVED FIELDS (NEW in v1.5)

**Purpose**: ML/AI-Derived Field Values Without Coupling to Implementation  
**Extends**: `dataType.fields`, `materialization`, `search`  
**Priority**: LOW

---

## Problem Statement

The current specification defines:
- `materialization` for derived state from source events
- Field types for storing values
- Search semantics for indexing (Addendum 15)

However, there is no grammar for expressing:
1. Fields whose values are computed by machine learning models
2. Model version binding for reproducibility
3. Vector embeddings for semantic search
4. Inference freshness and caching semantics
5. Fallback behavior when inference is unavailable

**Impact**: Applications requiring ML-derived features (sentiment analysis, classification, embeddings) must implement model binding and inference entirely in runtime code without specification guidance.

---

## Design Principle Alignment

### Intent-First

This addendum expresses **what** inference means, not **how** to run models:

| Intent-Level (This Spec) | Runtime Decides |
|--------------------------|-----------------|
| `model.name: sentiment_classifier` | TensorFlow, PyTorch, ONNX, cloud API |
| `freshness.strategy: on_write` | Sync inference, async queue, batch job |
| `type: vector, dimensions: 384` | float32 array, specific embedding format |
| `fallback.on_unavailable: default` | Fallback implementation, cache lookup |

### Transport-Agnostic

The grammar does not specify:
- ML framework or library
- Model hosting infrastructure
- Inference API protocol
- Hardware (CPU, GPU, TPU)

### Materialization-Compatible

Inferred values are derived state:
- Computed from source fields
- Rebuildable if model changes
- Versioned alongside model versions
- Stale values are acceptable with freshness budgets

---

## Intent-Level Grammar

### Inference Field Annotation

Add `inference` block to field definitions for ML-derived values:

```yaml
dataType:
  name: <string>
  version: <vN>
  fields:
    <field_name>:
      type: <output_type>
      
      inference:                         # NEW: ML-derived value
        enabled: true | false
        
        source:
          fields: [<field>, ...]         # Input fields
          transform: <expression>        # Optional pre-processing
          
        model:
          name: <model_name>             # Logical model identifier
          version: <model_version>       # Specific model version
          version_policy: exact | latest | compatible
          
        output:
          type: <output_type>            # Result type
          confidence_field: <field>      # Where to store confidence
          metadata_field: <field>        # Where to store inference metadata
          
        freshness:
          strategy: on_write | on_read | batch | manual
          staleness_budget: <duration>   # How stale is acceptable
          cache_ttl: <duration>          # How long to cache
          
        batching:
          enabled: true | false
          max_batch_size: <integer>
          max_wait: <duration>
          
        fallback:
          on_unavailable: null | default | cached | error
          default_value: <value>
          on_error: null | default | cached | propagate
          
        constraints:
          min_confidence: <decimal>      # Reject below threshold
          max_latency: <duration>        # Timeout
```

### Vector Embedding Type

Add `vector` as a semantic field type for embeddings:

```yaml
dataType:
  name: <string>
  version: <vN>
  fields:
    <field_name>:
      type: vector                       # NEW: Vector embedding type
      
      vector:
        dimensions: <integer>            # Vector size (e.g., 384, 768, 1536)
        
        source:
          field: <field>                 # What to embed
          transform: <expression>        # Optional pre-processing
          
        model:
          name: <embedding_model>
          version: <version>
          
        normalization: none | l2 | unit  # Vector normalization
        
        similarity:
          metric: cosine | euclidean | dot_product
          
        search:
          indexed: true | false
          strategy: semantic             # Enables semantic search
          
        freshness:
          strategy: on_write | batch
          staleness_budget: <duration>
```

---

## Model Registry Reference

Add `modelReference` for logical model definitions:

```yaml
modelReference:
  name: <string>
  version: <vN>
  description: <string>
  
  type: classification | regression | embedding | generation | extraction
  
  input:
    schema:
      <field>: <type>
    preprocessing: <expression>
    
  output:
    schema:
      <field>: <type>
    postprocessing: <expression>
    
  performance:
    latency_p50: <duration>
    latency_p99: <duration>
    throughput: <requests_per_second>
    
  versioning:
    deprecation_policy: <duration>       # Warn before deprecation
    compatibility: strict | backward | forward
```

---

## Materialization Integration

Extend `materialization` for inference-aware view building:

```yaml
materialization:
  name: <string>
  source: <DataState>
  targetState: <DataState>
  
  inference:                             # NEW: Inference in materialization
    enabled: true | false
    
    fields:
      - source_field: <field>
        target_field: <field>
        model: <model_ref>
        
    execution:
      strategy: inline | deferred | batch
      batch_size: <integer>
      parallelism: <integer>
      
    failure_handling:
      on_inference_error: skip | retry | fail
      max_retries: <integer>
      
    rebuild:
      on_model_update: full | incremental
      trigger: manual | automatic
```

---

## Search Integration

Extend search for semantic/vector search:

```yaml
searchIntent:
  name: <string>
  version: <vN>
  
  searches: <DataState>
  
  query:
    type: text | semantic | hybrid       # NEW: Query types
    
    # For semantic search
    semantic:
      embedding_field: <field>           # Vector field to search
      model: <embedding_model>           # Model for query embedding
      
    # For hybrid search
    hybrid:
      text_weight: <decimal>             # 0-1, weight for text match
      semantic_weight: <decimal>         # 0-1, weight for semantic
      
    similarity:
      threshold: <decimal>               # Minimum similarity score
      
  result:
    includes: [<field>, ...]
    score_field: <field>                 # Where to include similarity score
```

---

## Semantic Guarantees

When a field declares `inference`:

1. **Model Binding**: Inference MUST use declared model and version
2. **Freshness Compliance**: Values MUST be within staleness budget
3. **Fallback Behavior**: Fallback MUST be applied when inference unavailable
4. **Confidence Tracking**: If `confidence_field` declared, confidence MUST be stored
5. **Rebuild Capability**: Inference MUST be rebuildable from source fields

When a field declares `type: vector`:

1. **Dimension Match**: Vector MUST have declared dimensions
2. **Normalization**: Vector MUST be normalized per declared method
3. **Search Compatibility**: Vector MUST be searchable with declared metric

---

## Runtime Responsibilities

The runtime MUST:

1. **Invoke Models**: Call inference for declared fields
2. **Version Binding**: Use correct model version
3. **Apply Freshness**: Respect staleness budgets
4. **Handle Failures**: Apply fallback behavior
5. **Store Vectors**: Persist embeddings correctly

The runtime MAY:

1. Choose any ML framework or cloud API
2. Implement custom batching/caching
3. Optimize for GPU/TPU execution
4. Provide model A/B testing
5. Implement vector search with any engine

---

## Examples

### Example 1: Sentiment Classification

```yaml
dataType:
  name: CustomerFeedback
  version: v1
  fields:
    id:
      type: uuid
      
    feedback_text:
      type: string
      
    # Inferred sentiment
    sentiment:
      type: string
      inference:
        enabled: true
        source:
          fields: [feedback_text]
        model:
          name: sentiment_classifier
          version: v2
          version_policy: compatible
        output:
          type: enum
          values: [positive, neutral, negative]
          confidence_field: sentiment_confidence
        freshness:
          strategy: on_write
        fallback:
          on_unavailable: default
          default_value: neutral
          
    sentiment_confidence:
      type: decimal
      
    # Inferred category
    category:
      type: string
      inference:
        enabled: true
        source:
          fields: [feedback_text]
        model:
          name: feedback_categorizer
          version: v1
        output:
          type: enum
          values: [product, service, shipping, billing, other]
          confidence_field: category_confidence
        freshness:
          strategy: on_write
        fallback:
          on_unavailable: default
          default_value: other
          
    category_confidence:
      type: decimal
```

### Example 2: Semantic Search Embeddings

```yaml
dataType:
  name: Document
  version: v1
  fields:
    id:
      type: uuid
      
    title:
      type: string
      
    content:
      type: structured_content
      
    # Embedding for semantic search
    content_embedding:
      type: vector
      vector:
        dimensions: 768
        source:
          field: content
          transform: "extract_text(content)"
        model:
          name: document_embeddings
          version: v1
        normalization: l2
        similarity:
          metric: cosine
        search:
          indexed: true
          strategy: semantic
        freshness:
          strategy: on_write
          staleness_budget: 1h

searchIntent:
  name: SemanticDocumentSearch
  version: v1
  searches: DocumentState
  
  query:
    type: hybrid
    hybrid:
      text_weight: 0.3
      semantic_weight: 0.7
    semantic:
      embedding_field: content_embedding
      model: document_embeddings
    similarity:
      threshold: 0.7
      
  result:
    includes: [id, title, content]
    score_field: relevance_score
```

### Example 3: Entity Extraction

```yaml
dataType:
  name: Email
  version: v1
  fields:
    id:
      type: uuid
      
    body:
      type: string
      
    # Extracted entities
    mentioned_people:
      type: array
      items: { type: string }
      inference:
        enabled: true
        source:
          fields: [body]
        model:
          name: ner_extractor
          version: v1
        output:
          type: array
          filter: "entity.type == 'PERSON'"
        freshness:
          strategy: on_write
        fallback:
          on_unavailable: null
          
    mentioned_organizations:
      type: array
      items: { type: string }
      inference:
        enabled: true
        source:
          fields: [body]
        model:
          name: ner_extractor
          version: v1
        output:
          type: array
          filter: "entity.type == 'ORG'"
        freshness:
          strategy: on_write
```

### Example 4: Image Classification

```yaml
dataType:
  name: ProductImage
  version: v1
  fields:
    id:
      type: uuid
      
    image:
      type: asset
      asset:
        category: image
        constraints:
          max_size: 10MB
          mime_types: [image/jpeg, image/png]
          
    # Inferred tags
    auto_tags:
      type: array
      items: { type: string }
      inference:
        enabled: true
        source:
          fields: [image]
        model:
          name: image_tagger
          version: v2
        output:
          type: array
          max_items: 10
        freshness:
          strategy: on_write
        batching:
          enabled: true
          max_batch_size: 32
          max_wait: 5s
        fallback:
          on_unavailable: null
          
    # Image embedding for similarity search
    image_embedding:
      type: vector
      vector:
        dimensions: 512
        source:
          field: image
        model:
          name: image_encoder
          version: v1
        normalization: l2
        similarity:
          metric: cosine
        search:
          indexed: true
          strategy: semantic
```

### Example 5: Content Moderation

```yaml
dataType:
  name: UserPost
  version: v1
  fields:
    id:
      type: uuid
      
    content:
      type: string
      
    # Moderation inference
    moderation_result:
      type: object
      fields:
        is_safe: { type: boolean }
        categories: { type: array, items: { type: string } }
        severity: { type: string }
      inference:
        enabled: true
        source:
          fields: [content]
        model:
          name: content_moderator
          version: v1
        output:
          type: object
        freshness:
          strategy: on_write              # Must be immediate
        fallback:
          on_unavailable: error           # Block if moderation unavailable
          on_error: propagate
        constraints:
          max_latency: 500ms              # Must be fast
```

---

## Integration with Existing Grammar

### Extends

- `dataType.fields` - Adds `inference` annotation and `vector` type
- `materialization` - Adds inference-aware view building
- `searchIntent` - Adds semantic/hybrid search

### References

- `15-search-semantics.md` - Semantic search integration
- `14-asset-semantics.md` - Image/media inference
- `19-structured-content-type.md` - Content embedding

### Complements

- `02-view-materialization-contract.md` - Inference is derived state
- `16-scheduled-trigger-semantics.md` - Batch inference scheduling
- `10-presentation-hints.md` - Confidence display

---

## Validation Criteria

A conformant implementation MUST:

1. **Parse** inference and vector configurations without error
2. **Bind** to declared model versions
3. **Respect** freshness strategies
4. **Apply** fallback behavior
5. **Store** vectors with correct dimensions

A conformant implementation SHOULD:

1. Support multiple ML frameworks
2. Provide model version management
3. Implement efficient batching
4. Support vector search engines
5. Provide inference monitoring

---

## EBNF Grammar Addition

```ebnf
inference       ::= "inference:" NEWLINE INDENT
                    "enabled:" boolean NEWLINE
                    inference_source
                    inference_model
                    inference_output?
                    inference_freshness?
                    inference_batching?
                    inference_fallback?
                    inference_constraints?
                    DEDENT

inference_source ::= "source:" NEWLINE INDENT
                     "fields:" "[" field_list "]" NEWLINE
                     ( "transform:" expression NEWLINE )?
                     DEDENT

inference_model ::= "model:" NEWLINE INDENT
                    "name:" identifier NEWLINE
                    "version:" version NEWLINE
                    ( "version_policy:" version_policy NEWLINE )?
                    DEDENT

version_policy  ::= "exact" | "latest" | "compatible"

inference_output ::= "output:" NEWLINE INDENT
                     "type:" type_ref NEWLINE
                     ( "confidence_field:" identifier NEWLINE )?
                     ( "metadata_field:" identifier NEWLINE )?
                     DEDENT

inference_freshness ::= "freshness:" NEWLINE INDENT
                        "strategy:" freshness_strategy NEWLINE
                        ( "staleness_budget:" duration NEWLINE )?
                        ( "cache_ttl:" duration NEWLINE )?
                        DEDENT

freshness_strategy ::= "on_write" | "on_read" | "batch" | "manual"

inference_fallback ::= "fallback:" NEWLINE INDENT
                       ( "on_unavailable:" fallback_action NEWLINE )?
                       ( "default_value:" value NEWLINE )?
                       ( "on_error:" error_action NEWLINE )?
                       DEDENT

fallback_action ::= "null" | "default" | "cached" | "error"
error_action    ::= "null" | "default" | "cached" | "propagate"

vector_type     ::= "type:" "vector" NEWLINE
                    "vector:" NEWLINE INDENT
                    "dimensions:" integer NEWLINE
                    vector_source?
                    vector_model?
                    ( "normalization:" normalization NEWLINE )?
                    vector_similarity?
                    vector_search?
                    vector_freshness?
                    DEDENT

normalization   ::= "none" | "l2" | "unit"

vector_similarity ::= "similarity:" NEWLINE INDENT
                      "metric:" similarity_metric NEWLINE
                      DEDENT

similarity_metric ::= "cosine" | "euclidean" | "dot_product"
```

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial proposal |

---

## ADDENDUM AR: SPATIAL TYPE SEMANTICS (NEW in v1.5)

**Purpose**: Geographic and Spatial Data Types  
**Extends**: `dataType.fields`, `searchIntent`, `presentationView`, `constraint`  
**Priority**: LOW

---

## Problem Statement

The current specification defines:
- Basic field types (`string`, `decimal`, `integer`, etc.)
- Search semantics for text and semantic search (Addendum 15)
- Presentation hints for display (Addendum 10)

However, there is no grammar for expressing:
1. Geographic points (latitude/longitude coordinates)
2. Polygons and regions (delivery zones, service areas)
3. Spatial relationships (within, intersects, near)
4. Distance calculations and queries
5. Geofencing and boundary constraints

**Impact**: Applications requiring location-based features (maps, delivery, logistics, location-based services) must implement spatial logic entirely in runtime code without specification guidance.

---

## Design Principle Alignment

### Intent-First

This addendum expresses **what** spatial data means, not **how** to implement:

| Intent-Level (This Spec) | Runtime Decides |
|--------------------------|-----------------|
| `type: point` | PostGIS point, MongoDB 2dsphere, Elasticsearch geo_point |
| `constraint: bounds` | Geofencing implementation |
| `query: within_distance` | Haversine formula, spatial index, or external API |
| `presentation: map_marker` | Leaflet, Mapbox, Google Maps, static image |

### Transport-Agnostic

The grammar does not specify:
- Spatial database or index engine
- Map rendering library
- Coordinate transformation library
- Geocoding/reverse geocoding service

### Type-System Extension

Spatial types extend the type system naturally:
- Like `timestamp` represents time semantically
- Like `decimal` represents precise numbers
- Spatial types represent geographic data semantically

---

## Intent-Level Grammar

### Spatial Point Type

Add `point` as a semantic field type:

```yaml
dataType:
  name: <string>
  version: <vN>
  fields:
    <field_name>:
      type: point                        # NEW: Geographic point
      
      point:
        coordinate_system: wgs84 | web_mercator | custom
        
        precision:
          latitude: <decimal_places>     # Default 6 (~10cm)
          longitude: <decimal_places>
          
        constraints:
          bounds:                        # Geofence constraint
            type: polygon | circle | bbox
            coordinates: [...]           # For polygon/bbox
            center: { lat: <>, lng: <> } # For circle
            radius: <distance>           # For circle
            
          required: true | false
          
        search:
          indexed: true | false
          strategy: spatial
          
        presentation:
          display_as: map_marker | coordinates | address
          format: decimal | dms          # Degrees Minutes Seconds
```

### Spatial Polygon Type

Add `polygon` as a semantic field type:

```yaml
dataType:
  name: <string>
  version: <vN>
  fields:
    <field_name>:
      type: polygon                      # NEW: Geographic polygon
      
      polygon:
        coordinate_system: wgs84 | web_mercator | custom
        
        constraints:
          max_vertices: <integer>        # Complexity limit
          max_area: <area>               # Size limit
          min_area: <area>
          must_be_convex: true | false
          no_self_intersection: true | false
          
        search:
          indexed: true | false
          strategy: spatial
          
        presentation:
          display_as: region | outline | heatmap
          fill_opacity: <decimal>        # 0-1
          stroke_width: <integer>
```

### Spatial Circle Type

Add `circle` as a semantic field type for radius-based regions:

```yaml
dataType:
  name: <string>
  version: <vN>
  fields:
    <field_name>:
      type: circle                       # NEW: Circle region
      
      circle:
        center:
          from_field: <point_field>      # Dynamic center
          # OR
          fixed: { lat: <>, lng: <> }    # Static center
          
        radius:
          from_field: <field>            # Dynamic radius
          # OR
          fixed: <distance>              # Static radius
          unit: m | km | mi | ft
          
        constraints:
          max_radius: <distance>
          min_radius: <distance>
          
        search:
          indexed: true | false
          strategy: spatial
          
        presentation:
          display_as: circle | heatmap
```

### Spatial Line Type

Add `linestring` for paths and routes:

```yaml
dataType:
  name: <string>
  version: <vN>
  fields:
    <field_name>:
      type: linestring                   # NEW: Geographic line/path
      
      linestring:
        coordinate_system: wgs84 | web_mercator
        
        constraints:
          max_points: <integer>          # Complexity limit
          max_length: <distance>         # Total length
          
        presentation:
          display_as: path | route | polyline
          stroke_width: <integer>
```

---

## Distance and Area Units

```yaml
# Distance units
distance ::= <number> <unit>
unit     ::= m | km | mi | ft | nm      # Nautical miles

# Area units
area     ::= <number> <area_unit>
area_unit ::= sqm | sqkm | sqmi | acre | hectare
```

---

## Spatial Search Semantics

Extend `searchIntent` for spatial queries:

```yaml
searchIntent:
  name: <string>
  version: <vN>
  searches: <DataState>
  
  query:
    spatial:                             # NEW: Spatial query
      field: <spatial_field>
      
      type: within_distance | within_polygon | within_bbox | 
            intersects | contains | nearest
            
      # For within_distance
      within_distance:
        from: <expression>               # Origin point
        max_distance: <distance>
        min_distance: <distance>         # Optional donut query
        
      # For within_polygon
      within_polygon:
        polygon: <expression>            # Polygon definition
        
      # For within_bbox (bounding box)
      within_bbox:
        min_lat: <expression>
        max_lat: <expression>
        min_lng: <expression>
        max_lng: <expression>
        
      # For intersects
      intersects:
        geometry: <expression>           # Any geometry
        
      # For nearest (k-nearest neighbors)
      nearest:
        to: <expression>                 # Origin point
        limit: <integer>                 # How many to return
        max_distance: <distance>         # Optional cap
        
      sort:
        by_distance_from: <expression>
        direction: asc | desc
        
  result:
    includes: [<field>, ...]
    distance_field: <field>              # Include distance in result
```

---

## Spatial Constraints

Add spatial constraint types:

```yaml
constraint:
  name: <string>
  type: spatial                          # NEW: Spatial constraint
  
  spatial:
    # Within bounds
    within_bounds:
      field: <point_field>
      bounds: <polygon_ref | polygon_def>
      
    # Minimum distance between entities
    min_distance:
      from_field: <point_field>
      to: <expression>                   # Other entity's point
      distance: <distance>
      
    # Maximum distance from reference
    max_distance:
      from_field: <point_field>
      to: <expression>
      distance: <distance>
      
    # Non-overlapping regions
    no_overlap:
      field: <polygon_field>
      with: <expression>                 # Other polygons
```

---

## Geocoding Integration

Add `geocoding` for address-to-coordinate conversion:

```yaml
dataType:
  name: <string>
  version: <vN>
  fields:
    address:
      type: string
      
    location:
      type: point
      
      geocoding:                         # NEW: Geocoding derivation
        enabled: true | false
        
        source:
          address_field: <field>         # Field containing address
          # OR structured address
          structured:
            street: <field>
            city: <field>
            state: <field>
            postal_code: <field>
            country: <field>
            
        freshness:
          strategy: on_write | on_read | manual
          cache_ttl: <duration>
          
        fallback:
          on_unavailable: null | default | cached
          default_value: { lat: 0, lng: 0 }
          
        reverse:
          enabled: true | false
          target_field: <field>          # Store reverse geocode result
```

---

## Presentation View Integration

Extend `presentationView` for map display:

```yaml
presentationView:
  name: <string>
  version: <vN>
  
  fields:
    location:
      presentation:
        display_as: map
        
        map:
          type: marker | region | heatmap | cluster
          
          marker:
            icon: <icon_ref>
            color: <expression>          # Dynamic color
            label: <expression>          # Dynamic label
            popup: <field_list>          # Fields in popup
            
          region:
            fill_color: <expression>
            fill_opacity: <decimal>
            stroke_color: <expression>
            stroke_width: <integer>
            
          heatmap:
            weight_field: <field>
            radius: <integer>
            intensity: <decimal>
            
          cluster:
            enabled: true | false
            min_zoom: <integer>
            max_zoom: <integer>
            
          controls:
            zoom: true | false
            pan: true | false
            fullscreen: true | false
            layers: true | false
            
          initial_view:
            center: auto | { lat: <>, lng: <> }
            zoom: <integer>
            fit_bounds: true | false
```

---

## Semantic Guarantees

When a field declares a spatial type (`point`, `polygon`, `circle`, `linestring`):

1. **Coordinate Validity**: Coordinates MUST be valid for declared system
2. **Constraint Compliance**: Values MUST satisfy declared constraints
3. **Search Compatibility**: If indexed, MUST be searchable spatially
4. **Precision Compliance**: Coordinates MUST respect declared precision

When a spatial query is executed:

1. **Distance Accuracy**: Distance calculations MUST be accurate to within declared precision
2. **Containment Accuracy**: Within/contains MUST handle edge cases correctly
3. **Result Ordering**: If sorted by distance, MUST be correctly ordered

---

## Runtime Responsibilities

The runtime MUST:

1. **Validate Coordinates**: Ensure valid lat/lng ranges
2. **Enforce Constraints**: Apply bounds, area limits, etc.
3. **Execute Queries**: Perform spatial queries correctly
4. **Calculate Distances**: Use appropriate geodesic math

The runtime MAY:

1. Choose any spatial database (PostGIS, MongoDB, Elasticsearch)
2. Use any map rendering library
3. Implement custom spatial indexing
4. Integrate with geocoding services
5. Optimize with spatial caches

---

## Examples

### Example 1: Store Locator

```yaml
dataType:
  name: Store
  version: v1
  fields:
    id:
      type: uuid
      
    name:
      type: string
      
    address:
      type: string
      
    location:
      type: point
      point:
        coordinate_system: wgs84
        precision:
          latitude: 6
          longitude: 6
        search:
          indexed: true
          strategy: spatial
        presentation:
          display_as: map_marker
          
      geocoding:
        enabled: true
        source:
          address_field: address
        freshness:
          strategy: on_write
          cache_ttl: 30d
          
    delivery_zone:
      type: polygon
      polygon:
        coordinate_system: wgs84
        constraints:
          max_vertices: 100
          no_self_intersection: true
        search:
          indexed: true
          strategy: spatial
        presentation:
          display_as: region
          fill_opacity: 0.3

searchIntent:
  name: NearbyStores
  version: v1
  searches: StoreState
  
  query:
    spatial:
      field: location
      type: within_distance
      within_distance:
        from: "request.user_location"
        max_distance: 50km
      sort:
        by_distance_from: "request.user_location"
        direction: asc
        
  result:
    includes: [id, name, address, location]
    distance_field: distance_km
```

### Example 2: Delivery Service

```yaml
dataType:
  name: DeliveryZone
  version: v1
  fields:
    id:
      type: uuid
      
    name:
      type: string
      
    zone:
      type: polygon
      polygon:
        coordinate_system: wgs84
        constraints:
          max_vertices: 200
        search:
          indexed: true
          
    base_fee:
      type: decimal
      
    max_radius:
      type: circle
      circle:
        center:
          from_field: zone.centroid     # Computed center
        radius:
          fixed: 25km

# Check if address is within delivery zone
searchIntent:
  name: CheckDeliveryAvailability
  version: v1
  searches: DeliveryZoneState
  
  query:
    spatial:
      field: zone
      type: contains
      contains:
        point: "request.delivery_address"
        
  result:
    includes: [id, name, base_fee]
```

### Example 3: Fleet Tracking

```yaml
dataType:
  name: VehicleLocation
  version: v1
  
  lifecycle: append_only               # Location history
  
  fields:
    vehicle_id:
      type: uuid
      
    timestamp:
      type: timestamp
      
    location:
      type: point
      point:
        coordinate_system: wgs84
        precision:
          latitude: 7                   # ~1cm precision
          longitude: 7
        search:
          indexed: true
          
    speed:
      type: decimal
      
    heading:
      type: decimal                     # Degrees
      
    route:
      type: linestring
      linestring:
        coordinate_system: wgs84
        constraints:
          max_points: 10000
        presentation:
          display_as: path

# Geofence alert view
materializedView:
  name: GeofenceAlerts
  version: v1
  
  source: VehicleLocationState
  
  query:
    spatial:
      field: location
      type: within_polygon
      within_polygon:
        polygon: "NOT restricted_zones"  # Outside allowed area
```

### Example 4: Real Estate Listings

```yaml
dataType:
  name: Property
  version: v1
  fields:
    id:
      type: uuid
      
    address:
      type: string
      
    location:
      type: point
      point:
        coordinate_system: wgs84
        search:
          indexed: true
        presentation:
          display_as: map_marker
          
    price:
      type: decimal
      
    lot_boundary:
      type: polygon
      polygon:
        coordinate_system: wgs84
        constraints:
          max_vertices: 50
        presentation:
          display_as: region

# Search with bounding box (map viewport)
searchIntent:
  name: PropertiesInView
  version: v1
  searches: PropertyState
  
  query:
    spatial:
      field: location
      type: within_bbox
      within_bbox:
        min_lat: "request.viewport.sw.lat"
        max_lat: "request.viewport.ne.lat"
        min_lng: "request.viewport.sw.lng"
        max_lng: "request.viewport.ne.lng"
        
  result:
    includes: [id, address, price, location]
```

### Example 5: Ride-Sharing

```yaml
dataType:
  name: DriverLocation
  version: v1
  fields:
    driver_id:
      type: uuid
      
    location:
      type: point
      point:
        coordinate_system: wgs84
        search:
          indexed: true
          
    status:
      type: enum
      values: [available, busy, offline]
      
    vehicle_type:
      type: enum
      values: [economy, premium, xl]

# Find nearest available drivers
searchIntent:
  name: NearestDrivers
  version: v1
  searches: DriverLocationState
  
  query:
    filter:
      status: available
      vehicle_type: "request.vehicle_type"
      
    spatial:
      field: location
      type: nearest
      nearest:
        to: "request.pickup_location"
        limit: 10
        max_distance: 10km
        
  result:
    includes: [driver_id, location, vehicle_type]
    distance_field: eta_meters
```

---

## Integration with Existing Grammar

### Extends

- `dataType.fields` - Adds spatial types
- `searchIntent` - Adds spatial queries
- `presentationView` - Adds map display
- `constraint` - Adds spatial constraints

### References

- `15-search-semantics.md` - Spatial extends search
- `10-presentation-hints.md` - Map presentation hints
- `20-inference-derived-fields.md` - Geocoding as inference

### Complements

- `01-form-intent-binding.md` - Location picker in forms
- `17-notification-channel-semantics.md` - Location-based notifications
- `16-scheduled-trigger-semantics.md` - Geofence triggers

---

## Validation Criteria

A conformant implementation MUST:

1. **Parse** spatial type definitions without error
2. **Validate** coordinate ranges (lat: -90 to 90, lng: -180 to 180)
3. **Enforce** spatial constraints (bounds, max_vertices, etc.)
4. **Execute** spatial queries correctly
5. **Calculate** distances accurately

A conformant implementation SHOULD:

1. Support spatial indexing for performance
2. Provide map visualization components
3. Integrate with geocoding services
4. Support multiple coordinate systems
5. Handle dateline and pole edge cases

---

## EBNF Grammar Addition

```ebnf
spatial_type    ::= point_type | polygon_type | circle_type | linestring_type

point_type      ::= "type:" "point" NEWLINE
                    "point:" NEWLINE INDENT
                    ( "coordinate_system:" coord_system NEWLINE )?
                    point_precision?
                    point_constraints?
                    spatial_search?
                    spatial_presentation?
                    DEDENT

polygon_type    ::= "type:" "polygon" NEWLINE
                    "polygon:" NEWLINE INDENT
                    ( "coordinate_system:" coord_system NEWLINE )?
                    polygon_constraints?
                    spatial_search?
                    spatial_presentation?
                    DEDENT

circle_type     ::= "type:" "circle" NEWLINE
                    "circle:" NEWLINE INDENT
                    circle_center
                    circle_radius
                    circle_constraints?
                    spatial_search?
                    spatial_presentation?
                    DEDENT

linestring_type ::= "type:" "linestring" NEWLINE
                    "linestring:" NEWLINE INDENT
                    ( "coordinate_system:" coord_system NEWLINE )?
                    linestring_constraints?
                    spatial_presentation?
                    DEDENT

coord_system    ::= "wgs84" | "web_mercator" | "custom"

spatial_query   ::= "spatial:" NEWLINE INDENT
                    "field:" identifier NEWLINE
                    "type:" spatial_query_type NEWLINE
                    spatial_query_params
                    spatial_sort?
                    DEDENT

spatial_query_type ::= "within_distance" | "within_polygon" | "within_bbox"
                     | "intersects" | "contains" | "nearest"

distance        ::= number distance_unit
distance_unit   ::= "m" | "km" | "mi" | "ft" | "nm"

area            ::= number area_unit
area_unit       ::= "sqm" | "sqkm" | "sqmi" | "acre" | "hectare"
```

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial proposal |

---

## ADDENDUM AS: GRAPH QUERY SEMANTICS (NEW in v1.5)

**Purpose**: Graph Traversal and Path Queries Without Prescribing Graph Database  
**Extends**: `relationship`, `searchIntent`, `materialization`  
**Priority**: MEDIUM

---

## Problem Statement

The current specification defines:
- `relationship` grammar with cardinality (`one-to-one`, `one-to-many`, `many-to-many`)
- `searchIntent` for field-based discovery
- `materialization` for derived views

However, there is no grammar for expressing:
1. Graph traversal patterns (paths, connected components)
2. Recursive/self-referential relationships (hierarchies, org charts)
3. Multi-hop queries (2nd-degree connections, influence paths)
4. Graph aggregations (shortest path, degree counting)

**Impact**: Applications requiring social graphs, recommendation engines, knowledge graphs, or dependency analysis must implement traversal entirely in runtime code without specification guidance.

---

## Design Principle Alignment

### Intent-First

This addendum expresses **what** graph query means, not **how** to traverse:

| Intent-Level (This Spec) | Runtime Decides |
|--------------------------|-----------------|
| `pattern.type: path` | Recursive CTE, Cypher, Gremlin, custom |
| `max_hops: 10` | Query depth limit, cycle detection |
| `direction: up` | Edge traversal implementation |
| `aggregation: shortest_path` | Dijkstra, BFS, graph library |

### Transport-Agnostic

The grammar does not specify:
- Graph database or engine (Neo4j, ArangoDB, Dgraph, SQL recursive CTE)
- Query language (Cypher, Gremlin, SPARQL)
- Index structure (adjacency list, edge table)
- Traversal algorithm

### Entity-Scoped

Graph queries respect entity authority:
- Each node is an entity with its own authority
- Traversal reads from materialized views
- No cross-entity write coordination implied

---

## Intent-Level Grammar

### Recursive Relationship Extension

Extend `relationship` for self-referential semantics:

```yaml
relationship:
  name: <string>
  from: <model_ref>
  to: <model_ref>
  type: <identifier>
  cardinality: one-to-one | one-to-many | many-to-one | many-to-many
  
  recursion:                             # NEW: Self-referential semantics
    enabled: true | false
    max_depth: <integer>                 # Safety limit
    direction: up | down | both          # Traversal direction hint
    cycle_handling: detect | allow | error
```

### Graph Query Definition

Add `graphQuery` as a specialized search intent:

```yaml
graphQuery:
  name: <string>
  version: <vN>
  description: <string>
  
  traverses: <relationship_ref>          # Which relationship to traverse
  
  start:
    from: <dataState_ref>
    where: <expression>                  # Starting node filter
    
  pattern:
    type: path | neighbors | subgraph | connected_component | shortest_path
    
    # For path queries
    path:
      direction: up | down | both | any
      stop_when: <expression>            # Termination condition
      max_hops: <integer>
      min_hops: <integer>
      filter: <expression>               # Per-hop filter
      
    # For neighbor queries
    neighbors:
      depth: <integer>                   # How many hops
      direction: outbound | inbound | both
      
    # For shortest path
    shortest_path:
      to: <expression>                   # Target node filter
      weight_field: <field>              # Edge weight (optional)
      max_cost: <number>                 # Cost limit
      
  result:
    includes: [<field>, ...]
    path_field: <field>                  # Include traversal path
    depth_field: <field>                 # Include hop count
    
  aggregation:                           # Graph aggregations
    - type: count | sum | avg | collect
      field: <field>
      as: <result_field>
      
  limits:
    max_results: <integer>
    timeout: <duration>
```

### Graph Materialization

Extend `materialization` for graph-derived views:

```yaml
materialization:
  name: <string>
  source: <relationship_ref>             # Can source from relationship
  targetState: <DataState>
  
  graph:                                 # NEW: Graph materialization
    type: closure | degree | centrality | community
    
    # Transitive closure
    closure:
      relationship: <relationship_ref>
      max_depth: <integer>
      
    # Degree calculation  
    degree:
      relationship: <relationship_ref>
      direction: in | out | both
      
    # Centrality metrics
    centrality:
      algorithm: pagerank | betweenness | closeness
      damping: <decimal>                 # For PageRank
      iterations: <integer>
      
    # Community detection
    community:
      algorithm: louvain | label_propagation
      resolution: <decimal>
```

---

## Semantic Guarantees

When a `graphQuery` is declared:

1. **Depth Limiting**: Queries MUST respect `max_hops` to prevent runaway traversal
2. **Cycle Safety**: Recursive traversal MUST handle cycles per `cycle_handling`
3. **Result Ordering**: Path queries MUST return nodes in traversal order
4. **Timeout Enforcement**: Long traversals MUST respect `timeout`

When a `relationship.recursion` is declared:

1. **Self-Reference**: `from` and `to` MAY reference the same model
2. **Direction Clarity**: `direction` indicates parent/child semantics
3. **Finite Depth**: Traversal MUST terminate at `max_depth`

---

## Runtime Responsibilities

The runtime MUST:

1. **Traverse Relationships**: Follow declared relationship for queries
2. **Enforce Limits**: Respect max_hops, max_results, timeout
3. **Handle Cycles**: Implement cycle detection per policy
4. **Return Structure**: Include path/depth information as declared

The runtime MAY:

1. Choose any graph database or query engine
2. Implement custom indexing for performance
3. Cache transitive closures
4. Parallelize traversal

---

## Examples

### Example 1: Organizational Hierarchy

```yaml
relationship:
  name: Reports_To
  from: EmployeeState
  to: EmployeeState
  type: hierarchical
  cardinality: many-to-one
  semantics:
    causal: true
  recursion:
    enabled: true
    max_depth: 15
    direction: up
    cycle_handling: error

graphQuery:
  name: ManagementChainQuery
  version: v1
  description: "Find all managers above an employee"
  
  traverses: Reports_To
  
  start:
    from: EmployeeState
    where: "employee.id == request.employee_id"
    
  pattern:
    type: path
    path:
      direction: up
      stop_when: "employee.role == 'CEO'"
      max_hops: 15
      
  result:
    includes: [id, name, role, title]
    depth_field: management_level
    path_field: chain
```

### Example 2: Social Network Connections

```yaml
relationship:
  name: Follows
  from: UserState
  to: UserState
  type: social
  cardinality: many-to-many
  recursion:
    enabled: true
    max_depth: 6
    direction: both
    cycle_handling: detect

graphQuery:
  name: MutualConnectionsQuery
  version: v1
  description: "Find mutual connections between two users"
  
  traverses: Follows
  
  start:
    from: UserState
    where: "user.id == request.user_a_id"
    
  pattern:
    type: neighbors
    neighbors:
      depth: 1
      direction: outbound
      
  result:
    includes: [id, name, avatar]
    
  # Post-filter for mutual
  filter: "user.id IN followers_of(request.user_b_id)"
```

### Example 3: Shortest Path Finding

```yaml
graphQuery:
  name: ShortestRouteQuery
  version: v1
  description: "Find shortest path between locations"
  
  traverses: Connected_To
  
  start:
    from: LocationState
    where: "location.id == request.origin_id"
    
  pattern:
    type: shortest_path
    shortest_path:
      to: "location.id == request.destination_id"
      weight_field: distance_km
      max_cost: 1000
      
  result:
    includes: [id, name, coordinates]
    path_field: route
    
  aggregation:
    - type: sum
      field: distance_km
      as: total_distance
```

### Example 4: Transitive Closure Materialization

```yaml
materialization:
  name: AncestorView
  source: Reports_To
  targetState: AncestorState
  
  graph:
    type: closure
    closure:
      relationship: Reports_To
      max_depth: 15
      
  freshness:
    maxStaleness: 1h
    
  evolution:
    strategy: incremental
```

### Example 5: PageRank Materialization

```yaml
materialization:
  name: UserInfluenceView
  source: Follows
  targetState: UserInfluenceState
  
  graph:
    type: centrality
    centrality:
      algorithm: pagerank
      damping: 0.85
      iterations: 20
      
  freshness:
    maxStaleness: 24h
    
  evolution:
    strategy: rebuild
```

---

## Integration with Existing Grammar

### Extends

- `relationship` - Adds `recursion` block for self-referential
- `searchIntent` - `graphQuery` is a specialized search type
- `materialization` - Adds `graph` block for graph-derived views

### References

- `authority` - Graph nodes respect entity authority
- `policy` - Graph queries are policy-controlled
- `fieldPermissions` - Node field visibility respected

### Complements

- `15-search-semantics.md` - Graph + text search hybrid
- `21-spatial-type-semantics.md` - Geospatial graph (routes, networks)
- `02-view-materialization-contract.md` - Graph view retrieval

---

## Validation Criteria

A conformant implementation MUST:

1. **Parse** graphQuery and recursion definitions without error
2. **Traverse** relationships following declared pattern
3. **Enforce** depth and timeout limits
4. **Detect** cycles when `cycle_handling: detect` or `error`
5. **Return** path/depth information as declared

A conformant implementation SHOULD:

1. Optimize recursive queries with indexing
2. Support incremental closure updates
3. Provide query explain/planning
4. Cache common traversal patterns

---

## EBNF Grammar Addition

```ebnf
relationship_recursion ::= "recursion:" NEWLINE INDENT
                           "enabled:" boolean NEWLINE
                           ( "max_depth:" integer NEWLINE )?
                           ( "direction:" recursion_direction NEWLINE )?
                           ( "cycle_handling:" cycle_policy NEWLINE )?
                           DEDENT

recursion_direction    ::= "up" | "down" | "both"
cycle_policy           ::= "detect" | "allow" | "error"

graph_query            ::= "graphQuery:" NEWLINE INDENT
                           "name:" identifier NEWLINE
                           "version:" version NEWLINE
                           ( "description:" string NEWLINE )?
                           graph_traverses
                           graph_start
                           graph_pattern
                           graph_result?
                           graph_aggregation?
                           graph_limits?
                           DEDENT

graph_traverses        ::= "traverses:" relationship_ref NEWLINE

graph_start            ::= "start:" NEWLINE INDENT
                           "from:" data_state_ref NEWLINE
                           "where:" expression NEWLINE
                           DEDENT

graph_pattern          ::= "pattern:" NEWLINE INDENT
                           "type:" pattern_type NEWLINE
                           ( path_pattern | neighbors_pattern | shortest_path_pattern )?
                           DEDENT

pattern_type           ::= "path" | "neighbors" | "subgraph" | 
                           "connected_component" | "shortest_path"

path_pattern           ::= "path:" NEWLINE INDENT
                           ( "direction:" path_direction NEWLINE )?
                           ( "stop_when:" expression NEWLINE )?
                           ( "max_hops:" integer NEWLINE )?
                           ( "min_hops:" integer NEWLINE )?
                           ( "filter:" expression NEWLINE )?
                           DEDENT

path_direction         ::= "up" | "down" | "both" | "any"

graph_materialization  ::= "graph:" NEWLINE INDENT
                           "type:" graph_mat_type NEWLINE
                           ( closure_config | degree_config | 
                             centrality_config | community_config )?
                           DEDENT

graph_mat_type         ::= "closure" | "degree" | "centrality" | "community"
```

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial proposal |

---

## ADDENDUM AT: DURABLE WORKFLOW SEMANTICS (NEW in v1.5)

**Purpose**: Long-Running Process Orchestration with Human Tasks  
**Extends**: `smsFlow`, `inputIntent`, `dataState`  
**Priority**: MEDIUM

---

## Problem Statement

The current specification defines:
- `smsFlow` for workflow orchestration with steps and atomic groups
- `workUnitContract` for typed work unit specifications
- `compensation` for failure recovery

However, there is no grammar for expressing:
1. Workflows that span hours, days, or weeks (loan origination, insurance claims)
2. Human-in-the-loop task assignment and escalation
3. Await points where workflow pauses for external input
4. Named checkpoints for partial rollback
5. Process versioning for in-flight instances

**Impact**: Applications requiring approval workflows, multi-stage processes, or saga patterns with human interaction must implement durability and task management entirely in runtime code.

---

## Design Principle Alignment

### Intent-First

This addendum expresses **what** durable workflow means, not **how** to persist:

| Intent-Level (This Spec) | Runtime Decides |
|--------------------------|-----------------|
| `durability.enabled: true` | Temporal, Cadence, database, event sourcing |
| `await.trigger: inputIntent` | Task queue, human task service, email |
| `assignment.policy: RoundRobin` | Load balancer, queue implementation |
| `checkpoint: post_approval` | State snapshot, event position |

### Transport-Agnostic

The grammar does not specify:
- Workflow engine or runtime
- State persistence mechanism
- Task notification transport
- Timer implementation

### Entity-Scoped

Workflow state is an entity:
- Each workflow instance has independent authority
- Await points respect entity-scoped pausing
- Human tasks are inputIntents to the workflow entity

---

## Intent-Level Grammar

### Durable Flow Extension

Extend `smsFlow` for long-running semantics:

```yaml
smsFlow:
  name: <string>
  version: <vN>
  triggeredBy:
    inputIntent: <InputIntent>
    
  durability:                            # NEW: Long-running semantics
    enabled: true | false
    state: <dataState_ref>               # Where to persist flow state
    ttl: <duration>                      # Max process lifetime
    
    versioning:
      strategy: run_to_completion | upgrade_compatible | fork
      # run_to_completion: Old instances use old version
      # upgrade_compatible: Migrate to new version at checkpoints
      # fork: Run both versions, reconcile later
      
  steps:
    - <step_definition>
    
  checkpoints:                           # NEW: Named recovery points
    - name: <identifier>
      after_step: <step_name>
      
  resume:                                # NEW: Resume behavior
    on_restart: from_checkpoint | from_start | fail
    checkpoint_selection: latest | named
```

### Await Step Type

Add `await` as a step type for external triggers:

```yaml
steps:
  - await:                               # NEW: External wait point
      name: <identifier>
      description: <string>
      
      trigger:
        type: inputIntent | signal | timer | all_of | any_of
        
        # For inputIntent triggers (human tasks)
        intent: <inputIntent_ref>
        assignment:
          policy: direct | round_robin | least_loaded | skill_based
          pool: <pool_ref>               # Group of potential assignees
          subject: <expression>          # Specific assignee
          
        # For signal triggers
        signal:
          type: <signal_type>
          filter: <expression>
          
        # For timer triggers
        timer:
          duration: <duration>           # Relative delay
          at: <expression>               # Absolute time
          
        # For composite triggers
        all_of: [<trigger>, ...]         # All must fire
        any_of: [<trigger>, ...]         # First to fire wins
        
      sla:
        target: <duration>
        warning_at: <duration>
        
      escalation:
        - after: <duration>
          action: reassign | notify | escalate_pool
          to: <pool_ref> | <subject_expression>
          
      timeout: <duration>
      on_timeout: cancel | escalate | continue_default
      default_result: <value>            # For continue_default
      
      result:
        captures: [<field>, ...]         # Fields from trigger
```

### Human Task Pool Definition

Add `taskPool` for assignee grouping:

```yaml
taskPool:
  name: <string>
  version: <vN>
  description: <string>
  
  membership:
    type: role_based | explicit | dynamic
    
    # For role-based
    roles: [<role>, ...]
    
    # For explicit
    subjects: [<subject_id>, ...]
    
    # For dynamic
    query: <expression>
    
  capacity:
    max_concurrent_per_member: <integer>
    
  availability:
    schedule: <cron> | always
    timezone: <timezone>
    
  skills:                                # For skill-based routing
    - name: <skill>
      required: true | false
```

### Checkpoint Definition

Define explicit checkpoints for recovery:

```yaml
smsFlow:
  # ...
  
  checkpoints:
    - name: post_validation
      after_step: ValidateApplication
      
    - name: post_underwriting
      after_step: await_underwriter_decision
      state_snapshot: full | delta
      
  compensation:
    - from_checkpoint: post_underwriting  # NEW: Checkpoint-based rollback
      do: ReverseUnderwriterApproval
      
    - from_checkpoint: post_validation
      do: CancelApplication
```

---

## Semantic Guarantees

When a flow declares `durability.enabled: true`:

1. **State Persistence**: Flow state MUST survive process restarts
2. **TTL Enforcement**: Flows MUST complete or cancel within TTL
3. **Resume Capability**: Flows MUST resume from declared checkpoint policy
4. **Version Isolation**: In-flight instances follow versioning strategy

When an `await` step is declared:

1. **Blocking**: Flow MUST pause until trigger fires or timeout
2. **SLA Tracking**: SLA breaches MUST trigger warnings
3. **Escalation**: Escalation rules MUST execute at declared times
4. **Assignment**: Tasks MUST route per assignment policy

---

## Runtime Responsibilities

The runtime MUST:

1. **Persist State**: Store flow state durably
2. **Resume Flows**: Restart from checkpoints after failures
3. **Track SLAs**: Monitor and alert on SLA breaches
4. **Execute Escalations**: Reassign or notify as configured
5. **Enforce Timeouts**: Cancel or continue on timeout

The runtime MAY:

1. Choose any workflow engine (Temporal, Cadence, custom)
2. Implement any state storage
3. Provide any task UI or notification
4. Optimize checkpoint storage

---

## Examples

### Example 1: Loan Origination Workflow

```yaml
smsFlow:
  name: LoanOriginationFlow
  version: v1
  description: "Multi-week loan approval process"
  
  triggeredBy:
    inputIntent: SubmitLoanApplicationIntent
    
  durability:
    enabled: true
    state: LoanApplicationFlowState
    ttl: 60d
    versioning:
      strategy: run_to_completion
      
  steps:
    - work_unit: ValidateApplication
    
    - checkpoint: post_validation
    
    - work_unit: RunCreditCheck
    
    - work_unit: CalculateRiskScore
    
    - if: "risk_score > 700"
      then:
        - work_unit: AutoApprove
      else:
        - await:
            name: await_underwriter_review
            description: "Manual underwriter review required"
            trigger:
              type: inputIntent
              intent: UnderwriterDecisionIntent
              assignment:
                policy: skill_based
                pool: UnderwriterPool
                skills_required: [credit_analysis]
            sla:
              target: 24h
              warning_at: 20h
            escalation:
              - after: 24h
                action: notify
                to: UnderwriterManager
              - after: 48h
                action: escalate_pool
                to: SeniorUnderwriterPool
            timeout: 72h
            on_timeout: escalate
            result:
              captures: [decision, conditions, notes]
              
    - checkpoint: post_underwriting
    
    - if: "underwriter_decision == 'approved'"
      then:
        - work_unit: GenerateLoanDocuments
        
        - await:
            name: await_document_signing
            trigger:
              type: inputIntent
              intent: SignDocumentsIntent
              assignment:
                policy: direct
                subject: "application.borrower_id"
            sla:
              target: 7d
            timeout: 14d
            on_timeout: cancel
            
        - work_unit: FundLoan
        
      else:
        - work_unit: SendRejectionNotice
        
  checkpoints:
    - name: post_validation
      after_step: ValidateApplication
    - name: post_underwriting
      after_step: await_underwriter_review
      
  compensation:
    - from_checkpoint: post_underwriting
      do: ReverseApproval
    - from_checkpoint: post_validation
      do: CancelApplication

taskPool:
  name: UnderwriterPool
  version: v1
  
  membership:
    type: role_based
    roles: [underwriter, senior_underwriter]
    
  capacity:
    max_concurrent_per_member: 10
    
  skills:
    - name: credit_analysis
      required: true
    - name: high_value_loans
      required: false

taskPool:
  name: SeniorUnderwriterPool
  version: v1
  
  membership:
    type: role_based
    roles: [senior_underwriter]
    
  capacity:
    max_concurrent_per_member: 5
```

### Example 2: Approval Chain Workflow

```yaml
smsFlow:
  name: ExpenseApprovalFlow
  version: v1
  
  triggeredBy:
    inputIntent: SubmitExpenseIntent
    
  durability:
    enabled: true
    state: ExpenseApprovalState
    ttl: 30d
    
  steps:
    - work_unit: ValidateExpense
    
    - await:
        name: await_manager_approval
        trigger:
          type: inputIntent
          intent: ApproveExpenseIntent
          assignment:
            policy: direct
            subject: "expense.submitter.manager_id"
        sla:
          target: 48h
        escalation:
          - after: 48h
            action: notify
            to: "expense.submitter.manager.manager_id"
        timeout: 7d
        on_timeout: continue_default
        default_result:
          decision: "auto_approved"
          
    - if: "expense.amount > 5000 AND manager_decision == 'approved'"
      then:
        - await:
            name: await_finance_approval
            trigger:
              type: inputIntent
              intent: ApproveExpenseIntent
              assignment:
                policy: round_robin
                pool: FinanceApproverPool
            sla:
              target: 24h
            timeout: 72h
            on_timeout: escalate
            
    - if: "all_approvals_granted"
      then:
        - work_unit: ProcessReimbursement
      else:
        - work_unit: NotifyRejection
```

### Example 3: Multi-Signal Await

```yaml
steps:
  - await:
      name: await_payment_confirmation
      trigger:
        type: any_of
        any_of:
          - type: signal
            signal:
              type: payment_received
              filter: "signal.transaction_id == flow.payment_id"
          - type: timer
            timer:
              duration: 24h
      result:
        captures: [payment_status, payment_method]
        
  - if: "payment_status == 'received'"
    then:
      - work_unit: FulfillOrder
    else:
      - work_unit: CancelOrder
```

---

## Integration with Existing Grammar

### Extends

- `smsFlow` - Adds `durability`, `checkpoints`, `resume` blocks
- `steps` - Adds `await` step type
- `compensation` - Adds `from_checkpoint` reference

### References

- `inputIntent` - Human tasks are input intents
- `signal` - Await can trigger on signals
- `policy` - Task assignment uses policies

### Complements

- `16-scheduled-trigger-semantics.md` - Timer-based await
- `17-notification-channel-semantics.md` - Task notifications
- `04-session-contract.md` - Human task sessions

---

## Validation Criteria

A conformant implementation MUST:

1. **Parse** durability, await, checkpoint definitions without error
2. **Persist** flow state for durable flows
3. **Resume** from checkpoints on failure
4. **Enforce** SLA and escalation rules
5. **Route** tasks per assignment policy

A conformant implementation SHOULD:

1. Provide task inbox/dashboard
2. Support task delegation
3. Track SLA metrics
4. Enable workflow monitoring

---

## EBNF Grammar Addition

```ebnf
flow_durability    ::= "durability:" NEWLINE INDENT
                       "enabled:" boolean NEWLINE
                       ( "state:" data_state_ref NEWLINE )?
                       ( "ttl:" duration NEWLINE )?
                       durability_versioning?
                       DEDENT

durability_versioning ::= "versioning:" NEWLINE INDENT
                          "strategy:" versioning_strategy NEWLINE
                          DEDENT

versioning_strategy ::= "run_to_completion" | "upgrade_compatible" | "fork"

await_step         ::= "await:" NEWLINE INDENT
                       "name:" identifier NEWLINE
                       ( "description:" string NEWLINE )?
                       await_trigger
                       await_sla?
                       await_escalation?
                       ( "timeout:" duration NEWLINE )?
                       ( "on_timeout:" timeout_action NEWLINE )?
                       ( "default_result:" value NEWLINE )?
                       await_result?
                       DEDENT

await_trigger      ::= "trigger:" NEWLINE INDENT
                       "type:" trigger_type NEWLINE
                       ( intent_trigger | signal_trigger | timer_trigger |
                         composite_trigger )?
                       DEDENT

trigger_type       ::= "inputIntent" | "signal" | "timer" | "all_of" | "any_of"

intent_trigger     ::= "intent:" intent_ref NEWLINE
                       assignment_config?

assignment_config  ::= "assignment:" NEWLINE INDENT
                       "policy:" assignment_policy NEWLINE
                       ( "pool:" pool_ref NEWLINE )?
                       ( "subject:" expression NEWLINE )?
                       DEDENT

assignment_policy  ::= "direct" | "round_robin" | "least_loaded" | "skill_based"

await_escalation   ::= "escalation:" NEWLINE INDENT
                       escalation_rule+
                       DEDENT

escalation_rule    ::= "-" "after:" duration NEWLINE INDENT
                       "action:" escalation_action NEWLINE
                       "to:" ( pool_ref | expression ) NEWLINE
                       DEDENT

escalation_action  ::= "reassign" | "notify" | "escalate_pool"

task_pool          ::= "taskPool:" NEWLINE INDENT
                       "name:" identifier NEWLINE
                       "version:" version NEWLINE
                       ( "description:" string NEWLINE )?
                       pool_membership
                       pool_capacity?
                       pool_availability?
                       pool_skills?
                       DEDENT

checkpoint_def     ::= "checkpoints:" NEWLINE INDENT
                       checkpoint_entry+
                       DEDENT

checkpoint_entry   ::= "-" "name:" identifier NEWLINE INDENT
                       "after_step:" identifier NEWLINE
                       ( "state_snapshot:" snapshot_type NEWLINE )?
                       DEDENT

snapshot_type      ::= "full" | "delta"
```

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial proposal |

---

## ADDENDUM AU: DELEGATION AND CONSENT SEMANTICS (NEW in v1.5)

**Purpose**: Complex Authorization Models Including Delegation, Consent, and Purpose Binding  
**Extends**: `relationship`, `policy`, `dataState`, `subject`  
**Priority**: HIGH

---

### Problem Statement

The current specification defines:
- RBAC with `role` and `grants`
- ABAC with `policy.when` expressions
- Row-level security with `dataPolicy.row_level_security`

However, there is no grammar for expressing:
1. Delegation chains (User A grants User B access to their data)
2. Consent management (explicit opt-in/revocation tracking)
3. Purpose binding (access tied to specific use cases per GDPR)
4. Temporal access (time-bound or usage-limited permissions)
5. Federated identity resolution

**Impact**: Applications requiring healthcare consent (HIPAA), financial delegation (power of attorney), or privacy compliance (GDPR) must implement complex authorization entirely in runtime code.

---

### Design Principle Alignment

#### Intent-First

This addendum expresses **what** delegation and consent mean, not **how** to enforce:

| Intent-Level (This Spec) | Runtime Decides |
|--------------------------|-----------------|
| `delegation.transitive: false` | Policy chain evaluation algorithm |
| `consent.expires_after: 365d` | Timer implementation, expiration check |
| `purposes: [marketing, analytics]` | Purpose tagging, audit trail |
| `temporal.usage_limit: 10` | Counter storage, enforcement |

#### Transport-Agnostic

The grammar does not specify:
- Consent collection UI
- Delegation approval workflow
- Audit log storage
- Identity federation protocol

#### Entity-Scoped

Consent and delegation are entities:
- Each consent record has its own authority
- Delegation relationships respect entity boundaries
- Revocation is an entity state transition

---

### Intent-Level Grammar

#### Delegation Relationship

Extend `relationship` for delegation semantics:

```yaml
relationship:
  name: <string>
  from: <subject_model>
  to: <subject_model>
  type: delegation
  cardinality: one-to-many
  
  delegation:                            # NEW: Delegation semantics
    scope: [<permission>, ...]           # What can be delegated
    
    constraints:
      transitive: true | false           # Can delegatee re-delegate?
      max_depth: <integer>               # Max chain length
      same_realm: true | false           # Must be in same realm
      
    requires:
      approval: none | grantor | both    # Who must approve
      verification: none | identity | mfa
      
    validity:
      type: permanent | temporal | usage_limited
      duration: <duration>               # For temporal
      max_uses: <integer>                # For usage_limited
      
    revocation:
      by: grantor | either | admin_only
      cascade: true | false              # Revoke downstream?
```

#### Consent Record

Add `consentRecord` for explicit consent tracking:

```yaml
consentRecord:
  name: <string>
  version: <vN>
  description: <string>
  
  subject_field: <field>                 # Who is granting consent
  
  grantor:
    type: subject | guardian | authorized_representative
    
  scopes:
    - name: <identifier>
      description: <string>
      required: true | false             # Can user decline?
      default: true | false              # Default state
      revocable: true | false            # Can be withdrawn?
      
  purposes:                              # GDPR purpose binding
    <scope_name>:
      - <purpose_identifier>
      
  data_categories:                       # What data the consent covers
    <scope_name>:
      - <data_category>
      
  lifecycle:
    collection_point: <composition_ref>  # Where consent is collected
    expires_after: <duration>
    renewal:
      type: prompt | auto | required
      notice_before: <duration>
      
  withdrawal:
    effect: immediate | end_of_period
    cascade_to: [<related_consent>, ...]
    
  audit:
    track_changes: true | false
    retention: <duration>
```

#### Purpose Definition

Add `purpose` as a first-class concept for access control:

```yaml
purpose:
  name: <string>
  version: <vN>
  description: <string>
  
  category: essential | functional | analytics | marketing | third_party
  
  requires_consent: true | false
  
  data_access:
    allowed: [<dataState>, ...]
    fields:
      - dataState: <dataState_ref>
        include: [<field>, ...]
        exclude: [<field>, ...]
        
  retention:
    max_duration: <duration>
    after_purpose_fulfilled: delete | anonymize | archive
```

#### Policy Extension for Delegation and Consent

Extend `policy` for delegation-aware and consent-aware evaluation:

```yaml
policy:
  name: <string>
  version: <vN>
  appliesTo:
    type: dataState
    name: <string>
  effect: allow
  
  delegation:                            # NEW: Honor delegations
    enabled: true | false
    via_relationship: <relationship_ref>
    max_chain: <integer>
    require_active: true                 # Delegation must be active
    
  consent:                               # NEW: Require consent
    enabled: true | false
    record: <consentRecord_ref>
    scope: <scope_name>
    
  purpose:                               # NEW: Purpose binding
    required: true | false
    allowed: [<purpose>, ...]
    
  temporal:                              # NEW: Time-bound access
    valid_from: <expression>
    valid_until: <expression>
    usage_limit: <integer>
    usage_scope: per_subject | per_session | global
    
  when: <expression>                     # Standard ABAC condition
```

#### Temporal Access Token

Add `temporalAccess` for time-bound or usage-limited permissions:

```yaml
temporalAccess:
  name: <string>
  version: <vN>
  
  grants:
    policy: <policy_ref>
    to: <subject_expression>
    
  validity:
    type: time_bound | usage_bound | first_use_expiry
    
    # For time_bound
    starts_at: <expression>
    expires_at: <expression>
    
    # For usage_bound  
    max_uses: <integer>
    
    # For first_use_expiry
    expires_after_first_use: <duration>
    
  constraints:
    single_session: true | false
    ip_bound: true | false
    device_bound: true | false
    
  revocation:
    on_breach: immediate
    on_delegation_revoked: cascade
```

---

### Semantic Guarantees

When a `delegation` relationship is declared:

1. **Chain Limits**: Delegation chains MUST respect `max_depth`
2. **Transitivity**: Re-delegation only allowed if `transitive: true`
3. **Scope Bounds**: Delegatee CANNOT exceed delegator's permissions
4. **Revocation Cascade**: Downstream delegations revoked if `cascade: true`

When a `consentRecord` is declared:

1. **Purpose Binding**: Access MUST match declared purposes
2. **Scope Enforcement**: Only declared scopes are grantable
3. **Expiration**: Consent MUST expire per lifecycle
4. **Audit Trail**: Changes MUST be tracked if `audit.track_changes: true`

When `policy.consent` or `policy.delegation` is declared:

1. **Active Check**: Policy MUST verify consent/delegation is active
2. **Chain Evaluation**: Delegation chains evaluated up to max_chain
3. **Purpose Match**: Current purpose MUST be in allowed list

---

### Runtime Responsibilities

The runtime MUST:

1. **Evaluate Chains**: Traverse delegation relationships
2. **Check Consent**: Verify consent is active and not expired
3. **Enforce Purposes**: Match access to declared purpose
4. **Track Usage**: Count uses for usage-limited access
5. **Cascade Revocation**: Revoke downstream on grantor revocation

The runtime MAY:

1. Implement any consent collection UI
2. Choose any delegation approval workflow
3. Provide any audit storage mechanism
4. Integrate with external identity providers

---

### Examples

#### Example 1: Healthcare Data Delegation

```yaml
relationship:
  name: Authorizes_Access_To
  from: PatientState
  to: SubjectState
  type: delegation
  cardinality: one-to-many
  
  delegation:
    scope: [view_medical_records, view_prescriptions]
    constraints:
      transitive: false
      max_depth: 1
      same_realm: true
    requires:
      approval: grantor
      verification: identity
    validity:
      type: temporal
      duration: 365d
    revocation:
      by: grantor
      cascade: false

consentRecord:
  name: HIPAAConsent
  version: v1
  
  subject_field: patient_id
  
  grantor:
    type: subject
    
  scopes:
    - name: treatment
      description: "Share data for treatment purposes"
      required: true
      revocable: false
    - name: payment
      description: "Share data for payment processing"
      required: true
      revocable: false
    - name: research
      description: "Share anonymized data for research"
      required: false
      default: false
      revocable: true
      
  purposes:
    treatment: [diagnosis, treatment_planning, referral]
    payment: [insurance_claim, billing]
    research: [anonymized_studies]
    
  lifecycle:
    collection_point: PatientOnboardingComposition
    expires_after: 730d
    renewal:
      type: prompt
      notice_before: 30d
      
  audit:
    track_changes: true
    retention: 7y

policy:
  name: MedicalRecordAccess
  version: v1
  appliesTo:
    type: dataState
    name: MedicalRecordState
  effect: allow
  
  delegation:
    enabled: true
    via_relationship: Authorizes_Access_To
    max_chain: 1
    require_active: true
    
  consent:
    enabled: true
    record: HIPAAConsent
    scope: treatment
    
  purpose:
    required: true
    allowed: [diagnosis, treatment_planning]
```

#### Example 2: Financial Power of Attorney

```yaml
relationship:
  name: Has_Power_Of_Attorney_For
  from: SubjectState
  to: SubjectState
  type: delegation
  cardinality: one-to-many
  
  delegation:
    scope: [view_accounts, initiate_transfer, manage_investments]
    constraints:
      transitive: false
      max_depth: 1
    requires:
      approval: both
      verification: mfa
    validity:
      type: temporal
      duration: 1825d                    # 5 years
    revocation:
      by: either
      cascade: true

policy:
  name: AccountAccessWithPOA
  version: v1
  appliesTo:
    type: dataState
    name: AccountState
  effect: allow
  
  delegation:
    enabled: true
    via_relationship: Has_Power_Of_Attorney_For
    max_chain: 1
    
  when: |
    (subject.id == account.owner_id) OR
    (delegation.active == true AND delegation.scope CONTAINS 'view_accounts')
```

#### Example 3: Marketing Consent with Purpose Binding

```yaml
consentRecord:
  name: MarketingConsent
  version: v1
  
  subject_field: user_id
  
  grantor:
    type: subject
    
  scopes:
    - name: email_marketing
      description: "Receive promotional emails"
      required: false
      default: false
      revocable: true
    - name: personalization
      description: "Personalized recommendations"
      required: false
      default: true
      revocable: true
    - name: analytics
      description: "Usage analytics for product improvement"
      required: false
      default: true
      revocable: true
      
  purposes:
    email_marketing: [promotional_campaigns, newsletters]
    personalization: [product_recommendations, content_curation]
    analytics: [usage_patterns, feature_adoption]
    
  data_categories:
    email_marketing: [email_address, preferences]
    personalization: [browse_history, purchase_history]
    analytics: [usage_events, session_data]
    
  lifecycle:
    collection_point: OnboardingComposition
    expires_after: 365d
    renewal:
      type: auto
      
  withdrawal:
    effect: immediate
    cascade_to: [ThirdPartyDataSharing]

purpose:
  name: promotional_campaigns
  version: v1
  category: marketing
  requires_consent: true
  
  data_access:
    allowed: [UserProfileState, PreferenceState]
    fields:
      - dataState: UserProfileState
        include: [email, first_name]
        exclude: [phone, address]
        
  retention:
    max_duration: 365d
    after_purpose_fulfilled: anonymize

policy:
  name: MarketingDataAccess
  version: v1
  appliesTo:
    type: dataState
    name: UserProfileState
  effect: allow
  
  consent:
    enabled: true
    record: MarketingConsent
    scope: email_marketing
    
  purpose:
    required: true
    allowed: [promotional_campaigns, newsletters]
    
  when: "requester.role == 'marketing_system'"
```

#### Example 4: Time-Bound Guest Access

```yaml
temporalAccess:
  name: PropertyGuestAccess
  version: v1
  
  grants:
    policy: PropertyAccessPolicy
    to: "request.guest_email"
    
  validity:
    type: time_bound
    starts_at: "booking.check_in_time"
    expires_at: "booking.check_out_time"
    
  constraints:
    device_bound: true
    
  revocation:
    on_breach: immediate
```

---

### Integration with Existing Grammar

#### Extends

- `relationship` - Adds `delegation` block
- `policy` - Adds `delegation`, `consent`, `purpose`, `temporal` blocks
- `subject` - References for grantor/grantee

#### References

- `dataState` - Consent records are data states
- `inputIntent` - Consent collection via intents
- `audit` - Audit trail for consent changes

#### Complements

- `07-mask-expression-grammar.md` - Field masking by purpose
- `27-data-governance-semantics.md` - Retention and erasure
- `04-session-contract.md` - Session-bound access

---

### Validation Criteria

A conformant implementation MUST:

1. **Parse** delegation, consent, purpose definitions without error
2. **Evaluate** delegation chains up to declared depth
3. **Enforce** consent scope requirements
4. **Bind** access to declared purposes
5. **Expire** temporal access as declared

A conformant implementation SHOULD:

1. Provide consent management UI
2. Support delegation approval workflows
3. Generate consent audit reports
4. Integrate with external consent managers

---

### EBNF Grammar Addition

```ebnf
delegation_config  ::= "delegation:" NEWLINE INDENT
                       "scope:" "[" permission_list "]" NEWLINE
                       delegation_constraints?
                       delegation_requires?
                       delegation_validity?
                       delegation_revocation?
                       DEDENT

delegation_constraints ::= "constraints:" NEWLINE INDENT
                           ( "transitive:" boolean NEWLINE )?
                           ( "max_depth:" integer NEWLINE )?
                           ( "same_realm:" boolean NEWLINE )?
                           DEDENT

consent_record     ::= "consentRecord:" NEWLINE INDENT
                       "name:" identifier NEWLINE
                       "version:" version NEWLINE
                       ( "description:" string NEWLINE )?
                       "subject_field:" identifier NEWLINE
                       consent_grantor
                       consent_scopes
                       consent_purposes?
                       consent_lifecycle?
                       consent_withdrawal?
                       consent_audit?
                       DEDENT

consent_scope      ::= "-" "name:" identifier NEWLINE INDENT
                       ( "description:" string NEWLINE )?
                       ( "required:" boolean NEWLINE )?
                       ( "default:" boolean NEWLINE )?
                       ( "revocable:" boolean NEWLINE )?
                       DEDENT

purpose_def        ::= "purpose:" NEWLINE INDENT
                       "name:" identifier NEWLINE
                       "version:" version NEWLINE
                       ( "description:" string NEWLINE )?
                       "category:" purpose_category NEWLINE
                       "requires_consent:" boolean NEWLINE
                       purpose_data_access?
                       purpose_retention?
                       DEDENT

purpose_category   ::= "essential" | "functional" | "analytics" | 
                       "marketing" | "third_party"

policy_delegation  ::= "delegation:" NEWLINE INDENT
                       "enabled:" boolean NEWLINE
                       ( "via_relationship:" relationship_ref NEWLINE )?
                       ( "max_chain:" integer NEWLINE )?
                       ( "require_active:" boolean NEWLINE )?
                       DEDENT

policy_consent     ::= "consent:" NEWLINE INDENT
                       "enabled:" boolean NEWLINE
                       "record:" consent_record_ref NEWLINE
                       "scope:" identifier NEWLINE
                       DEDENT

policy_purpose     ::= "purpose:" NEWLINE INDENT
                       "required:" boolean NEWLINE
                       ( "allowed:" "[" purpose_list "]" NEWLINE )?
                       DEDENT

temporal_access    ::= "temporalAccess:" NEWLINE INDENT
                       "name:" identifier NEWLINE
                       "version:" version NEWLINE
                       temporal_grants
                       temporal_validity
                       temporal_constraints?
                       temporal_revocation?
                       DEDENT
```

---

### Changelog

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial proposal |

---

## ADDENDUM AV: CONVERSATIONAL EXPERIENCE SEMANTICS (NEW in v1.5)

**Purpose**: Voice and Chatbot Interfaces as Presentation Modality  
**Extends**: `experience`, `presentationComposition`, `inputIntent`  
**Priority**: MEDIUM

---

### Problem Statement

The current specification defines:
- `experience` for application-level user journeys
- `presentationView` for UI data binding
- `presentationComposition` for semantic view grouping
- `accessibility` hints for screen readers

However, there is no grammar for expressing:
1. Conversational UI as a modality (voice assistants, chatbots)
2. Dialog flow with turn-taking semantics
3. Slot filling for structured data collection
4. Multi-turn conversation context
5. Speech prompts and reprompts

**Impact**: Applications requiring Alexa skills, Google Actions, or customer service chatbots must implement dialog management entirely in runtime code.

---

### Design Principle Alignment

#### Intent-First

This addendum expresses **what** conversational interaction means, not **how** to implement:

| Intent-Level (This Spec) | Runtime Decides |
|--------------------------|-----------------|
| `modality.type: voice` | Alexa, Google Assistant, custom ASR |
| `slot.synonyms: [personal, tiny]` | NLU synonym matching algorithm |
| `turn_taking.model: mixed` | Barge-in handling, turn detection |
| `prompt: "What size?"` | TTS engine, voice synthesis |

#### Transport-Agnostic

The grammar does not specify:
- Voice platform (Alexa, Google, Siri)
- NLU engine (Dialogflow, Lex, Rasa)
- TTS provider
- Chat platform (Slack, Teams, web)

#### Modality as Presentation Layer

Conversation is a presentation modality:
- Dialog flows map to compositions
- Slots map to inputIntent fields
- Responses map to view data
- Context maps to session state

---

### Intent-Level Grammar

#### Modality Declaration

Add `modality` to experience for non-visual interfaces:

```yaml
experience:
  name: <string>
  version: <vN>
  
  modality:                              # NEW: Interface modality
    type: visual | voice | text | multimodal
    
    # For voice modality
    voice:
      wake_word: <string>                # Optional custom wake word
      languages: [<locale>, ...]
      
    # For text modality (chatbot)
    text:
      platforms: [web, slack, teams, sms, whatsapp]
      
  conversation:                          # NEW: Conversation settings
    context:
      ttl: <duration>                    # Context expiration
      carry_forward: [<field>, ...]      # What persists between turns
      
    turn_taking:
      model: user_initiated | system_initiated | mixed
      timeout: <duration>                # Silence timeout
      
    error_handling:
      no_match: reprompt | fallback | escalate
      no_input: reprompt | close
      max_reprompts: <integer>
      
    disambiguation:
      strategy: confirm | list_options | implicit_first
      max_options: <integer>
      
  entry_point: <dialog_ref>
```

#### Dialog Composition

Extend `presentationComposition` for dialog semantics:

```yaml
presentationComposition:
  name: <string>
  version: <vN>
  
  dialog:                                # NEW: Dialog semantics
    type: question | statement | confirmation | handoff
    
    prompts:
      initial: <prompt_ref>
      reprompt: <prompt_ref>
      help: <prompt_ref>
      
    slots:                               # Structured data collection
      - name: <identifier>
        maps_to: <inputIntent_field>
        required: true | false
        
        elicitation:
          prompt: <prompt_ref>
          reprompt: <prompt_ref>
          
        type: <slot_type>
        
        validation:
          expression: <expression>
          on_invalid: reprompt | confirm | skip
          
        synonyms:                        # Value synonyms
          <canonical_value>:
            - <synonym>
            - <synonym>
            
    confirmation:
      required: true | false
      prompt: <prompt_ref>
      
    fulfillment:
      intent: <inputIntent_ref>
      response: <prompt_ref>
      
  transitions:
    on_complete: <dialog_ref>
    on_cancel: <dialog_ref>
    on_help: <dialog_ref>
    on_fallback: <dialog_ref>
```

#### Prompt Definition

Add `prompt` for speech/text output:

```yaml
prompt:
  name: <string>
  version: <vN>
  
  content:
    # For voice
    speech:
      ssml: <ssml_string>                # Rich speech markup
      plain: <string>                    # Fallback plain text
      
    # For text/chat
    text: <string>
    
    # For visual (cards, etc.)
    display:
      title: <string>
      body: <string>
      image: <asset_ref>
      
  variations:                            # Random selection for variety
    - <content>
    - <content>
    
  context_variables:                     # Dynamic content
    - name: <variable>
      from: session | slot | view
      field: <field_path>
```

#### Slot Types

Standard slot types for structured extraction:

```yaml
slot:
  type: text | number | date | time | datetime | duration | 
        boolean | enum | entity | currency | phone | email |
        address | list
        
  # For enum type
  enum:
    values: [<value>, ...]
    
  # For entity type (custom)  
  entity:
    entity_type: <entity_ref>
    
  # For list type
  list:
    item_type: <slot_type>
    min_items: <integer>
    max_items: <integer>
```

---

### Semantic Guarantees

When an experience declares `modality`:

1. **Prompt Delivery**: All prompts MUST be delivered in appropriate format
2. **Context Persistence**: Session context MUST persist per TTL
3. **Turn Timeout**: System MUST respect turn timeout
4. **Error Handling**: No-match/no-input MUST follow declared policy

When a dialog declares `slots`:

1. **Elicitation**: Required slots MUST be elicited before fulfillment
2. **Validation**: Slot values MUST pass declared validation
3. **Synonym Mapping**: Synonyms MUST map to canonical values
4. **Confirmation**: Confirmation MUST be requested if required

---

### Runtime Responsibilities

The runtime MUST:

1. **Interpret Input**: Parse user speech/text to intent and slots
2. **Manage Context**: Track conversation state per session
3. **Elicit Slots**: Prompt for missing required slots
4. **Deliver Prompts**: Synthesize speech or format text
5. **Handle Errors**: Apply no-match/no-input policies

The runtime MAY:

1. Choose any NLU engine
2. Implement any voice platform
3. Provide custom slot types
4. Optimize dialog flow

---

### Examples

#### Example 1: Pizza Ordering Voice Assistant

```yaml
experience:
  name: PizzaOrderingVoice
  version: v1
  
  modality:
    type: voice
    voice:
      languages: [en-US, es-US]
      
  conversation:
    context:
      ttl: 10m
      carry_forward: [order_id, customer_id]
    turn_taking:
      model: mixed
      timeout: 8s
    error_handling:
      no_match: reprompt
      no_input: reprompt
      max_reprompts: 2
    disambiguation:
      strategy: list_options
      max_options: 3
      
  entry_point: WelcomeDialog

presentationComposition:
  name: WelcomeDialog
  version: v1
  
  dialog:
    type: statement
    prompts:
      initial: WelcomePrompt
    transitions:
      on_complete: MainMenuDialog

presentationComposition:
  name: OrderPizzaDialog
  version: v1
  
  dialog:
    type: question
    
    prompts:
      initial: OrderPrompt
      reprompt: OrderReprompt
      help: OrderHelpPrompt
      
    slots:
      - name: size
        maps_to: OrderPizzaIntent.size
        required: true
        elicitation:
          prompt: SizePrompt
          reprompt: SizeReprompt
        type: enum
        enum:
          values: [small, medium, large]
        synonyms:
          small: [personal, individual, single]
          medium: [regular, normal]
          large: [family, extra large, xl, party]
          
      - name: toppings
        maps_to: OrderPizzaIntent.toppings
        required: true
        elicitation:
          prompt: ToppingsPrompt
        type: list
        list:
          item_type: enum
          enum:
            values: [pepperoni, mushrooms, olives, onions, peppers, sausage]
          max_items: 5
          
      - name: crust
        maps_to: OrderPizzaIntent.crust
        required: false
        type: enum
        enum:
          values: [thin, regular, thick, stuffed]
        synonyms:
          thin: [crispy, light]
          thick: [deep dish, pan]
          
    confirmation:
      required: true
      prompt: OrderConfirmationPrompt
      
    fulfillment:
      intent: OrderPizzaIntent
      response: OrderConfirmedPrompt
      
  transitions:
    on_complete: AnythingElseDialog
    on_cancel: MainMenuDialog
    on_help: PizzaHelpDialog

prompt:
  name: WelcomePrompt
  version: v1
  content:
    speech:
      ssml: |
        <speak>
          Welcome to Pizza Palace! 
          <break time="300ms"/>
          Would you like to order a pizza, check your order, or hear our specials?
        </speak>
      plain: "Welcome to Pizza Palace! Would you like to order a pizza, check your order, or hear our specials?"
    display:
      title: "Welcome to Pizza Palace"
      body: "How can I help you today?"

prompt:
  name: SizePrompt
  version: v1
  content:
    speech:
      ssml: "What size pizza would you like? We have small, medium, or large."
  variations:
    - speech:
        ssml: "Would you like a small, medium, or large pizza?"
    - speech:
        ssml: "What size? Small, medium, or large?"

prompt:
  name: OrderConfirmationPrompt
  version: v1
  content:
    speech:
      ssml: |
        <speak>
          Just to confirm, that's a {{size}} pizza with {{toppings}}
          <break time="200ms"/>
          on {{crust}} crust. Is that correct?
        </speak>
  context_variables:
    - name: size
      from: slot
      field: size
    - name: toppings
      from: slot
      field: toppings
    - name: crust
      from: slot
      field: crust
```

#### Example 2: Customer Service Chatbot

```yaml
experience:
  name: CustomerServiceBot
  version: v1
  
  modality:
    type: text
    text:
      platforms: [web, slack]
      
  conversation:
    context:
      ttl: 30m
      carry_forward: [customer_id, ticket_id, issue_category]
    turn_taking:
      model: user_initiated
    error_handling:
      no_match: fallback
      max_reprompts: 3
      
  entry_point: GreetingDialog

presentationComposition:
  name: IssueClassificationDialog
  version: v1
  
  dialog:
    type: question
    
    prompts:
      initial: IssuePrompt
      
    slots:
      - name: issue_category
        maps_to: CreateTicketIntent.category
        required: true
        type: enum
        enum:
          values: [billing, technical, shipping, returns, other]
        synonyms:
          billing: [payment, charge, invoice, refund]
          technical: [bug, error, not working, broken, crash]
          shipping: [delivery, tracking, package, lost]
          returns: [return, exchange, wrong item]
          
      - name: description
        maps_to: CreateTicketIntent.description
        required: true
        elicitation:
          prompt: DescriptionPrompt
        type: text
        validation:
          expression: "length(description) >= 10"
          on_invalid: reprompt
          
  transitions:
    on_complete: TicketCreatedDialog
    on_fallback: HandoffToAgentDialog

presentationComposition:
  name: HandoffToAgentDialog
  version: v1
  
  dialog:
    type: handoff
    
    prompts:
      initial: HandoffPrompt
      
    handoff:
      to: live_agent
      queue: CustomerServiceQueue
      context_transfer: [customer_id, issue_category, conversation_history]
```

#### Example 3: Multimodal Banking

```yaml
experience:
  name: BankingAssistant
  version: v1
  
  modality:
    type: multimodal
    
  conversation:
    context:
      ttl: 5m
      carry_forward: [account_id, authenticated]
      
presentationComposition:
  name: BalanceInquiryDialog
  version: v1
  
  dialog:
    type: question
    
    slots:
      - name: account
        maps_to: BalanceInquiryIntent.account_id
        required: true
        type: entity
        entity:
          entity_type: AccountEntity
        elicitation:
          prompt: WhichAccountPrompt
          
    fulfillment:
      intent: BalanceInquiryIntent
      response: BalanceResponsePrompt

prompt:
  name: BalanceResponsePrompt
  version: v1
  content:
    speech:
      ssml: |
        <speak>
          Your {{account_name}} balance is 
          <say-as interpret-as="currency">{{balance}}</say-as>.
        </speak>
    display:
      title: "Account Balance"
      body: "{{account_name}}: {{balance}}"
  context_variables:
    - name: account_name
      from: view
      field: account.name
    - name: balance
      from: view
      field: account.balance
```

---

### Integration with Existing Grammar

#### Extends

- `experience` - Adds `modality` and `conversation` blocks
- `presentationComposition` - Adds `dialog` block
- `inputIntent` - Slots map to intent fields

#### References

- `session` - Conversation context in session
- `policy` - Dialog access control
- `notification` - Proactive dialog triggers

#### Complements

- `12-accessibility-preferences.md` - Voice as accessibility modality
- `04-session-contract.md` - Conversation session
- `01-form-intent-binding.md` - Slot-to-intent mapping

---

### Validation Criteria

A conformant implementation MUST:

1. **Parse** modality, dialog, prompt definitions without error
2. **Manage** conversation context per session
3. **Elicit** required slots in order
4. **Map** synonyms to canonical values
5. **Deliver** prompts in appropriate format

A conformant implementation SHOULD:

1. Support SSML for voice
2. Provide rich cards for visual
3. Support handoff to live agents
4. Track dialog analytics

---

### EBNF Grammar Addition

```ebnf
experience_modality ::= "modality:" NEWLINE INDENT
                        "type:" modality_type NEWLINE
                        ( voice_config | text_config )?
                        DEDENT

modality_type       ::= "visual" | "voice" | "text" | "multimodal"

voice_config        ::= "voice:" NEWLINE INDENT
                        ( "wake_word:" string NEWLINE )?
                        ( "languages:" "[" locale_list "]" NEWLINE )?
                        DEDENT

conversation_config ::= "conversation:" NEWLINE INDENT
                        conversation_context?
                        turn_taking?
                        error_handling?
                        disambiguation?
                        DEDENT

dialog_def          ::= "dialog:" NEWLINE INDENT
                        "type:" dialog_type NEWLINE
                        dialog_prompts?
                        dialog_slots?
                        dialog_confirmation?
                        dialog_fulfillment?
                        DEDENT

dialog_type         ::= "question" | "statement" | "confirmation" | "handoff"

dialog_slot         ::= "-" "name:" identifier NEWLINE INDENT
                        ( "maps_to:" field_ref NEWLINE )?
                        ( "required:" boolean NEWLINE )?
                        slot_elicitation?
                        "type:" slot_type NEWLINE
                        slot_validation?
                        slot_synonyms?
                        DEDENT

slot_type           ::= "text" | "number" | "date" | "time" | "datetime" |
                        "duration" | "boolean" | "enum" | "entity" |
                        "currency" | "phone" | "email" | "address" | "list"

prompt_def          ::= "prompt:" NEWLINE INDENT
                        "name:" identifier NEWLINE
                        "version:" version NEWLINE
                        prompt_content
                        prompt_variations?
                        prompt_context_vars?
                        DEDENT

prompt_content      ::= "content:" NEWLINE INDENT
                        ( prompt_speech | prompt_text | prompt_display )+
                        DEDENT
```

---

### Changelog

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial proposal |

---

## ADDENDUM AW: EXTERNAL INTEGRATION SEMANTICS (NEW in v1.5)

**Purpose**: External API Dependencies, Webhooks, and ETL Pipelines  
**Extends**: `workUnitContract`, `inputIntent`, `materialization`  
**Priority**: HIGH

---

### Problem Statement

The current specification defines:
- `workUnitContract` with input/output schemas and side effects
- `inputIntent` for user/system proposals
- `materialization` for derived state from internal sources

However, there is no grammar for expressing:
1. External API dependencies (third-party services)
2. Inbound webhooks from external systems
3. External data sources for materialization
4. Credential and secret binding
5. Rate limiting and circuit breaker policies

**Impact**: Applications requiring payment processing, shipping APIs, or third-party data feeds must implement integration contracts entirely in runtime code.

---

### Design Principle Alignment

#### Intent-First

This addendum expresses **what** external integration means, not **how** to connect:

| Intent-Level (This Spec) | Runtime Decides |
|--------------------------|-----------------|
| `capability: charge_payment` | Stripe, Braintree, Adyen API |
| `circuit_breaker.threshold: 5` | Resilience4j, Polly, custom |
| `credentials.type: oauth` | OAuth flow, token refresh |
| `signature.type: hmac_sha256` | Verification implementation |

#### Transport-Agnostic

The grammar does not specify:
- HTTP client library
- OAuth library
- Message format (JSON, XML, protobuf)
- Retry implementation

#### Boundary-Aware

External integrations are trust boundaries:
- External calls cross trust boundaries
- Responses require validation
- Failures require explicit handling

---

### Intent-Level Grammar

#### External Dependency Declaration

Add `externalDependency` for third-party API contracts:

```yaml
externalDependency:
  name: <string>
  version: <vN>
  description: <string>
  
  type: api | data_feed | message_queue | storage
  
  capabilities:                          # What this dependency provides
    - <capability_name>
    
  contract:                              # Interface contract
    <capability_name>:
      input:
        schema:
          <field>: { type: <type>, required: <boolean> }
      output:
        schema:
          <field>: { type: <type> }
      errors:
        - code: <error_code>
          retryable: <boolean>
          
  sla:                                   # Expected service levels
    availability: <percentage>
    latency_p99: <duration>
    
  resilience:
    timeout: <duration>
    
    retry:
      enabled: true | false
      max_attempts: <integer>
      backoff:
        strategy: exponential | linear | fixed
        initial: <duration>
        max: <duration>
        jitter: <boolean>
        
    circuit_breaker:
      enabled: true | false
      threshold: <integer>               # Failures before open
      reset_after: <duration>
      half_open_requests: <integer>
      
    bulkhead:
      enabled: true | false
      max_concurrent: <integer>
      queue_size: <integer>
      
  fallback:
    on_unavailable: queue | fail | degrade | cache
    queue_ttl: <duration>
    cache_ttl: <duration>
    degrade_to: <work_unit>
    
  rate_limit:
    requests_per_second: <integer>
    requests_per_minute: <integer>
    on_exceeded: queue | reject | throttle
    
  credentials:
    type: api_key | oauth2 | basic | mtls | custom
    secret_ref: <secret_name>            # Reference, never value
    
    # For OAuth2
    oauth2:
      grant_type: client_credentials | authorization_code
      token_endpoint: <url_template>
      scopes: [<scope>, ...]
      
  health_check:
    enabled: true | false
    endpoint: <capability_name>
    interval: <duration>
    timeout: <duration>
```

#### Webhook Receiver Definition

Add `webhookReceiver` for inbound external events:

```yaml
webhookReceiver:
  name: <string>
  version: <vN>
  description: <string>
  
  source: <external_dependency_ref>      # Who sends this
  
  maps_to:
    inputIntent: <inputIntent_ref>       # What intent it creates
    
  authentication:
    type: signature | bearer | basic | ip_whitelist | none
    
    # For signature verification
    signature:
      algorithm: hmac_sha256 | hmac_sha512 | rsa_sha256
      header: <header_name>
      secret_ref: <secret_name>
      encoding: hex | base64
      include_body: true | false
      timestamp_tolerance: <duration>    # Replay protection
      
    # For bearer token
    bearer:
      secret_ref: <secret_name>
      
    # For IP whitelist
    ip_whitelist:
      allowed: [<cidr>, ...]
      
  event_mapping:
    - external_event: <event_type>
      intent_type: <intent_name>
      field_mapping:
        <intent_field>: <json_path>
        
  idempotency:
    key_path: <json_path>
    ttl: <duration>
    
  replay:
    enabled: true | false
    window: <duration>
```

#### External Data Source for Materialization

Extend `materialization` for external sources:

```yaml
materialization:
  name: <string>
  
  external_source:                       # NEW: External data feed
    dependency: <external_dependency_ref>
    capability: <capability_name>
    
    sync:
      strategy: poll | push | hybrid
      
      # For polling
      poll:
        interval: <duration>
        full_sync_interval: <duration>   # Periodic full refresh
        cursor_field: <field>
        
      # For push (webhook)
      push:
        receiver: <webhook_receiver_ref>
        
    transformation:
      - source_field: <json_path>
        target_field: <field>
        transform: <expression>
        
    conflict:
      resolution: source_wins | target_wins | merge | manual
      
    freshness:
      maxStaleness: <duration>
```

#### Outbound Webhook/Event

Add `webhookSender` for outbound notifications:

```yaml
webhookSender:
  name: <string>
  version: <vN>
  
  triggers:
    - on_transition:
        from: <dataState>
        to: <dataState>
    - on_event:
        type: <signal_type>
        
  destination:
    url_template: <url_template>         # May include entity fields
    method: POST | PUT
    
  payload:
    format: json | form | xml
    schema:
      <field>: <expression>
      
  authentication:
    type: bearer | basic | signature | custom
    secret_ref: <secret_name>
    
  delivery:
    guarantee: at_least_once | best_effort
    retry:
      max_attempts: <integer>
      backoff: exponential | fixed
    timeout: <duration>
    
  signing:
    enabled: true | false
    algorithm: hmac_sha256
    secret_ref: <secret_name>
    header: <header_name>
```

---

### Semantic Guarantees

When an `externalDependency` is declared:

1. **Timeout Enforcement**: Calls MUST timeout per declared duration
2. **Retry Policy**: Retries MUST follow declared backoff
3. **Circuit Breaker**: Circuit MUST open after threshold failures
4. **Rate Limiting**: Requests MUST be throttled per limits
5. **Credential Safety**: Credentials MUST be resolved from secret store

When a `webhookReceiver` is declared:

1. **Authentication**: Requests MUST be authenticated per policy
2. **Signature Verification**: Signatures MUST be validated if configured
3. **Idempotency**: Duplicate events MUST be deduplicated
4. **Event Mapping**: Events MUST map to declared intents

---

### Runtime Responsibilities

The runtime MUST:

1. **Resolve Credentials**: Fetch secrets from secret store
2. **Enforce Resilience**: Apply timeout, retry, circuit breaker
3. **Verify Webhooks**: Validate signatures and authentication
4. **Track Health**: Monitor external dependency health
5. **Rate Limit**: Throttle outbound requests

The runtime MAY:

1. Choose any HTTP client
2. Implement any secret store
3. Provide custom credential types
4. Aggregate health metrics

---

### Examples

#### Example 1: Payment Processor Integration

```yaml
externalDependency:
  name: PaymentProcessor
  version: v1
  description: "Stripe payment processing"
  
  type: api
  
  capabilities:
    - create_payment_intent
    - capture_payment
    - refund_payment
    - get_payment_status
    
  contract:
    create_payment_intent:
      input:
        schema:
          amount: { type: integer, required: true }
          currency: { type: string, required: true }
          customer_id: { type: string, required: false }
      output:
        schema:
          payment_intent_id: { type: string }
          client_secret: { type: string }
          status: { type: string }
      errors:
        - code: card_declined
          retryable: false
        - code: insufficient_funds
          retryable: false
        - code: rate_limited
          retryable: true
          
    refund_payment:
      input:
        schema:
          payment_intent_id: { type: string, required: true }
          amount: { type: integer, required: false }
      output:
        schema:
          refund_id: { type: string }
          status: { type: string }
          
  sla:
    availability: 99.99%
    latency_p99: 2s
    
  resilience:
    timeout: 30s
    retry:
      enabled: true
      max_attempts: 3
      backoff:
        strategy: exponential
        initial: 100ms
        max: 5s
        jitter: true
    circuit_breaker:
      enabled: true
      threshold: 5
      reset_after: 60s
      half_open_requests: 3
      
  fallback:
    on_unavailable: queue
    queue_ttl: 24h
    
  rate_limit:
    requests_per_second: 100
    on_exceeded: throttle
    
  credentials:
    type: api_key
    secret_ref: stripe_api_key
    
  health_check:
    enabled: true
    endpoint: get_payment_status
    interval: 30s
    timeout: 5s

webhookReceiver:
  name: StripeWebhook
  version: v1
  description: "Receive Stripe payment events"
  
  source: PaymentProcessor
  
  maps_to:
    inputIntent: PaymentEventReceivedIntent
    
  authentication:
    type: signature
    signature:
      algorithm: hmac_sha256
      header: Stripe-Signature
      secret_ref: stripe_webhook_secret
      encoding: hex
      timestamp_tolerance: 5m
      
  event_mapping:
    - external_event: payment_intent.succeeded
      intent_type: PaymentSucceeded
      field_mapping:
        payment_id: "$.data.object.id"
        amount: "$.data.object.amount"
        currency: "$.data.object.currency"
        customer_id: "$.data.object.customer"
        
    - external_event: payment_intent.payment_failed
      intent_type: PaymentFailed
      field_mapping:
        payment_id: "$.data.object.id"
        error_code: "$.data.object.last_payment_error.code"
        error_message: "$.data.object.last_payment_error.message"
        
  idempotency:
    key_path: "$.id"
    ttl: 24h
```

#### Example 2: Shipping Provider Integration

```yaml
externalDependency:
  name: ShippingProvider
  version: v1
  description: "Shippo shipping rates and labels"
  
  type: api
  
  capabilities:
    - get_rates
    - create_shipment
    - create_label
    - track_package
    
  contract:
    get_rates:
      input:
        schema:
          from_address: { type: object, required: true }
          to_address: { type: object, required: true }
          parcel: { type: object, required: true }
      output:
        schema:
          rates: { type: array }
          
  resilience:
    timeout: 10s
    retry:
      enabled: true
      max_attempts: 2
      backoff:
        strategy: fixed
        initial: 500ms
    circuit_breaker:
      enabled: true
      threshold: 10
      reset_after: 30s
      
  fallback:
    on_unavailable: cache
    cache_ttl: 1h
    
  credentials:
    type: api_key
    secret_ref: shippo_api_key

webhookSender:
  name: OrderShippedNotification
  version: v1
  
  triggers:
    - on_transition:
        from: OrderPendingState
        to: OrderShippedState
        
  destination:
    url_template: "{{customer.webhook_url}}"
    method: POST
    
  payload:
    format: json
    schema:
      event_type: "'order_shipped'"
      order_id: "order.id"
      tracking_number: "shipment.tracking_number"
      carrier: "shipment.carrier"
      estimated_delivery: "shipment.estimated_delivery"
      
  authentication:
    type: signature
    secret_ref: customer_webhook_secret
    
  delivery:
    guarantee: at_least_once
    retry:
      max_attempts: 5
      backoff: exponential
    timeout: 30s
    
  signing:
    enabled: true
    algorithm: hmac_sha256
    secret_ref: webhook_signing_key
    header: X-Signature
```

#### Example 3: External Data Feed Materialization

```yaml
externalDependency:
  name: ExchangeRateFeed
  version: v1
  type: data_feed
  
  capabilities:
    - get_rates
    - get_historical_rates
    
  sla:
    availability: 99.9%
    
  resilience:
    timeout: 5s
    circuit_breaker:
      enabled: true
      threshold: 3
      reset_after: 60s
      
  credentials:
    type: api_key
    secret_ref: exchange_rate_api_key

materialization:
  name: ExchangeRateView
  
  external_source:
    dependency: ExchangeRateFeed
    capability: get_rates
    
    sync:
      strategy: poll
      poll:
        interval: 5m
        full_sync_interval: 24h
        
    transformation:
      - source_field: "$.rates.USD"
        target_field: usd_rate
      - source_field: "$.rates.EUR"
        target_field: eur_rate
      - source_field: "$.timestamp"
        target_field: rate_timestamp
        transform: "parse_timestamp(value)"
        
    conflict:
      resolution: source_wins
      
  freshness:
    maxStaleness: 10m
    
  targetState: ExchangeRateState
```

---

### Integration with Existing Grammar

#### Extends

- `workUnitContract` - External calls as dependencies
- `inputIntent` - Webhooks create intents
- `materialization` - External data sources

#### References

- `signal` - External health as signals
- `policy` - External access policies
- `boundary` - Trust boundary semantics

#### Complements

- `03-intent-delivery-contract.md` - Delivery guarantees
- `16-scheduled-trigger-semantics.md` - Polling schedules
- `17-notification-channel-semantics.md` - Outbound notifications

---

### Validation Criteria

A conformant implementation MUST:

1. **Parse** external dependency definitions without error
2. **Enforce** timeout and resilience policies
3. **Verify** webhook signatures
4. **Throttle** per rate limits
5. **Resolve** credentials from secret store

A conformant implementation SHOULD:

1. Provide external dependency dashboards
2. Track SLA compliance
3. Support credential rotation
4. Enable circuit breaker monitoring

---

### EBNF Grammar Addition

```ebnf
external_dependency ::= "externalDependency:" NEWLINE INDENT
                        "name:" identifier NEWLINE
                        "version:" version NEWLINE
                        ( "description:" string NEWLINE )?
                        "type:" dependency_type NEWLINE
                        dependency_capabilities
                        dependency_contract?
                        dependency_sla?
                        dependency_resilience?
                        dependency_fallback?
                        dependency_rate_limit?
                        dependency_credentials
                        dependency_health_check?
                        DEDENT

dependency_type     ::= "api" | "data_feed" | "message_queue" | "storage"

dependency_resilience ::= "resilience:" NEWLINE INDENT
                          ( "timeout:" duration NEWLINE )?
                          resilience_retry?
                          resilience_circuit_breaker?
                          resilience_bulkhead?
                          DEDENT

resilience_circuit_breaker ::= "circuit_breaker:" NEWLINE INDENT
                               "enabled:" boolean NEWLINE
                               ( "threshold:" integer NEWLINE )?
                               ( "reset_after:" duration NEWLINE )?
                               ( "half_open_requests:" integer NEWLINE )?
                               DEDENT

webhook_receiver    ::= "webhookReceiver:" NEWLINE INDENT
                        "name:" identifier NEWLINE
                        "version:" version NEWLINE
                        ( "description:" string NEWLINE )?
                        ( "source:" dependency_ref NEWLINE )?
                        webhook_maps_to
                        webhook_authentication
                        webhook_event_mapping
                        webhook_idempotency?
                        DEDENT

webhook_authentication ::= "authentication:" NEWLINE INDENT
                           "type:" auth_type NEWLINE
                           ( signature_config | bearer_config | ip_whitelist_config )?
                           DEDENT

auth_type           ::= "signature" | "bearer" | "basic" | "ip_whitelist" | "none"

webhook_sender      ::= "webhookSender:" NEWLINE INDENT
                        "name:" identifier NEWLINE
                        "version:" version NEWLINE
                        sender_triggers
                        sender_destination
                        sender_payload
                        sender_authentication?
                        sender_delivery?
                        sender_signing?
                        DEDENT
```

---

### Changelog

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial proposal |

---

## ADDENDUM AX: DATA GOVERNANCE SEMANTICS (NEW in v1.5)

**Purpose**: Retention, Erasure, Residency, and Compliance Requirements  
**Extends**: `dataState`, `inputIntent`, `policy`  
**Priority**: HIGH

---

### Problem Statement

The current specification defines:
- `dataState` with lifecycle (intermediate, persistent, materialized)
- `realm` for multi-tenant isolation
- `authority` with regional constraints

However, there is no grammar for expressing:
1. Data retention policies with automatic expiration
2. Right-to-erasure (GDPR Article 17) semantics
3. Data residency and sovereignty constraints
4. Anonymization and pseudonymization transforms
5. Audit trail requirements

**Impact**: Applications requiring GDPR, CCPA, HIPAA, or SOX compliance must implement data governance entirely in runtime code.

---

### Design Principle Alignment

#### Intent-First

This addendum expresses **what** data governance means, not **how** to implement:

| Intent-Level (This Spec) | Runtime Decides |
|--------------------------|-----------------|
| `retention.duration: 7y` | Scheduled deletion, archival jobs |
| `residency.primary_region` | Data routing, replication config |
| `anonymization.method: hash` | SHA-256, bcrypt, Argon2 |
| `erasure.cascade_to: [...]` | Dependency resolution, deletion order |

#### Transport-Agnostic

The grammar does not specify:
- Archival storage implementation
- Deletion mechanism
- Encryption algorithm
- Compliance reporting tools

#### Entity-Scoped

Governance respects entity boundaries:
- Retention applies per entity
- Erasure respects entity authority
- Anonymization transforms entity fields

---

### Intent-Level Grammar

#### Data Governance Extension for DataState

Extend `dataState` with governance semantics:

```yaml
dataState:
  name: <string>
  type: <DataType>
  lifecycle: persistent
  
  governance:                            # NEW: Governance semantics
    classification: pii | phi | pci | financial | public | internal | confidential
    
    retention:
      policy: time_based | event_based | indefinite | legal_hold
      
      # For time_based
      duration: <duration>
      
      # For event_based
      after_event: <event_type>
      grace_period: <duration>
      
      archive:
        enabled: true | false
        after: <duration>
        tier: warm | cold | glacier
        
      on_expiry: delete | anonymize | archive
      
    residency:
      allowed_regions: [<region>, ...]
      forbidden_regions: [<region>, ...]
      primary_region: <expression>       # Dynamic based on subject
      
      transfer:
        allowed: true | false
        requires_consent: true | false
        mechanisms: [standard_clauses, adequacy, binding_rules]
        
    audit:
      access_logging: true | false
      modification_logging: true | false
      retention_for_logs: <duration>
      immutable: true | false
      
    encryption:
      at_rest: required | optional | none
      in_transit: required | optional | none
      key_rotation: <duration>
```

#### Erasure Intent

Add `erasureRequest` for right-to-deletion:

```yaml
erasureRequest:
  name: <string>
  version: <vN>
  description: <string>
  
  scope:
    subject_field: <field>               # Who is requesting erasure
    
  strategy: delete | anonymize | hybrid
  
  hybrid:
    delete: [<field>, ...]               # Fields to fully delete
    anonymize: [<field>, ...]            # Fields to anonymize
    retain: [<field>, ...]               # Fields to keep (legal requirement)
    
  cascade:
    behavior: automatic | manual_review | refuse
    entities: [<dataState>, ...]
    relationship_handling: delete | orphan | reassign
    
  verification:
    required: true | false
    methods: [email_confirmation, identity_verification, mfa]
    timeout: <duration>
    
  processing:
    sla: <duration>                      # GDPR: 30 days
    notification:
      on_complete: true | false
      on_partial: true | false
      
  exceptions:
    - reason: legal_hold
      action: defer
    - reason: ongoing_contract
      action: defer_until_completion
    - reason: legal_obligation
      action: refuse
      
  completion:
    certificate: true | false
    retention_of_proof: <duration>
```

#### Anonymization Transform

Add `anonymizationProfile` for data transformation:

```yaml
anonymizationProfile:
  name: <string>
  version: <vN>
  
  applies_to: <dataState>
  
  transforms:
    - field: <field_name>
      method: <anonymization_method>
      params: { ... }
      
  reversible: true | false               # Pseudonymization vs anonymization
  
  # For reversible (pseudonymization)
  key_management:
    key_ref: <secret_ref>
    rotation: <duration>
```

#### Anonymization Methods

| Method | Description | Reversible | Params |
|--------|-------------|------------|--------|
| `hash` | One-way hash | No | `algorithm`, `salt_source` |
| `pseudonymize` | Keyed encryption | Yes | `key_ref` |
| `generalize` | Reduce precision | No | `to` (e.g., birth_year) |
| `redact` | Remove entirely | No | `placeholder` |
| `mask` | Partial visibility | No | `reveal`, `character` |
| `round` | Numeric rounding | No | `precision` |
| `date_shift` | Random date offset | No | `max_shift` |
| `k_anonymize` | k-anonymity grouping | No | `k`, `quasi_identifiers` |
| `noise_add` | Add statistical noise | No | `distribution`, `scale` |
| `tokenize` | Replace with token | Yes | `token_store_ref` |

#### Legal Hold

Add `legalHold` for compliance preservation:

```yaml
legalHold:
  name: <string>
  version: <vN>
  
  scope:
    entities: [<dataState>, ...]
    filter: <expression>
    
  reason: <string>
  reference: <string>                    # Case number, matter ID
  
  custodian: <subject_ref>
  
  preserves:
    - dataState: <dataState>
      fields: all | [<field>, ...]
      
  overrides:
    retention: true                      # Suspend normal retention
    erasure: true                        # Block erasure requests
    
  lifecycle:
    start_date: <timestamp>
    review_interval: <duration>
    
  release:
    requires_approval: true | false
    approvers: [<role>, ...]
```

#### Data Processing Agreement

Add `dataProcessingAgreement` for third-party compliance:

```yaml
dataProcessingAgreement:
  name: <string>
  version: <vN>
  
  parties:
    controller: <entity_ref>
    processor: <external_dependency_ref>
    
  data_categories: [<classification>, ...]
  
  purposes: [<purpose>, ...]
  
  sub_processors:
    allowed: true | false
    requires_notification: true | false
    
  security:
    requirements: [encryption, access_control, audit_logging]
    certifications: [SOC2, ISO27001, HIPAA]
    
  data_location:
    allowed_regions: [<region>, ...]
    
  retention:
    controller_instruction: true | false
    max_duration: <duration>
    
  breach_notification:
    sla: <duration>
    contacts: [<email>, ...]
    
  audit:
    rights: true | false
    frequency: <duration>
    
  termination:
    data_return: true | false
    data_deletion: true | false
    certification: true | false
```

---

### Semantic Guarantees

When `governance.retention` is declared:

1. **Expiration**: Data MUST be processed per `on_expiry` after duration
2. **Archival**: Data MUST be moved to declared tier after archive period
3. **Legal Hold**: Retention MUST be suspended during legal hold

When `governance.residency` is declared:

1. **Region Restriction**: Data MUST NOT be stored in forbidden regions
2. **Primary Region**: Write authority MUST respect primary region
3. **Transfer Rules**: Cross-border transfers MUST follow declared mechanisms

When an `erasureRequest` is processed:

1. **SLA Compliance**: Erasure MUST complete within declared SLA
2. **Cascade**: Related entities MUST be handled per cascade policy
3. **Verification**: Subject MUST be verified if required
4. **Certificate**: Deletion proof MUST be generated if required

---

### Runtime Responsibilities

The runtime MUST:

1. **Track Retention**: Monitor data age against retention policy
2. **Execute Expiry**: Delete, anonymize, or archive as declared
3. **Enforce Residency**: Route data to allowed regions
4. **Process Erasure**: Complete erasure requests within SLA
5. **Honor Legal Hold**: Suspend normal processing during hold

The runtime MAY:

1. Implement any archival storage
2. Choose any anonymization algorithm
3. Provide any compliance reporting
4. Integrate with external DLP tools

---

### Examples

#### Example 1: GDPR User Data Governance

```yaml
dataState:
  name: UserProfileState
  type: UserProfile
  lifecycle: persistent
  
  governance:
    classification: pii
    
    retention:
      policy: event_based
      after_event: account_closure
      grace_period: 30d
      archive:
        enabled: true
        after: 2y
        tier: cold
      on_expiry: anonymize
      
    residency:
      allowed_regions: [eu-west-1, eu-central-1]
      forbidden_regions: [us-east-1, ap-southeast-1]
      primary_region: "user.country IN ['DE', 'FR', 'IT'] ? 'eu-central-1' : 'eu-west-1'"
      transfer:
        allowed: false
        
    audit:
      access_logging: true
      modification_logging: true
      retention_for_logs: 7y
      immutable: true
      
    encryption:
      at_rest: required
      in_transit: required
      key_rotation: 365d

erasureRequest:
  name: GDPRDeletionRequest
  version: v1
  description: "GDPR Article 17 Right to Erasure"
  
  scope:
    subject_field: user_id
    
  strategy: hybrid
  
  hybrid:
    delete: [email, phone, address, ip_addresses]
    anonymize: [purchase_history, support_tickets]
    retain: [account_id, account_created_date]      # Legal requirement
    
  cascade:
    behavior: automatic
    entities: [OrderState, PaymentState, SupportTicketState]
    relationship_handling: orphan
    
  verification:
    required: true
    methods: [email_confirmation]
    timeout: 30d
    
  processing:
    sla: 720h                            # 30 days
    notification:
      on_complete: true
      on_partial: true
      
  exceptions:
    - reason: legal_hold
      action: defer
    - reason: ongoing_dispute
      action: defer_until_completion
      
  completion:
    certificate: true
    retention_of_proof: 3y

anonymizationProfile:
  name: UserAnonymization
  version: v1
  
  applies_to: UserProfileState
  
  transforms:
    - field: name
      method: pseudonymize
      params:
        key_ref: pii_anonymization_key
        
    - field: email
      method: hash
      params:
        algorithm: sha256
        salt_source: entity_id
        
    - field: birth_date
      method: generalize
      params:
        to: birth_year
        
    - field: address
      method: redact
      params:
        placeholder: "[REDACTED]"
        
    - field: ip_address
      method: mask
      params:
        reveal: first2_octets
        
    - field: purchase_amounts
      method: round
      params:
        precision: 100
        
  reversible: false
```

#### Example 2: Healthcare PHI Governance

```yaml
dataState:
  name: PatientRecordState
  type: PatientRecord
  lifecycle: persistent
  
  governance:
    classification: phi
    
    retention:
      policy: time_based
      duration: 2555d                    # 7 years
      archive:
        enabled: true
        after: 365d
        tier: cold
      on_expiry: archive
      
    residency:
      allowed_regions: [us-east-1, us-west-2]
      transfer:
        allowed: false
        
    audit:
      access_logging: true
      modification_logging: true
      retention_for_logs: 2555d
      immutable: true
      
    encryption:
      at_rest: required
      in_transit: required

legalHold:
  name: PatientLitigationHold
  version: v1
  
  scope:
    entities: [PatientRecordState, TreatmentState, BillingState]
    filter: "patient.id IN hold_list"
    
  reason: "Pending malpractice litigation"
  reference: "Case #2026-MED-1234"
  
  custodian: legal_department
  
  preserves:
    - dataState: PatientRecordState
      fields: all
    - dataState: TreatmentState
      fields: all
      
  overrides:
    retention: true
    erasure: true
    
  lifecycle:
    start_date: "2026-01-15T00:00:00Z"
    review_interval: 90d
    
  release:
    requires_approval: true
    approvers: [legal_counsel, compliance_officer]
```

#### Example 3: Financial Data Retention

```yaml
dataState:
  name: TransactionState
  type: Transaction
  lifecycle: persistent
  
  governance:
    classification: financial
    
    retention:
      policy: time_based
      duration: 2555d                    # 7 years for SOX
      archive:
        enabled: true
        after: 365d
        tier: cold
      on_expiry: archive                 # Never delete, only archive
      
    audit:
      access_logging: true
      modification_logging: true
      retention_for_logs: 2555d
      immutable: true

dataProcessingAgreement:
  name: PaymentProcessorDPA
  version: v1
  
  parties:
    controller: OurCompany
    processor: PaymentProcessor
    
  data_categories: [pci, financial]
  
  purposes: [payment_processing, fraud_detection]
  
  sub_processors:
    allowed: true
    requires_notification: true
    
  security:
    requirements: [encryption, access_control, audit_logging]
    certifications: [PCI-DSS, SOC2]
    
  data_location:
    allowed_regions: [us-east-1, eu-west-1]
    
  breach_notification:
    sla: 72h
    contacts: [security@company.com, legal@company.com]
    
  audit:
    rights: true
    frequency: 365d
```

---

### Integration with Existing Grammar

#### Extends

- `dataState` - Adds `governance` block
- `inputIntent` - Erasure as intent
- `authority` - Residency constraints

#### References

- `realm` - Governance per realm
- `policy` - Governance enforcement policies
- `24-delegation-consent-semantics.md` - Consent for data processing

#### Complements

- `24-delegation-consent-semantics.md` - Consent lifecycle
- `07-mask-expression-grammar.md` - Field masking
- `17-notification-channel-semantics.md` - Breach notifications

---

### Validation Criteria

A conformant implementation MUST:

1. **Parse** governance configurations without error
2. **Enforce** retention expiration
3. **Restrict** data to allowed regions
4. **Process** erasure within SLA
5. **Suspend** retention during legal hold

A conformant implementation SHOULD:

1. Provide retention dashboards
2. Generate compliance reports
3. Support automated data discovery
4. Integrate with DLP tools

---

### EBNF Grammar Addition

```ebnf
governance_block    ::= "governance:" NEWLINE INDENT
                        ( "classification:" classification NEWLINE )?
                        governance_retention?
                        governance_residency?
                        governance_audit?
                        governance_encryption?
                        DEDENT

classification      ::= "pii" | "phi" | "pci" | "financial" | 
                        "public" | "internal" | "confidential"

governance_retention ::= "retention:" NEWLINE INDENT
                         "policy:" retention_policy NEWLINE
                         ( "duration:" duration NEWLINE )?
                         ( "after_event:" identifier NEWLINE )?
                         ( "grace_period:" duration NEWLINE )?
                         retention_archive?
                         ( "on_expiry:" expiry_action NEWLINE )?
                         DEDENT

retention_policy    ::= "time_based" | "event_based" | "indefinite" | "legal_hold"
expiry_action       ::= "delete" | "anonymize" | "archive"

governance_residency ::= "residency:" NEWLINE INDENT
                         ( "allowed_regions:" "[" region_list "]" NEWLINE )?
                         ( "forbidden_regions:" "[" region_list "]" NEWLINE )?
                         ( "primary_region:" expression NEWLINE )?
                         residency_transfer?
                         DEDENT

erasure_request     ::= "erasureRequest:" NEWLINE INDENT
                        "name:" identifier NEWLINE
                        "version:" version NEWLINE
                        ( "description:" string NEWLINE )?
                        erasure_scope
                        erasure_strategy
                        erasure_cascade?
                        erasure_verification?
                        erasure_processing?
                        erasure_exceptions?
                        erasure_completion?
                        DEDENT

erasure_strategy    ::= "strategy:" ( "delete" | "anonymize" | "hybrid" ) NEWLINE
                        ( erasure_hybrid )?

erasure_hybrid      ::= "hybrid:" NEWLINE INDENT
                        ( "delete:" "[" field_list "]" NEWLINE )?
                        ( "anonymize:" "[" field_list "]" NEWLINE )?
                        ( "retain:" "[" field_list "]" NEWLINE )?
                        DEDENT

anonymization_profile ::= "anonymizationProfile:" NEWLINE INDENT
                          "name:" identifier NEWLINE
                          "version:" version NEWLINE
                          "applies_to:" data_state_ref NEWLINE
                          anon_transforms
                          ( "reversible:" boolean NEWLINE )?
                          anon_key_management?
                          DEDENT

anon_transform      ::= "-" "field:" identifier NEWLINE INDENT
                        "method:" anon_method NEWLINE
                        ( "params:" params_map NEWLINE )?
                        DEDENT

anon_method         ::= "hash" | "pseudonymize" | "generalize" | "redact" |
                        "mask" | "round" | "date_shift" | "k_anonymize" |
                        "noise_add" | "tokenize"

legal_hold          ::= "legalHold:" NEWLINE INDENT
                        "name:" identifier NEWLINE
                        "version:" version NEWLINE
                        hold_scope
                        "reason:" string NEWLINE
                        "reference:" string NEWLINE
                        "custodian:" subject_ref NEWLINE
                        hold_preserves
                        hold_overrides?
                        hold_lifecycle?
                        hold_release?
                        DEDENT
```

---

### Changelog

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial proposal |

---

## ADDENDUM AY: EDGE DEVICE SEMANTICS (NEW in v1.5)

**Purpose**: IoT Device Integration, Sensors, and Actuators  
**Extends**: `worker_class`, `inputIntent`, `workUnitContract`, `signal`  
**Priority**: LOW

---

### Problem Statement

The current specification defines:
- `worker_class` with runtime and resource requirements
- `placement` for worker distribution
- `signal` for operational observations
- `inputIntent` for system proposals

However, there is no grammar for expressing:
1. Device capabilities (sensors, actuators)
2. Connectivity patterns (intermittent, battery-constrained)
3. Local buffering and sync semantics
4. Safety constraints for physical actions
5. Device health and telemetry

**Impact**: Applications requiring IoT, industrial control, or smart home integration must implement device management entirely in runtime code.

---

### Design Principle Alignment

#### Intent-First

This addendum expresses **what** device integration means, not **how** to connect:

| Intent-Level (This Spec) | Runtime Decides |
|--------------------------|-----------------|
| `capabilities: read_temperature` | I2C, SPI, analog read, MQTT |
| `connectivity: intermittent` | LoRa, NB-IoT, BLE, WiFi |
| `safety.max_temperature: 80` | PLC logic, firmware limits |
| `buffering.on_offline` | Flash storage, EEPROM, RAM |

#### Transport-Agnostic

The grammar does not specify:
- IoT protocol (MQTT, CoAP, LwM2M)
- Hardware interface
- Firmware implementation
- Edge runtime

#### Entity-Scoped

Devices are workers with entity authority:
- Each device has independent operation
- Device state follows entity patterns
- Telemetry respects entity boundaries

---

### Intent-Level Grammar

#### Device Worker Class

Extend `worker_class` for device semantics:

```yaml
worker_class:
  name: <string>
  runtime: embedded | edge | gateway
  
  device:                                # NEW: Device semantics
    type: sensor | actuator | gateway | hybrid
    
    capabilities:
      - <capability_name>
      
    hardware:
      category: industrial | consumer | medical | automotive
      certifications: [<certification>, ...]
      
    constraints:
      power:
        source: battery | mains | solar | poe
        battery_capacity: <mAh>
        sleep_supported: true | false
        
      connectivity:
        type: always_on | intermittent | scheduled
        protocols: [wifi, ble, lora, nbiot, zigbee, thread, ethernet]
        
      processing:
        level: minimal | standard | edge_compute
        
      environment:
        temperature_range: { min: <celsius>, max: <celsius> }
        humidity_range: { min: <percent>, max: <percent> }
        ip_rating: <ip_code>
        
    health:
      heartbeat_interval: <duration>
      offline_threshold: <duration>
      battery_low_threshold: <percent>
      
    firmware:
      update_supported: true | false
      update_method: ota | manual | staged
```

#### Device Capability Declaration

Add `deviceCapability` for sensor/actuator contracts:

```yaml
deviceCapability:
  name: <string>
  version: <vN>
  
  type: sensor | actuator | diagnostic
  
  # For sensors
  sensor:
    measures: temperature | humidity | pressure | motion | light | 
              proximity | acceleration | gps | current | voltage |
              flow | level | vibration | sound | air_quality | custom
              
    output:
      type: <data_type>
      unit: <unit>
      precision: <decimal_places>
      range: { min: <value>, max: <value> }
      
    sampling:
      min_interval: <duration>
      max_interval: <duration>
      
  # For actuators
  actuator:
    controls: switch | dimmer | motor | valve | lock | hvac | custom
    
    input:
      type: <data_type>
      range: { min: <value>, max: <value> }
      
    feedback:
      confirmation: required | optional | none
      timeout: <duration>
      
  # Safety constraints
  safety:
    constraints:
      - expression: <expression>
        on_violation: reject | clamp | alert | emergency_stop
        
    clamp_to: { min: <value>, max: <value> }
    
    interlocks:
      - condition: <expression>
        action: prevent | warn
```

#### Device Telemetry State

Add `deviceTelemetry` for streaming sensor data:

```yaml
deviceTelemetry:
  name: <string>
  version: <vN>
  
  source:
    device_type: <worker_class_ref>
    capability: <capability_ref>
    
  collection:
    mode: continuous | on_change | threshold | scheduled
    
    # For continuous
    continuous:
      interval: <duration>
      
    # For on_change
    on_change:
      tolerance: <value>                 # Minimum change to report
      
    # For threshold
    threshold:
      above: <value>
      below: <value>
      
    # For scheduled
    scheduled:
      cron: <cron_expression>
      
  buffering:
    on_offline: buffer_local | discard | aggregate
    max_buffer_size: <integer>
    max_buffer_duration: <duration>
    sync_on_reconnect: immediate | batched | background
    
  aggregation:
    enabled: true | false
    window: <duration>
    functions: [min, max, avg, count, sum, stddev]
    
  delivery:
    guarantee: best_effort | at_least_once
    compression: none | gzip | lz4
    batching:
      enabled: true | false
      max_size: <integer>
      max_delay: <duration>
```

#### Actuator Command Intent

Add `actuatorCommand` for device control:

```yaml
actuatorCommand:
  name: <string>
  version: <vN>
  
  target:
    device_type: <worker_class_ref>
    capability: <capability_ref>
    selector: <expression>               # Which device(s)
    
  command:
    type: set_state | toggle | pulse | sequence
    
    # For set_state
    set_state:
      value: <expression>
      
    # For toggle
    toggle:
      duration: <duration>               # Optional auto-revert
      
    # For pulse
    pulse:
      duration: <duration>
      
    # For sequence
    sequence:
      steps:
        - value: <value>
          hold: <duration>
          
  safety:
    constraints:
      - <expression>
    on_violation: reject | clamp | alert
    
  delivery:
    guarantee: at_least_once | exactly_once
    timeout: <duration>
    retry_on_offline: queue | reject
    queue_ttl: <duration>
    
  confirmation:
    required: true | false
    timeout: <duration>
    on_no_confirmation: retry | alert | rollback
```

#### Device Group

Add `deviceGroup` for fleet management:

```yaml
deviceGroup:
  name: <string>
  version: <vN>
  
  membership:
    type: static | dynamic | hierarchical
    
    # For static
    devices: [<device_id>, ...]
    
    # For dynamic
    filter: <expression>
    
    # For hierarchical
    parent: <device_group_ref>
    
  properties:
    location: <spatial_ref>
    zone: <string>
    owner: <subject_ref>
    
  commands:
    broadcast: true | false              # Command all at once
    sequential: true | false             # Command one at a time
    
  health:
    aggregate: true | false
    unhealthy_threshold: <percentage>
```

---

### Semantic Guarantees

When a `device` worker class is declared:

1. **Capability Binding**: Device MUST provide declared capabilities
2. **Health Reporting**: Device MUST heartbeat per interval
3. **Offline Detection**: System MUST detect offline after threshold
4. **Power Awareness**: Operations MUST respect power constraints

When `deviceTelemetry` is declared:

1. **Collection Mode**: Data MUST be collected per declared mode
2. **Buffering**: Data MUST be buffered during offline per policy
3. **Sync**: Data MUST sync on reconnect per policy

When `actuatorCommand` is executed:

1. **Safety Enforcement**: Commands MUST respect safety constraints
2. **Delivery**: Commands MUST be delivered per guarantee
3. **Confirmation**: Confirmation MUST be awaited if required

---

### Runtime Responsibilities

The runtime MUST:

1. **Track Devices**: Maintain device registry with health
2. **Collect Telemetry**: Receive and store sensor data
3. **Enforce Safety**: Reject or clamp unsafe commands
4. **Buffer Offline**: Store data during disconnection
5. **Sync Reconnect**: Replay buffered data on reconnect

The runtime MAY:

1. Choose any IoT protocol
2. Implement any edge runtime
3. Provide custom capabilities
4. Optimize for power/bandwidth

---

### Examples

#### Example 1: Temperature Sensor Device

```yaml
worker_class:
  name: TemperatureSensorDevice
  runtime: embedded
  
  device:
    type: sensor
    
    capabilities:
      - read_temperature
      - read_humidity
      - read_battery
      
    hardware:
      category: industrial
      certifications: [CE, FCC, IP67]
      
    constraints:
      power:
        source: battery
        battery_capacity: 2400
        sleep_supported: true
        
      connectivity:
        type: intermittent
        protocols: [lora]
        
      processing:
        level: minimal
        
      environment:
        temperature_range: { min: -40, max: 85 }
        humidity_range: { min: 0, max: 100 }
        ip_rating: IP67
        
    health:
      heartbeat_interval: 1h
      offline_threshold: 4h
      battery_low_threshold: 20
      
    firmware:
      update_supported: true
      update_method: ota

deviceCapability:
  name: read_temperature
  version: v1
  type: sensor
  
  sensor:
    measures: temperature
    output:
      type: decimal
      unit: celsius
      precision: 1
      range: { min: -40, max: 85 }
    sampling:
      min_interval: 1s
      max_interval: 1h

deviceTelemetry:
  name: TemperatureReading
  version: v1
  
  source:
    device_type: TemperatureSensorDevice
    capability: read_temperature
    
  collection:
    mode: threshold
    threshold:
      above: 30
      below: 10
      
  buffering:
    on_offline: buffer_local
    max_buffer_size: 1000
    max_buffer_duration: 24h
    sync_on_reconnect: batched
    
  aggregation:
    enabled: true
    window: 5m
    functions: [min, max, avg]
    
  delivery:
    guarantee: at_least_once
    batching:
      enabled: true
      max_size: 100
      max_delay: 1m
```

#### Example 2: Smart Thermostat Actuator

```yaml
worker_class:
  name: SmartThermostat
  runtime: edge
  
  device:
    type: hybrid
    
    capabilities:
      - read_temperature
      - read_humidity
      - set_target_temperature
      - set_mode
      
    hardware:
      category: consumer
      
    constraints:
      power:
        source: mains
        
      connectivity:
        type: always_on
        protocols: [wifi, thread]
        
      processing:
        level: edge_compute
        
    health:
      heartbeat_interval: 1m
      offline_threshold: 5m

deviceCapability:
  name: set_target_temperature
  version: v1
  type: actuator
  
  actuator:
    controls: hvac
    input:
      type: decimal
      range: { min: 50, max: 90 }
    feedback:
      confirmation: required
      timeout: 30s
      
  safety:
    constraints:
      - expression: "target >= 50 AND target <= 90"
        on_violation: clamp
      - expression: "device.mode != 'maintenance'"
        on_violation: reject
    clamp_to: { min: 50, max: 90 }
    
    interlocks:
      - condition: "window_sensor.open == true"
        action: warn

actuatorCommand:
  name: SetTemperatureCommand
  version: v1
  
  target:
    device_type: SmartThermostat
    capability: set_target_temperature
    selector: "device.zone == request.zone"
    
  command:
    type: set_state
    set_state:
      value: "request.target_temperature"
      
  safety:
    constraints:
      - "request.target_temperature >= 50"
      - "request.target_temperature <= 90"
    on_violation: clamp
    
  delivery:
    guarantee: at_least_once
    timeout: 10s
    retry_on_offline: queue
    queue_ttl: 1h
    
  confirmation:
    required: true
    timeout: 30s
    on_no_confirmation: retry
```

#### Example 3: Industrial Valve Controller

```yaml
worker_class:
  name: IndustrialValve
  runtime: embedded
  
  device:
    type: actuator
    
    capabilities:
      - set_valve_position
      - read_valve_position
      - emergency_close
      
    hardware:
      category: industrial
      certifications: [ATEX, SIL2]
      
    constraints:
      power:
        source: poe
        
      connectivity:
        type: always_on
        protocols: [ethernet]
        
      environment:
        temperature_range: { min: -20, max: 60 }
        ip_rating: IP68
        
    health:
      heartbeat_interval: 1s
      offline_threshold: 5s

deviceCapability:
  name: set_valve_position
  version: v1
  type: actuator
  
  actuator:
    controls: valve
    input:
      type: integer
      range: { min: 0, max: 100 }
    feedback:
      confirmation: required
      timeout: 5s
      
  safety:
    constraints:
      - expression: "pressure_sensor.value < max_pressure"
        on_violation: emergency_stop
      - expression: "rate_of_change(position) < max_rate"
        on_violation: reject
        
    interlocks:
      - condition: "upstream_valve.closed == true"
        action: prevent

deviceCapability:
  name: emergency_close
  version: v1
  type: actuator
  
  actuator:
    controls: valve
    input:
      type: boolean
    feedback:
      confirmation: required
      timeout: 2s
      
  safety:
    constraints: []                      # No safety constraints - this IS safety
```

#### Example 4: Device Fleet Management

```yaml
deviceGroup:
  name: BuildingA_Thermostats
  version: v1
  
  membership:
    type: dynamic
    filter: "device.building == 'A' AND device.type == 'SmartThermostat'"
    
  properties:
    location:
      type: building
      building_id: "building-a"
    zone: HVAC_Zone_1
    
  commands:
    broadcast: true
    sequential: false
    
  health:
    aggregate: true
    unhealthy_threshold: 20%

actuatorCommand:
  name: SetBuildingTemperature
  version: v1
  
  target:
    device_type: SmartThermostat
    capability: set_target_temperature
    selector: "device IN group('BuildingA_Thermostats')"
    
  command:
    type: set_state
    set_state:
      value: "request.target_temperature"
      
  delivery:
    guarantee: at_least_once
    timeout: 30s
```

---

### Integration with Existing Grammar

#### Extends

- `worker_class` - Adds `device` block
- `inputIntent` - Actuator commands as intents
- `signal` - Device health as signals
- `placement` - Device placement constraints

#### References

- `21-spatial-type-semantics.md` - Device location
- `16-scheduled-trigger-semantics.md` - Scheduled collection
- `26-external-integration-semantics.md` - Gateway integration

#### Complements

- `22-graph-query-semantics.md` - Device relationship graphs
- `20-inference-derived-fields.md` - Edge ML inference
- `17-notification-channel-semantics.md` - Device alerts

---

### Validation Criteria

A conformant implementation MUST:

1. **Parse** device configurations without error
2. **Track** device health per heartbeat
3. **Enforce** safety constraints on commands
4. **Buffer** telemetry during offline
5. **Sync** on reconnect per policy

A conformant implementation SHOULD:

1. Support OTA firmware updates
2. Provide device dashboards
3. Support edge compute
4. Enable fleet management

---

### EBNF Grammar Addition

```ebnf
device_block        ::= "device:" NEWLINE INDENT
                        "type:" device_type NEWLINE
                        device_capabilities
                        device_hardware?
                        device_constraints?
                        device_health?
                        device_firmware?
                        DEDENT

device_type         ::= "sensor" | "actuator" | "gateway" | "hybrid"

device_capabilities ::= "capabilities:" NEWLINE INDENT
                        ( "-" identifier NEWLINE )+
                        DEDENT

device_constraints  ::= "constraints:" NEWLINE INDENT
                        device_power?
                        device_connectivity?
                        device_processing?
                        device_environment?
                        DEDENT

device_capability   ::= "deviceCapability:" NEWLINE INDENT
                        "name:" identifier NEWLINE
                        "version:" version NEWLINE
                        "type:" capability_type NEWLINE
                        ( sensor_config | actuator_config )
                        capability_safety?
                        DEDENT

capability_type     ::= "sensor" | "actuator" | "diagnostic"

sensor_config       ::= "sensor:" NEWLINE INDENT
                        "measures:" measurement_type NEWLINE
                        sensor_output
                        sensor_sampling?
                        DEDENT

actuator_config     ::= "actuator:" NEWLINE INDENT
                        "controls:" control_type NEWLINE
                        actuator_input
                        actuator_feedback?
                        DEDENT

device_telemetry    ::= "deviceTelemetry:" NEWLINE INDENT
                        "name:" identifier NEWLINE
                        "version:" version NEWLINE
                        telemetry_source
                        telemetry_collection
                        telemetry_buffering?
                        telemetry_aggregation?
                        telemetry_delivery?
                        DEDENT

actuator_command    ::= "actuatorCommand:" NEWLINE INDENT
                        "name:" identifier NEWLINE
                        "version:" version NEWLINE
                        command_target
                        command_spec
                        command_safety?
                        command_delivery?
                        command_confirmation?
                        DEDENT

device_group        ::= "deviceGroup:" NEWLINE INDENT
                        "name:" identifier NEWLINE
                        "version:" version NEWLINE
                        group_membership
                        group_properties?
                        group_commands?
                        group_health?
                        DEDENT
```

---

### Changelog

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial proposal |

---

## v1.5 Addendum Cross-Reference Table

This table maps all v1.5 addendums with their original proposal IDs, priorities, and primary grammar extensions:

| v1.5 ID | Proposal ID | Addendum Name | Priority | Extends |
|---------|-------------|---------------|----------|---------|
| **X** | 01 | Form Intent Binding | CRITICAL | `inputIntent`, `presentationComposition` |
| **Y** | 02 | View Materialization Contract | HIGH | `presentationView`, `materialization` |
| **Z** | 03 | Intent Delivery Contract | HIGH | `inputIntent`, `workUnitContract` |
| **AA** | 04 | Session Contract | MEDIUM | `session`, `experience` |
| **AB** | 05 | Indicator Source Binding | MEDIUM | `indicator`, `materialization` |
| **AC** | 06 | Composition Fetch Semantics | HIGH | `presentationComposition`, `materialization` |
| **AD** | 07 | Mask Expression Grammar | MEDIUM | `presentationView`, `policy` |
| **AE** | 08 | Navigation Semantics | LOW | `experience`, `presentationComposition` |
| **AF** | 09 | View Availability Policy | LOW | `presentationView`, `policy` |
| **AG** | 10 | Presentation Hints | MEDIUM | `presentationView`, `presentationComposition` |
| **AH** | 11 | Surface Container Semantics | MEDIUM | `experience`, `presentationComposition` |
| **AI** | 12 | Accessibility Preferences | HIGH | `subject`, `presentationView` |
| **AJ** | 13 | End-to-End Reference Flow | REFERENCE | Multiple (integration example) |
| **AK** | 14 | Asset Semantics | HIGH | `asset`, `presentationView` |
| **AL** | 15 | Search Semantics | HIGH | `search`, `materialization` |
| **AM** | 16 | Scheduled Trigger Semantics | MEDIUM | `trigger`, `workUnitContract` |
| **AN** | 17 | Notification Channel Semantics | MEDIUM | `notification`, `inputIntent` |
| **AO** | 18 | Collaborative Session Semantics | MEDIUM | `session`, `signal` |
| **AP** | 19 | Structured Content Type | MEDIUM | `dataState`, `presentationView` |
| **AQ** | 20 | Inference Derived Fields | LOW | `materialization`, `workUnitContract` |
| **AR** | 21 | Spatial Type Semantics | LOW | `dataState`, `materialization` |
| **AS** | 22 | Graph Query Semantics | MEDIUM | `materialization`, `relationship` |
| **AT** | 23 | Durable Workflow Semantics | MEDIUM | `workUnitContract`, `flow` |
| **AU** | 24 | Delegation Consent Semantics | HIGH | `relationship`, `policy`, `dataState` |
| **AV** | 25 | Conversational Experience Semantics | MEDIUM | `experience`, `presentationComposition` |
| **AW** | 26 | External Integration Semantics | HIGH | `workUnitContract`, `inputIntent`, `materialization` |
| **AX** | 27 | Data Governance Semantics | HIGH | `dataState`, `inputIntent`, `policy` |
| **AY** | 28 | Edge Device Semantics | LOW | `worker_class`, `inputIntent`, `signal` |

---

## Addendum Priority Summary

### CRITICAL Priority (1)
- **X** - Form Intent Binding: Essential for user input collection

### HIGH Priority (10)
- **Y** - View Materialization Contract
- **Z** - Intent Delivery Contract
- **AC** - Composition Fetch Semantics
- **AI** - Accessibility Preferences
- **AK** - Asset Semantics
- **AL** - Search Semantics
- **AU** - Delegation Consent Semantics
- **AW** - External Integration Semantics
- **AX** - Data Governance Semantics

### MEDIUM Priority (12)
- **AA** - Session Contract
- **AB** - Indicator Source Binding
- **AD** - Mask Expression Grammar
- **AG** - Presentation Hints
- **AH** - Surface Container Semantics
- **AM** - Scheduled Trigger Semantics
- **AN** - Notification Channel Semantics
- **AO** - Collaborative Session Semantics
- **AP** - Structured Content Type
- **AS** - Graph Query Semantics
- **AT** - Durable Workflow Semantics
- **AV** - Conversational Experience Semantics

### LOW Priority (4)
- **AE** - Navigation Semantics
- **AF** - View Availability Policy
- **AQ** - Inference Derived Fields
- **AR** - Spatial Type Semantics
- **AY** - Edge Device Semantics

### REFERENCE (1)
- **AJ** - End-to-End Reference Flow: Integration example, not a grammar extension

---

## Addendum Interdependencies

Key relationships between addendums:

### Presentation Layer Dependencies
- **X** (Form Intent Binding) ← Required by **Y** (View Materialization)
- **Y** (View Materialization) ← Required by **AC** (Composition Fetch)
- **AG** (Presentation Hints) ↔ **AI** (Accessibility Preferences)
- **AV** (Conversational Experience) ← Uses **X** (Form Intent Binding)

### Data Layer Dependencies
- **AU** (Delegation Consent) ↔ **AX** (Data Governance)
- **AX** (Data Governance) → Complements **AD** (Mask Expression)
- **AP** (Structured Content) ← Used by **AK** (Asset Semantics)
- **AR** (Spatial Types) → Used by **AY** (Edge Devices)

### Integration Layer Dependencies
- **AW** (External Integration) ↔ **Z** (Intent Delivery)
- **AW** (External Integration) → Enables **Y** (View Materialization from external sources)
- **AM** (Scheduled Triggers) ← Used by **AW** (polling schedules)
- **AN** (Notification Channels) ← Used by **AW** (outbound webhooks)

### Workflow Layer Dependencies
- **AT** (Durable Workflow) ↔ **Z** (Intent Delivery)
- **AS** (Graph Query) → Complements **AL** (Search Semantics)
- **AO** (Collaborative Session) ← Uses **AA** (Session Contract)

### Device & Edge Dependencies
- **AY** (Edge Devices) ↔ **AM** (Scheduled Triggers)
- **AY** (Edge Devices) → Uses **AR** (Spatial Types) for location
- **AQ** (Inference Derived Fields) → Edge ML for **AY**

---

## Conclusion

The SMS v1.5 specification now includes **28 addendums** (X-AY) that extend the v1.4 core grammar to provide complete end-to-end application development capabilities. These addendums maintain the specification's core philosophy of **Intent-First, Runtime-Decided** while enabling:

### Complete Application Stack
1. **Presentation Layer** (X, Y, AC, AD, AE, AF, AG, AH, AI, AV) - Forms, views, navigation, accessibility, conversational UI
2. **Business Logic Layer** (Z, AT, AU, AS) - Intent delivery, workflows, authorization, graph queries
3. **Data Layer** (AP, AQ, AR, AX) - Content types, inference, spatial data, governance
4. **Integration Layer** (AW, AM, AN) - External APIs, webhooks, scheduled tasks, notifications
5. **Infrastructure Layer** (AA, AB, AO, AY) - Sessions, indicators, collaboration, edge devices
6. **Asset Management** (AK, AL) - Media handling, search capabilities

### Key Capabilities Enabled
- **User Experience**: Form binding, accessibility, conversational interfaces
- **Data Governance**: GDPR/CCPA compliance, retention, erasure, residency
- **Authorization**: Delegation chains, consent management, purpose binding
- **Integration**: External APIs, webhooks, ETL pipelines, circuit breakers
- **Workflows**: Durable processes, human-in-the-loop, compensation
- **IoT & Edge**: Sensors, actuators, telemetry, fleet management
- **Search & Query**: Full-text search, graph queries, spatial queries
- **Real-time**: Collaborative sessions, notifications, live updates

### Implementation Approach

**Phase 1: Foundation** (CRITICAL + HIGH)
- Start with **X** (Form Intent Binding) - essential for any UI
- Add **Z** (Intent Delivery Contract) - core intent processing
- Add **Y** (View Materialization Contract) - data projection

**Phase 2: Core Features** (HIGH)
- **AC** (Composition Fetch) - efficient data loading
- **AI** (Accessibility) - inclusive design
- **AK** (Asset Semantics) - media handling
- **AL** (Search) - content discovery

**Phase 3: Compliance & Integration** (HIGH)
- **AU** (Delegation Consent) - authorization models
- **AX** (Data Governance) - compliance requirements
- **AW** (External Integration) - third-party services

**Phase 4: Advanced Features** (MEDIUM)
- Session management, workflows, notifications
- Presentation hints, surface containers
- Collaborative features

**Phase 5: Specialized** (LOW)
- Inference, spatial types, edge devices
- Advanced navigation, availability policies

### Backward Compatibility

All v1.5 addendums are **additive only**:
- No breaking changes to v1.4 grammar
- Existing v1.4 applications remain valid
- Runtimes can implement addendums incrementally
- Graceful degradation for unsupported features

### Next Steps

For implementers:
1. Review the addendum cross-reference table
2. Identify priority addendums for your use case
3. Implement addendums in dependency order
4. Use the comprehensive examples as implementation guides
5. Leverage EBNF grammar additions for parsers

For specification evolution:
1. Gather implementation feedback
2. Refine addendum interdependencies
3. Develop conformance test suites
4. Create implementation guides per addendum
5. Document best practices and patterns

---

## Document Metadata

**Version**: v1.5 Draft  
**Date**: 2026-01-17  
**Status**: Integration Complete  
**Addendums Integrated**: 28 (X through AY)  
**Total Length**: Comprehensive specification with examples  
**Target Audience**: Runtime implementers, application developers, specification contributors

---
