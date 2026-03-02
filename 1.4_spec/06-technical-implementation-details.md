# System Mechanics Specification - Technical Implementation Details v1.4

## Overview

This document captures deep technical nuances, implementation patterns, and operational details discussed throughout the specification development. It serves as a comprehensive technical reference for implementers.

### v1.4 Additions

This version adds:
- UI Composition runtime implementation (Part XX)
- Experience routing patterns (Part XX)
- Field permission enforcement (Part XX)
- Navigation state management (Part XX)

### v1.3 Additions

- Authority implementation patterns (Part XVI)
- Storage role implementation (Part XVII)
- View worker authority patterns (Part XVIII)
- Multi-region data patterns (Part XIX)

---

## PART I: NATS INTEGRATION DEEP DIVE

### 1.1 NATS Role & Boundaries

**What NATS IS Used For**:
1. **Control Plane Distribution**
   - Spec artifacts (DataTypes, Flows, Policies)
   - Lifecycle events (promote, deprecate, revoke)
   - Configuration updates

2. **Worker Discovery & Registration**
   - Endpoint registration via KV
   - TTL-based liveness
   - Capability advertisement

3. **Coordination & Signaling**
   - Operational signals (capacity, health, latency)
   - Policy evaluation results (shadow divergence)
   - Flow coordination events

4. **Cross-Boundary Execution (Fallback)**
   - When direct invocation unavailable
   - When constraints require remote execution
   - When no local worker exists

**What NATS IS NOT Used For**:
1. **Direct Execution Path (When Local Available)**
   - Co-located workers use direct invocation
   - Unix sockets, loopback HTTP preferred
   - NATS only when remote required

2. **Large Data Transfer**
   - Bulk data uses object storage references
   - Streaming uses side channels (WebRTC, gRPC)
   - NATS carries control messages only

3. **Authorization Decisions**
   - Policies distributed via NATS
   - Evaluation happens locally
   - No synchronous auth over NATS

4. **Durable Business State**
   - Flow state may be persisted via JetStream
   - Business data lives in databases
   - NATS is coordination, not storage

### 1.2 Subject Hierarchy Design

**Hierarchical Structure**:
```
<category>.<domain>.<component>.<entity>.<version>
```

**Categories**:

#### Control Plane Subjects
```
spec.order.dataType.Order.v3
spec.order.inputIntent.CreateOrder.v2
spec.order.smsFlow.CheckoutFlow.v1
spec.order.presentationView.OrderList.v2

policy.order.worker.OrderService.v2
policy.order.ui.OrderList.v1
policy.global.auth.FinanceAccess.v1

topology.order.placement.api_workers
topology.order.scaling.api_scale_policy
```

#### Execution Subjects
```
intent.order.create
intent.order.cancel
intent.payment.apply

work.order.validate
work.order.persist
work.inventory.reserve
```

#### Signal Subjects
```
signal.worker.OrderService.capacity
signal.worker.OrderService.latency
signal.worker.OrderService.health
signal.sms.CheckoutFlow.backpressure
signal.policy.ModifyOrder.shadow
```

#### Discovery Subjects (KV)
```
KV Bucket: WORKER_REGISTRY_order
Keys: worker.OrderService.instance-abc123
```

### 1.3 JetStream Configuration Details

#### Control Plane Streams

**SPEC_DEFS Stream**:
```yaml
name: SPEC_DEFS
subjects:
  - spec.>
storage: file
retention: limits
max_msgs_per_subject: 1
discard: old
max_age: 0  # Keep indefinitely
replicas: 3
```

**Rationale**:
- Keep only latest version per subject
- Persist indefinitely for replay
- Replicate for availability

**POLICY_DEFS Stream**:
```yaml
name: POLICY_DEFS
subjects:
  - policy.>
storage: file
retention: limits
max_msgs_per_subject: 1
discard: old
max_age: 0
replicas: 3
```

**LIFECYCLE_EVENTS Stream**:
```yaml
name: LIFECYCLE_EVENTS
subjects:
  - lifecycle.>
storage: file
retention: interest  # Keep until consumed
max_age: 7d
replicas: 3
```

**Rationale**: Lifecycle transitions need ordered delivery

#### Signal Streams (Optional)

**SIGNALS Stream** (for analysis, not real-time):
```yaml
name: SIGNALS
subjects:
  - signal.>
storage: file
retention: limits
max_age: 24h
max_bytes: 10GB
replicas: 1  # Single replica sufficient
```

**Rationale**: Historical analysis only, not hot path

### 1.4 KV Bucket Configuration

**Worker Registry**:
```yaml
bucket: WORKER_REGISTRY
ttl: 15s  # Workers must heartbeat within 15s
history: 1
replicas: 3
```

**Registration Entry**:
```json
{
  "worker": "OrderService",
  "version": "v3",
  "instance_id": "instance-abc123",
  "execution_domain": "node-123",
  "endpoints": [
    {
      "type": "unix",
      "address": "/var/run/order.sock",
      "priority": 1
    },
    {
      "type": "http",
      "address": "http://127.0.0.1:8080",
      "priority": 2
    },
    {
      "type": "nats",
      "subject": "work.order.>",
      "priority": 3
    }
  ],
  "capabilities": [
    "order_management",
    "inventory_check"
  ],
  "compatible_versions": ["v2", "v3"],
  "health": "healthy",
  "load": {
    "inflight": 23,
    "max_capacity": 100,
    "queue_depth": 5
  },
  "registered_at": "2024-01-15T14:23:45Z"
}
```

**TTL Behavior**:
- Refreshed by heartbeat every 5s
- Expires after 15s without heartbeat
- Auto-cleanup, no manual intervention
- Routing caches updated on expiry

---

## PART II: LOCALITY-FIRST INVOCATION PATTERNS

### 2.1 Routing Cache Design

**Cache Structure**:
```go
type RoutingCache struct {
    // Indexed by work unit name + version
    endpoints map[string][]WorkerEndpoint
    
    // Indexed by execution domain
    localWorkers map[string][]WorkerEndpoint
    
    // Policy-filtered cache
    authorizedEndpoints map[string]map[string][]WorkerEndpoint  // [workUnit][subject]
    
    // Mutex for updates
    mu sync.RWMutex
    
    // Last update time
    lastUpdate time.Time
}

type WorkerEndpoint struct {
    WorkerID    string
    Version     string
    Domain      string
    Endpoints   []TransportEndpoint
    Health      HealthStatus
    Load        LoadMetrics
    LastSeen    time.Time
}

type TransportEndpoint struct {
    Type     TransportType  // unix, http, nats
    Address  string
    Priority int
}
```

**Cache Update Triggers**:
1. Worker registration (KV watch)
2. Worker deregistration (TTL expiry)
3. Heartbeat with health change
4. Policy lifecycle change
5. Periodic refresh (every 30s)

**Cache Lookup Algorithm**:
```go
func (r *RoutingCache) SelectEndpoint(
    workUnit string,
    version string,
    constraints Constraints,
    context ExecutionContext,
) (*WorkerEndpoint, error) {
    
    r.mu.RLock()
    defer r.mu.RUnlock()
    
    // 1. Filter by capability and version
    candidates := r.filterByCapability(workUnit, version)
    
    // 2. Filter by execution domain (locality)
    local := r.filterByDomain(candidates, context.ExecutionDomain)
    if len(local) > 0 {
        candidates = local
    }
    
    // 3. Filter by policy (pre-computed)
    authorized := r.filterByPolicy(candidates, context.Actor)
    if len(authorized) == 0 {
        return nil, ErrNotAuthorized
    }
    
    // 4. Filter by health
    healthy := r.filterHealthy(authorized)
    if len(healthy) == 0 {
        return nil, ErrNoHealthyWorker
    }
    
    // 5. Score by load and proximity
    best := r.selectLowestLoad(healthy)
    
    return best, nil
}
```

**Scoring Function**:
```go
func (r *RoutingCache) scoreEndpoint(endpoint WorkerEndpoint, localDomain string) float64 {
    score := 0.0
    
    // Locality bonus
    if endpoint.Domain == localDomain {
        score += 100.0  // Strong preference for local
    }
    
    // Load factor
    utilization := float64(endpoint.Load.Inflight) / float64(endpoint.Load.MaxCapacity)
    score += (1.0 - utilization) * 50.0
    
    // Queue depth penalty
    score -= float64(endpoint.Load.QueueDepth) * 2.0
    
    // Health factor
    if endpoint.Health != Healthy {
        score -= 200.0  // Strong penalty for unhealthy
    }
    
    return score
}
```

### 2.2 Transport Selection Algorithm

**Decision Tree**:
```go
func (e *Executor) InvokeWorkUnit(
    workUnit string,
    input Data,
    context ExecutionContext,
) (Data, error) {
    
    // 1. Lookup from cache (no IO)
    endpoint := e.cache.SelectEndpoint(workUnit, context)
    
    // 2. Try endpoints in priority order
    for _, transport := range endpoint.Endpoints {
        
        switch transport.Type {
        
        case Unix:
            // Highest priority for co-located
            if e.canReachUnix(transport.Address) {
                return e.invokeUnix(transport.Address, input, context)
            }
        
        case HTTP:
            // Second priority for same-node
            if e.canReachHTTP(transport.Address) {
                return e.invokeHTTP(transport.Address, input, context)
            }
        
        case NATS:
            // Fallback for remote or when others unavailable
            return e.invokeNATS(transport.Subject, input, context)
        }
    }
    
    return nil, ErrNoAvailableTransport
}
```

**Key Principle**: Try local first, fall back gracefully.

### 2.3 Heartbeat Implementation

**Heartbeat Loop**:
```go
func (w *Worker) StartHeartbeatLoop(ctx context.Context) {
    ticker := time.NewTicker(5 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            w.emitHeartbeat()
        }
    }
}

func (w *Worker) emitHeartbeat() {
    // 1. Collect current metrics
    load := w.getCurrentLoad()
    health := w.assessHealth()
    
    // 2. Update KV registration (refreshes TTL)
    registration := WorkerRegistration{
        Worker: w.name,
        Version: w.version,
        InstanceID: w.instanceID,
        Endpoints: w.endpoints,
        Health: health,
        Load: load,
        RegisteredAt: time.Now(),
    }
    
    key := fmt.Sprintf("worker.%s.%s", w.name, w.instanceID)
    w.kv.Put(key, registration)
    
    // 3. Emit capacity signal (for schedulers)
    signal := Signal{
        Type: Capacity,
        Source: Source{
            Component: "worker",
            Name: w.name,
            Region: w.region,
        },
        Metrics: map[string]interface{}{
            "inflight": load.Inflight,
            "max_capacity": load.MaxCapacity,
            "queue_depth": load.QueueDepth,
            "cpu_usage": load.CPUUsage,
        },
        Severity: calculateSeverity(load),
        Timestamp: time.Now(),
    }
    
    w.nats.Publish("signal.worker."+w.name+".capacity", signal)
}
```

