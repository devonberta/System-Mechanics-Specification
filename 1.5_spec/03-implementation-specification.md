# System Mechanics Specification - Implementation Reference v1.5

## Overview

This document provides detailed implementation guidance for building systems conformant with the System Mechanics Specification. It covers runtime architecture, NATS integration patterns, control plane design, authority management, multi-region survivability, UI composition runtime, and operational concerns.

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

## PART XVIII: ANTI-PATTERNS & GOTCHAS

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

## PART XVII: UI COMPOSITION RUNTIME (NEW in v1.4)

### Overview

The UI Composition Runtime resolves compositions into renderable view trees, manages navigation state, enforces field permissions, and routes between experiences.

### Composition Resolver

The composition resolver transforms specification compositions into runtime view structures:

```
┌─────────────────────────────────────────────────┐
│ Composition Resolver                            │
├─────────────────────────────────────────────────┤
│ Input: Composition spec + context              │
│                                                 │
│ 1. Resolve primary view                        │
│ 2. Evaluate navigation policy                  │
│ 3. Resolve required related views              │
│ 4. Apply data scopes                           │
│ 5. Build view tree                             │
│                                                 │
│ Output: Resolved view tree with data bindings  │
└─────────────────────────────────────────────────┘
```

**Resolution Algorithm**:

```
resolve_composition(composition, context):
  # 1. Check composition access
  IF NOT evaluate_policy(composition.navigation.policy_set, context.subject):
    RETURN access_denied(composition.on_unauthorized)
  
  # 2. Resolve primary view
  primary = resolve_view(composition.primary, context)
  IF NOT primary:
    RETURN error("Primary view not found")
  
  # 3. Build view tree
  tree = {primary: primary, related: []}
  
  FOR each related in composition.related:
    # Evaluate related view policy
    IF related.policy AND NOT evaluate_policy(related.policy, context.subject):
      IF related.required:
        RETURN error("Required view access denied")
      CONTINUE  # Skip optional views
    
    # Apply data scope
    scoped_context = apply_data_scope(context, related.data_scope)
    
    # Resolve related view
    view = resolve_view(related.view, scoped_context)
    IF NOT view AND related.required:
      RETURN error("Required view not found")
    
    tree.related.append({view: view, relationship: related.relationship})
  
  RETURN tree
```

### Navigation Tree Builder

The navigation tree builder constructs the navigation hierarchy from composition parent relationships:

```
build_navigation_tree(experience):
  roots = []
  nodes = {}
  
  # Index all compositions
  FOR each composition in experience.includes:
    nodes[composition.name] = {
      composition: composition,
      children: [],
      parent: null
    }
  
  # Build parent-child relationships
  FOR each name, node in nodes:
    parent_ref = node.composition.navigation.parent
    IF parent_ref:
      parent_node = nodes[parent_ref]
      IF NOT parent_node:
        ERROR("Parent composition not found: " + parent_ref)
      parent_node.children.append(node)
      node.parent = parent_node
    ELSE:
      roots.append(node)
  
  # Validate acyclicity
  IF has_cycle(roots):
    ERROR("Navigation hierarchy contains cycle")
  
  RETURN roots
```

### Experience Router

The experience router manages experience selection and navigation:

```
┌─────────────────────────────────────────────────┐
│ Experience Router                               │
├─────────────────────────────────────────────────┤
│ 1. Identify target experience                  │
│ 2. Check authentication                        │
│ 3. Evaluate experience policy                  │
│ 4. Resolve target composition                  │
│ 5. Handle on_unauthorized behavior             │
└─────────────────────────────────────────────────┘
```

**Routing Algorithm**:

```
route_to_experience(experience_name, composition_name, params, subject):
  experience = get_experience(experience_name)
  
  # Check authentication
  IF NOT subject.authenticated:
    IF experience.unauthenticated:
      RETURN redirect(experience.unauthenticated.redirect_to)
    RETURN error("Authentication required")
  
  # Check experience policy
  IF NOT evaluate_policy(experience.policy_set, subject):
    RETURN handle_unauthorized(experience.on_unauthorized)
  
  # Validate composition in experience
  IF composition_name NOT IN experience.includes:
    RETURN error("Composition not in experience")
  
  # Resolve and return composition
  context = {subject: subject, params: params, experience: experience}
  RETURN resolve_composition(get_composition(composition_name), context)
```

### Field Permission Enforcer

The field permission enforcer applies visibility and masking rules:

```
enforce_field_permissions(view, data, subject):
  result = {}
  
  FOR each field, value in data:
    permission = view.fieldPermissions.find(field)
    
    # Default: visible, no mask
    IF NOT permission:
      result[field] = value
      CONTINUE
    
    # Check visibility
    visible = evaluate_visibility(permission.visible, subject)
    IF NOT visible:
      CONTINUE  # Omit field
    
    # Check masking
    IF permission.mask:
      IF NOT evaluate_expression(permission.mask.unless, subject):
        result[field] = apply_mask(value, permission.mask)
      ELSE:
        result[field] = value
    ELSE:
      result[field] = value
  
  RETURN result

apply_mask(value, mask_spec):
  SWITCH mask_spec.type:
    CASE "full":
      RETURN mask_full(value)
    CASE "partial":
      RETURN mask_partial(value, mask_spec.reveal)
    CASE "hash":
      RETURN hash(value)
```

### Indicator Evaluator

The indicator evaluator updates navigation indicators based on data:

```
evaluate_indicators(navigation_tree, data_context):
  FOR each node in navigation_tree:
    IF node.composition.navigation.indicator:
      indicator = node.composition.navigation.indicator
      IF evaluate_expression(indicator.when, data_context):
        node.indicator_active = true
        node.indicator_severity = indicator.severity
      ELSE:
        node.indicator_active = false
    
    # Recurse to children
    evaluate_indicators(node.children, data_context)
```

### Caching Strategy

**Composition Resolution Cache**:
- Key: `(composition_name, composition_version, subject_id)`
- TTL: 5 minutes (configurable)
- Invalidation: On policy change, on composition change

**Navigation Tree Cache**:
- Key: `(experience_name, experience_version)`
- TTL: 15 minutes (configurable)
- Invalidation: On experience change, on included composition change

**Field Permission Cache**:
- Key: `(view_name, view_version, subject_roles)`
- TTL: 5 minutes (configurable)
- Invalidation: On policy change, on view change

### Integration with Policy Engine

The UI Composition Runtime integrates with the policy engine for authorization:

```
┌─────────────────────────────────────────────────┐
│ Policy Integration                              │
├─────────────────────────────────────────────────┤
│ Experience Access:                             │
│   policy_set → experience default              │
│                                                 │
│ Composition Access:                            │
│   policy_set → composition.navigation.policy_set │
│   Falls back to: experience.policy_set         │
│                                                 │
│ Related View Access:                           │
│   policy → related.policy or related.policy_set │
│   Falls back to: composition policy            │
│                                                 │
│ Field Visibility:                              │
│   policy → fieldPermission.visible.policy      │
│   No fallback (default: visible)               │
└─────────────────────────────────────────────────┘
```

---

## PART XIX: V1.5 RUNTIME COMPONENTS (NEW in v1.5)

### Overview

