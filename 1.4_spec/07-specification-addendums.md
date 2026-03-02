# System Mechanics Specification - Required Addendums v1.4

## Overview

This document contains critical addendums identified during specification review. These fill semantic gaps that would cause incorrect implementation if omitted. All addendums are **normative** and **mandatory** for conformant implementations.

### Addendums Index

**v1.1 Addendums (A-N)**: Core specification gaps
**v1.3 Addendums (O-T)**: Authority, storage, and invariant semantics
**v1.4 Addendums (U-W)**: UI Composition, experience, and field permission semantics

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