**Health Assessment**:
```go
func (w *Worker) assessHealth() HealthStatus {
    // Check multiple dimensions
    if w.panicRecoveryCount > 3 {
        return Unhealthy
    }
    if w.consecutiveFailures > 10 {
        return Degraded
    }
    if w.memoryPressure > 0.9 {
        return Degraded
    }
    if w.isShuttingDown {
        return Draining
    }
    return Healthy
}
```

### 2.4 Graceful Draining Details

**Shutdown Sequence**:
```go
func (w *Worker) Shutdown(ctx context.Context) error {
    log.Info("Beginning graceful shutdown")
    
    // 1. Update status to draining
    w.status = Draining
    w.emitHeartbeat()  // Immediately notify routing caches
    
    // 2. Unsubscribe from new work
    for _, sub := range w.subscriptions {
        sub.Unsubscribe()
    }
    log.Info("Stopped accepting new work")
    
    // 3. Wait for in-flight work with timeout
    done := make(chan struct{})
    go func() {
        w.wg.Wait()  // Wait for all work units to complete
        close(done)
    }()
    
    select {
    case <-done:
        log.Info("All in-flight work completed")
    case <-ctx.Done():
        log.Warn("Drain timeout exceeded, forcing shutdown",
            "remaining", w.inflightCount)
        // Fall through to cleanup
    }
    
    // 4. Emit final status
    w.status = Inactive
    w.emitHeartbeat()
    
    // 5. Close connections
    w.nats.Drain()  // Drain NATS connection
    
    // KV registration expires automatically via TTL
    
    return nil
}
```

**Routing Cache Response to Draining**:
```go
func (r *RoutingCache) OnHeartbeat(hb Heartbeat) {
    r.mu.Lock()
    defer r.mu.Unlock()
    
    endpoint := r.endpoints[hb.WorkerID]
    oldHealth := endpoint.Health
    endpoint.Health = hb.Health
    endpoint.Load = hb.Load
    endpoint.LastSeen = time.Now()
    
    // If transitioning to draining, mark in cache
    if oldHealth != Draining && hb.Health == Draining {
        endpoint.AcceptingWork = false
        log.Info("Worker draining, routing new work elsewhere",
            "worker", hb.WorkerID)
    }
    
    // If inactive, remove from cache
    if hb.Health == Inactive {
        delete(r.endpoints, hb.WorkerID)
        log.Info("Worker deregistered", "worker", hb.WorkerID)
    }
}
```

---

## PART II: POLICY ENFORCEMENT PATTERNS

### 2.1 Policy Registry Implementation

**Registry Structure**:
```go
type PolicyRegistry struct {
    // Active policies by name and version
    enforced map[string]map[string]Policy  // [name][version]
    
    // Shadow policies
    shadow map[string]map[string]Policy
    
    // Deprecated policies (grace period)
    deprecated map[string]map[string]Policy
    
    // Index by target type
    byTarget map[PolicyTargetType][]PolicyRef
    
    // Compiled expressions (cached)
    compiled map[string]CompiledExpression
    
    mu sync.RWMutex
}
```

**Policy Loading**:
```go
func (p *PolicyRegistry) LoadFromStream(stream JetStream) error {
    // 1. Subscribe to policy stream
    sub, err := stream.Subscribe("policy.>", func(msg *nats.Msg) {
        artifact := ParsePolicyArtifact(msg.Data)
        p.updatePolicy(artifact)
    })
    
    // 2. Replay historical messages
    // This ensures late-joining components get all policies
    
    return nil
}

func (p *PolicyRegistry) updatePolicy(artifact PolicyArtifact) {
    p.mu.Lock()
    defer p.mu.Unlock()
    
    name := artifact.Policy.Name
    version := artifact.Policy.Version
    
    switch artifact.Lifecycle.State {
    
    case Shadow:
        p.shadow[name][version] = artifact.Policy
        
    case Enforce:
        p.enforced[name][version] = artifact.Policy
        
        // Remove superseded version
        if artifact.Lifecycle.Supersedes != "" {
            delete(p.enforced[name], artifact.Lifecycle.Supersedes)
        }
        
    case Deprecated:
        p.deprecated[name][version] = artifact.Policy
        // Keep enforcing but warn
        
    case Revoked:
        delete(p.enforced[name], version)
        delete(p.shadow[name], version)
        delete(p.deprecated[name], version)
    }
    
    // Invalidate compiled cache
    delete(p.compiled, name+":"+version)
}
```

### 2.2 Dual Evaluation (Shadow + Enforce)

**Evaluation Function**:
```go
func (p *PolicyRegistry) Evaluate(
    ctx ExecutionContext,
    target Target,
) PolicyDecision {
    
    p.mu.RLock()
    defer p.mu.RUnlock()
    
    // 1. Evaluate enforced policies
    enforcedDecision := p.evaluateSet(p.enforced, ctx, target)
    
    // 2. Evaluate shadow policies
    shadowDecision := p.evaluateSet(p.shadow, ctx, target)
    
    // 3. Record divergence if any
    if enforcedDecision.Allow != shadowDecision.Allow {
        p.emitDivergenceSignal(
            target,
            enforcedDecision,
            shadowDecision,
        )
    }
    
    // 4. Return enforced decision only
    return enforcedDecision
}
```

**Divergence Signal**:
```go
func (p *PolicyRegistry) emitDivergenceSignal(
    target Target,
    enforced PolicyDecision,
    shadow PolicyDecision,
) {
    signal := Signal{
        Type: PolicyDivergence,
        Metrics: map[string]interface{}{
            "policy": shadow.Policy,
            "enforced_decision": enforced.Allow,
            "shadow_decision": shadow.Allow,
            "target": target.String(),
        },
    }
    
    p.nats.Publish("signal.policy."+shadow.Policy+".divergence", signal)
}
```

### 2.3 Policy Expression Compilation

**Lazy Compilation**:
```go
func (p *PolicyRegistry) getCompiledExpression(
    policy Policy,
) CompiledExpression {
    
    key := policy.Name + ":" + policy.Version
    
    if expr, ok := p.compiled[key]; ok {
        return expr
    }
    
    // Compile expression to bytecode or AST
    expr := compileExpression(policy.When)
    
    p.compiled[key] = expr
    
    return expr
}
```

**Evaluation Context**:
```go
type EvaluationContext struct {
    Subject    Subject        // Actor information
    Data       Data           // Data being accessed
    Environment map[string]any // Runtime context
    Target     Target         // What's being authorized
}

func (e CompiledExpression) Evaluate(ctx EvaluationContext) bool {
    // Execute compiled expression
    // Access ctx.Subject.Role, ctx.Data.field, etc.
    return result
}
```

---

## PART III: FLOW EXECUTION ENGINE INTERNALS

### 3.1 Flow State Machine Implementation

**State Representation**:
```go
type FlowInstance struct {
    // Identity
    FlowID          string
    FlowSpec        FlowSpec
    ExecutionContext ExecutionContext
    
    // State machine
    CurrentStep     int
    StepStates      map[string]StepExecutionState
    
    // Data
    InitialInput    Data
    CurrentData     Data
    StepOutputs     map[string]Data
    
    // Metadata
    StartedAt       time.Time
    LastTransition  time.Time
    
    // Synchronization
    mu sync.Mutex
}

type StepExecutionState struct {
    Status      StepStatus
    Attempts    int
    LastError   error
    StartedAt   time.Time
    CompletedAt time.Time
}

type StepStatus int
const (
    StepPending StepStatus = iota
    StepRunning
    StepCompleted
    StepFailed
    StepCompensating
)
```

**State Transitions**:
```go
func (f *FlowInstance) AdvanceToNextStep(result StepResult) error {
    f.mu.Lock()
    defer f.mu.Unlock()
    
    // 1. Update current step state
    currentStepName := f.FlowSpec.Steps[f.CurrentStep].Name
    f.StepStates[currentStepName] = StepExecutionState{
        Status: StepCompleted,
        CompletedAt: time.Now(),
    }
    
    // 2. Store step output
    f.StepOutputs[currentStepName] = result.Output
    
    // 3. Merge output into current data
    f.CurrentData = mergeData(f.CurrentData, result.Output)
    
    // 4. Advance cursor
    f.CurrentStep++
    
    // 5. Update timestamp
    f.LastTransition = time.Now()
    
    return nil
}
```

### 3.2 Atomic Group Execution Details

**Placement Decision**:
```go
func (e *FlowEngine) executeAtomicGroup(
    group AtomicGroup,
    input Data,
    context ExecutionContext,
) (Data, error) {
    
    // 1. Make placement decision ONCE
    location := e.router.SelectExecutionLocation(
        group.Boundary,
        context,
    )
    
    // 2. Create local context for group
    groupContext := context.WithBoundary(group.Boundary)
    
    // 3. Execute steps sequentially within location
    state := input
    for i, step := range group.Steps {
        
        // Execute at selected location
        result, err := e.executeStepAt(
            location,
            step,
            state,
            groupContext,
        )
        
        if err != nil {
            // Fail entire group immediately
            return nil, fmt.Errorf("atomic group failed at step %d: %w", i, err)
        }
        
        // Pass output to next step
        state = result
    }
    
    // 4. Only return after all steps complete
    return state, nil
}
```

**Cross-Runtime Atomic Groups**:
```go
func (e *FlowEngine) executeStepAt(
    location ExecutionLocation,
    step Step,
    input Data,
    context ExecutionContext,
) (Data, error) {
    
    if location.Local {
        // In-process execution
        workUnit := e.registry.Get(step.WorkUnit)
        return workUnit(context, input)
    } else {
        // Remote execution (still atomic from flow perspective)
        return e.transport.InvokeRemote(
            location.Endpoint,
            step.WorkUnit,
            input,
            context,
        )
    }
}
```