Version 1.5 introduces comprehensive application development capabilities including form intent binding, session management, search semantics, scheduled triggers, external integrations, and advanced features for building complete end-to-end applications. This section provides implementation guidance for these new runtime components.

---

### Form Intent Binding Runtime

**Purpose**: Enable UI forms to submit intents with field mapping and validation.

**Runtime Responsibilities**:

1. **Field Mapping Resolution**
   ```go
   type FormBinder struct {
       view          PresentationView
       intent        InputIntent
       fieldMappings []FieldMapping
   }
   
   func (fb *FormBinder) BindFormToIntent(formData map[string]interface{}, context ExecutionContext) (Intent, error) {
       intentData := make(map[string]interface{})
       
       for _, mapping := range fb.fieldMappings {
           var value interface{}
           
           switch mapping.Source {
           case "input":
               value = formData[mapping.ViewField]
           case "context":
               value = context.Get(mapping.ViewField)
           case "computed":
               value = fb.evaluateExpression(mapping.ComputedExpr, formData, context)
           }
           
           // Apply validation
           if mapping.Validation != nil {
               if err := fb.validateField(value, mapping.Validation); err != nil {
                   return Intent{}, FieldValidationError{
                       Field:   mapping.ViewField,
                       Message: mapping.Validation.Message,
                   }
               }
           }
           
           intentData[mapping.SuppliesField] = value
       }
       
       return Intent{
           Name:    fb.intent.Name,
           Version: fb.intent.Version,
           Data:    intentData,
       }, nil
   }
   ```

2. **UI-Level Validation**
   - Validate constraints before intent submission
   - Provide immediate feedback to users
   - Reduce unnecessary server round-trips
   
   ```go
   func (fb *FormBinder) validateField(value interface{}, validation Validation) error {
       result := fb.expressionEvaluator.Evaluate(validation.Constraint, map[string]interface{}{
           "value": value,
       })
       
       if !result.(bool) {
           return errors.New(validation.Message)
       }
       
       return nil
   }
   ```

3. **Confirmation Handling**
   ```go
   func (fb *FormBinder) RequiresConfirmation(formData map[string]interface{}) (bool, string) {
       if !fb.view.Triggers.Confirmation.Required {
           return false, ""
       }
       
       message := fb.interpolateMessage(
           fb.view.Triggers.Confirmation.Message,
           formData,
       )
       
       return true, message
   }
   ```

4. **Success/Error Handling**
   ```go
   func (ui *UIRuntime) HandleIntentSubmission(result IntentResult, triggers Triggers) NavigationAction {
       if result.Success {
           // Handle success
           switch triggers.OnSuccess.Action {
           case "navigate_back":
               return ui.navigationStack.Pop()
           case "navigate_to":
               target := ui.resolveComposition(triggers.OnSuccess.Target)
               params := ui.evaluateParams(triggers.OnSuccess.Params, result.Data)
               return ui.navigationStack.Push(target, params)
           case "refresh":
               return ui.refreshCurrentView()
           case "stay":
               return ui.showNotification(triggers.OnSuccess.Notification)
           }
       } else {
           // Handle error
           switch triggers.OnError.Action {
           case "display_inline":
               return ui.showFieldErrors(result.Errors, triggers.OnError.ShowFieldErrors)
           case "display_modal":
               return ui.showErrorModal(result.Errors)
           case "retry":
               return ui.showRetryDialog()
           }
       }
   }
   ```

**Integration Points**:
- **Policy Engine**: Validate intent submission against policies before form binding
- **Session Management**: Inject session context into form data
- **Validation Engine**: Reuse constraint validation logic from data layer

---

### View Materialization & Retrieval Runtime

**Purpose**: Implement retrieval contracts, caching strategies, and temporal streaming for materialized views.

**Runtime Responsibilities**:

1. **Retrieval Mode Handling**
   ```go
   type ViewRetriever interface {
       GetByEntity(viewName string, entityKey string, context ExecutionContext) (ViewData, error)
       GetByQuery(viewName string, queryParams map[string]interface{}) ([]ViewData, error)
       GetSingleton(viewName string, context ExecutionContext) (ViewData, error)
   }
   
   type NATSViewRetriever struct {
       kvStore nats.KeyValue
       cache   *ViewCache
   }
   
   func (r *NATSViewRetriever) GetByEntity(viewName string, entityKey string, ctx ExecutionContext) (ViewData, error) {
       // Construct key
       key := fmt.Sprintf("%s.%s", viewName, entityKey)
       
       // Check cache first
       if cached, ok := r.cache.Get(key); ok {
           if !r.isStale(cached, ctx) {
               return cached.Data, nil
           }
       }
       
       // Fetch from KV
       entry, err := r.kvStore.Get(key)
       if err != nil {
           return r.handleUnavailable(viewName, entityKey, ctx)
       }
       
       var data ViewData
       if err := json.Unmarshal(entry.Value(), &data); err != nil {
           return ViewData{}, err
       }
       
       // Update cache
       r.cache.Set(key, CachedView{
           Data:      data,
           Timestamp: entry.Created(),
       })
       
       return data, nil
   }
   ```

2. **Unavailability Handling**
   ```go
   func (r *NATSViewRetriever) handleUnavailable(viewName string, entityKey string, ctx ExecutionContext) (ViewData, error) {
       materialization := r.registry.GetMaterialization(viewName)
       
       switch materialization.Retrieval.OnUnavailable.Strategy {
       case "use_cached":
           cached, ok := r.cache.Get(fmt.Sprintf("%s.%s", viewName, entityKey))
           if !ok {
               return ViewData{}, ErrViewNotAvailable
           }
           return cached.Data, nil
           
       case "fail":
           return ViewData{}, ErrViewNotAvailable
           
       case "degrade":
           fallbackView := materialization.Retrieval.OnUnavailable.DegradeTo
           return r.GetByEntity(fallbackView, entityKey, ctx)
           
       case "wait":
           timeout := materialization.Retrieval.OnUnavailable.WaitTimeout
           return r.waitForView(viewName, entityKey, timeout)
       }
       
       return ViewData{}, ErrViewNotAvailable
   }
   ```

3. **Temporal Window Processing**
   ```go
   type TemporalViewProcessor struct {
       windowType string
       windowSize time.Duration
       aggregator Aggregator
   }
   
   func (p *TemporalViewProcessor) ProcessWindow(events []Event, materialization Materialization) (ViewData, error) {
       windows := p.createWindows(events, materialization.Temporal.Window)
       results := []WindowResult{}
       
       for _, window := range windows {
           result := WindowResult{
               StartTime: window.Start,
               EndTime:   window.End,
           }
           
           // Apply aggregations
           for _, agg := range materialization.Temporal.Aggregation {
               value := p.aggregator.Aggregate(
                   window.Events,
                   agg.Field,
                   agg.Function,
               )
               result.Data[agg.As] = value
           }
           
           results = append(results, result)
       }
       
       return ViewData{Windows: results}, nil
   }
   
   func (p *TemporalViewProcessor) createWindows(events []Event, windowSpec WindowSpec) []Window {
       switch windowSpec.Type {
       case "tumbling":
           return p.createTumblingWindows(events, windowSpec.Size)
       case "sliding":
           return p.createSlidingWindows(events, windowSpec.Size, windowSpec.Slide)
       case "session":
           return p.createSessionWindows(events, windowSpec.Gap)
       case "hopping":
           return p.createHoppingWindows(events, windowSpec.Size, windowSpec.Slide)
       }
       return nil
   }
   ```

