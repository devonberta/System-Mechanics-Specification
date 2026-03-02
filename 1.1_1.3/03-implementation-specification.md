# System Mechanics Specification - Implementation Reference v1.3

## Overview

This document provides detailed implementation guidance for building systems conformant with the System Mechanics Specification. It covers runtime architecture, NATS integration patterns, control plane design, authority management, multi-region survivability, and operational concerns.

---

## PART I: ARCHITECTURAL PRINCIPLES

### Core Separations

The system maintains strict separation between:

1. **Intent vs Execution**
   - Grammar defines intent
   - Runtime determines execution strategy

2. **Control Plane vs Data Plane**
   - Control plane: Policy, topology, metadata distribution
   - Data plane: Work execution, data flow

3. **Scheduling vs Execution**
   - Schedulers decide worker existence
   - Runtime decides work routing
   - Infrastructure provisions resources

4. **Authorization vs Execution**
   - Policies distributed centrally
   - Enforcement evaluated locally
   - No synchronous auth calls in hot path

### Authority Model

**Single Authority Per Scope**
- One authoritative scheduler per topology scope
- One authoritative policy enforcer per component
- No distributed consensus required
- Leader election for high availability only

**Entity-Scoped Authority (NEW in v1.3)**
- Write authority resolved at entity level, not model level
- Each entity has exactly one authoritative region at any time
- Authority may change over entity lifetime via explicit transition
- Authority epoch ensures CAS correctness across migrations

**Local Enforcement**
- All decisions made with local state
- Control plane absence does not halt execution
- Fail-closed on ambiguity

---

## PART II: RUNTIME ARCHITECTURE

### Runtime Binary Structure

Each runtime (flow engine, worker, UI) follows this canonical structure:

```
┌─────────────────────────────────┐
│ Runtime Binary                  │
├─────────────────────────────────┤
│ Subscription Manager            │  ← NATS integration
│ Policy Registry (local)         │  ← In-memory, versioned
│ Data Schema Registry            │  ← Type definitions
│ Intent Router                   │  ← Routing cache
│ Execution Engine                │  ← Work execution
│ Materialization Engine          │  ← View generation
│ Observability Hooks             │  ← Signals, traces
└─────────────────────────────────┘
```

**Key Properties**:
- Libraries, not services
- Composable based on role
- Stateless except for caches
- Long-lived process

### Component Responsibilities

#### Flow Engine (SMS Runtime)
**Responsibilities**:
- Execute SMS flows as FSMs
- Gate flows with policy sets
- Coordinate work unit invocation
- Track correlation context
- Emit signals

**Does NOT**:
- Store business state
- Make scheduling decisions
- Directly provision workers
- Enforce data policies (delegates to workers)

#### Worker Runtime
**Responsibilities**:
- Execute work units
- Enforce data policies locally
- Emit capacity/health signals
- Advertise capabilities
- Handle graceful shutdown

**Does NOT**:
- Coordinate with other workers
- Know about other workers
- Make topology decisions
- Store durable state (delegates to storage)

#### UI Runtime
**Responsibilities**:
- Render presentation views
- Bind to materialized data versions
- Gate user actions with policies
- Submit input intents

**Does NOT**:
- Execute business logic
- Store authoritative data
- Make authorization decisions (evaluates policies)

---

## PART III: CONTROL PLANE ARCHITECTURE

### Control Plane Components

```
┌──────────────────────────────────────┐
│ Control Plane                        │
├──────────────────────────────────────┤
│ Spec Artifact Manager                │
│ Policy Artifact Manager              │
│ Topology Configuration Manager       │
│ Lifecycle State Manager              │
│ Distribution Layer (NATS)            │
└──────────────────────────────────────┘
```

**Deployment Model**:
- Logically centralized
- Physically replicated (active-active)
- Stateless at runtime
- Git/object storage backed

**Where It Runs**:
- One instance per bounded domain
- Regional replicas for availability
- NOT in request path
- Can disappear without halting execution

### Control Plane Responsibilities

**DOES**:
- Define worker topology intent
- Define policy artifacts
- Publish immutable, versioned configuration
- Manage lifecycle state transitions
- Validate artifact correctness

**DOES NOT**:
- Execute work
- Schedule workers (delegates to infrastructure)
- Make real-time decisions
- Store business data

---