**Critical Invariant**: Atomic group executes under single authority, even if that authority is remote.

### 3.3 Compensation Execution

**Compensation State Machine**:
```go
func (e *FlowEngine) executeWithCompensation(
    flow FlowSpec,
    input Data,
    context ExecutionContext,
) (Data, error) {
    
    var completed []CompletedStep
    state := input
    
    for i, step := range flow.Steps {
        result, err := e.executeStep(step, state, context)
        
        if err != nil {
            log.Warn("Step failed, executing compensations",
                "step", step.Name,
                "error", err)
            
            // Execute compensations in reverse order
            e.compensate(flow, completed, context)
            
            return nil, fmt.Errorf("flow failed at step %s: %w", step.Name, err)
        }
        
        completed = append(completed, CompletedStep{
            StepIndex: i,
            StepName: step.Name,
            Input: state,
            Output: result,
            CompletedAt: time.Now(),
        })
        
        state = result
    }
    
    return state, nil
}

func (e *FlowEngine) compensate(
    flow FlowSpec,
    completed []CompletedStep,
    context ExecutionContext,
) {
    
    // Reverse order
    for i := len(completed) - 1; i >= 0; i-- {
        step := completed[i]
        
        // Find compensation for this step
        comp := flow.FindCompensation(step.StepName)
        if comp == nil {
            continue
        }
        
        log.Info("Executing compensation",
            "for_step", step.StepName,
            "compensation", comp.WorkUnit)
        
        // Execute compensation (best-effort)
        _, err := e.executeStep(comp, step.Input, context)
        if err != nil {
            // Log but continue with other compensations
            log.Error("Compensation failed",
                "step", step.StepName,
                "error", err)
        }
    }
}
```

---

## PART IV: VERSIONING & EVOLUTION MECHANICS

### 4.1 Multi-Version Worker Implementation

**Worker Version Support**:
```go
type MultiVersionWorker struct {
    name string
    
    // Handlers by version
    handlers map[string]WorkUnitHandler  // [version]
    
    // Schema registry
    schemas map[string]DataTypeSchema  // [version]
}

func (w *MultiVersionWorker) Handle(
    version string,
    input Data,
    context ExecutionContext,
) (Data, error) {
    
    // 1. Select version-specific handler
    handler, ok := w.handlers[version]
    if !ok {
        return nil, ErrVersionNotSupported
    }
    
    // 2. Parse input according to version schema
    schema := w.schemas[version]
    parsed, err := schema.Parse(input)
    if err != nil {
        return nil, fmt.Errorf("schema validation failed: %w", err)
    }
    
    // 3. Execute
    result, err := handler(context, parsed)
    if err != nil {
        return nil, err
    }
    
    // 4. Serialize output with version
    output := schema.Serialize(result)
    output.Version = version
    
    return output, nil
}
```

### 4.2 Shadow Write Implementation

**Dual Write Pattern**:
```go
func (w *Worker) PersistOrderWithShadow(order Order, context ExecutionContext) error {
    
    // 1. Write to authoritative version
    authVersion := "v2"
    err := w.storage.Write(
        "order:"+order.ID,
        serializeOrder(order, authVersion),
        authVersion,
    )
    if err != nil {
        return fmt.Errorf("authoritative write failed: %w", err)
    }
    
    // 2. Shadow write to new version (non-blocking)
    shadowVersion := "v3"
    go func() {
        shadowErr := w.storage.Write(
            "order:"+order.ID+":shadow",
            serializeOrder(order, shadowVersion),
            shadowVersion,
        )
        
        if shadowErr != nil {
            // Emit signal but don't fail request
            w.emitShadowWriteError(order.ID, shadowErr)
        } else {
            // Validate shadow data
            w.validateShadowData(order.ID, shadowVersion)
        }
    }()
    
    return nil
}
```

**Shadow Validation**:
```go
func (w *Worker) validateShadowData(orderID string, version string) {
    // Read both versions
    auth := w.storage.Read("order:" + orderID)
    shadow := w.storage.Read("order:" + orderID + ":shadow")
    
    // Compare semantically
    divergence := compareOrders(auth, shadow)
    
    if divergence {
        signal := Signal{
            Type: DataDivergence,
            Metrics: map[string]interface{}{
                "order_id": orderID,
                "auth_version": auth.Version,
                "shadow_version": version,
                "divergence_fields": divergence.Fields,
            },
        }
        w.nats.Publish("signal.data.shadow_divergence", signal)
    }
}
```

### 4.3 Version Negotiation

**Content Negotiation Pattern**:
```go
func (w *Worker) NegotiateVersion(
    requested []string,
    supported []string,
) (string, error) {
    
    // 1. Try to match latest compatible version
    for i := len(requested) - 1; i >= 0; i-- {
        reqVersion := requested[i]
        for j := len(supported) - 1; j >= 0; j-- {
            supVersion := supported[j]
            if reqVersion == supVersion {
                return reqVersion, nil
            }
            if w.isCompatible(reqVersion, supVersion) {
                return supVersion, nil
            }
        }
    }
    
    return "", ErrNoCompatibleVersion
}
```

**Compatibility Check**:
```go
func (w *Worker) isCompatible(requested string, supported string) bool {
    // Check evolution declarations
    evolution := w.schemas.GetEvolution(requested, supported)
    if evolution == nil {
        return false
    }
    
    // Check if worker can handle compatibility mode
    return evolution.Compatibility.Read == Backward ||
           evolution.Compatibility.Write == Forward
}
```

---

## PART V: SCHEDULER INTEGRATION PATTERNS

### 5.1 Signal Aggregation

**Scheduler Signal Consumer**:
```go
type SignalAggregator struct {
    // Time-windowed metrics
    windows map[string]*MetricWindow  // [worker_name]
    
    // Current snapshot
    current map[string]AggregatedMetrics
    
    mu sync.RWMutex
}

func (s *SignalAggregator) ConsumeSignals(stream JetStream) {
    sub, _ := stream.Subscribe("signal.worker.>", func(msg *nats.Msg) {
        signal := ParseSignal(msg.Data)
        s.aggregateSignal(signal)
    })
}

func (s *SignalAggregator) aggregateSignal(signal Signal) {
    s.mu.Lock()
    defer s.mu.Unlock()
    
    worker := signal.Source.Name
    window := s.windows[worker]
    
    if window == nil {
        window = NewMetricWindow(60 * time.Second)
        s.windows[worker] = window
    }
    
    // Add to time window
    window.Add(signal.Timestamp, signal.Metrics)
    
    // Update current snapshot
    s.current[worker] = AggregatedMetrics{
        QueueDepth: window.Percentile("queue_depth", 95),
        Latency: window.Percentile("latency_p95", 95),
        ErrorRate: window.Average("error_rate"),
        Capacity: window.Latest("inflight") / window.Latest("max_capacity"),
    }
}
```

### 5.2 Scaling Decision Engine

**Evaluation Loop**:
```go
func (s *Scheduler) EvaluationLoop(ctx context.Context) {
    ticker := time.NewTicker(10 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            s.evaluateAndScale()
        }
    }
}

func (s *Scheduler) evaluateAndScale() {
    topology := s.loadTopology()
    metrics := s.aggregator.CurrentMetrics()
    currentCounts := s.getCurrentWorkerCounts()
    
    for _, placement := range topology.Placements {
        current := currentCounts[placement.Name]
        desired := s.calculateDesiredCount(
            placement,
            metrics[placement.WorkerClass],
            current,
        )
        
        if desired != current {
            s.emitScalingIntent(placement.Name, desired)
        }
    }
}

func (s *Scheduler) calculateDesiredCount(
    placement Placement,
    metrics AggregatedMetrics,
    current int,
) int {
    
    desired := current
    
    policy := s.findScalingPolicy(placement.Name)
    if policy == nil {
        return desired  // No policy, no change
    }
    
    // Apply scale-up rules
    for _, rule := range policy.ScaleUpRules {
        if s.evaluateCondition(rule.Condition, metrics) {
            desired += rule.Delta
        }
    }
    
    // Apply scale-down rules
    for _, rule := range policy.ScaleDownRules {
        if s.evaluateCondition(rule.Condition, metrics) {
            desired -= rule.Delta
        }
    }
    
    // Clamp to bounds
    if desired < placement.Min {
        desired = placement.Min
    }
    if desired > placement.Max {
        desired = placement.Max
    }
    
    return desired
}
```

### 5.3 Infrastructure Adapter

**Kubernetes HPA Adapter**:
```go
type KubernetesAdapter struct {
    clientset kubernetes.Interface
    namespace string
}

func (k *KubernetesAdapter) ApplyDesiredState(
    workerClass string,
    location string,
    desired int,
) error {
    
    // Map worker class to K8s Deployment
    deploymentName := workerClass + "-" + location
    
    // Get current deployment
    deployment, err := k.clientset.AppsV1().
        Deployments(k.namespace).
        Get(context.TODO(), deploymentName, metav1.GetOptions{})
    
    if err != nil {
        return err
    }
    
    // Update replica count
    current := *deployment.Spec.Replicas
    if int(current) == desired {
        return nil  // Already at desired
    }
    
    log.Info("Scaling deployment",
        "deployment", deploymentName,
        "current", current,
        "desired", desired)
    
    deployment.Spec.Replicas = int32Ptr(int32(desired))
    
    _, err = k.clientset.AppsV1().
        Deployments(k.namespace).
        Update(context.TODO(), deployment, metav1.UpdateOptions{})
    
    return err
}
```

---

## PART VI: DATA EVOLUTION PATTERNS

### 6.1 Backward Read Implementation

**Read Adapter**:
```go
func (s *Storage) ReadWithCompatibility(
    key string,
    targetVersion string,
) (Data, error) {
    
    // 1. Read stored data
    stored, version, err := s.readWithVersion(key)
    if err != nil {
        return nil, err
    }
    
    // 2. If versions match, return directly
    if version == targetVersion {
        return stored, nil
    }
    
    // 3. Check if upgrade path exists
    if s.canUpgrade(version, targetVersion) {
        return s.upgrade(stored, version, targetVersion)
    }
    
    // 4. Check if downgrade is safe
    if s.canDowngrade(version, targetVersion) {
        return s.downgrade(stored, version, targetVersion)
    }
    
    return nil, ErrIncompatibleVersion
}

func (s *Storage) upgrade(data Data, from string, to string) (Data, error) {
    evolution := s.getEvolution(from, to)
    
    result := data.Clone()
    
    for _, change := range evolution.Changes {
        switch change.Type {
        case AddField:
            result[change.FieldName] = change.Default
        case ExtendEnum:
            // Existing values still valid
        }
    }
    
    return result, nil
}
```