4. **Freshness Tracking**
   ```go
   func (r *NATSViewRetriever) isStale(cached CachedView, ctx ExecutionContext) bool {
       viewSpec := r.registry.GetView(ctx.ViewName)
       maxStaleness := viewSpec.Retrieval.Freshness.MaxStaleness
       
       age := time.Since(cached.Timestamp)
       return age > maxStaleness
   }
   ```

**Caching Strategy**:
- **L1 Cache**: In-memory cache per runtime instance
- **TTL**: Based on `freshness.max_staleness` specification
- **Invalidation**: On view update events, policy changes, or explicit flush

---

### Intent Delivery & Response Runtime

**Purpose**: Implement delivery guarantees, acknowledgments, and structured responses for intent routing.

**Runtime Responsibilities**:

1. **Delivery Guarantee Enforcement**
   ```go
   type IntentDeliverer interface {
       Deliver(intent Intent, context ExecutionContext) (IntentResult, error)
   }
   
   type NATSIntentDeliverer struct {
       js           nats.JetStreamContext
       nc           *nats.Conn
       idempotency  IdempotencyTracker
   }
   
   func (d *NATSIntentDeliverer) Deliver(intent Intent, ctx ExecutionContext) (IntentResult, error) {
       spec := d.registry.GetIntent(intent.Name, intent.Version)
       
       switch spec.Delivery.Guarantee {
       case "at_least_once":
           return d.deliverAtLeastOnce(intent, spec, ctx)
       case "at_most_once":
           return d.deliverAtMostOnce(intent, spec, ctx)
       case "exactly_once":
           return d.deliverExactlyOnce(intent, spec, ctx)
       }
       
       return IntentResult{}, ErrUnknownGuarantee
   }
   
   func (d *NATSIntentDeliverer) deliverExactlyOnce(intent Intent, spec InputIntent, ctx ExecutionContext) (IntentResult, error) {
       // Check idempotency
       idempotencyKey := d.calculateIdempotencyKey(intent, spec)
       
       if result, exists := d.idempotency.Check(idempotencyKey); exists {
           return result, nil
       }
       
       // Deliver with JetStream deduplication
       msg := &nats.Msg{
           Subject: d.constructSubject(intent),
           Data:    d.serialize(intent),
           Header: nats.Header{
               "Idempotency-Key": []string{idempotencyKey},
               "Trace-ID":        []string{ctx.TraceID},
           },
       }
       
       ack, err := d.js.PublishMsg(msg)
       if err != nil {
           return IntentResult{}, err
       }
       
       // Wait for response with timeout
       result, err := d.waitForResponse(idempotencyKey, spec.Delivery.Timeout)
       if err != nil {
           return IntentResult{}, err
       }
       
       // Store result for future idempotency checks
       d.idempotency.Store(idempotencyKey, result)
       
       return result, nil
   }
   ```

2. **Acknowledgment Handling**
   ```go
   func (d *NATSIntentDeliverer) deliverWithAck(intent Intent, spec InputIntent) (IntentResult, error) {
       switch spec.Delivery.Acknowledgment {
       case "required":
           return d.requestReply(intent, spec)
       case "optional":
           return d.requestReplyOptional(intent, spec)
       case "fire_and_forget":
           return d.publish(intent, spec)
       }
       return IntentResult{}, ErrInvalidAckMode
   }
   ```

3. **Retry Logic**
   ```go
   func (d *NATSIntentDeliverer) deliverWithRetry(intent Intent, spec InputIntent) (IntentResult, error) {
       if !spec.Delivery.Retry.Allowed {
           return d.deliver(intent, spec)
       }
       
       var lastErr error
       for attempt := 1; attempt <= spec.Delivery.Retry.MaxAttempts; attempt++ {
           result, err := d.deliver(intent, spec)
           if err == nil {
               return result, nil
           }
           
           lastErr = err
           
           // Calculate backoff
           backoff := d.calculateBackoff(attempt, spec.Delivery.Retry.Backoff)
           time.Sleep(backoff)
       }
       
       return IntentResult{}, fmt.Errorf("max retries exceeded: %w", lastErr)
   }
   
   func (d *NATSIntentDeliverer) calculateBackoff(attempt int, strategy string) time.Duration {
       switch strategy {
       case "linear":
           return time.Duration(attempt) * time.Second
       case "exponential":
           return time.Duration(math.Pow(2, float64(attempt))) * time.Second
       case "fixed":
           return 1 * time.Second
       }
       return 0
   }
   ```

4. **Structured Response Handling**
   ```go
   type IntentResult struct {
       Success       bool
       EntityID      string
       Data          map[string]interface{}
       Errors        []FieldError
       ErrorCode     string
       ErrorMessage  string
       Guarantees    []string
   }
   
   func (w *Worker) HandleIntent(intent Intent, spec InputIntent) IntentResult {
       // Execute work unit
       outcome, err := w.executeWorkUnit(intent)
       
       if err != nil {
           return IntentResult{
               Success:      false,
               ErrorCode:    w.classifyError(err),
               ErrorMessage: err.Error(),
               Errors:       w.extractFieldErrors(err),
           }
       }
       
       // Construct success response
       result := IntentResult{
           Success:    true,
           Guarantees: spec.Response.OnSuccess.Guarantees,
       }
       
       // Include specified fields
       for _, field := range spec.Response.OnSuccess.Includes {
           result.Data[field] = outcome.Data[field]
       }
       
       return result
   }
   ```

---

### Session Management Runtime

**Purpose**: Manage user sessions with authentication state, preferences, and temporal policies.

**Runtime Responsibilities**:

1. **Session Lifecycle**
   ```go
   type SessionManager struct {
       store      SessionStore
       validator  TokenValidator
       policies   PolicyRegistry
   }
   
   type Session struct {
       ID             string
       Subject        Subject
       CreatedAt      time.Time
       LastAccessedAt time.Time
       ExpiresAt      time.Time
       Preferences    map[string]interface{}
       State          map[string]interface{}
   }
   
   func (sm *SessionManager) CreateSession(subject Subject, duration time.Duration) (*Session, error) {
       session := &Session{
           ID:             generateSessionID(),
           Subject:        subject,
           CreatedAt:      time.Now(),
           LastAccessedAt: time.Now(),
           ExpiresAt:      time.Now().Add(duration),
           Preferences:    make(map[string]interface{}),
           State:          make(map[string]interface{}),
       }
       
       if err := sm.store.Store(session); err != nil {
           return nil, err
       }
       
       return session, nil
   }
   
   func (sm *SessionManager) ValidateSession(sessionID string) (*Session, error) {
       session, err := sm.store.Get(sessionID)
       if err != nil {
           return nil, ErrSessionNotFound
       }
       
       // Check expiry
       if time.Now().After(session.ExpiresAt) {
           sm.store.Delete(sessionID)
           return nil, ErrSessionExpired
       }
       
       // Update last accessed
       session.LastAccessedAt = time.Now()
       sm.store.Update(session)
       
       return session, nil
   }
   
   func (sm *SessionManager) RevokeSession(sessionID string) error {
       return sm.store.Delete(sessionID)
   }
   ```