## PART IV: NATS INTEGRATION PATTERNS

### NATS Role Clarification

**NATS is:**
- Coordination substrate
- Signal transport
- Policy distribution channel
- Worker discovery registry
- Control plane backbone

**NATS is NOT:**
- Mandatory execution path
- Data storage
- Transaction coordinator
- Authorization service

### Subject Design Patterns

#### Intent Subjects (Execution)
```
intent.<domain>.<action>
```

Examples:
```
intent.order.create
intent.order.cancel
intent.payment.apply
```

**Properties**:
- Workers subscribe based on capability
- Queue groups for load balancing
- Version negotiation via headers

#### Control Plane Subjects (Configuration)
```
spec.<domain>.<type>.<name>.<version>
policy.<domain>.<component>.<policyName>.<version>
```

Examples:
```
spec.order.dataType.Order.v3
policy.order.worker.OrderService.modify_order.v2
```

**Properties**:
- JetStream backed
- Replay enabled
- Immutable per subject
- Latest retention

#### Signal Subjects (Observability)
```
signal.<source>.<name>.<type>
```

Examples:
```
signal.worker.OrderService.capacity
signal.sms.OrderFlow.backpressure
signal.policy.ModifyOrder.shadow
```

**Properties**:
- Fire-and-forget
- Optionally persisted for analysis
- Not request-reply
- High frequency acceptable

#### Discovery Subjects (Registry)
NATS KV bucket per execution domain:
```
WORKER_REGISTRY_<domain>
```

Keys:
```
worker.<name>.<instance_id>
```

Value:
```json
{
  "worker": "OrderService",
  "version": "v3",
  "executionDomain": "node-123",
  "endpoints": [
    {"type": "unix", "address": "/var/run/order.sock"},
    {"type": "http", "address": "http://127.0.0.1:8080"}
  ],
  "capabilities": ["order_management"],
  "health": "healthy",
  "ttl": 10
}
```

**TTL Behavior**:
- Refreshed by heartbeats
- Auto-expires on worker death
- No manual cleanup needed

### JetStream Configuration

#### Policy Stream
```yaml
stream:
  name: POLICY_DEFS
  subjects: ["policy.>"]
  retention: limits
  max_msgs_per_subject: 1
  discard: old
  storage: file
  replicas: 3
```

#### Spec Stream
```yaml
stream:
  name: SPEC_DEFS
  subjects: ["spec.>"]
  retention: limits
  max_msgs_per_subject: 1
  discard: old
  storage: file
  replicas: 3
```

**Rationale**: Keep only latest version per subject

### Worker Discovery Pattern

**Registration Flow**:
```
1. Worker starts
2. Loads local spec + policy registries from JetStream
3. Publishes endpoint to NATS KV
4. Subscribes to intent subjects
5. Starts heartbeat loop
6. Begins accepting work
```

**Heartbeat Loop**:
```
Every 5s:
  1. Emit health signal
  2. Refresh KV registration (TTL)
  3. Update local metrics
```

**Shutdown Flow**:
```
1. Receive SIGTERM
2. Stop accepting new intents (unsubscribe)
3. Finish in-flight work (bounded timeout)
4. Emit final signal
5. Exit cleanly (KV expires automatically)
```

### Locality-First Invocation

**Decision Tree**:
```
1. Check routing cache for target work unit
2. If local endpoint exists and constraints satisfied:
   → Use direct invocation (unix socket, loopback HTTP, in-process)
3. Else if remote endpoint exists and constraints satisfied:
   → Use remote invocation (HTTP, gRPC)
4. Else:
   → Publish to NATS subject (brokered execution)
5. If no response:
   → Fail with clear error
```

**Cache Update Events**:
- Worker registration
- Worker deregistration (TTL expiry)
- Heartbeat with state change
- Policy change affecting routing

**Cache Invalidation**:
- Policy lifecycle change
- Topology reconfiguration
- Health degradation
- Manual flush (operational)

---

## PART V: POLICY ENFORCEMENT PATTERNS

### Local Policy Registry

Each component maintains in-memory policy registry:

```go
type PolicyRegistry interface {
    // Load policies from control plane
    Load(stream JetStream) error
    
    // Evaluate policy against context
    Evaluate(ctx ExecutionContext, target Target) Decision
    
    // Get active policies
    ActivePolicies() []PolicyArtifact
    
    // Get shadow policies
    ShadowPolicies() []PolicyArtifact
}
```