### 6.2 Promotion Algorithm

**Shadow to Production Promotion**:
```go
func (c *ControlPlane) PromoteVersion(
    dataType string,
    from string,
    to string,
) error {
    
    // 1. Verify shadow write success rate
    stats := c.getShadowStats(dataType, to)
    if stats.ErrorRate > 0.01 {
        return ErrShadowQualityInsufficient
    }
    
    // 2. Verify shadow read compatibility
    divergence := c.getShadowDivergence(dataType, to)
    if divergence > 0.05 {
        return ErrShadowDivergenceTooHigh
    }
    
    // 3. Update routing rules to prefer new version
    c.updateRoutingPreference(dataType, to)
    
    // 4. Wait for traffic to shift (observe signals)
    time.Sleep(30 * time.Second)
    
    // 5. Verify new version handling traffic well
    if !c.verifyVersionHealth(dataType, to) {
        c.rollbackRouting(dataType, from)
        return ErrPromotionFailed
    }
    
    // 6. Make new version authoritative
    c.setAuthoritativeVersion(dataType, to)
    
    // 7. Begin draining old version
    c.deprecateVersion(dataType, from)
    
    log.Info("Version promoted successfully",
        "dataType", dataType,
        "from", from,
        "to", to)
    
    return nil
}
```

---

## PART VII: FAILURE HANDLING PATTERNS

### 7.1 Failure Classification

**Error Types**:
```go
type ExecutionError struct {
    Type       ErrorType
    Retryable  bool
    Reason     string
    Context    map[string]interface{}
}

type ErrorType int
const (
    // User/client errors (don't retry)
    ErrorValidation ErrorType = iota
    ErrorAuthorization
    ErrorConflict
    ErrorNotFound
    
    // Infrastructure errors (retry)
    ErrorTimeout
    ErrorNetworkFailure
    ErrorServiceUnavailable
    ErrorResourceExhausted
    
    // System errors (investigate)
    ErrorInternalFailure
    ErrorDataCorruption
    ErrorInvariantViolation
)
```

**Retry Decision**:
```go
func (e *FlowEngine) shouldRetry(err ExecutionError, attempts int) bool {
    // Never retry non-retryable errors
    if !err.Retryable {
        return false
    }
    
    // Respect max attempts
    if attempts >= e.maxRetries {
        return false
    }
    
    // Check error type
    switch err.Type {
    case ErrorValidation, ErrorAuthorization:
        return false  // Client errors
    case ErrorTimeout, ErrorServiceUnavailable:
        return true   // Infrastructure errors
    case ErrorInternalFailure:
        return true   // System errors (may be transient)
    }
    
    return false
}
```

### 7.2 Circuit Breaker Pattern

**Worker-Level Circuit Breaker**:
```go
type CircuitBreaker struct {
    state          CircuitState
    failureCount   int
    successCount   int
    lastFailure    time.Time
    threshold      int
    timeout        time.Duration
    mu sync.Mutex
}

type CircuitState int
const (
    Closed CircuitState = iota  // Normal operation
    Open                        // Failing, reject fast
    HalfOpen                    // Testing recovery
)

func (cb *CircuitBreaker) Call(fn func() error) error {
    cb.mu.Lock()
    
    switch cb.state {
    
    case Open:
        // Check if timeout elapsed
        if time.Since(cb.lastFailure) < cb.timeout {
            cb.mu.Unlock()
            return ErrCircuitOpen
        }
        // Try recovery
        cb.state = HalfOpen
        cb.mu.Unlock()
        
    case HalfOpen:
        cb.mu.Unlock()
        
    case Closed:
        cb.mu.Unlock()
    }
    
    // Execute
    err := fn()
    
    cb.mu.Lock()
    defer cb.mu.Unlock()
    
    if err != nil {
        cb.failureCount++
        cb.successCount = 0
        cb.lastFailure = time.Now()
        
        if cb.failureCount >= cb.threshold {
            cb.state = Open
            log.Warn("Circuit breaker opened",
                "failures", cb.failureCount)
        }
    } else {
        cb.successCount++
        cb.failureCount = 0
        
        if cb.state == HalfOpen && cb.successCount >= 3 {
            cb.state = Closed
            log.Info("Circuit breaker closed (recovered)")
        }
    }
    
    return err
}
```

### 7.3 Backpressure Handling

**Worker Backpressure**:
```go
func (w *Worker) HandleIntent(msg *nats.Msg) {
    // Check capacity before accepting
    if w.isOverloaded() {
        // Don't ACK message, let it requeue
        msg.Nak()
        
        // Emit backpressure signal
        w.emitBackpressureSignal()
        
        return
    }
    
    // Accept work
    w.inflightCount++
    
    // Process
    result, err := w.processIntent(msg.Data)
    
    w.inflightCount--
    
    if err != nil {
        msg.Nak()
        return
    }
    
    msg.Ack()
    w.publishResult(result)
}

func (w *Worker) isOverloaded() bool {
    return w.inflightCount >= w.maxCapacity ||
           w.queueDepth > w.maxQueueDepth ||
           w.memoryPressure > 0.9
}
```

**SMS Backpressure**:
```go
func (sms *FlowEngine) emitBackpressureSignal() {
    if sms.blockedTransitions > sms.backpressureThreshold {
        signal := Signal{
            Type: Backpressure,
            Source: Source{
                Component: "sms",
                Name: sms.name,
            },
            Metrics: map[string]interface{}{
                "blocked_transitions": sms.blockedTransitions,
                "wait_time_avg_ms": sms.calculateAvgWait(),
            },
            Severity: Critical,
        }
        
        sms.nats.Publish("signal.sms."+sms.name+".backpressure", signal)
    }
}
```

---

## PART VIII: OBSERVABILITY IMPLEMENTATION

### 8.1 Distributed Tracing

**Context Propagation**:
```go
func (e *FlowEngine) ExecuteWithTracing(
    flow FlowSpec,
    input Data,
) (Data, error) {
    
    // Create root span
    trace := NewTrace()
    ctx := ExecutionContext{
        TraceID: trace.ID,
        CausationID: trace.ID,
        Actor: input.Actor,
    }
    
    // Emit flow start event
    e.tracer.RecordEvent(Event{
        Type: "flow_started",
        TraceID: ctx.TraceID,
        Flow: flow.Name,
        Timestamp: time.Now(),
    })
    
    state := input.Data
    
    for i, step := range flow.Steps {
        // Create child span for step
        stepCtx := ctx.WithNewCausation()
        
        e.tracer.RecordEvent(Event{
            Type: "step_started",
            TraceID: stepCtx.TraceID,
            CausationID: stepCtx.CausationID,
            Step: step.Name,
        })
        
        result, err := e.executeStepWithContext(step, state, stepCtx)
        
        if err != nil {
            e.tracer.RecordEvent(Event{
                Type: "step_failed",
                TraceID: stepCtx.TraceID,
                Error: err.Error(),
            })
            return nil, err
        }
        
        e.tracer.RecordEvent(Event{
            Type: "step_completed",
            TraceID: stepCtx.TraceID,
            Duration: time.Since(stepCtx.StartTime),
        })
        
        state = result
    }
    
    e.tracer.RecordEvent(Event{
        Type: "flow_completed",
        TraceID: ctx.TraceID,
    })
    
    return state, nil
}
```

**NATS Header Propagation**:
```go
func (w *Worker) InvokeRemoteWithContext(
    subject string,
    input Data,
    ctx ExecutionContext,
) (Data, error) {
    
    msg := &nats.Msg{
        Subject: subject,
        Data: serialize(input),
        Header: nats.Header{
            "X-Trace-ID": []string{ctx.TraceID},
            "X-Causation-ID": []string{ctx.CausationID},
            "X-Actor-Subject": []string{ctx.Actor.Subject},
            "X-Actor-Roles": ctx.Actor.Roles,
        },
    }
    
    resp, err := w.nats.RequestMsg(msg, 5*time.Second)
    if err != nil {
        return nil, err
    }
    
    return parse(resp.Data), nil
}
```

### 8.2 Metrics Collection

**Worker Metrics**:
```go
type WorkerMetrics struct {
    // Counters
    RequestsTotal      int64
    RequestsSuccessful int64
    RequestsFailed     int64
    
    // Gauges
    InflightRequests   int64
    QueueDepth         int64
    
    // Histograms
    RequestDuration    *prometheus.HistogramVec
    
    // Labels
    WorkerName  string
    Version     string
    Region      string
}

func (w *Worker) RecordExecution(
    workUnit string,
    duration time.Duration,
    err error,
) {
    atomic.AddInt64(&w.metrics.RequestsTotal, 1)
    
    if err != nil {
        atomic.AddInt64(&w.metrics.RequestsFailed, 1)
    } else {
        atomic.AddInt64(&w.metrics.RequestsSuccessful, 1)
    }
    
    w.metrics.RequestDuration.
        WithLabelValues(workUnit, w.metrics.Version).
        Observe(duration.Seconds())
}
```

---

## PART IX: CODE GENERATION PATTERNS

### 9.1 From DataType to Go Struct

**Generator Template**:
```go
// Generated from: dataType Order v3

package generated

import (
    "github.com/google/uuid"
    "github.com/shopspring/decimal"
)

// OrderV3 represents Order at version v3
type OrderV3 struct {
    ID          uuid.UUID       `json:"id"`
    CustomerID  uuid.UUID       `json:"customer_id"`
    Subtotal    decimal.Decimal `json:"subtotal"`
    Tax         decimal.Decimal `json:"tax"`
    Discount    decimal.Decimal `json:"discount"`
    Total       decimal.Decimal `json:"total"`
    PaidAmount  decimal.Decimal `json:"paid_amount"`
    Status      OrderStatus     `json:"status"`
}

type OrderStatus string
const (
    OrderStatusDraft         OrderStatus = "draft"
    OrderStatusPartiallyPaid OrderStatus = "partially_paid"
    OrderStatusPaid          OrderStatus = "paid"
    OrderStatusFulfilled     OrderStatus = "fulfilled"
)

// Version returns the schema version
func (o OrderV3) Version() string {
    return "v3"
}

// Validate checks constraints
func (o OrderV3) Validate() error {
    // Generated from constraint: order_total
    if !o.Total.Equal(o.Subtotal.Add(o.Tax).Sub(o.Discount)) {
        return ErrOrderTotalInvalid
    }
    
    // Generated from constraint: payment_validity
    if o.PaidAmount.GreaterThan(o.Total) {
        return ErrPaidAmountExceedsTotal
    }
    
    return nil
}
```