2. **Authentication Integration**
   ```go
   func (sm *SessionManager) AuthenticateAndCreateSession(credentials Credentials) (*Session, string, error) {
       // Validate credentials
       subject, err := sm.authenticator.Authenticate(credentials)
       if err != nil {
           return nil, "", err
       }
       
       // Load subject policies
       policies := sm.policies.ForSubject(subject)
       
       // Determine session duration from policy
       duration := sm.determineSessionDuration(subject, policies)
       
       // Create session
       session, err := sm.CreateSession(subject, duration)
       if err != nil {
           return nil, "", err
       }
       
       // Generate token
       token, err := sm.tokenGenerator.GenerateToken(session)
       if err != nil {
           sm.store.Delete(session.ID)
           return nil, "", err
       }
       
       return session, token, nil
   }
   ```

3. **Session State Management**
   ```go
   func (sm *SessionManager) GetSessionState(sessionID string, key string) (interface{}, error) {
       session, err := sm.ValidateSession(sessionID)
       if err != nil {
           return nil, err
       }
       
       value, exists := session.State[key]
       if !exists {
           return nil, ErrKeyNotFound
       }
       
       return value, nil
   }
   
   func (sm *SessionManager) SetSessionState(sessionID string, key string, value interface{}) error {
       session, err := sm.ValidateSession(sessionID)
       if err != nil {
           return err
       }
       
       session.State[key] = value
       return sm.store.Update(session)
   }
   ```

4. **Idle Timeout & Sliding Expiry**
   ```go
   func (sm *SessionManager) RefreshSession(sessionID string, slidingWindow time.Duration) error {
       session, err := sm.store.Get(sessionID)
       if err != nil {
           return err
       }
       
       // Check if session is still valid
       if time.Now().After(session.ExpiresAt) {
           return ErrSessionExpired
       }
       
       // Extend expiry (sliding window)
       session.ExpiresAt = time.Now().Add(slidingWindow)
       session.LastAccessedAt = time.Now()
       
       return sm.store.Update(session)
   }
   ```

**Storage Backends**:
- **NATS KV**: Distributed session store with TTL
- **Redis**: High-performance session cache
- **Database**: Persistent session storage for audit

---

### Search Runtime

**Purpose**: Index and query materialized views with full-text and structured search.

**Runtime Responsibilities**:

1. **Index Management**
   ```go
   type SearchIndexer struct {
       backend SearchBackend
       analyzer TextAnalyzer
   }
   
   func (si *SearchIndexer) IndexView(view MaterializedView, spec SearchSpec) error {
       index := Index{
           Name:   spec.IndexName,
           Schema: si.buildSchema(spec),
       }
       
       // Create or update index
       if err := si.backend.CreateIndex(index); err != nil {
           return err
       }
       
       // Index documents
       for _, doc := range view.Data {
           searchDoc := si.transformDocument(doc, spec)
           if err := si.backend.IndexDocument(index.Name, searchDoc); err != nil {
               return err
           }
       }
       
       return nil
   }
   
   func (si *SearchIndexer) buildSchema(spec SearchSpec) IndexSchema {
       schema := IndexSchema{Fields: []FieldSchema{}}
       
       for _, field := range spec.Fields {
           fieldSchema := FieldSchema{
               Name: field.Name,
               Type: field.Type,
           }
           
           if field.Indexed {
               fieldSchema.Indexed = true
               fieldSchema.Analyzer = field.Analyzer
           }
           
           if field.Faceted {
               fieldSchema.Faceted = true
           }
           
           if field.Sortable {
               fieldSchema.Sortable = true
           }
           
           schema.Fields = append(schema.Fields, fieldSchema)
       }
       
       return schema
   }
   ```

2. **Query Execution**
   ```go
   type SearchExecutor struct {
       backend SearchBackend
       policies PolicyRegistry
   }
   
   func (se *SearchExecutor) ExecuteSearch(query SearchQuery, context ExecutionContext) (SearchResults, error) {
       spec := se.registry.GetSearchSpec(query.Index)
       
       // Validate search permissions
       if !se.policies.EvaluateSearch(spec, context.Subject) {
           return SearchResults{}, ErrSearchDenied
       }
       
       // Apply data scopes
       query = se.applyDataScopes(query, spec, context)
       
       // Execute search
       results, err := se.backend.Search(query)
       if err != nil {
           return SearchResults{}, err
       }
       
       // Apply field permissions
       results = se.applyFieldPermissions(results, spec, context)
       
       return results, nil
   }
   
   func (se *SearchExecutor) applyDataScopes(query SearchQuery, spec SearchSpec, ctx ExecutionContext) SearchQuery {
       for _, scope := range spec.Scopes {
           filter := se.evaluateScope(scope, ctx)
           query.Filters = append(query.Filters, filter)
       }
       return query
   }
   ```

3. **Real-Time Indexing**
   ```go
   func (si *SearchIndexer) StartRealtimeIndexing(viewName string, spec SearchSpec) error {
       // Subscribe to view update events
       sub, err := si.nats.Subscribe(fmt.Sprintf("view.%s.updated", viewName), func(msg *nats.Msg) {
           var update ViewUpdate
           if err := json.Unmarshal(msg.Data, &update); err != nil {
               return
           }
           
           switch update.Operation {
           case "create", "update":
               doc := si.transformDocument(update.Data, spec)
               si.backend.IndexDocument(spec.IndexName, doc)
           case "delete":
               si.backend.DeleteDocument(spec.IndexName, update.EntityID)
           }
       })
       
       if err != nil {
           return err
       }
       
       si.subscriptions[viewName] = sub
       return nil
   }
   ```

4. **Faceted Search**
   ```go
   func (se *SearchExecutor) ExecuteFacetedSearch(query SearchQuery) (FacetedResults, error) {
       results := FacetedResults{
           Hits:   []SearchHit{},
           Facets: make(map[string][]FacetValue),
       }
       
       // Execute main query
       hits, err := se.backend.Search(query)
       if err != nil {
           return results, err
       }
       results.Hits = hits
       
       // Compute facets
       for _, facetField := range query.Facets {
           facetValues, err := se.backend.ComputeFacet(query, facetField)
           if err != nil {
               continue
           }
           results.Facets[facetField] = facetValues
       }
       
       return results, nil
   }
   ```

**Search Backends**:
- **Elasticsearch**: Full-text search with analytics
- **MeiliSearch**: Fast, typo-tolerant search
- **Typesense**: Tuned for instant search experiences
- **PostgreSQL FTS**: SQL-based full-text search

---

### Scheduled Trigger Runtime

**Purpose**: Execute intents and flows on time-based schedules.

**Runtime Responsibilities**:

1. **Schedule Manager**
   ```go
   type ScheduleManager struct {
       scheduler   Scheduler
       intentDeliverer IntentDeliverer
       registry    SpecRegistry
   }
   
   func (sm *ScheduleManager) RegisterScheduledIntent(intent InputIntent) error {
       if !intent.Schedule.Enabled {
           return nil
       }
       
       switch intent.Schedule.Trigger.Type {
       case "cron":
           return sm.registerCronSchedule(intent)
       case "interval":
           return sm.registerIntervalSchedule(intent)
       case "delay":
           return sm.registerDelayedIntent(intent)
       }
       
       return ErrUnknownTriggerType
   }
   
   func (sm *ScheduleManager) registerCronSchedule(intent InputIntent) error {
       cronSpec := intent.Schedule.Trigger.Cron
       
       job := &ScheduledJob{
           ID:       generateJobID(intent),
           IntentSpec: intent,
           Handler: func() error {
               // Create intent instance
               intentInstance := Intent{
                   Name:    intent.Name,
                   Version: intent.Version,
                   Data:    sm.generateScheduledData(intent),
               }
               
               // Deliver intent
               ctx := ExecutionContext{
                   TraceID: generateTraceID(),
                   Actor:   Subject{Type: "system", ID: "scheduler"},
               }
               
               _, err := sm.intentDeliverer.Deliver(intentInstance, ctx)
               return err
           },
       }
       
       // Parse cron expression with timezone
       location, err := time.LoadLocation(cronSpec.Timezone)
       if err != nil {
           return err
       }
       
       return sm.scheduler.AddCronJob(cronSpec.Expression, location, job)
   }
   
   func (sm *ScheduleManager) registerIntervalSchedule(intent InputIntent) error {
       intervalSpec := intent.Schedule.Trigger.Interval
       
       job := &ScheduledJob{
           ID:       generateJobID(intent),
           IntentSpec: intent,
           Handler: func() error {
               intentInstance := Intent{
                   Name:    intent.Name,
                   Version: intent.Version,
                   Data:    sm.generateScheduledData(intent),
               }
               
               ctx := ExecutionContext{
                   TraceID: generateTraceID(),
                   Actor:   Subject{Type: "system", ID: "scheduler"},
               }
               
               _, err := sm.intentDeliverer.Deliver(intentInstance, ctx)
               return err
           },
       }
       
       return sm.scheduler.AddPeriodicJob(intervalSpec.Every, job)
   }
   ```

2. **Distributed Lock for Exactly-Once**
   ```go
   func (sm *ScheduleManager) executeWithLock(job *ScheduledJob) error {
       lockKey := fmt.Sprintf("schedule-lock:%s", job.ID)
       
       // Acquire distributed lock
       lock, err := sm.lockManager.TryLock(lockKey, 5*time.Minute)
       if err != nil {
           // Another instance is executing this job
           return nil
       }
       defer lock.Unlock()
       
       // Execute job
       return job.Handler()
   }
   ```

3. **Batch Window Processing**
   ```go
   func (sm *ScheduleManager) registerBatchWindow(intent InputIntent) error {
       batchSpec := intent.Schedule.Trigger.EventWindow
       
       collector := &BatchCollector{
           windowSize: batchSpec.WindowSize,
           maxEvents:  batchSpec.MaxEvents,
           events:     []Event{},
       }
       
       // Start collection
       go sm.collectBatchEvents(collector, intent)
       
       // Schedule batch processing
       ticker := time.NewTicker(batchSpec.WindowSize)
       go func() {
           for range ticker.C {
               sm.processBatch(collector, intent)
           }
       }()
       
       return nil
   }
   
   func (sm *ScheduleManager) processBatch(collector *BatchCollector, intent InputIntent) error {
       events := collector.DrainEvents()
       if len(events) == 0 {
           return nil
       }
       
       // Create batch intent
       intentInstance := Intent{
           Name:    intent.Name,
           Version: intent.Version,
           Data: map[string]interface{}{
               "events": events,
               "count":  len(events),
           },
       }
       
       ctx := ExecutionContext{
           TraceID: generateTraceID(),
           Actor:   Subject{Type: "system", ID: "batch-processor"},
       }
       
       _, err := sm.intentDeliverer.Deliver(intentInstance, ctx)
       return err
   }
   ```

4. **Signal-Based Scheduling**
   ```go
   func (sm *ScheduleManager) registerSignalTrigger(intent InputIntent) error {
       signalSpec := intent.Schedule.Trigger.Signal
       
       // Subscribe to signal
       subject := fmt.Sprintf("signal.%s", signalSpec.Pattern)
       sub, err := sm.nats.Subscribe(subject, func(msg *nats.Msg) {
           var signal Signal
           if err := json.Unmarshal(msg.Data, &signal); err != nil {
               return
           }
           
           // Evaluate condition
           if !sm.evaluateSignalCondition(signal, signalSpec.Condition) {
               return
           }
           
           // Trigger intent
           intentInstance := Intent{
               Name:    intent.Name,
               Version: intent.Version,
               Data:    sm.extractSignalData(signal, signalSpec),
           }
           
           ctx := ExecutionContext{
               TraceID:     generateTraceID(),
               CausationID: signal.ID,
               Actor:       Subject{Type: "system", ID: "signal-trigger"},
           }
           
           sm.intentDeliverer.Deliver(intentInstance, ctx)
       })
       
       return err
   }
   ```

**Scheduler Backends**:
- **Embedded**: In-process cron scheduler
- **Kubernetes CronJob**: Native k8s scheduling
- **Temporal**: Durable workflow scheduler
- **AWS EventBridge**: Cloud-based scheduling

---

### Durable Workflow Runtime

**Purpose**: Execute long-running workflows with human tasks and durable state.

**Runtime Responsibilities**:

1. **Workflow State Persistence**
   ```go
   type DurableFlowEngine struct {
       store       WorkflowStore
       taskManager TaskManager
       engine      FlowEngine
   }
   
   type WorkflowInstance struct {
       ID              string
       FlowSpec        FlowSpec
       CurrentStep     int
       StepStates      map[string]StepState
       ExecutionContext ExecutionContext
       Data            map[string]interface{}
       Status          WorkflowStatus
       Checkpoints     []Checkpoint
       CreatedAt       time.Time
       UpdatedAt       time.Time
   }
   
   func (dfe *DurableFlowEngine) StartWorkflow(flow FlowSpec, input Intent) (string, error) {
       instance := &WorkflowInstance{
           ID:              generateWorkflowID(),
           FlowSpec:        flow,
           CurrentStep:     0,
           StepStates:      make(map[string]StepState),
           ExecutionContext: NewExecutionContext(input),
           Data:            input.Data,
           Status:          "running",
           CreatedAt:       time.Now(),
           UpdatedAt:       time.Now(),
       }
       
       // Persist initial state
       if err := dfe.store.Save(instance); err != nil {
           return "", err
       }
       
       // Start execution
       go dfe.executeWorkflow(instance)
       
       return instance.ID, nil
   }
   
   func (dfe *DurableFlowEngine) executeWorkflow(instance *WorkflowInstance) {
       for !instance.IsComplete() {
           step := instance.NextStep()
           
           // Check for await points
           if step.Await != nil {
               if err := dfe.handleAwaitPoint(instance, step); err != nil {
                   instance.Status = "failed"
                   dfe.store.Save(instance)
                   return
               }
               // Workflow paused, will resume when await is satisfied
               return
           }
           
           // Execute step
           result, err := dfe.engine.ExecuteStep(instance, step)
           if err != nil {
               instance.Status = "failed"
               dfe.store.Save(instance)
               return
           }
           
           // Update state
           instance.AdvanceWithResult(result)
           instance.UpdatedAt = time.Now()
           
           // Persist state after each step
           if err := dfe.store.Save(instance); err != nil {
               instance.Status = "failed"
               return
           }
           
           // Check for checkpoint
           if step.Checkpoint != "" {
               dfe.createCheckpoint(instance, step.Checkpoint)
           }
       }
       
       instance.Status = "completed"
       dfe.store.Save(instance)
   }
   ```