### Enforcement Points

#### Input Gate (UI/API)
```
User submits InputIntent
  ↓
Evaluate InputIntent policies
  ↓
If denied → Reject with reason
If allowed → Submit to SMS
```

#### SMS Gate
```
InputIntent received
  ↓
Evaluate PolicySet for flow
  ↓
If denied → Reject flow
If allowed → Begin execution
```

#### Worker Gate
```
Work unit invocation received
  ↓
Evaluate DataPolicy for read/write/transition
  ↓
If denied → Return rejection outcome
If allowed → Execute work unit
```

### Shadow Policy Evaluation

**Dual Evaluation Pattern**:
```
For each request:
  1. Evaluate enforced policy → Binding decision
  2. Evaluate shadow policy → Record divergence
  3. If divergence:
     → Emit policy signal with metrics
  4. Return enforced decision
```

**Signal Format**:
```yaml
signal:
  type: policy
  metrics:
    shadow_allows: 1456
    shadow_denies: 14
    divergence_rate: 0.0095
```

**Promotion Criteria**:
- Divergence rate < threshold (e.g., 1%)
- Sustained over time window (e.g., 24h)
- Manual approval (optional)

### Policy Change Handling

#### Immediate Effect Changes
- Policy revocation
- Security hotfixes
- Compliance requirements

**Behavior**:
```
1. Receive lifecycle event (revoke)
2. Update local registry immediately
3. Reject new requests matching policy
4. Log affected requests
```

#### Graceful Changes
- Policy refinement
- Feature rollout
- Permission expansion

**Behavior**:
```
1. Deploy shadow policy
2. Monitor divergence
3. Promote when safe
4. Deprecate old policy
5. Retire after drain period
```

---

## PART VI: DATA & STATE MANAGEMENT

### State Storage Patterns

#### Persistent State
**Characteristics**:
- Authoritative source of truth
- Durable storage (database, object store)
- Strict consistency enforcement
- Versioned schema

**Implementation Pattern**:
```
Worker receives persist operation
  ↓
Validate constraints locally
  ↓
Write to storage with version
  ↓
Emit state change event (optional)
  ↓
Return success outcome
```

#### Intermediate State
**Characteristics**:
- Transient during flow execution
- May live in-memory or cache
- Deferred constraint validation
- Not exposed externally

**Implementation Pattern**:
```
Flow begins with InputIntent
  ↓
Create intermediate state
  ↓
Pass through enrichment steps
  ↓
Validate constraints
  ↓
Transition to persistent state
```

#### Materialized State
**Characteristics**:
- Derived from persistent state
- Regenerable on demand
- Eventual consistency acceptable
- Optimized for reads

**Implementation Pattern**:
```
Persistent state changes
  ↓
Trigger materialization (async)
  ↓
Apply transformations + filters
  ↓
Write to read-optimized store
  ↓
Emit freshness signal
```

### Data Evolution Implementation

#### Backward Read Compatibility
**New code reads old data**:
```go
func ReadOrder(version string, data []byte) (Order, error) {
    switch version {
    case "v3":
        return parseV3(data)
    case "v2":
        // Read v2, provide defaults for v3 fields
        v2 := parseV2(data)
        return v2.ToV3WithDefaults(), nil
    }
}
```

#### Forward Write Compatibility
**Old code writes, new code reads**:
```go
func WriteOrder(order Order) error {
    // Always write latest version
    data := serialize(order, "v3")
    return store.Put("order:"+order.ID, data, "v3")
}
```

#### Shadow Write Pattern
**Zero-downtime migration**:
```go
func WriteOrderWithShadow(order Order) error {
    // Write to old version (authoritative)
    err := store.Put("order:"+order.ID, serializeV2(order), "v2")
    if err != nil {
        return err
    }
    
    // Shadow write to new version (validation)
    shadowErr := store.Put("order:"+order.ID+":shadow", serializeV3(order), "v3")
    if shadowErr != nil {
        emitShadowError(shadowErr)
    }
    
    return nil
}
```

### Constraint Enforcement Timing

#### Strict Enforcement (Synchronous)
```
User submits InputIntent
  ↓
Validate all guarantees immediately
  ↓
If violation → Reject with details
If pass → Continue
```