### 9.2 From InputIntent to Handler

**Generated Interface**:
```go
// Generated from: inputIntent CreateOrder v2

package generated

// CreateOrderV2Input represents the input for CreateOrder v2
type CreateOrderV2Input struct {
    // Required fields
    CustomerID uuid.UUID       `json:"customer_id" validate:"required"`
    Items      []OrderItem     `json:"items" validate:"required,min=1"`
    Subtotal   decimal.Decimal `json:"subtotal" validate:"required,gte=0"`
    
    // Optional fields
    Discount   *decimal.Decimal `json:"discount,omitempty"`
    Notes      *string          `json:"notes,omitempty"`
}

// CreateOrderV2Handler handles CreateOrder v2 intents
type CreateOrderV2Handler interface {
    Handle(ctx ExecutionContext, input CreateOrderV2Input) (OrderV3, error)
}

// CreateOrderV2Flow is the generated flow executor
type CreateOrderV2Flow struct {
    handler CreateOrderV2Handler
    policies *PolicyRegistry
}

func (f *CreateOrderV2Flow) Execute(
    ctx ExecutionContext,
    input CreateOrderV2Input,
) (OrderV3, error) {
    
    // 1. Validate input constraints
    if err := validateCreateOrderV2Input(input); err != nil {
        return OrderV3{}, ExecutionError{
            Type: ErrorValidation,
            Retryable: false,
            Reason: err.Error(),
        }
    }
    
    // 2. Check authorization
    decision := f.policies.Evaluate(ctx, Target{
        Type: "inputIntent",
        Name: "CreateOrder",
        Version: "v2",
    })
    if decision.Denied {
        return OrderV3{}, ExecutionError{
            Type: ErrorAuthorization,
            Retryable: false,
            Reason: decision.Reason,
        }
    }
    
    // 3. Delegate to handler
    return f.handler.Handle(ctx, input)
}

func validateCreateOrderV2Input(input CreateOrderV2Input) error {
    // Generated from constraints.guarantees
    if input.Subtotal.LessThan(decimal.Zero) {
        return errors.New("subtotal must be >= 0")
    }
    if len(input.Items) == 0 {
        return errors.New("items must not be empty")
    }
    return nil
}
```

### 9.3 From Policy to Evaluator

**Generated Evaluator**:
```go
// Generated from: policy ModifyOrder v2

package generated

func EvaluateModifyOrderV2(
    ctx ExecutionContext,
    target Target,
) PolicyDecision {
    
    // Generated from: when expression
    allowed := (ctx.Actor.Role == "admin") ||
               (ctx.Actor.Role == "customer" && target.Data["customer_id"] == ctx.Actor.Subject) ||
               (ctx.Actor.Role == "support" && ctx.Actor.Attributes["department"] == "order_management")
    
    return PolicyDecision{
        Policy: "ModifyOrder",
        Version: "v2",
        Allow: allowed,
        Effect: "allow",
        EvaluatedAt: time.Now(),
    }
}
```

---

## PART X: MULTI-REGION IMPLEMENTATION

### 10.1 Regional Deployment Model

**Topology**:
```
┌─────────────────────────────────────────────┐
│ Global Control Plane (Optional)             │
│ - Policy definitions                        │
│ - Topology intent                           │
│ - Cross-region coordination                 │
└──────────────┬──────────────────────────────┘
               │ (advisory only)
        ┌──────┴──────┐
        │             │
┌───────▼──────┐  ┌───▼───────────┐
│ US-EAST      │  │ EU-WEST       │
│              │  │               │
│ Controller   │  │ Controller    │
│ Workers      │  │ Workers       │
│ NATS Cluster │  │ NATS Cluster  │
└──────────────┘  └───────────────┘
```

**Communication**:
- Control plane → regions: Spec/policy distribution
- Region ← region: None (isolated by default)
- Global optional: Advisory coordination only

### 10.2 Cross-Region Signal Aggregation

**Regional Aggregator**:
```go
type RegionalAggregator struct {
    region string
    
    // Regional metrics only
    metrics map[string]AggregatedMetrics
}

func (r *RegionalAggregator) ConsumeSignals() {
    // Subscribe to regional signals only
    subject := fmt.Sprintf("signal.worker.*.*.%s", r.region)
    
    r.nats.Subscribe(subject, func(msg *nats.Msg) {
        signal := ParseSignal(msg.Data)
        r.aggregate(signal)
    })
}
```

**Global Aggregator (Optional)**:
```go
type GlobalAggregator struct {
    // Cross-region metrics
    byRegion map[string]AggregatedMetrics
}

func (g *GlobalAggregator) ConsumeSignals() {
    // Subscribe to all regions
    g.nats.Subscribe("signal.worker.>", func(msg *nats.Msg) {
        signal := ParseSignal(msg.Data)
        g.aggregateByRegion(signal)
    })
}
```

### 10.3 Partition Handling

**Regional Isolation Pattern**:
```go
func (r *RegionalScheduler) EvaluationLoop(ctx context.Context) {
    ticker := time.NewTicker(10 * time.Second)
    
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            // Only use regional signals
            signals := r.aggregator.GetRegionalMetrics(r.region)
            
            // Never assume global state
            // Never compensate for missing remote signals
            
            decisions := r.evaluateScaling(signals)
            r.applyDecisions(decisions)
        }
    }
}
```

**Partition Detection**:
```go
func (r *RegionalScheduler) detectPartition() bool {
    // Check if global control plane reachable
    timeout := 5 * time.Second
    _, err := r.nats.Request("healthcheck.global", nil, timeout)
    
    if err != nil {
        // Partition detected
        log.Warn("Global control plane unreachable, continuing autonomously")
        r.mode = Autonomous
        return true
    }
    
    r.mode = Connected
    return false
}
```

---

## PART XI: PERFORMANCE OPTIMIZATION

### 11.1 Routing Cache Optimization

**Cache Warming**:
```go
func (r *RoutingCache) WarmCache(ctx context.Context) error {
    // Load all current registrations
    entries, err := r.kv.Keys()
    if err != nil {
        return err
    }
    
    for _, key := range entries {
        entry, _ := r.kv.Get(key)
        registration := ParseRegistration(entry.Value())
        r.addEndpoint(registration)
    }
    
    log.Info("Cache warmed", "endpoints", len(r.endpoints))
    
    // Watch for updates
    watcher, _ := r.kv.WatchAll()
    go r.processUpdates(ctx, watcher)
    
    return nil
}
```

**Update Processing (Hot Path Optimization)**:
```go
func (r *RoutingCache) processUpdates(ctx context.Context, watcher nats.KeyWatcher) {
    for {
        select {
        case <-ctx.Done():
            return
        case update := <-watcher.Updates():
            
            if update == nil {
                continue
            }
            
            // Non-blocking update
            switch update.Operation() {
            case nats.KeyValuePut:
                registration := ParseRegistration(update.Value())
                r.addEndpointAsync(registration)
                
            case nats.KeyValueDelete:
                workerID := extractWorkerID(update.Key())
                r.removeEndpointAsync(workerID)
            }
        }
    }
}

func (r *RoutingCache) addEndpointAsync(reg WorkerRegistration) {
    // Use lock-free data structures or batch updates
    r.updateQueue <- CacheUpdate{
        Type: Add,
        Registration: reg,
    }
}
```

**Batched Cache Updates**:
```go
func (r *RoutingCache) processBatchUpdates(ctx context.Context) {
    batch := make([]CacheUpdate, 0, 100)
    ticker := time.NewTicker(100 * time.Millisecond)
    
    for {
        select {
        case update := <-r.updateQueue:
            batch = append(batch, update)
            
            if len(batch) >= 100 {
                r.applyBatch(batch)
                batch = batch[:0]
            }
            
        case <-ticker.C:
            if len(batch) > 0 {
                r.applyBatch(batch)
                batch = batch[:0]
            }
        }
    }
}
```

### 11.2 In-Process Optimization

**Direct Function Call Pattern**:
```go
func (e *Executor) InvokeLocalWorkUnit(
    workUnit string,
    input Data,
    context ExecutionContext,
) (Data, error) {
    
    // No serialization needed
    // No network hop
    // No transport overhead
    
    fn := e.localRegistry.Get(workUnit)
    if fn == nil {
        return nil, ErrNotLocal
    }
    
    return fn(context, input)
}
```

**Atomic Group In-Process**:
```go
func (e *FlowEngine) executeAtomicGroupLocal(
    group AtomicGroup,
    input Data,
    context ExecutionContext,
) (Data, error) {
    
    // All work units local - pure function composition
    state := input
    
    for _, step := range group.Steps {
        fn := e.localRegistry.Get(step.WorkUnit)
        
        result, err := fn(context, state)
        if err != nil {
            return nil, err
        }
        
        state = result
    }
    
    return state, nil
}
```

**Benchmark**:
- Local invocation: ~5-20 microseconds
- NATS invocation: ~1-5 milliseconds
- HTTP invocation: ~2-10 milliseconds

**When to Optimize**: Flows with high-frequency invocation (>1000/sec)

---

## PART XII: STREAMING & REAL-TIME PATTERNS

### 12.1 Side-Channel Streaming

**Pattern**: NATS for signaling, direct connection for data.

**WebRTC Negotiation**:
```go
func (w *Worker) HandleStreamingIntent(msg *nats.Msg) {
    intent := ParseIntent(msg.Data)
    
    // 1. Create WebRTC endpoint
    endpoint, offer := w.createWebRTCEndpoint()
    
    // 2. Send offer via NATS
    response := map[string]interface{}{
        "type": "stream_offer",
        "offer": offer,
        "endpoint": endpoint,
    }
    msg.Respond(serialize(response))
    
    // 3. Client establishes direct WebRTC connection
    // 4. Stream data flows outside NATS
    // 5. Completion signal sent via NATS
}
```