2. **Await Point Handling**
   ```go
   func (dfe *DurableFlowEngine) handleAwaitPoint(instance *WorkflowInstance, step Step) error {
       switch step.Await.Trigger {
       case "inputIntent":
           return dfe.awaitInputIntent(instance, step)
       case "timeout":
           return dfe.awaitTimeout(instance, step)
       case "signal":
           return dfe.awaitSignal(instance, step)
       }
       return ErrUnknownAwaitTrigger
   }
   
   func (dfe *DurableFlowEngine) awaitInputIntent(instance *WorkflowInstance, step Step) error {
       task := HumanTask{
           WorkflowID:  instance.ID,
           StepID:      step.Name,
           IntentName:  step.Await.IntentName,
           AssignedTo:  dfe.resolveAssignment(instance, step.Await.Assignment),
           CreatedAt:   time.Now(),
           Timeout:     step.Await.Timeout,
           Status:      "pending",
       }
       
       // Create task
       if err := dfe.taskManager.CreateTask(task); err != nil {
           return err
       }
       
       // Update workflow status
       instance.Status = "awaiting_input"
       instance.UpdatedAt = time.Now()
       
       return dfe.store.Save(instance)
   }
   
   func (dfe *DurableFlowEngine) ResumeWorkflow(workflowID string, intent Intent) error {
       // Load workflow instance
       instance, err := dfe.store.Get(workflowID)
       if err != nil {
           return err
       }
       
       if instance.Status != "awaiting_input" {
           return ErrWorkflowNotAwaiting
       }
       
       // Inject intent data
       instance.Data = mergeData(instance.Data, intent.Data)
       instance.Status = "running"
       instance.UpdatedAt = time.Now()
       
       // Resume execution
       go dfe.executeWorkflow(instance)
       
       return nil
   }
   ```

3. **Human Task Management**
   ```go
   type TaskManager struct {
       store       TaskStore
       notifier    Notifier
       assigner    TaskAssigner
   }
   
   func (tm *TaskManager) CreateTask(task HumanTask) error {
       // Assign task
       assignee, err := tm.assigner.AssignTask(task)
       if err != nil {
           return err
       }
       task.AssignedTo = assignee
       
       // Store task
       if err := tm.store.Save(task); err != nil {
           return err
       }
       
       // Notify assignee
       if err := tm.notifier.NotifyTaskCreated(task); err != nil {
           // Log but don't fail
           log.Warn("Failed to notify task assignee", "task", task.ID, "error", err)
       }
       
       // Schedule timeout
       if task.Timeout > 0 {
           tm.scheduleTimeout(task)
       }
       
       return nil
   }
   
   func (tm *TaskManager) CompleteTask(taskID string, result Intent) error {
       task, err := tm.store.Get(taskID)
       if err != nil {
           return err
       }
       
       if task.Status != "pending" {
           return ErrTaskNotPending
       }
       
       // Update task
       task.Status = "completed"
       task.CompletedAt = time.Now()
       task.Result = result
       
       if err := tm.store.Save(task); err != nil {
           return err
       }
       
       // Resume workflow
       return tm.workflowEngine.ResumeWorkflow(task.WorkflowID, result)
   }
   
   func (tm *TaskManager) EscalateTask(taskID string) error {
       task, err := tm.store.Get(taskID)
       if err != nil {
           return err
       }
       
       // Reassign to escalation target
       newAssignee := tm.assigner.EscalateTask(task)
       task.AssignedTo = newAssignee
       task.EscalatedAt = time.Now()
       
       if err := tm.store.Save(task); err != nil {
           return err
       }
       
       // Notify new assignee
       return tm.notifier.NotifyTaskEscalated(task)
   }
   ```

4. **Checkpoint & Rollback**
   ```go
   func (dfe *DurableFlowEngine) createCheckpoint(instance *WorkflowInstance, name string) error {
       checkpoint := Checkpoint{
           Name:         name,
           WorkflowID:   instance.ID,
           StepIndex:    instance.CurrentStep,
           Data:         deepCopy(instance.Data),
           StepStates:   deepCopy(instance.StepStates),
           CreatedAt:    time.Now(),
       }
       
       instance.Checkpoints = append(instance.Checkpoints, checkpoint)
       return dfe.store.Save(instance)
   }
   
   func (dfe *DurableFlowEngine) RollbackToCheckpoint(workflowID string, checkpointName string) error {
       instance, err := dfe.store.Get(workflowID)
       if err != nil {
           return err
       }
       
       // Find checkpoint
       var checkpoint *Checkpoint
       for i := range instance.Checkpoints {
           if instance.Checkpoints[i].Name == checkpointName {
               checkpoint = &instance.Checkpoints[i]
               break
           }
       }
       
       if checkpoint == nil {
           return ErrCheckpointNotFound
       }
       
       // Restore state
       instance.CurrentStep = checkpoint.StepIndex
       instance.Data = checkpoint.Data
       instance.StepStates = checkpoint.StepStates
       instance.Status = "running"
       instance.UpdatedAt = time.Now()
       
       // Resume execution
       if err := dfe.store.Save(instance); err != nil {
           return err
       }
       
       go dfe.executeWorkflow(instance)
       return nil
   }
   ```

**Storage Backends**:
- **Temporal**: Built-in workflow durability
- **Database**: PostgreSQL/MySQL with JSON state
- **Event Store**: Event-sourced workflow state

---

### External Integration Runtime

**Purpose**: Connect to external APIs, webhooks, and data sources with resilience patterns.

**Runtime Responsibilities**:

1. **External Dependency Client**
   ```go
   type ExternalClient struct {
       dependency   ExternalDependency
       httpClient   *http.Client
       credentials  CredentialManager
       circuitBreaker CircuitBreaker
       rateLimiter  RateLimiter
   }
   
   func (ec *ExternalClient) Call(capability string, input map[string]interface{}) (map[string]interface{}, error) {
       contract := ec.dependency.Contract[capability]
       
       // Check circuit breaker
       if !ec.circuitBreaker.Allow() {
           return nil, ErrCircuitOpen
       }
       
       // Check rate limit
       if err := ec.rateLimiter.Wait(); err != nil {
           return nil, ErrRateLimited
       }
       
       // Get credentials
       creds, err := ec.credentials.Get(ec.dependency.Name)
       if err != nil {
           return nil, err
       }
       
       // Build request
       req, err := ec.buildRequest(capability, input, contract, creds)
       if err != nil {
           return nil, err
       }
       
       // Execute with retry
       var resp *http.Response
       var lastErr error
       
       for attempt := 1; attempt <= contract.Retry.MaxAttempts; attempt++ {
           resp, lastErr = ec.httpClient.Do(req)
           if lastErr == nil && resp.StatusCode < 500 {
               break
           }
           
           // Backoff before retry
           backoff := ec.calculateBackoff(attempt, contract.Retry.Strategy)
           time.Sleep(backoff)
       }
       
       if lastErr != nil {
           ec.circuitBreaker.RecordFailure()
           return nil, lastErr
       }
       
       ec.circuitBreaker.RecordSuccess()
       
       // Parse response
       output, err := ec.parseResponse(resp, contract)
       if err != nil {
           return nil, err
       }
       
       // Validate output schema
       if err := ec.validateOutput(output, contract.Output.Schema); err != nil {
           return nil, ErrInvalidResponse
       }
       
       return output, nil
   }
   ```