**Use Cases**:
- Financial transactions
- Compliance requirements
- Data integrity

#### Deferred Enforcement (Asynchronous)
```
User submits InputIntent
  ↓
Accept with warning
  ↓
Process in background
  ↓
Validate deferred constraints
  ↓
If violation → Compensate or alert
```

**Use Cases**:
- External API calls
- Complex computations
- Risk scoring

#### Best Effort Enforcement
```
User submits InputIntent
  ↓
Attempt validation
  ↓
If validator unavailable → Log and continue
If validation fails → Log and continue
```

**Use Cases**:
- Analytics
- Audit logs
- Non-critical metadata

---

## PART VII: FLOW EXECUTION IMPLEMENTATION

### Flow State Machine

**State Representation**:
```go
type FlowInstance struct {
    ID              string
    FlowSpec        FlowSpec
    CurrentStep     int
    StepStates      map[string]StepState
    ExecutionContext ExecutionContext
    Data            map[string]interface{}
}

type StepState string
const (
    Pending   StepState = "pending"
    Running   StepState = "running"
    Completed StepState = "completed"
    Failed    StepState = "failed"
)
```

**Execution Loop**:
```go
func (e *FlowEngine) Execute(flow FlowSpec, input Intent) Result {
    instance := NewFlowInstance(flow, input)
    
    for !instance.IsComplete() {
        step := instance.NextStep()
        
        // Evaluate policies
        decision := e.policies.Evaluate(instance.Context, step)
        if decision.Denied {
            return Rejection(decision.Reason)
        }
        
        // Execute step
        result, err := e.executeStep(instance, step)
        if err != nil {
            return e.handleFailure(instance, step, err)
        }
        
        // Advance
        instance.AdvanceWithResult(result)
    }
    
    return Success(instance.Data)
}
```

### Atomic Group Execution

**Enforcement Pattern**:
```go
func (e *FlowEngine) executeAtomicGroup(group AtomicGroup, input Data) (Data, error) {
    // Select execution location ONCE
    location := e.router.SelectLocation(group.Boundary)
    
    state := input
    for _, step := range group.Steps {
        // Execute within same boundary
        result, err := e.executeAt(location, step, state)
        if err != nil {
            // Any failure fails entire group
            return nil, err
        }
        state = result
    }
    
    // Only expose result after all steps complete
    return state, nil
}
```

**Key Properties**:
- Single placement decision
- Sequential execution within group
- No external visibility of intermediate state
- All-or-nothing semantics

### Compensation Pattern

**Implementation**:
```go
func (e *FlowEngine) executeWithCompensation(flow FlowSpec, input Intent) Result {
    var completedSteps []CompletedStep
    
    for _, step := range flow.Steps {
        result, err := e.executeStep(step, input)
        if err != nil {
            // Failure: run compensations in reverse order
            for i := len(completedSteps) - 1; i >= 0; i-- {
                comp := findCompensation(completedSteps[i])
                if comp != nil {
                    e.executeStep(comp, completedSteps[i].Output)
                }
            }
            return Failure(err)
        }
        completedSteps = append(completedSteps, CompletedStep{step, result})
        input = result
    }
    
    return Success(input)
}
```

---

## PART VIII: SIGNAL & SCHEDULING IMPLEMENTATION

### Signal Emission Patterns

#### Worker Capacity Signal
```go
func (w *Worker) emitCapacitySignal() {
    signal := Signal{
        Type: Capacity,
        Source: Source{
            Component: "worker",
            Name: w.Name,
            Region: w.Region,
        },
        Metrics: map[string]interface{}{
            "inflight": w.inflightCount,
            "max_capacity": w.maxCapacity,
            "queue_depth": w.queueDepth,
        },
        Severity: w.calculateSeverity(),
        Timestamp: time.Now(),
    }
    
    w.nats.Publish("signal.worker."+w.Name+".capacity", signal)
}
```

#### SMS Backpressure Signal
```go
func (sms *FlowEngine) emitBackpressureSignal(flow FlowSpec) {
    signal := Signal{
        Type: Backpressure,
        Source: Source{
            Component: "sms",
            Name: flow.Name,
        },
        Metrics: map[string]interface{}{
            "blocked_transitions": sms.blockedCount,
            "wait_time_avg_ms": sms.avgWaitTime,
        },
    }
    
    sms.nats.Publish("signal.sms."+flow.Name+".backpressure", signal)
}
```