**gRPC Streaming**:
```go
func (w *Worker) HandleStreamingWorkUnit(
    stream grpc.ServerStream,
    context ExecutionContext,
) error {
    
    for {
        // Receive chunks
        chunk, err := stream.Recv()
        if err == io.EOF {
            break
        }
        
        // Process
        result := w.processChunk(chunk, context)
        
        // Send result
        stream.Send(result)
    }
    
    // Send completion via NATS
    w.emitCompletionSignal(context)
    
    return nil
}
```

### 12.2 Interactive Human-in-Loop

**Pattern**:
```yaml
sms:
  flows:
    - name: RefundApprovalFlow
      steps:
        - work_unit: calculate_refund
        - work_unit: request_approval
          interaction:
            type: human
            timeout: 5m
        - work_unit: process_refund
```

**Implementation**:
```go
func (w *Worker) HandleHumanInteraction(
    workUnit string,
    input Data,
    context ExecutionContext,
) (Data, error) {
    
    // 1. Publish interaction request
    request := InteractionRequest{
        FlowID: context.FlowID,
        WorkUnit: workUnit,
        Input: input,
        Timeout: 5 * time.Minute,
    }
    
    w.nats.Publish("interaction.request", request)
    
    // 2. Wait for response (with timeout)
    responseChan := make(chan InteractionResponse)
    
    sub, _ := w.nats.Subscribe("interaction.response."+context.FlowID, func(msg *nats.Msg) {
        response := ParseInteractionResponse(msg.Data)
        responseChan <- response
    })
    defer sub.Unsubscribe()
    
    select {
    case response := <-responseChan:
        return response.Data, nil
        
    case <-time.After(5 * time.Minute):
        return nil, ErrInteractionTimeout
    }
}
```

**UI Integration**:
```go
// UI subscribes to interaction requests
func (ui *UIRuntime) ListenForInteractions() {
    ui.nats.Subscribe("interaction.request", func(msg *nats.Msg) {
        request := ParseInteractionRequest(msg.Data)
        
        // Display to user
        ui.displayApprovalRequest(request)
    })
}

// User responds
func (ui *UIRuntime) SubmitApproval(flowID string, approved bool) {
    response := InteractionResponse{
        FlowID: flowID,
        Approved: approved,
        Data: map[string]interface{}{
            "approved": approved,
            "approved_by": ui.currentUser,
            "approved_at": time.Now(),
        },
    }
    
    ui.nats.Publish("interaction.response."+flowID, response)
}
```

---

## PART XIII: TESTING IMPLEMENTATION

### 13.1 Chaos Testing Framework

**Failure Injection**:
```go
type ChaosEngine struct {
    killWorkerProbability   float64
    networkDelayRange       time.Duration
    policyCorruptionRate    float64
}

func (c *ChaosEngine) InjectFailures(ctx context.Context) {
    ticker := time.NewTicker(1 * time.Second)
    
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            c.maybeKillWorker()
            c.maybeDelayNetwork()
            c.maybeCorruptPolicy()
        }
    }
}

func (c *ChaosEngine) maybeKillWorker() {
    if rand.Float64() < c.killWorkerProbability {
        workers := c.listWorkers()
        if len(workers) > 0 {
            victim := workers[rand.Intn(len(workers))]
            
            log.Info("[CHAOS] Killing worker", "worker", victim)
            c.killWorker(victim)
        }
    }
}
```

**Validation**:
```go
func TestChaosResilience(t *testing.T) {
    // Setup system
    system := SetupTestSystem()
    chaos := NewChaosEngine()
    
    // Start chaos
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Minute)
    defer cancel()
    
    go chaos.InjectFailures(ctx)
    
    // Run workload
    results := system.ExecuteWorkload(1000)
    
    // Assertions
    assert.Zero(t, results.DataLoss, "No data loss allowed")
    assert.Zero(t, results.Duplicates, "No duplicate processing")
    assert.GreaterOrEqual(t, results.SuccessRate, 0.99, "99% success rate required")
}
```

### 13.2 Policy Testing

**Shadow Divergence Testing**:
```go
func TestPolicyShadowDivergence(t *testing.T) {
    registry := NewPolicyRegistry()
    
    // Load both versions
    registry.LoadEnforced(modifyOrderV1)
    registry.LoadShadow(modifyOrderV2)
    
    // Run test cases
    testCases := loadTestCases()
    
    divergences := 0
    for _, tc := range testCases {
        decision := registry.Evaluate(tc.Context, tc.Target)
        
        if decision.Diverged {
            divergences++
            log.Info("Divergence detected",
                "case", tc.Name,
                "enforced", decision.Enforced,
                "shadow", decision.Shadow)
        }
    }
    
    divergenceRate := float64(divergences) / float64(len(testCases))
    
    assert.Less(t, divergenceRate, 0.01, "Divergence must be <1%")
}
```

---

## PART XIV: OPERATIONAL PROCEDURES

### 14.1 Deployment Procedure

**Rolling Update**:
```
1. Deploy new worker version (green)
   - Workers auto-register with version: v4
   - Begin heartbeat loop
   - No traffic yet (not preferred)

2. Deploy shadow policy (if applicable)
   - Evaluate but don't enforce
   - Monitor divergence

3. Update routing preference
   - Prefer v4 for compatible intents
   - v3 workers still handle existing flows

4. Monitor signals
   - Error rate
   - Latency
   - Divergence
   - Queue depth

5. If healthy after 10 minutes:
   - Promote routing to v4 fully
   - Begin draining v3 workers

6. Drain v3 workers
   - Send SIGTERM
   - Wait for graceful shutdown
   - Max drain time: 5 minutes

7. Decommission v3
   - Scale to 0
   - Remove from topology
```

### 14.2 Rollback Procedure

**Instant Rollback**:
```
1. Detect issue (high error rate, policy violations)

2. Revert routing preference
   - Prefer v3 over v4
   - Takes effect within cache refresh (30s)

3. Scale up v3 if needed
   - Scheduler responds to load shift
   - Auto-scaling triggers

4. Investigate v4 issue
   - Check signals
   - Review policy divergence
   - Analyze traces

5. Fix and redeploy
   - Or deprecate v4 entirely
```

### 14.3 Policy Revocation (Emergency)

**Immediate Revocation**:
```
1. Operator identifies policy issue

2. Publish revocation event
   - lifecycle.policy.ModifyOrder.v2.revoke

3. Workers receive event
   - Update local registry immediately
   - Remove from enforced set
   - Begin denying requests

4. Effect within 1 second globally

5. Fallback behavior
   - If policy set becomes empty, fail closed
   - If superseded policy exists, revert to it
```

---

## PART XV: CRITICAL IMPLEMENTATION NOTES

### 15.1 Idempotency Implementation

**Key Generation**:
```go
func GenerateIdempotencyKey(intent InputIntent, data Data) string {
    parts := []string{}
    
    for _, keyPart := range intent.Idempotency.Key {
        value := evaluateKeyExpression(keyPart, data)
        parts = append(parts, value)
    }
    
    // Deterministic hash
    hash := sha256.Sum256([]byte(strings.Join(parts, "|")))
    return hex.EncodeToString(hash[:])
}
```

**Deduplication Check**:
```go
func (w *Worker) ExecuteWithIdempotency(
    workUnit string,
    input Data,
    context ExecutionContext,
) (Data, error) {
    
    key := GenerateIdempotencyKey(input.Intent, input.Data)
    
    // Check if already processed
    if result, ok := w.dedupeCache.Get(key); ok {
        log.Info("Idempotent replay, returning cached result",
            "key", key)
        return result, nil
    }
    
    // Execute
    result, err := w.executeWorkUnit(workUnit, input, context)
    if err != nil {
        return nil, err
    }
    
    // Cache result
    w.dedupeCache.Put(key, result, 24*time.Hour)
    
    return result, nil
}
```

### 15.2 Boundary Enforcement

**Runtime Validation**:
```go
func (e *FlowEngine) validateBoundary(
    step Step,
    currentBoundary string,
) error {
    
    if step.Boundary == "" {
        return nil  // No boundary constraint
    }
    
    if step.Boundary != currentBoundary {
        return ExecutionError{
            Type: ErrorInvariantViolation,
            Retryable: false,
            Reason: fmt.Sprintf(
                "boundary violation: step %s requires %s but executing in %s",
                step.Name,
                step.Boundary,
                currentBoundary,
            ),
        }
    }
    
    return nil
}
```

### 15.3 Time Constraint Enforcement

**Deadline Enforcement**:
```go
func (e *FlowEngine) executeStepWithDeadline(
    step Step,
    input Data,
    context ExecutionContext,
) (Data, error) {
    
    deadline := context.Deadline
    if deadline.IsZero() {
        deadline = time.Now().Add(30 * time.Second)  // Default
    }
    
    // Create context with deadline
    ctx, cancel := context.WithDeadline(context.Background(), deadline)
    defer cancel()
    
    // Execute with deadline
    resultChan := make(chan executeResult)
    go func() {
        result, err := e.executeStep(step, input, context)
        resultChan <- executeResult{result, err}
    }()
    
    select {
    case result := <-resultChan:
        return result.Data, result.Error
        
    case <-ctx.Done():
        return nil, ExecutionError{
            Type: ErrorTimeout,
            Retryable: true,
            Reason: "deadline exceeded",
        }
    }
}
```

---

## PART XVI: AUTHORITY IMPLEMENTATION (NEW in v1.3)

### 16.1 JetStream KV for Authority State

Authority state stored in dedicated KV bucket with strong consistency.

```go
const AuthorityBucket = "SMS_AUTHORITY"

type AuthorityStore struct {
    kv nats.KeyValue
}

func (s *AuthorityStore) GetAuthority(entityID string) (*AuthorityState, error) {
    entry, err := s.kv.Get(entityID)
    if err != nil {
        return nil, ErrAuthorityNotFound
    }
    
    var state AuthorityState
    if err := json.Unmarshal(entry.Value(), &state); err != nil {
        return nil, err
    }
    
    return &state, nil
}

func (s *AuthorityStore) SetAuthority(state *AuthorityState) error {
    data, _ := json.Marshal(state)
    _, err := s.kv.Put(state.EntityID, data)
    return err
}

func (s *AuthorityStore) TransitionAuthority(entityID string, newRegion string) error {
    // Atomic update using revision
    entry, err := s.kv.Get(entityID)
    if err != nil {
        return err
    }
    
    var state AuthorityState
    json.Unmarshal(entry.Value(), &state)
    
    // Increment epoch, set transitioning
    state.AuthorityEpoch++
    state.Status = "TRANSITIONING"
    state.TargetRegion = newRegion
    
    data, _ := json.Marshal(state)
    _, err = s.kv.Update(entityID, data, entry.Revision())
    return err
}
```