2. **Circuit Breaker**
   ```go
   type CircuitBreaker struct {
       state         string // closed, open, half_open
       failureCount  int
       successCount  int
       threshold     int
       timeout       time.Duration
       lastFailTime  time.Time
   }
   
   func (cb *CircuitBreaker) Allow() bool {
       switch cb.state {
       case "closed":
           return true
       case "open":
           if time.Since(cb.lastFailTime) > cb.timeout {
               cb.state = "half_open"
               return true
           }
           return false
       case "half_open":
           return true
       }
       return false
   }
   
   func (cb *CircuitBreaker) RecordSuccess() {
       if cb.state == "half_open" {
           cb.successCount++
           if cb.successCount >= 3 {
               cb.state = "closed"
               cb.failureCount = 0
               cb.successCount = 0
           }
       }
   }
   
   func (cb *CircuitBreaker) RecordFailure() {
       cb.lastFailTime = time.Now()
       cb.failureCount++
       
       if cb.failureCount >= cb.threshold {
           cb.state = "open"
       }
   }
   ```

3. **Webhook Handler**
   ```go
   type WebhookHandler struct {
       registry    ExternalRegistry
       validator   SignatureValidator
       processor   WebhookProcessor
   }
   
   func (wh *WebhookHandler) HandleWebhook(r *http.Request, dependencyName string) error {
       dependency := wh.registry.GetDependency(dependencyName)
       
       // Read body
       body, err := io.ReadAll(r.Body)
       if err != nil {
           return err
       }
       
       // Verify signature
       signature := r.Header.Get("X-Signature")
       if !wh.validator.Verify(body, signature, dependency.Webhook.Signature) {
           return ErrInvalidSignature
       }
       
       // Parse webhook payload
       var payload map[string]interface{}
       if err := json.Unmarshal(body, &payload); err != nil {
           return err
       }
       
       // Extract event type
       eventType := wh.extractEventType(payload, dependency.Webhook.EventTypeField)
       
       // Find mapped intent
       intentName := dependency.Webhook.EventMapping[eventType]
       if intentName == "" {
           // Unknown event type, log and ignore
           log.Info("Ignoring unknown webhook event", "type", eventType)
           return nil
       }
       
       // Transform payload to intent
       intent, err := wh.transformWebhookToIntent(payload, intentName, dependency)
       if err != nil {
           return err
       }
       
       // Deliver intent
       ctx := ExecutionContext{
           TraceID: generateTraceID(),
           Actor:   Subject{Type: "external", ID: dependencyName},
       }
       
       _, err = wh.intentDeliverer.Deliver(intent, ctx)
       return err
   }
   ```

4. **Credential Management**
   ```go
   type CredentialManager struct {
       store SecretStore
       cache *CredentialCache
   }
   
   func (cm *CredentialManager) Get(dependencyName string) (Credentials, error) {
       // Check cache
       if cached, ok := cm.cache.Get(dependencyName); ok {
           if !cached.Expired() {
               return cached.Credentials, nil
           }
       }
       
       // Load from secret store
       creds, err := cm.store.Get(dependencyName)
       if err != nil {
           return Credentials{}, err
       }
       
       // Refresh if needed
       if creds.Type == "oauth" && creds.NeedsRefresh() {
           creds, err = cm.refreshOAuthToken(creds)
           if err != nil {
               return Credentials{}, err
           }
           cm.store.Update(dependencyName, creds)
       }
       
       // Cache
       cm.cache.Set(dependencyName, creds)
       
       return creds, nil
   }
   
   func (cm *CredentialManager) refreshOAuthToken(creds Credentials) (Credentials, error) {
       req := &http.Request{
           Method: "POST",
           URL:    creds.OAuth.TokenURL,
           Body: io.NopCloser(strings.NewReader(
               fmt.Sprintf("grant_type=refresh_token&refresh_token=%s&client_id=%s&client_secret=%s",
                   creds.OAuth.RefreshToken,
                   creds.OAuth.ClientID,
                   creds.OAuth.ClientSecret,
               ),
           )),
           Header: http.Header{
               "Content-Type": []string{"application/x-www-form-urlencoded"},
           },
       }
       
       resp, err := http.DefaultClient.Do(req)
       if err != nil {
           return Credentials{}, err
       }
       defer resp.Body.Close()
       
       var tokenResp OAuthTokenResponse
       if err := json.NewDecoder(resp.Body).Decode(&tokenResp); err != nil {
           return Credentials{}, err
       }
       
       creds.OAuth.AccessToken = tokenResp.AccessToken
       creds.OAuth.ExpiresAt = time.Now().Add(time.Duration(tokenResp.ExpiresIn) * time.Second)
       
       return creds, nil
   }
   ```

**Integration Patterns**:
- **Sync Request-Response**: Direct API calls with timeout
- **Async Webhook**: Inbound events trigger intents
- **Polling**: Periodic checks for external state changes
- **Streaming**: Real-time data feeds

---

### Data Governance Runtime

**Purpose**: Enforce compliance, audit trails, and data lifecycle policies.

**Runtime Responsibilities**:

1. **Audit Logger**
   ```go
   type AuditLogger struct {
       stream   nats.JetStreamContext
       enricher AuditEnricher
   }
   
   type AuditEvent struct {
       EventID       string
       Timestamp     time.Time
       Actor         Subject
       Action        string
       Resource      Resource
       Result        string
       DataChanges   []DataChange
       Justification string
       TraceID       string
       CausationID   string
   }
   
   func (al *AuditLogger) LogAccess(ctx ExecutionContext, resource Resource, action string, result string) error {
       event := AuditEvent{
           EventID:     generateEventID(),
           Timestamp:   time.Now(),
           Actor:       ctx.Actor,
           Action:      action,
           Resource:    resource,
           Result:      result,
           TraceID:     ctx.TraceID,
           CausationID: ctx.CausationID,
       }
       
       // Enrich with governance metadata
       event = al.enricher.Enrich(event)
       
       // Publish to audit stream
       data, err := json.Marshal(event)
       if err != nil {
           return err
       }
       
       subject := fmt.Sprintf("audit.%s.%s", resource.Type, action)
       _, err = al.stream.Publish(subject, data)
       return err
   }
   
   func (al *AuditLogger) LogDataChange(ctx ExecutionContext, resource Resource, before, after map[string]interface{}) error {
       changes := al.computeDataChanges(before, after)
       
       event := AuditEvent{
           EventID:     generateEventID(),
           Timestamp:   time.Now(),
           Actor:       ctx.Actor,
           Action:      "modify",
           Resource:    resource,
           Result:      "success",
           DataChanges: changes,
           TraceID:     ctx.TraceID,
           CausationID: ctx.CausationID,
       }
       
       event = al.enricher.Enrich(event)
       
       data, err := json.Marshal(event)
       if err != nil {
           return err
       }
       
       subject := fmt.Sprintf("audit.%s.modify", resource.Type)
       _, err = al.stream.Publish(subject, data)
       return err
   }
   ```