### Scheduler Integration

**External Scheduler Pattern**:
```
Scheduler (Kubernetes HPA, Nomad, custom)
  ↓
Subscribes to: signal.worker.>
  ↓
Aggregates metrics by worker class
  ↓
Applies scaling rules (from WTS)
  ↓
Issues scaling commands to infrastructure
  ↓
Infrastructure provisions/deprovisions workers
  ↓
Workers auto-register with NATS
```

**Scaling Decision Algorithm**:
```python
def should_scale_up(signals, policy):
    for rule in policy.signals:
        if evaluate(rule.condition, signals):
            if rule.action == "scale_up":
                return True
    return False

def calculate_target_count(current, signals, policy):
    target = current
    
    # Apply scale-up rules
    if should_scale_up(signals, policy):
        target += policy.scale_increment
    
    # Apply scale-down rules
    if should_scale_down(signals, policy):
        target -= policy.scale_increment
    
    # Clamp to bounds
    return max(policy.min, min(policy.max, target))
```

### Graceful Draining

**Worker Shutdown Sequence**:
```go
func (w *Worker) Shutdown(ctx context.Context) error {
    // 1. Stop accepting new work
    w.subscription.Unsubscribe()
    
    // 2. Mark as draining
    w.updateStatus("draining")
    w.emitCapacitySignal()
    
    // 3. Wait for in-flight work (with timeout)
    done := w.waitForInflight()
    select {
    case <-done:
        // Clean completion
    case <-ctx.Done():
        // Timeout: log and exit
        w.logger.Warn("forced shutdown with in-flight work")
    }
    
    // 4. Emit final signal
    w.emitCapacitySignal()
    
    // 5. Exit (KV registration auto-expires)
    return nil
}
```

---

## PART IX: OBSERVABILITY IMPLEMENTATION

### Correlation Propagation

**Context Injection**:
```go
type ExecutionContext struct {
    TraceID     string
    CausationID string
    Actor       Actor
    Metadata    map[string]string
}

func (e *FlowEngine) Execute(flow FlowSpec, input Intent) Result {
    ctx := NewExecutionContext(input)
    
    for _, step := range flow.Steps {
        // Propagate context
        result := e.executeStepWithContext(ctx, step)
        
        // Create causation chain
        ctx.CausationID = ctx.TraceID
        ctx.TraceID = generateNewTraceID()
    }
}
```

**NATS Header Propagation**:
```go
func (w *Worker) InvokeViaNATS(subject string, payload Data, ctx ExecutionContext) {
    msg := &nats.Msg{
        Subject: subject,
        Data: serialize(payload),
        Header: nats.Header{
            "Trace-ID": []string{ctx.TraceID},
            "Causation-ID": []string{ctx.CausationID},
            "Actor-Subject": []string{ctx.Actor.Subject},
        },
    }
    w.nats.PublishMsg(msg)
}
```

### Observability Hooks

**Required Events**:
1. Flow started
2. Step started
3. Step completed
4. Policy evaluated
5. Routing decision made
6. Flow completed
7. Flow failed

**Event Structure**:
```json
{
  "event": "step_started",
  "trace_id": "550e8400-...",
  "causation_id": "660e8400-...",
  "timestamp": "2024-01-15T14:23:45Z",
  "component": "sms",
  "flow": "OrderCreationFlow",
  "step": "reserve_inventory",
  "metadata": {}
}
```

---

## PART X: MULTI-REGION PATTERNS

### Region-Aware Signal Aggregation

**Per-Region Schedulers**:
```
Region: us-east
  ↓
Scheduler subscribes: signal.worker.*.*.us-east
  ↓
Aggregates only regional signals
  ↓
Makes scaling decisions for us-east workers
```

### Cross-Region Failover

**Automatic Failover Pattern**:
```
1. Workers in us-east stop emitting heartbeats (outage)
2. KV registrations expire (TTL)
3. Routing caches update (remove dead endpoints)
4. New requests route to us-west workers automatically
5. Scheduler in us-west sees increased load
6. Scheduler scales up us-west workers
```

**No coordination required** - emergent behavior from local decisions

### Regional Policy Distribution