### 16.2 Epoch Management

```go
type EpochManager struct {
    authorityStore *AuthorityStore
}

func (m *EpochManager) ValidateEpoch(entityID string, providedEpoch int64) error {
    current, err := m.authorityStore.GetAuthority(entityID)
    if err != nil {
        return err
    }
    
    if providedEpoch < current.AuthorityEpoch {
        return ErrStaleEpoch
    }
    
    if providedEpoch > current.AuthorityEpoch {
        return ErrFutureEpoch
    }
    
    return nil
}

func (m *EpochManager) IncrementEpoch(entityID string) (int64, error) {
    current, err := m.authorityStore.GetAuthority(entityID)
    if err != nil {
        return 0, err
    }
    
    newEpoch := current.AuthorityEpoch + 1
    current.AuthorityEpoch = newEpoch
    
    if err := m.authorityStore.SetAuthority(current); err != nil {
        return 0, err
    }
    
    return newEpoch, nil
}
```

### 16.3 Lease Validation

```go
type LeaseValidator struct {
    leaseKV nats.KeyValue
}

func (v *LeaseValidator) Valid(leaseID string) bool {
    entry, err := v.leaseKV.Get(leaseID)
    if err != nil {
        return false
    }
    
    var lease Lease
    json.Unmarshal(entry.Value(), &lease)
    
    return time.Now().Before(lease.ExpiresAt)
}

func (v *LeaseValidator) GrantLease(entityID string, region string, duration time.Duration) (string, error) {
    lease := Lease{
        ID:        uuid.New().String(),
        EntityID:  entityID,
        Region:    region,
        GrantedAt: time.Now(),
        ExpiresAt: time.Now().Add(duration),
    }
    
    data, _ := json.Marshal(lease)
    _, err := v.leaseKV.Put(lease.ID, data)
    if err != nil {
        return "", err
    }
    
    return lease.ID, nil
}
```

### 16.4 CAS Token Encoding

```go
type CASToken struct {
    EntityID        string `json:"entity_id"`
    EntityVersion   int64  `json:"entity_version"`
    AuthorityEpoch  int64  `json:"authority_epoch"`
    AuthorityRegion string `json:"authority_region"`
    LeaseID         string `json:"lease_id"`
    Signature       []byte `json:"signature"`
}

func (t *CASToken) Encode(secret []byte) string {
    payload, _ := json.Marshal(t)
    
    mac := hmac.New(sha256.New, secret)
    mac.Write(payload)
    t.Signature = mac.Sum(nil)
    
    finalPayload, _ := json.Marshal(t)
    return base64.URLEncoding.EncodeToString(finalPayload)
}

func DecodeCASToken(encoded string, secret []byte) (*CASToken, error) {
    payload, err := base64.URLEncoding.DecodeString(encoded)
    if err != nil {
        return nil, ErrInvalidToken
    }
    
    var token CASToken
    if err := json.Unmarshal(payload, &token); err != nil {
        return nil, ErrInvalidToken
    }
    
    // Verify signature
    tokenWithoutSig := token
    tokenWithoutSig.Signature = nil
    verifyPayload, _ := json.Marshal(tokenWithoutSig)
    
    mac := hmac.New(sha256.New, secret)
    mac.Write(verifyPayload)
    expectedMAC := mac.Sum(nil)
    
    if !hmac.Equal(token.Signature, expectedMAC) {
        return nil, ErrInvalidSignature
    }
    
    return &token, nil
}
```

---

## PART XVII: STORAGE ROLE IMPLEMENTATION (NEW in v1.3)

### 17.1 Control Storage Patterns

Control storage requires strong consistency and should be small.

```go
// Control storage buckets
const (
    BucketAuthority    = "SMS_AUTHORITY"    // Entity authority state
    BucketLeases       = "SMS_LEASES"       // Authority leases
    BucketWorkers      = "SMS_WORKERS"      // Worker registrations
    BucketPolicies     = "SMS_POLICIES"     // Active policy versions
    BucketCoordination = "SMS_COORDINATION" // Distributed locks
)

type ControlStorageConfig struct {
    Replicas     int           // Typically 3 for quorum
    MaxValueSize int64         // Small: 1MB max
    TTL          time.Duration // Varies by use case
}

var DefaultControlConfig = ControlStorageConfig{
    Replicas:     3,
    MaxValueSize: 1 * 1024 * 1024, // 1MB
    TTL:          0,               // No default TTL
}
```

### 17.2 Data Storage Patterns

Data storage is append-only and rebuildable.

```go
// Data storage streams
const (
    StreamEvents       = "SMS_EVENTS"       // Domain events
    StreamMaterialized = "SMS_MATERIALIZED" // View snapshots
    StreamAudit        = "SMS_AUDIT"        // Audit trail
)

type DataStorageConfig struct {
    Retention    time.Duration
    MaxMsgSize   int64
    Replicas     int
    DenyDelete   bool // Append-only
    DenyPurge    bool
}

var DefaultDataConfig = DataStorageConfig{
    Retention:  365 * 24 * time.Hour, // 1 year
    MaxMsgSize: 8 * 1024 * 1024,      // 8MB
    Replicas:   3,
    DenyDelete: true,
    DenyPurge:  true,
}
```

### 17.3 Rebuild Mechanics

```go
type ViewRebuilder struct {
    sourceStream nats.JetStream
    targetKV     nats.KeyValue
}

func (r *ViewRebuilder) Rebuild(ctx context.Context, viewName string) error {
    // Subscribe from beginning
    sub, err := r.sourceStream.Subscribe(
        fmt.Sprintf("events.*.%s.*", viewName),
        r.handleEvent,
        nats.DeliverAll(),
    )
    if err != nil {
        return err
    }
    defer sub.Unsubscribe()
    
    // Wait for completion or timeout
    select {
    case <-ctx.Done():
        return ctx.Err()
    case err := <-r.completionChan:
        return err
    }
}
```

---

## PART XVIII: VIEW WORKER AUTHORITY PATTERNS (NEW in v1.3)

### 18.1 Authority-Agnostic Consumption

```go
type AuthorityAgnosticConsumer struct {
    stream   nats.JetStream
    regions  []string
}

func (c *AuthorityAgnosticConsumer) Subscribe(handler EventHandler) error {
    // Subscribe to events from all regions
    for _, region := range c.regions {
        subject := fmt.Sprintf("events.%s.*", region)
        
        _, err := c.stream.Subscribe(subject, func(msg *nats.Msg) {
            event := ParseEvent(msg)
            
            // Process regardless of originating region
            handler.Handle(event)
            
            msg.Ack()
        })
        
        if err != nil {
            return err
        }
    }
    
    return nil
}
```

### 18.2 Epoch Ordering

```go
type EpochOrderedProcessor struct {
    lastEpoch map[string]int64 // per entity
    buffer    map[string][]Event
}

func (p *EpochOrderedProcessor) Process(event Event) error {
    entityID := event.EntityID
    
    lastSeen := p.lastEpoch[entityID]
    
    if event.Epoch < lastSeen {
        // Old epoch, safe to process (historical)
        return p.processEvent(event)
    }
    
    if event.Epoch == lastSeen {
        // Same epoch, normal processing
        return p.processEvent(event)
    }
    
    // New epoch detected - ensure we've seen epoch boundary
    if !p.epochBoundaryReceived(entityID, event.Epoch) {
        p.buffer[entityID] = append(p.buffer[entityID], event)
        return nil
    }
    
    // Process buffered then current
    for _, buffered := range p.buffer[entityID] {
        p.processEvent(buffered)
    }
    p.buffer[entityID] = nil
    p.lastEpoch[entityID] = event.Epoch
    
    return p.processEvent(event)
}
```

### 18.3 Multi-Region Event Merging

```go
type RegionalMerger struct {
    regions     map[string]*RegionalStream
    outputChan  chan Event
}

func (m *RegionalMerger) MergeStreams() <-chan Event {
    for _, rs := range m.regions {
        go func(stream *RegionalStream) {
            for event := range stream.Events() {
                // Add region metadata
                event.SourceRegion = stream.Region
                m.outputChan <- event
            }
        }(rs)
    }
    
    return m.outputChan
}

// View worker consumes merged stream
func (v *ViewWorker) Run() {
    merger := NewRegionalMerger(v.regions)
    events := merger.MergeStreams()
    
    for event := range events {
        // Process event regardless of source region
        v.updateView(event)
    }
}
```

---

## PART XX: UI RUNTIME IMPLEMENTATION (NEW in v1.4)

### 20.1 Composition State Management

The UI runtime maintains composition state for active user sessions:

```go
type CompositionState struct {
    CompositionName    string
    CompositionVersion string
    PrimaryViewState   *ViewState
    RelatedViewStates  map[string]*ViewState
    NavigationContext  *NavigationContext
    LastResolved       time.Time
    Subject            *Subject
}

type ViewState struct {
    ViewName    string
    ViewVersion string
    DataScope   map[string]any
    Data        any
    FieldMasks  map[string]MaskResult
    LastFetched time.Time
}

type NavigationContext struct {
    ExperienceName string
    BreadcrumbPath []*BreadcrumbEntry
    ActiveParams   map[string]string
}
```

### 20.2 Navigation Hierarchy Caching

Navigation trees are cached to avoid repeated computation:

```go
type NavigationCache struct {
    mu    sync.RWMutex
    trees map[string]*CachedNavigationTree
}

type CachedNavigationTree struct {
    ExperienceName    string
    ExperienceVersion string
    RootNodes         []*NavigationNode
    NodeIndex         map[string]*NavigationNode // composition_name -> node
    ComputedAt        time.Time
    TTL               time.Duration
}

func (c *NavigationCache) GetTree(experienceName, version string) *CachedNavigationTree {
    c.mu.RLock()
    key := experienceName + ":" + version
    tree := c.trees[key]
    c.mu.RUnlock()
    
    if tree != nil && time.Since(tree.ComputedAt) < tree.TTL {
        return tree
    }
    
    // Rebuild if expired or missing
    return c.rebuildTree(experienceName, version)
}

func (c *NavigationCache) Invalidate(experienceName string) {
    c.mu.Lock()
    // Remove all versions of this experience
    for key := range c.trees {
        if strings.HasPrefix(key, experienceName+":") {
            delete(c.trees, key)
        }
    }
    c.mu.Unlock()
}
```