2. **Data Classification**
   ```go
   type DataClassifier struct {
       rules []ClassificationRule
   }
   
   func (dc *DataClassifier) Classify(field Field, value interface{}) DataClassification {
       for _, rule := range dc.rules {
           if rule.Matches(field, value) {
               return rule.Classification
           }
       }
       return DataClassification{Level: "public"}
   }
   
   type DataClassification struct {
       Level          string // public, internal, confidential, restricted
       Sensitivity    []string // pii, phi, pci, etc.
       RetentionDays  int
       EncryptionReq  bool
       AccessControls []string
   }
   
   func (dc *DataClassifier) EnforceClassification(data map[string]interface{}, schema DataSchema) (map[string]interface{}, error) {
       classified := make(map[string]interface{})
       
       for field, value := range data {
           fieldDef := schema.GetField(field)
           classification := dc.Classify(fieldDef, value)
           
           // Apply encryption if required
           if classification.EncryptionReq {
               encrypted, err := dc.encryptor.Encrypt(value)
               if err != nil {
                   return nil, err
               }
               classified[field] = encrypted
           } else {
               classified[field] = value
           }
           
           // Store classification metadata
           classified[field+"_classification"] = classification.Level
       }
       
       return classified, nil
   }
   ```

3. **Retention Policy Enforcer**
   ```go
   type RetentionEnforcer struct {
       policies    []RetentionPolicy
       scheduler   Scheduler
       dataDeleter DataDeleter
   }
   
   func (re *RetentionEnforcer) EnforceRetention() {
       for _, policy := range re.policies {
           cutoffDate := time.Now().Add(-time.Duration(policy.RetentionDays) * 24 * time.Hour)
           
           // Find records to delete
           records, err := re.dataStore.QueryByDateRange(policy.DataType, time.Time{}, cutoffDate)
           if err != nil {
               log.Error("Failed to query records for retention", "error", err)
               continue
           }
           
           // Delete or archive
           for _, record := range records {
               if policy.ArchiveBeforeDelete {
                   if err := re.archiver.Archive(record); err != nil {
                       log.Error("Failed to archive record", "error", err)
                       continue
                   }
               }
               
               if err := re.dataDeleter.Delete(record.ID); err != nil {
                   log.Error("Failed to delete record", "error", err)
               }
           }
       }
   }
   
   func (re *RetentionEnforcer) ScheduleRetentionJobs() {
       // Run daily retention enforcement
       re.scheduler.AddCronJob("0 2 * * *", time.UTC, &ScheduledJob{
           ID:      "retention-enforcer",
           Handler: re.EnforceRetention,
       })
   }
   ```

4. **Data Subject Rights**
   ```go
   type DataSubjectRightsManager struct {
       dataStore   DataStore
       auditLogger AuditLogger
   }
   
   func (dsrm *DataSubjectRightsManager) HandleAccessRequest(subjectID string) (SubjectData, error) {
       // Collect all data for subject
       data := SubjectData{
           PersonalInfo: dsrm.dataStore.GetPersonalInfo(subjectID),
           ActivityLog:  dsrm.auditLogger.GetActivityLog(subjectID),
           Preferences:  dsrm.dataStore.GetPreferences(subjectID),
       }
       
       // Log access request
       dsrm.auditLogger.LogAccess(ExecutionContext{
           Actor: Subject{Type: "system", ID: "dsr-manager"},
       }, Resource{
           Type: "subject_data",
           ID:   subjectID,
       }, "export", "success")
       
       return data, nil
   }
   
   func (dsrm *DataSubjectRightsManager) HandleErasureRequest(subjectID string, ctx ExecutionContext) error {
       // Validate request
       if !dsrm.canErase(subjectID) {
           return ErrCannotErase
       }
       
       // Anonymize or delete data
       if err := dsrm.dataStore.AnonymizeSubject(subjectID); err != nil {
           return err
       }
       
       // Log erasure
       dsrm.auditLogger.LogAccess(ctx, Resource{
           Type: "subject_data",
           ID:   subjectID,
       }, "erase", "success")
       
       return nil
   }
   ```

**Compliance Frameworks**:
- **GDPR**: Right to access, erasure, portability
- **HIPAA**: Audit trails, encryption, access controls
- **SOC2**: Change logging, access reviews
- **PCI DSS**: Encryption, tokenization, audit logs

---

### Additional v1.5 Runtime Components

#### Asset Management Runtime
- **Asset Storage**: S3, CDN, local filesystem
- **Asset Processing**: Image resizing, video transcoding
- **Asset Delivery**: Signed URLs, CDN integration
- **Asset Metadata**: Content type, size, checksum

#### Notification Channel Runtime
- **Channel Routing**: Email, SMS, push, in-app
- **Template Rendering**: Mustache, Handlebars
- **Delivery Tracking**: Sent, delivered, opened, clicked
- **Preference Management**: User notification preferences

#### Collaborative Session Runtime
- **Presence Management**: Who's online, typing indicators
- **Conflict Resolution**: Operational transformation, CRDTs
- **Change Broadcasting**: WebSocket, SSE, long polling
- **Lock Management**: Pessimistic locking for critical sections

#### Conversational Experience Runtime
- **Intent Recognition**: NLU for text/voice input
- **Dialog Management**: Conversation state, context
- **Response Generation**: Template-based, generative
- **Multi-turn Handling**: Follow-up questions, clarifications

#### Edge Device Runtime
- **Offline Support**: Local storage, sync when online
- **Conflict Merge**: Divergent offline changes
- **Selective Sync**: Only relevant data to edge
- **Edge Computation**: Local policy evaluation

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
14. **Composition resolution** for semantic view grouping (NEW in v1.4)
15. **Experience routing** for application navigation (NEW in v1.4)
16. **Field permission enforcement** with visibility and masking (NEW in v1.4)
17. **Navigation tree building** from parent relationships (NEW in v1.4)
18. **Indicator evaluation** for dynamic navigation state (NEW in v1.4)
19. **Form intent binding** for UI form submission (NEW in v1.5)
20. **View materialization & retrieval** with caching strategies (NEW in v1.5)
21. **Intent delivery contracts** with guarantees and responses (NEW in v1.5)
22. **Session management** with authentication and state (NEW in v1.5)
23. **Search runtime** for indexing and querying views (NEW in v1.5)
24. **Scheduled trigger runtime** for time-based execution (NEW in v1.5)
25. **Durable workflow runtime** for long-running processes (NEW in v1.5)
26. **External integration** with circuit breakers and webhooks (NEW in v1.5)
27. **Data governance** with audit trails and compliance (NEW in v1.5)
28. **Asset management** for storage and delivery (NEW in v1.5)
29. **Notification channels** for multi-channel messaging (NEW in v1.5)
30. **Collaborative sessions** for real-time co-editing (NEW in v1.5)
31. **Conversational experiences** for chat/voice interfaces (NEW in v1.5)
32. **Edge device support** for offline-first applications (NEW in v1.5)

**Key Principle**: Every implementation decision preserves the grammar's intent-first, runtime-decided philosophy.

**Version History**:
- **v1.1**: Core runtime, NATS integration, policy enforcement
- **v1.3**: Entity-scoped authority, multi-region survivability, CAS tokens
- **v1.4**: UI composition, experience routing, field permissions, navigation
- **v1.5**: Complete application development support with forms, sessions, search, schedules, workflows, integrations, governance, and advanced features