**Policy Replication**:
```
Control Plane (global)
  ↓
Publishes policy to POLICY_DEFS stream
  ↓
JetStream replicates to all regions
  ↓
Workers in each region replay stream on startup
  ↓
Local enforcement with global consistency
```

---

## PART XI: TESTING STRATEGIES

### Grammar Validation Testing

**Static Analysis**:
- Parse all YAML artifacts
- Validate against schema
- Check cross-references
- Verify version sequences

**Example**:
```go
func TestGrammarValidity(t *testing.T) {
    spec := LoadSpec("order-system.yaml")
    
    // Structural validation
    assert.NoError(t, ValidateStructure(spec))
    
    // Semantic validation
    assert.NoError(t, ValidateSemantics(spec))
    
    // Cross-reference validation
    assert.NoError(t, ValidateReferences(spec))
}
```

### Policy Shadow Testing

**Pattern**:
1. Deploy new policy in shadow mode
2. Run production traffic
3. Compare enforced vs shadow decisions
4. Analyze divergence
5. Refine policy
6. Promote when divergence < threshold

### Flow Replay Testing

**Pattern**:
```go
func TestFlowReplay(t *testing.T) {
    // Record flow execution
    recorder := NewFlowRecorder()
    result1 := engine.ExecuteWithRecorder(flow, input, recorder)
    
    // Replay with same inputs
    replayer := NewFlowReplayer(recorder.Events())
    result2 := replayer.Replay()
    
    // Results must match
    assert.Equal(t, result1, result2)
}
```

### Chaos Testing

**Scenarios**:
1. Kill random workers mid-execution
2. Partition network between regions
3. Delay policy distribution
4. Corrupt KV registrations
5. Inject slow workers

**Assertions**:
- No data loss
- No duplicate processing (idempotency)
- Graceful degradation
- Recovery without manual intervention

---

## PART XII: OPERATIONAL PATTERNS

### Deployment Strategies

#### Blue-Green Deployment
```
1. Deploy new worker version (green)
2. Workers auto-register with version tag
3. Update routing rules to prefer green
4. Drain blue workers
5. Decommission blue after drain period
```

#### Canary Deployment
```
1. Deploy canary workers (5%)
2. Route 5% of traffic via version routing
3. Monitor signals (error rate, latency)
4. If healthy: gradually increase to 100%
5. If unhealthy: roll back instantly
```

### Incident Response

#### Policy Revocation
```
Incident: Security breach detected
  ↓
Operator issues policy revocation
  ↓
Control plane publishes lifecycle event
  ↓
All components update registries immediately
  ↓
New requests denied
  ↓
Crisis contained within seconds
```

#### Worker Quarantine
```
Incident: Worker behaving erratically
  ↓
Operator marks worker unhealthy
  ↓
Worker emits health signal
  ↓
Routing caches update
  ↓
No new work routed to unhealthy worker
  ↓
Drain and replace
```

### Monitoring & Alerting

**Key Metrics**:
1. Policy divergence rate (shadow)
2. Flow completion rate
3. Worker capacity utilization
4. Signal latency
5. Cache hit rate
6. Idempotency collisions
7. Regional health score

**Critical Alerts**:
- Policy distribution lag > 5s
- Worker drain timeout exceeded
- Idempotency violation detected
- Cross-boundary atomic group violation
- Constraint enforcement failure

---

## PART XIII: CODE GENERATION PATTERNS

### From Grammar to Go Types

**DataType → Go Struct**:
```yaml
# Input
dataType:
  name: Order
  version: v3
  fields:
    id: uuid
    total: decimal
```

```go
// Generated
type OrderV3 struct {
    ID    uuid.UUID     `json:"id"`
    Total decimal.Decimal `json:"total"`
}
```

### From InputIntent to Handlers

**InputIntent → Handler Interface**:
```yaml
# Input
inputIntent:
  name: CreateOrder
  version: v2
```

```go
// Generated
type CreateOrderV2Handler interface {
    Handle(ctx ExecutionContext, input CreateOrderV2Input) (OrderRecord, error)
}

// User implements
type OrderService struct {}

func (s *OrderService) Handle(ctx ExecutionContext, input CreateOrderV2Input) (OrderRecord, error) {
    // Business logic here
}
```

### From Policy to Evaluation Functions

**Policy → Evaluator**:
```yaml
# Input
policy:
  name: ModifyOrder
  when: subject.role == "admin"
```