### 20.3 Field Masking Implementation

Efficient field masking with precompiled policies:

```go
type FieldMasker struct {
    policyCache *PolicyCache
}

type MaskResult struct {
    Visible bool
    Masked  bool
    Value   any
}

func (m *FieldMasker) ApplyMasks(
    view *PresentationView,
    data map[string]any,
    subject *Subject,
) map[string]MaskResult {
    results := make(map[string]MaskResult)
    
    for fieldName, value := range data {
        result := MaskResult{Visible: true, Masked: false, Value: value}
        
        perm := view.GetFieldPermission(fieldName)
        if perm == nil {
            results[fieldName] = result
            continue
        }
        
        // Check visibility
        visible := m.evaluateVisibility(perm.Visible, subject)
        if !visible {
            continue // Omit field entirely
        }
        result.Visible = true
        
        // Check masking
        if perm.Mask != nil {
            shouldMask := !m.evaluateExpression(perm.Mask.Unless, subject)
            if shouldMask {
                result.Masked = true
                result.Value = m.applyMask(value, perm.Mask)
            }
        }
        
        results[fieldName] = result
    }
    
    return results
}

func (m *FieldMasker) applyMask(value any, mask *MaskSpec) any {
    str := fmt.Sprintf("%v", value)
    
    switch mask.Type {
    case "full":
        return strings.Repeat("*", len(str))
    case "partial":
        return m.partialMask(str, mask.Reveal)
    case "hash":
        h := sha256.Sum256([]byte(str))
        return hex.EncodeToString(h[:8]) // First 8 bytes
    default:
        return "***"
    }
}

func (m *FieldMasker) partialMask(str string, reveal string) string {
    switch reveal {
    case "last4":
        if len(str) <= 4 {
            return str
        }
        return strings.Repeat("*", len(str)-4) + str[len(str)-4:]
    case "first3":
        if len(str) <= 3 {
            return str
        }
        return str[:3] + strings.Repeat("*", len(str)-3)
    default:
        return strings.Repeat("*", len(str))
    }
}
```

### 20.4 Experience Context Propagation

Experience context is propagated through the request chain:

```go
type ExperienceContext struct {
    ExperienceName     string
    ExperienceVersion  string
    DefaultPolicySet   string
    OnUnauthorized     string // "conceal", "indicate", "redirect", "deny"
    AuthenticatedAs    *Subject
    ActiveComposition  string
    NavigationParams   map[string]string
}

// Middleware to inject experience context
func ExperienceMiddleware(router *ExperienceRouter) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // Extract experience from path or header
            experienceName := extractExperienceName(r)
            
            // Get subject from auth context
            subject := getSubject(r.Context())
            
            // Build experience context
            ctx := &ExperienceContext{
                ExperienceName:    experienceName,
                AuthenticatedAs:   subject,
                NavigationParams:  extractParams(r),
            }
            
            // Resolve experience
            experience, err := router.GetExperience(experienceName)
            if err != nil {
                http.Error(w, "Experience not found", 404)
                return
            }
            
            ctx.ExperienceVersion = experience.Version
            ctx.DefaultPolicySet = experience.PolicySet
            ctx.OnUnauthorized = experience.OnUnauthorized
            
            // Add to request context
            r = r.WithContext(context.WithValue(r.Context(), experienceContextKey, ctx))
            
            next.ServeHTTP(w, r)
        })
    }
}
```

### 20.5 View Relationship Resolution

Resolving related views based on relationship type:

```go
type RelationshipResolver struct {
    dataLayer        DataLayer
    relationshipDefs map[string]*RelationshipDefinition
}

func (r *RelationshipResolver) ResolveRelated(
    primary *ViewState,
    related *RelatedViewSpec,
    context *CompositionContext,
) (*ViewState, error) {
    // Apply data scope
    scopedContext := r.applyDataScope(context, related.DataScope)
    
    // Resolve basis relationship
    if related.Basis != "" {
        relDef := r.relationshipDefs[related.Basis]
        if relDef != nil {
            scopedContext = r.applyRelationshipBasis(scopedContext, primary, relDef)
        }
    }
    
    // Fetch data based on cardinality
    var data any
    var err error
    
    if related.Cardinality == "many" {
        data, err = r.dataLayer.FetchMany(related.View, scopedContext)
    } else {
        data, err = r.dataLayer.FetchOne(related.View, scopedContext)
    }
    
    if err != nil {
        if related.Required {
            return nil, fmt.Errorf("required view %s failed: %w", related.View, err)
        }
        return nil, nil // Optional view failed silently
    }
    
    return &ViewState{
        ViewName:    related.View,
        DataScope:   scopedContext.Params,
        Data:        data,
        LastFetched: time.Now(),
    }, nil
}

func (r *RelationshipResolver) applyRelationshipBasis(
    ctx *CompositionContext,
    primary *ViewState,
    rel *RelationshipDefinition,
) *CompositionContext {
    // Extract FK from primary based on relationship
    newCtx := ctx.Clone()
    
    // e.g., for "transaction_debits_account", extract account_id from transaction
    switch rel.Cardinality {
    case "many-to-one":
        // Primary has FK to related
        fkField := rel.ForeignKeyField
        if fkValue, ok := primary.Data.(map[string]any)[fkField]; ok {
            newCtx.Params[rel.TargetField] = fmt.Sprintf("%v", fkValue)
        }
    case "one-to-many":
        // Related has FK to primary
        pkField := rel.PrimaryKeyField
        if pkValue, ok := primary.Data.(map[string]any)[pkField]; ok {
            newCtx.Params[rel.ForeignKeyField] = fmt.Sprintf("%v", pkValue)
        }
    }
    
    return newCtx
}
```

### 20.6 Indicator Real-Time Updates

For real-time indicator updates, use WebSocket or SSE:

```go
type IndicatorService struct {
    evaluator   *ExpressionEvaluator
    subscribers map[string][]chan IndicatorUpdate
    mu          sync.RWMutex
}

type IndicatorUpdate struct {
    CompositionName string
    Active          bool
    Severity        string
    Timestamp       time.Time
}

func (s *IndicatorService) Subscribe(sessionID string, compositions []string) <-chan IndicatorUpdate {
    ch := make(chan IndicatorUpdate, 10)
    
    s.mu.Lock()
    for _, comp := range compositions {
        s.subscribers[comp] = append(s.subscribers[comp], ch)
    }
    s.mu.Unlock()
    
    return ch
}

func (s *IndicatorService) EvaluateAndNotify(compositionName string, dataContext map[string]any) {
    composition := s.getComposition(compositionName)
    if composition.Navigation.Indicator == nil {
        return
    }
    
    indicator := composition.Navigation.Indicator
    active := s.evaluator.Evaluate(indicator.When, dataContext)
    
    update := IndicatorUpdate{
        CompositionName: compositionName,
        Active:          active,
        Severity:        indicator.Severity,
        Timestamp:       time.Now(),
    }
    
    s.mu.RLock()
    subscribers := s.subscribers[compositionName]
    s.mu.RUnlock()
    
    for _, ch := range subscribers {
        select {
        case ch <- update:
        default:
            // Skip if buffer full
        }
    }
}
```

### 20.7 Performance Optimization

Key performance considerations for UI runtime:

| Operation | Target Latency | Strategy |
|-----------|---------------|----------|
| Composition resolution | < 5ms | Cached policy + parallel view fetch |
| Navigation tree build | < 10ms | Cached with TTL |
| Field permission eval | < 1ms | Precompiled expressions |
| Indicator evaluation | < 2ms | Async with debounce |
| Experience routing | < 1ms | Cached routing table |

**Optimization Patterns**:

1. **Parallel View Fetching**: Fetch all related views concurrently
2. **Policy Precompilation**: Compile expressions at load time
3. **Hierarchical Caching**: Experience → Composition → View → Field
4. **Lazy Related Views**: Fetch non-essential views on demand
5. **Indicator Batching**: Evaluate multiple indicators together

---

## SUMMARY OF KEY IMPLEMENTATION PATTERNS

### Pattern Catalog

1. **Locality-First Routing**: Check local, fallback to remote
2. **Shadow Everything**: Test changes with real traffic before committing
3. **TTL-Based Lifecycle**: No manual cleanup, automatic expiry
4. **Dual Evaluation**: Enforced + shadow policies evaluated simultaneously
5. **Batched Updates**: Amortize cache update cost
6. **Fail-Closed**: Deny by default on ambiguity
7. **Correlation Everywhere**: Trace ID propagated end-to-end
8. **Graceful Draining**: Finish in-flight before shutdown
9. **Circuit Breakers**: Fail fast when downstream unhealthy
10. **Version Negotiation**: Runtime resolution of compatible versions

### Critical Performance Numbers

- **Cache lookup**: < 10 microseconds (in-memory)
- **Local invocation**: 5-20 microseconds
- **NATS invocation**: 1-5 milliseconds
- **Policy evaluation**: < 100 microseconds (compiled)
- **Heartbeat frequency**: 5 seconds
- **TTL expiry**: 15 seconds
- **Cache refresh**: 30 seconds
- **Scaling evaluation**: 10 seconds
- **Policy propagation**: < 1 second globally

### Failure Recovery Times

- **Worker death**: Detected within 15s (TTL), traffic shifted within 30s (cache refresh)
- **Region outage**: Auto-failover within 30s
- **Control plane outage**: No impact (last-known-good continues)
- **Policy corruption**: Rejected immediately, fallback to previous version
- **Network partition**: Regions operate autonomously, recover on reconnect

---

This document captures the deep technical implementation details required to build a production-grade system conformant with the System Mechanics Specification v1.4.

### v1.4 Addition Highlights

- **Composition State Management**: Session-scoped state for active compositions
- **Navigation Hierarchy Caching**: Cached tree with automatic invalidation
- **Field Masking**: Precompiled policies with partial/full/hash masks
- **Experience Context Propagation**: Request-scoped context with policy defaults
- **View Relationship Resolution**: Basis-aware data fetching
- **Indicator Real-Time Updates**: WebSocket/SSE for live navigation state