```go
// Generated
func EvaluateModifyOrderV1(ctx ExecutionContext, target Target) Decision {
    return Decision{
        Allow: ctx.Actor.Role == "admin",
        Policy: "ModifyOrder v1",
    }
}
```

---

## PART XIV: IMPLEMENTATION CHECKLIST

### Minimum Viable Implementation

**Phase 1: Single Binary**
- [x] Load spec from YAML
- [x] In-process work unit execution
- [x] Basic constraint validation
- [x] No NATS, no distribution

**Phase 2: Control Plane**
- [x] NATS integration
- [x] Spec distribution via JetStream
- [x] Worker registration via KV
- [x] Heartbeat loop

**Phase 3: Policy**
- [x] Policy loading from stream
- [x] Local policy evaluation
- [x] Shadow policy support
- [x] Lifecycle management

**Phase 4: Multi-Worker**
- [x] Intent-based routing
- [x] Version-aware routing
- [x] Locality-first invocation
- [x] Graceful draining

**Phase 5: Observability**
- [x] Signal emission
- [x] Correlation propagation
- [x] Metric collection
- [x] Trace reconstruction

**Phase 6: Production**
- [x] Multi-region deployment
- [x] Scheduler integration
- [x] Chaos testing
- [x] Incident runbooks

---

## PART XV: AUTHORITY CONTROL PLANE (NEW in v1.3)

### Authority Control Worker

The Authority Control Worker manages entity authority state.

**Responsibilities**:
- Maintain authority state in `SMS.AUTHORITY` KV bucket
- Process authority transition requests
- Validate governance constraints
- Grant and revoke authority leases

**NATS Integration**:
```
# Authority KV bucket
SMS.AUTHORITY.{entity_type}.{entity_id}

# Authority transition requests
SMS.AUTHORITY.TRANSITION.{entity_type}

# Authority lease renewals
SMS.AUTHORITY.LEASE.{entity_id}
```

### Authority State Storage

```go
type AuthorityState struct {
    EntityID        string    `json:"entity_id"`
    AuthorityRegion string    `json:"authority_region"`
    AuthorityEpoch  int64     `json:"authority_epoch"`
    AuthorityLease  string    `json:"authority_lease"`
    Status          string    `json:"status"` // ACTIVE | TRANSITIONING
    EntityVersion   int64     `json:"entity_version"`
    UpdatedAt       time.Time `json:"updated_at"`
}
```

### Write Worker Authority Validation

```go
func (w *WriteWorker) ValidateAuthority(entityID string) error {
    state, err := w.authorityKV.Get(entityID)
    if err != nil {
        return ErrAuthorityUnknown
    }
    
    if state.AuthorityRegion != w.region {
        return ErrWrongRegion
    }
    
    if state.Status == "TRANSITIONING" {
        return ErrAuthorityTransitioning
    }
    
    if !w.leaseValidator.Valid(state.AuthorityLease) {
        return ErrLeaseExpired
    }
    
    return nil
}
```

### CAS Token Validation

```go
func (w *WriteWorker) ValidateCASToken(token CASToken, current AuthorityState) error {
    if token.AuthorityEpoch != current.AuthorityEpoch {
        return ErrEpochMismatch
    }
    if token.EntityVersion != current.EntityVersion {
        return ErrVersionMismatch
    }
    if token.AuthorityRegion != current.AuthorityRegion {
        return ErrWrongRegion
    }
    return nil
}
```

---

## PART XVI: MULTI-REGION SURVIVABILITY (NEW in v1.3)

### Regional Isolation Pattern

Each region is self-contained with:
- Regional event streams
- Regional view workers
- Regional KV storage
- Regional authority for local entities

```
US-EAST
 ├─ STREAM user.profile.v3.us
 ├─ STREAM activity.v1.us
 ├─ KV user_dashboard.v2.us
 ├─ View workers (us-only)
 └─ UI workers (us-only)

EU-WEST
 ├─ STREAM user.profile.v3.eu
 ├─ STREAM activity.v1.eu
 ├─ KV user_dashboard.v2.eu
 ├─ View workers (eu-only)
 └─ UI workers (eu-only)
```

### View-Only Replication

Raw data never crosses regions. Only derived, policy-sanitized views replicate.

```go
type ViewReplicator struct {
    sourceRegion string
    targetRegion string
}

func (r *ViewReplicator) Replicate(view ViewSnapshot) error {
    // Validate view is marked for replication
    if !view.ReplicationAllowed {
        return ErrReplicationNotAllowed
    }
    
    // Validate no PII
    if view.ContainsPII {
        return ErrCannotReplicatePII
    }
    
    // Replicate to global view stream
    return r.globalStream.Publish(view)
}
```

### Failover Behavior

**Regional Failure Impact**:
| Entity Authority | Impact |
|------------------|--------|
| Owned by failed region | Writes unavailable, reads degraded |
| Owned by healthy region | Fully operational |

**Degraded Mode Behavior**:
```go
func (w *UIWorker) HandleRead(entityID string) (Response, error) {
    view, err := w.viewKV.Get(entityID)
    
    if err == ErrRegionUnavailable {
        // Serve stale cached view with freshness metadata
        cached := w.cache.Get(entityID)
        return Response{
            Data:      cached.Data,
            Freshness: "stale",
            AsOf:      cached.Timestamp,
        }, nil
    }
    
    return Response{Data: view.Data, Freshness: "current"}, nil
}
```

### Authority Transition Under Failure

```go
func (c *AuthorityController) HandleTransitionFailure(
    entityID string, 
    fromRegion string,
    toRegion string,
    epoch int64,
) error {
    // Determine current state
    state, err := c.authorityKV.Get(entityID)
    if err != nil {
        return err
    }
    
    // If transition not committed, revert to original
    if state.AuthorityEpoch < epoch {
        log.Info("Transition not committed, authority remains", 
            "entity", entityID, 
            "region", fromRegion)
        return nil
    }
    
    // If transition committed, new region is authoritative
    if state.AuthorityEpoch == epoch && state.AuthorityRegion == toRegion {
        log.Info("Transition committed, new authority active",
            "entity", entityID,
            "region", toRegion)
        return nil
    }
    
    return ErrAmbiguousAuthorityState
}
```

---

## PART XVII: ANTI-PATTERNS & GOTCHAS

### Anti-Pattern: Model-Scoped Authority (NEW in v1.3)

❌ **Wrong**: Assign authority to entire model, causing availability collapse

✅ **Right**: Entity-scoped authority, failures affect only specific entities

### Anti-Pattern: Cross-Entity CAS (NEW in v1.3)

❌ **Wrong**: Multi-entity CAS requiring global coordination

✅ **Right**: Entity-local CAS with relationship invariants in views

### Anti-Pattern: Central Coordinator

❌ **Wrong**: Single service that knows about all workers and routes everything

✅ **Right**: Distributed routing via NATS subjects + local caches

### Anti-Pattern: Synchronous Policy Checks

❌ **Wrong**: Call auth service on every request

✅ **Right**: Evaluate policies locally from cached registry

### Anti-Pattern: Tight Coupling to NATS

❌ **Wrong**: Hardcode NATS publish/subscribe in business logic

✅ **Right**: Abstract transport behind interfaces

### Anti-Pattern: Version Skipping

❌ **Wrong**: Jump from Order v1 → v3

✅ **Right**: Evolve through v2 with compatibility declarations

### Anti-Pattern: Hidden State

❌ **Wrong**: Worker stores state in-memory

✅ **Right**: Worker delegates to external storage

### Anti-Pattern: Manual Cleanup

❌ **Wrong**: Operator manually deletes KV entries

✅ **Right**: TTL-based expiry handles cleanup automatically

---

## SUMMARY

This implementation specification provides:

1. **Architectural patterns** for runtime components
2. **NATS integration** for control plane and execution
3. **Policy enforcement** with local evaluation
4. **Data evolution** with zero-downtime patterns
5. **Flow execution** as finite state machines
6. **Signal-driven scheduling** with operational feedback
7. **Multi-region patterns** with automatic failover
8. **Testing strategies** including chaos engineering
9. **Operational patterns** for deployment and incident response
10. **Code generation** patterns for spec → implementation
11. **Authority control plane** for entity-scoped write ownership (NEW in v1.3)
12. **Multi-region survivability** with view-only replication (NEW in v1.3)
13. **CAS token validation** for linearizable writes (NEW in v1.3)

**Key Principle**: Every implementation decision preserves the grammar's intent-first, runtime-decided philosophy.

