# SMS v1.5 Implementation Guide

## For LLM Implementers

This guide provides comprehensive implementation patterns for building conformant SMS v1.5 runtimes. It merges high-level architectural guidance with deep technical implementation details, organized hierarchically from conceptual foundations to specific patterns.

**How to use this guide:**
- Start with Part I (Runtime Architecture) for system-wide design patterns
- Reference Part II-VII for specific subsystem implementations (Storage, Multi-Region, Worker, Policy, UI/Experience, V1.5 Features)
- Use Part VIII-X for code generation, operational patterns, and anti-patterns
- Consult Part XI-XVII for detailed engine mechanics, versioning, and performance
- Reference Part XVIII-XX for v1.3 authority and storage deep patterns
- Use Part XXI-XXV for v1.5 feature deep implementations (Sessions, Assets, Search, Integration, Governance)

**Related Documents:**
- **SMS-v1.5-Specification.md**: Normative specification with grammar definitions
- **SMS-v1.5-EBNF-Grammar.md**: Formal grammar for parser/tooling development
- **SMS-v1.5-Reference-Examples.md**: Modeling patterns and complete worked examples
- **SMS-v1.5-Quick-Reference.md**: Rapid-lookup cheat sheets and decision trees

---

## PART I: RUNTIME ARCHITECTURE

### 1.1 Core Architectural Principles

The system maintains strict separation between:

**1. Intent vs Execution**
- Grammar defines intent
- Runtime determines execution strategy

**2. Control Plane vs Data Plane**
- Control plane: Policy, topology, metadata distribution
- Data plane: Work execution, data flow

**3. Scheduling vs Execution**
- Schedulers decide worker existence
- Runtime decides work routing
- Infrastructure provisions resources

**4. Authorization vs Execution**
- Policies distributed centrally
- Enforcement evaluated locally
- No synchronous auth calls in hot path

### 1.2 Authority Model

**Single Authority Per Scope**
- One authoritative scheduler per topology scope
- One authoritative policy enforcer per component
- No distributed consensus required
- Leader election for high availability only

**Entity-Scoped Authority (v1.3)**
- Write authority resolved at entity level, not model level
- Each entity has exactly one authoritative region at any time
- Authority may change over entity lifetime via explicit transition
- Authority epoch ensures CAS correctness across migrations

**Local Enforcement**
- All decisions made with local state
- Control plane absence does not halt execution
- Fail-closed on ambiguity

### 1.3 Runtime Binary Structure

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

**Key Properties:**
- Libraries, not services
- Composable based on role
- Stateless except for caches
- Long-lived process

### 1.4 Component Responsibilities

#### Flow Engine (SMS Runtime)
**Responsibilities:**
- Execute SMS flows as FSMs
- Gate flows with policy sets
- Coordinate work unit invocation
- Track correlation context
- Emit signals

**Does NOT:**
- Store business state
- Make scheduling decisions
- Directly provision workers
- Enforce data policies (delegates to workers)

#### Worker Runtime
**Responsibilities:**
- Execute work units
- Enforce data policies locally
- Emit capacity/health signals
- Advertise capabilities
- Handle graceful shutdown

**Does NOT:**
- Coordinate with other workers
- Know about other workers
- Make topology decisions
- Store durable state (delegates to storage)

#### UI Runtime
**Responsibilities:**
- Render presentation views
- Bind to materialized data versions
- Gate user actions with policies
- Submit input intents

**Does NOT:**
- Execute business logic
- Store authoritative data
- Make authorization decisions (evaluates policies)

---

## PART II: STORAGE & PERSISTENCE

### 2.1 NATS Integration Patterns

#### NATS Role Clarification

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

#### Subject Design Patterns

**Intent Subjects (Execution)**
```
intent.<domain>.<action>

Examples:
intent.order.create
intent.order.cancel
intent.payment.apply
```

**Properties:**
- Workers subscribe based on capability
- Queue groups for load balancing
- Version negotiation via headers

**Control Plane Subjects (Configuration)**
```
spec.<domain>.<type>.<name>.<version>
policy.<domain>.<component>.<policyName>.<version>

Examples:
spec.order.dataType.Order.v3
policy.order.worker.OrderService.modify_order.v2
```

**Properties:**
- JetStream backed
- Replay enabled
- Immutable per subject
- Latest retention

**Signal Subjects (Observability)**
```
signal.<source>.<name>.<type>

Examples:
signal.worker.OrderService.capacity
signal.sms.OrderFlow.backpressure
signal.policy.ModifyOrder.shadow
```

**Properties:**
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

**TTL Behavior:**
- Refreshed by heartbeats
- Auto-expires on worker death
- No manual cleanup needed

### 2.2 JetStream Configuration

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

**Rationale:** Keep only latest version per subject

#### Lifecycle Events Stream
```yaml
stream:
  name: LIFECYCLE_EVENTS
  subjects: ["lifecycle.>"]
  storage: file
  retention: interest
  max_age: 7d
  replicas: 3
```

**Rationale:** Lifecycle transitions need ordered delivery

### 2.3 Authority KV Implementation (v1.3)

**Authority State Storage:**
```go
type AuthorityState struct {
    EntityID        string
    AuthorityRegion string
    AuthorityEpoch  int64
    AuthorityLease  string
    Status          string // ACTIVE | TRANSITIONING
    EntityVersion   int64
    UpdatedAt       time.Time
}
```

**Authority Validation:**
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

**CAS Token Validation:**
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

### 2.4 View Materialization Workers

**Materialization Pattern:**
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

**Worker Implementation:**
```go
type ViewMaterializer struct {
    sourceStream  nats.JetStreamContext
    targetKV      nats.KeyValue
    viewSpec      PresentationView
}

func (vm *ViewMaterializer) Materialize(event StateChangeEvent) error {
    // 1. Apply view logic (filter, transform, aggregate)
    viewData := vm.applyViewLogic(event.Data)
    
    // 2. Store in KV
    key := vm.constructKey(viewData)
    vm.targetKV.Put(key, viewData)
    
    // 3. Emit freshness signal
    vm.emitFreshnessSignal(key, time.Now())
    
    return nil
}
```

---

## PART III: MULTI-REGION IMPLEMENTATION

### 3.1 Regional Isolation Pattern

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

### 3.2 Authority Management

**Authority Control Worker:**
The Authority Control Worker manages entity authority state.

**Responsibilities:**
- Maintain authority state in `SMS.AUTHORITY` KV bucket
- Process authority transition requests
- Validate governance constraints
- Grant and revoke authority leases

**NATS Integration:**
```
# Authority KV bucket
SMS.AUTHORITY.{entity_type}.{entity_id}

# Authority transition requests
SMS.AUTHORITY.TRANSITION.{entity_type}

# Authority lease renewals
SMS.AUTHORITY.LEASE.{entity_id}
```

### 3.3 Regional Failover

**Automatic Failover Pattern:**
```
1. Workers in us-east stop emitting heartbeats (outage)
2. KV registrations expire (TTL)
3. Routing caches update (remove dead endpoints)
4. New requests route to us-west workers automatically
5. Scheduler in us-west sees increased load
6. Scheduler scales up us-west workers
```

**No coordination required** - emergent behavior from local decisions

### 3.4 Cross-Region Views

**View-Only Replication:**
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

**Failover Behavior:**

| Entity Authority | Impact |
|------------------|--------|
| Owned by failed region | Writes unavailable, reads degraded |
| Owned by healthy region | Fully operational |

**Degraded Mode:**
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

### 3.5 Global Context Pattern

For truly global resources (e.g., global counters, system-wide state):

**Implementation:**
```go
type GlobalContextManager struct {
    localCache     map[string]interface{}
    globalStream   nats.JetStreamContext
    syncInterval   time.Duration
}

func (gcm *GlobalContextManager) Get(key string) (interface{}, error) {
    // Try local cache first
    if val, ok := gcm.localCache[key]; ok {
        return val, nil
    }
    
    // Fetch from global stream
    msg, err := gcm.globalStream.GetLastMsg("global."+key)
    if err != nil {
        return nil, err
    }
    
    // Cache locally
    gcm.localCache[key] = msg.Data
    return msg.Data, nil
}
```

---

## PART IV: WORKER IMPLEMENTATION

### 4.1 Worker Registration

**Registration Flow:**
```
1. Worker starts
2. Loads local spec + policy registries from JetStream
3. Publishes endpoint to NATS KV
4. Subscribes to intent subjects
5. Starts heartbeat loop
6. Begins accepting work
```

**Implementation:**
```go
func (w *Worker) Register(ctx context.Context) error {
    // 1. Load specs and policies
    if err := w.loadSpecifications(); err != nil {
        return err
    }
    if err := w.loadPolicies(); err != nil {
        return err
    }
    
    // 2. Publish registration
    registration := WorkerRegistration{
        Worker:       w.name,
        Version:      w.version,
        InstanceID:   w.instanceID,
        Endpoints:    w.endpoints,
        Capabilities: w.capabilities,
        Health:       "healthy",
    }
    
    key := fmt.Sprintf("worker.%s.%s", w.name, w.instanceID)
    if err := w.kv.Put(key, registration); err != nil {
        return err
    }
    
    // 3. Subscribe to intents
    if err := w.subscribeToIntents(); err != nil {
        return err
    }
    
    // 4. Start heartbeat
    go w.heartbeatLoop(ctx)
    
    return nil
}
```

### 4.2 Routing & Discovery

**Locality-First Routing Cache:**
```go
type RoutingCache struct {
    // Indexed by work unit name + version
    endpoints map[string][]WorkerEndpoint
    
    // Indexed by execution domain
    localWorkers map[string][]WorkerEndpoint
    
    // Policy-filtered cache
    authorizedEndpoints map[string]map[string][]WorkerEndpoint
    
    mu sync.RWMutex
    lastUpdate time.Time
}
```

**Cache Lookup Algorithm:**
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

**Transport Selection:**
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

### 4.3 Scaling & Placement

**Signal-Driven Scaling:**
Workers emit capacity signals that schedulers consume to make scaling decisions.

**Worker Capacity Signal:**
```go
func (w *Worker) emitCapacitySignal() {
    signal := Signal{
        Type: Capacity,
        Source: Source{
            Component: "worker",
            Name: w.Name,
            Region: w.region,
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

**Scheduler Integration:**
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

### 4.4 Graceful Shutdown

**Shutdown Sequence:**
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

## PART V: POLICY RUNTIME

### 5.1 Policy Distribution

**Policy Loading:**
```go
type PolicyRegistry struct {
    // Active policies by name and version
    enforced map[string]map[string]Policy
    
    // Shadow policies
    shadow map[string]map[string]Policy
    
    // Deprecated policies (grace period)
    deprecated map[string]map[string]Policy
    
    // Compiled expressions (cached)
    compiled map[string]CompiledExpression
    
    mu sync.RWMutex
}

func (p *PolicyRegistry) LoadFromStream(stream JetStream) error {
    // Subscribe to policy stream
    sub, err := stream.Subscribe("policy.>", func(msg *nats.Msg) {
        artifact := ParsePolicyArtifact(msg.Data)
        p.updatePolicy(artifact)
    })
    
    // Replay historical messages
    // This ensures late-joining components get all policies
    
    return nil
}
```

### 5.2 Local Enforcement

**Enforcement Points:**

**Input Gate (UI/API):**
```
User submits InputIntent
  ↓
Evaluate InputIntent policies
  ↓
If denied → Reject with reason
If allowed → Submit to SMS
```

**SMS Gate:**
```
InputIntent received
  ↓
Evaluate PolicySet for flow
  ↓
If denied → Reject flow
If allowed → Begin execution
```

**Worker Gate:**
```
Work unit invocation received
  ↓
Evaluate DataPolicy for read/write/transition
  ↓
If denied → Return rejection outcome
If allowed → Execute work unit
```

**Policy Evaluation:**
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
        p.emitDivergenceSignal(target, enforcedDecision, shadowDecision)
    }
    
    // 4. Return enforced decision only
    return enforcedDecision
}
```

### 5.3 Shadow Evaluation

**Dual Evaluation Pattern:**
```
For each request:
  1. Evaluate enforced policy → Binding decision
  2. Evaluate shadow policy → Record divergence
  3. If divergence:
     → Emit policy signal with metrics
  4. Return enforced decision
```

**Signal Format:**
```yaml
signal:
  type: policy
  metrics:
    shadow_allows: 1456
    shadow_denies: 14
    divergence_rate: 0.0095
```

**Promotion Criteria:**
- Divergence rate < threshold (e.g., 1%)
- Sustained over time window (e.g., 24h)
- Manual approval (optional)

### 5.4 Delegation Chains (v1.5)

**Delegation Validation:**
```go
func (p *PolicyEngine) ValidateDelegation(
    delegator Subject,
    delegate Subject,
    scope DelegationScope,
) error {
    // Check delegator has permission to delegate
    if !p.canDelegate(delegator, scope) {
        return ErrCannotDelegate
    }
    
    // Check delegation depth
    chain := p.getDelegationChain(delegate)
    if len(chain) >= p.maxDelegationDepth {
        return ErrMaxDelegationDepth
    }
    
    // Check scope restrictions
    if !p.isValidScope(scope) {
        return ErrInvalidScope
    }
    
    return nil
}
```

---

## PART VI: UI & EXPERIENCE RUNTIME

### 6.1 View Composition

**Composition Resolver:**
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

**Resolution Algorithm:**
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

### 6.2 Experience Routing

**Routing Algorithm:**
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

### 6.3 Field Masking

**Field Permission Enforcement:**
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
```

### 6.4 Indicator Evaluation

**Navigation Indicator Update:**
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

---

## PART VII: V1.5 FEATURE IMPLEMENTATION

### 7.1 Form-Intent Binding

**Field Mapping Resolution:**
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

**Success/Error Handling:**
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

### 7.2 View Materialization Contracts

**Retrieval Mode Handling:**
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

**Unavailability Handling:**
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

### 7.3 Session Management

**Session Lifecycle:**
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
```

### 7.4 Asset Upload & Transformation

**Asset Storage:**
```go
type AssetManager struct {
    storage      StorageBackend
    processor    AssetProcessor
    cdn          CDNProvider
}

func (am *AssetManager) UploadAsset(
    file io.Reader,
    metadata AssetMetadata,
    spec AssetSpec,
) (AssetReference, error) {
    // 1. Validate file
    if err := am.validateAsset(file, spec); err != nil {
        return AssetReference{}, err
    }
    
    // 2. Generate asset ID
    assetID := generateAssetID()
    
    // 3. Apply transformations
    transformed, err := am.processor.Transform(file, spec.Transformations)
    if err != nil {
        return AssetReference{}, err
    }
    
    // 4. Store to backend
    path := am.constructPath(assetID, metadata)
    if err := am.storage.Store(path, transformed); err != nil {
        return AssetReference{}, err
    }
    
    // 5. Optionally upload to CDN
    var cdnURL string
    if spec.CDN.Enabled {
        cdnURL, err = am.cdn.Upload(transformed, spec.CDN)
        if err != nil {
            log.Warn("CDN upload failed", "error", err)
        }
    }
    
    // 6. Create reference
    ref := AssetReference{
        ID:       assetID,
        URL:      am.constructURL(path),
        CDNURL:   cdnURL,
        Metadata: metadata,
    }
    
    return ref, nil
}
```

### 7.5 Search Indexing

**Index Management:**
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
```

**Query Execution:**
```go
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
```

### 7.6 Scheduled Triggers

**Schedule Manager:**
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
```

### 7.7 Durable Workflows

**Workflow State Persistence:**
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
```

**Await Point Handling:**
```go
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

### 7.8 External Integration

**External Dependency Client:**
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
    
    return output, nil
}
```

**Webhook Handler:**
```go
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

### 7.9 Data Governance

**Audit Logger:**
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
```

**Data Classification:**
```go
type DataClassifier struct {
    rules []ClassificationRule
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

---

## PART VIII: CODE GENERATION

### 8.1 Parser Implementation

**Grammar to AST:**
```go
type Parser struct {
    lexer   *Lexer
    tokens  []Token
    current int
}

func (p *Parser) ParseSpec(yaml []byte) (*Specification, error) {
    // 1. Lex YAML
    p.tokens = p.lexer.Tokenize(yaml)
    
    // 2. Parse document structure
    spec := &Specification{}
    
    for p.current < len(p.tokens) {
        switch p.currentToken().Type {
        case "dataType":
            dt, err := p.parseDataType()
            if err != nil {
                return nil, err
            }
            spec.DataTypes = append(spec.DataTypes, dt)
            
        case "inputIntent":
            intent, err := p.parseInputIntent()
            if err != nil {
                return nil, err
            }
            spec.InputIntents = append(spec.InputIntents, intent)
            
        // ... other grammar elements
        }
    }
    
    return spec, nil
}
```

### 8.2 Validator Implementation

**Semantic Validation:**
```go
type Validator struct {
    spec *Specification
    errors []ValidationError
}

func (v *Validator) Validate() []ValidationError {
    v.errors = []ValidationError{}
    
    // 1. Validate cross-references
    v.validateReferences()
    
    // 2. Validate version sequences
    v.validateVersioning()
    
    // 3. Validate invariants
    v.validateInvariants()
    
    // 4. Validate policies
    v.validatePolicies()
    
    return v.errors
}

func (v *Validator) validateReferences() {
    // Check all referenced types exist
    for _, dt := range v.spec.DataTypes {
        for _, field := range dt.Fields {
            if field.Type == "ref" {
                if !v.typeExists(field.RefType) {
                    v.addError("Unknown type reference: %s", field.RefType)
                }
            }
        }
    }
}
```

### 8.3 Client SDK Generation

**DataType → Go Struct:**
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

**InputIntent → Handler Interface:**
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

---

## PART IX: OPERATIONAL PATTERNS

### 9.1 Monitoring & Observability

**Key Metrics:**
1. Policy divergence rate (shadow)
2. Flow completion rate
3. Worker capacity utilization
4. Signal latency
5. Cache hit rate
6. Idempotency collisions
7. Regional health score

**Critical Alerts:**
- Policy distribution lag > 5s
- Worker drain timeout exceeded
- Idempotency violation detected
- Cross-boundary atomic group violation
- Constraint enforcement failure

**Correlation Propagation:**
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

### 9.2 Deployment Strategies

**Blue-Green Deployment:**
```
1. Deploy new worker version (green)
2. Workers auto-register with version tag
3. Update routing rules to prefer green
4. Drain blue workers
5. Decommission blue after drain period
```

**Canary Deployment:**
```
1. Deploy canary workers (5%)
2. Route 5% of traffic via version routing
3. Monitor signals (error rate, latency)
4. If healthy: gradually increase to 100%
5. If unhealthy: roll back instantly
```

### 9.3 Testing Strategies

**Grammar Validation Testing:**
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

**Policy Shadow Testing:**
```
1. Deploy new policy in shadow mode
2. Run production traffic
3. Compare enforced vs shadow decisions
4. Analyze divergence
5. Refine policy
6. Promote when divergence < threshold
```

**Chaos Testing Scenarios:**
1. Kill random workers mid-execution
2. Partition network between regions
3. Delay policy distribution
4. Corrupt KV registrations
5. Inject slow workers

**Assertions:**
- No data loss
- No duplicate processing (idempotency)
- Graceful degradation
- Recovery without manual intervention

### 9.4 Performance Optimization

**Caching Strategy:**
- **Routing Cache**: In-memory, TTL 30s, invalidate on policy/topology change
- **Policy Cache**: In-memory, compiled expressions, invalidate on lifecycle event
- **View Cache**: Per-runtime, TTL based on freshness spec, invalidate on update events
- **Schema Cache**: In-memory, persistent across requests, invalidate on version change

**Connection Pooling:**
```go
type ConnectionPool struct {
    connections chan *Connection
    maxSize     int
    minSize     int
}

func (p *ConnectionPool) Get() (*Connection, error) {
    select {
    case conn := <-p.connections:
        if conn.IsHealthy() {
            return conn, nil
        }
        return p.createConnection()
    default:
        return p.createConnection()
    }
}
```

**Batch Processing:**
```go
type BatchProcessor struct {
    batchSize    int
    flushTimeout time.Duration
    buffer       []Event
}

func (bp *BatchProcessor) Process(event Event) {
    bp.buffer = append(bp.buffer, event)
    
    if len(bp.buffer) >= bp.batchSize {
        bp.flush()
    }
}
```

---

## PART X: ANTI-PATTERNS & GOTCHAS

### Anti-Pattern: Model-Scoped Authority (v1.3)
❌ **Wrong**: Assign authority to entire model, causing availability collapse
✅ **Right**: Entity-scoped authority, failures affect only specific entities

### Anti-Pattern: Cross-Entity CAS (v1.3)
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

## IMPLEMENTATION CHECKLIST

### Phase 1: Single Binary
- [x] Load spec from YAML
- [x] In-process work unit execution
- [x] Basic constraint validation
- [x] No NATS, no distribution

### Phase 2: Control Plane
- [x] NATS integration
- [x] Spec distribution via JetStream
- [x] Worker registration via KV
- [x] Heartbeat loop

### Phase 3: Policy
- [x] Policy loading from stream
- [x] Local policy evaluation
- [x] Shadow policy support
- [x] Lifecycle management

### Phase 4: Multi-Worker
- [x] Intent-based routing
- [x] Version-aware routing
- [x] Locality-first invocation
- [x] Graceful draining

### Phase 5: Observability
- [x] Signal emission
- [x] Correlation propagation
- [x] Metric collection
- [x] Trace reconstruction

### Phase 6: Production
- [x] Multi-region deployment
- [x] Scheduler integration
- [x] Chaos testing
- [x] Incident runbooks

### Phase 7: v1.3 Features
- [x] Entity-scoped authority
- [x] CAS token validation
- [x] Authority migration
- [x] View-only replication

### Phase 8: v1.4 Features
- [x] UI composition runtime
- [x] Experience routing
- [x] Field permissions
- [x] Navigation indicators

### Phase 9: v1.5 Features
- [x] Form-intent binding
- [x] Session management
- [x] Asset management
- [x] Search indexing
- [x] Scheduled triggers
- [x] Durable workflows
- [x] External integration
- [x] Data governance
- [x] Notification channels
- [x] Collaborative sessions
- [x] Edge device support

---

## SUMMARY

This implementation guide provides comprehensive patterns for building SMS v1.5 conformant runtimes, covering:

1. **Runtime Architecture**: Component structure, responsibilities, authority model
2. **Storage & Persistence**: NATS integration, JetStream configuration, authority KV, view materialization
3. **Multi-Region**: Regional isolation, authority management, failover, cross-region views
4. **Worker Implementation**: Registration, routing, scaling, graceful shutdown
5. **Policy Runtime**: Distribution, local enforcement, shadow evaluation, delegation
6. **UI & Experience**: View composition, routing, field masking, indicators
7. **V1.5 Features**: Form binding, view contracts, sessions, assets, search, schedules, workflows, integrations, governance
8. **Code Generation**: Parser, validator, SDK generation
9. **Operational Patterns**: Monitoring, deployment, testing, performance optimization
10. **Anti-Patterns**: Common mistakes and correct approaches

**Key Principle**: Every implementation decision preserves the grammar's intent-first, runtime-decided philosophy.

**Version History:**
- **v1.1**: Core runtime, NATS integration, policy enforcement
- **v1.3**: Entity-scoped authority, multi-region survivability, CAS tokens
- **v1.4**: UI composition, experience routing, field permissions, navigation
- **v1.5**: Complete application development support with forms, sessions, search, schedules, workflows, integrations, governance, and advanced features

---

## PART XI: FLOW EXECUTION ENGINE DETAILS

### 11.1 Flow State Machine Implementation

**State Representation:**
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

**State Transitions:**
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

### 11.2 Atomic Group Execution Details

**Placement Decision:**
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

**Cross-Runtime Atomic Groups:**
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

**Critical Invariant:** Atomic group executes under single authority, even if that authority is remote.

### 11.3 Compensation Execution

**Compensation State Machine:**
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

## PART XII: VERSIONING & EVOLUTION MECHANICS

### 12.1 Multi-Version Worker Implementation

**Worker Version Support:**
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

### 12.2 Shadow Write Implementation

**Dual Write Pattern:**
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

**Shadow Validation:**
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

### 12.3 Version Negotiation

**Content Negotiation Pattern:**
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

**Compatibility Check:**
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

## PART XIII: SCHEDULER INTEGRATION PATTERNS

### 13.1 Signal Aggregation

**Scheduler Signal Consumer:**
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

### 13.2 Scaling Decision Engine

**Evaluation Loop:**
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

### 13.3 Infrastructure Adapter

**Kubernetes HPA Adapter:**
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

## PART XIV: FAILURE HANDLING PATTERNS

### 14.1 Failure Classification

**Error Types:**
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

**Retry Decision:**
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

### 14.2 Circuit Breaker Pattern

**Worker-Level Circuit Breaker:**
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

### 14.3 Backpressure Handling

**Worker Backpressure:**
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

---

## PART XV: OBSERVABILITY IMPLEMENTATION

### 15.1 Distributed Tracing

**Context Propagation:**
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

**NATS Header Propagation:**
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

### 15.2 Metrics Collection

**Worker Metrics:**
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

## PART XVI: PERFORMANCE OPTIMIZATION

### 16.1 Routing Cache Optimization

**Cache Warming:**
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

**Batched Cache Updates:**
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

### 16.2 In-Process Optimization

**Direct Function Call Pattern:**
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

**Benchmark:**
- Local invocation: ~5-20 microseconds
- NATS invocation: ~1-5 milliseconds
- HTTP invocation: ~2-10 milliseconds

**When to Optimize:** Flows with high-frequency invocation (>1000/sec)

---

## PART XVII: CRITICAL IMPLEMENTATION NOTES

### 17.1 Idempotency Implementation

**Key Generation:**
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

**Deduplication Check:**
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

### 17.2 Boundary Enforcement

**Runtime Validation:**
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

### 17.3 Time Constraint Enforcement

**Deadline Enforcement:**
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

## PART XVIII: AUTHORITY IMPLEMENTATION (v1.3 Deep Patterns)

This section provides deep implementation patterns for entity-scoped authority introduced in v1.3.

### 18.1 JetStream KV for Authority State

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

### 18.2 Epoch Management

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

### 18.3 Lease Validation

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

### 18.4 CAS Token Encoding

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

## PART XIX: STORAGE ROLE IMPLEMENTATION (v1.3 Deep Patterns)

### 19.1 Control Storage Patterns

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

### 19.2 Data Storage Patterns

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

### 19.3 Rebuild Mechanics

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

## PART XX: VIEW WORKER AUTHORITY PATTERNS (v1.3 Deep Patterns)

### 20.1 Authority-Agnostic Consumption

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

### 20.2 Epoch Ordering

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

### 20.3 Multi-Region Event Merging

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

## PART XXI: SESSION CONTRACT DEEP IMPLEMENTATION (v1.5)

### 21.1 Distributed Session Store

Sessions must work across multiple runtime instances.

**Session Store Interface**:
```go
type SessionStore interface {
    Create(session *Session) error
    Get(sessionID string) (*Session, error)
    Update(sessionID string, updates map[string]any) error
    Delete(sessionID string) error
    Touch(sessionID string) error
    Expire(sessionID string, after time.Duration) error
}

type Session struct {
    ID              string
    Subject         *Subject
    CreatedAt       time.Time
    LastAccessedAt  time.Time
    ExpiresAt       time.Time
    Data            map[string]any
    Metadata        map[string]string
}
```

**Redis Implementation**:
```go
type RedisSessionStore struct {
    client *redis.Client
    prefix string
    ttl    time.Duration
}

func (s *RedisSessionStore) Create(session *Session) error {
    key := s.prefix + session.ID
    
    data, err := json.Marshal(session)
    if err != nil {
        return err
    }
    
    return s.client.Set(context.Background(), key, data, s.ttl).Err()
}

func (s *RedisSessionStore) Get(sessionID string) (*Session, error) {
    key := s.prefix + sessionID
    
    data, err := s.client.Get(context.Background(), key).Bytes()
    if err != nil {
        if err == redis.Nil {
            return nil, ErrSessionNotFound
        }
        return nil, err
    }
    
    var session Session
    if err := json.Unmarshal(data, &session); err != nil {
        return nil, err
    }
    
    return &session, nil
}

func (s *RedisSessionStore) Touch(sessionID string) error {
    key := s.prefix + sessionID
    
    // Refresh TTL
    return s.client.Expire(context.Background(), key, s.ttl).Err()
}
```

### 21.2 Activity Tracking

Track user activity for analytics and session management.

**Activity Tracker**:
```go
type ActivityTracker struct {
    store    ActivityStore
    buffer   chan ActivityEvent
    batchSize int
}

type ActivityEvent struct {
    SessionID   string
    Subject     string
    EventType   string
    Composition string
    Timestamp   time.Time
    Metadata    map[string]any
}

func (t *ActivityTracker) Track(event ActivityEvent) {
    select {
    case t.buffer <- event:
        // Buffered successfully
    default:
        // Buffer full, log warning
        log.Warn("Activity buffer full, dropping event")
    }
}

func (t *ActivityTracker) processBatch() {
    batch := make([]ActivityEvent, 0, t.batchSize)
    
    for event := range t.buffer {
        batch = append(batch, event)
        
        if len(batch) >= t.batchSize {
            if err := t.store.WriteBatch(batch); err != nil {
                log.Error("Failed to write activity batch", "error", err)
            }
            batch = batch[:0]
        }
    }
    
    // Flush remaining
    if len(batch) > 0 {
        t.store.WriteBatch(batch)
    }
}
```

### 21.3 Session Timeout Handling

Implement both idle timeout and absolute timeout.

**Timeout Manager**:
```go
type SessionTimeoutManager struct {
    store        SessionStore
    idleTimeout  time.Duration
    maxDuration  time.Duration
    checkInterval time.Duration
}

func (m *SessionTimeoutManager) CheckSession(sessionID string) (bool, error) {
    session, err := m.store.Get(sessionID)
    if err != nil {
        return false, err
    }
    
    now := time.Now()
    
    // Check absolute timeout
    if now.After(session.ExpiresAt) {
        m.store.Delete(sessionID)
        return false, ErrSessionExpired
    }
    
    // Check idle timeout
    idleFor := now.Sub(session.LastAccessedAt)
    if idleFor > m.idleTimeout {
        m.store.Delete(sessionID)
        return false, ErrSessionIdle
    }
    
    // Touch session to extend TTL
    m.store.Touch(sessionID)
    
    return true, nil
}

func (m *SessionTimeoutManager) CleanupLoop(ctx context.Context) {
    ticker := time.NewTicker(m.checkInterval)
    defer ticker.Stop()
    
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            m.cleanupExpiredSessions()
        }
    }
}
```

---

## PART XXII: ASSET SEMANTICS DEEP IMPLEMENTATION (v1.5)

### 22.1 Asset Storage Abstraction

Support multiple storage backends (S3, GCS, Azure Blob, local).

**Storage Provider Interface**:
```go
type AssetStorageProvider interface {
    Upload(ctx context.Context, asset *Asset, reader io.Reader) (*StorageLocation, error)
    Download(ctx context.Context, location *StorageLocation) (io.ReadCloser, error)
    Delete(ctx context.Context, location *StorageLocation) error
    GetMetadata(ctx context.Context, location *StorageLocation) (*AssetMetadata, error)
    GenerateURL(ctx context.Context, location *StorageLocation, expiry time.Duration) (string, error)
}

type Asset struct {
    ID          string
    Name        string
    ContentType string
    Size        int64
    Checksum    string
    UploadedBy  string
    UploadedAt  time.Time
}

type StorageLocation struct {
    Provider string
    Bucket   string
    Key      string
    Region   string
}
```

**S3 Provider Implementation**:
```go
type S3StorageProvider struct {
    client *s3.Client
    bucket string
}

func (p *S3StorageProvider) Upload(
    ctx context.Context,
    asset *Asset,
    reader io.Reader,
) (*StorageLocation, error) {
    
    key := p.generateKey(asset)
    
    _, err := p.client.PutObject(ctx, &s3.PutObjectInput{
        Bucket:      aws.String(p.bucket),
        Key:         aws.String(key),
        Body:        reader,
        ContentType: aws.String(asset.ContentType),
        Metadata: map[string]string{
            "asset-id":    asset.ID,
            "uploaded-by": asset.UploadedBy,
            "checksum":    asset.Checksum,
        },
    })
    
    if err != nil {
        return nil, err
    }
    
    return &StorageLocation{
        Provider: "s3",
        Bucket:   p.bucket,
        Key:      key,
    }, nil
}

func (p *S3StorageProvider) GenerateURL(
    ctx context.Context,
    location *StorageLocation,
    expiry time.Duration,
) (string, error) {
    
    presignClient := s3.NewPresignClient(p.client)
    
    req, err := presignClient.PresignGetObject(ctx, &s3.GetObjectInput{
        Bucket: aws.String(location.Bucket),
        Key:    aws.String(location.Key),
    }, s3.WithPresignExpires(expiry))
    
    if err != nil {
        return "", err
    }
    
    return req.URL, nil
}
```

### 22.2 Image Transformation Pipeline

On-demand image transformations (resize, crop, format conversion).

**Transformation Engine**:
```go
type ImageTransformer struct {
    cache    TransformCache
    storage  AssetStorageProvider
}

type TransformSpec struct {
    Width   int
    Height  int
    Crop    string // "fill", "fit", "scale"
    Format  string // "jpeg", "png", "webp"
    Quality int
}

func (t *ImageTransformer) Transform(
    ctx context.Context,
    assetID string,
    spec *TransformSpec,
) (io.ReadCloser, error) {
    
    // Check cache first
    cacheKey := t.generateCacheKey(assetID, spec)
    if cached, ok := t.cache.Get(cacheKey); ok {
        return cached, nil
    }
    
    // Download original
    original, err := t.storage.Download(ctx, &StorageLocation{
        Key: assetID,
    })
    if err != nil {
        return nil, err
    }
    defer original.Close()
    
    // Decode image
    img, _, err := image.Decode(original)
    if err != nil {
        return nil, err
    }
    
    // Apply transformations
    transformed := t.applyTransformations(img, spec)
    
    // Encode in target format
    var buf bytes.Buffer
    if err := t.encode(&buf, transformed, spec); err != nil {
        return nil, err
    }
    
    // Cache result
    t.cache.Set(cacheKey, buf.Bytes())
    
    return io.NopCloser(bytes.NewReader(buf.Bytes())), nil
}
```

### 22.3 CDN Integration

Serve assets through CDN with automatic cache invalidation.

**CDN Manager**:
```go
type CDNManager struct {
    provider      CDNProvider
    originStorage AssetStorageProvider
    urlPrefix     string
}

type CDNProvider interface {
    Invalidate(paths []string) error
    GetStats(path string) (*CDNStats, error)
}

func (m *CDNManager) GetAssetURL(assetID string) string {
    return m.urlPrefix + "/" + assetID
}

func (m *CDNManager) InvalidateAsset(assetID string) error {
    paths := []string{
        "/" + assetID,
        "/" + assetID + "/thumb",
        "/" + assetID + "/preview",
    }
    
    return m.provider.Invalidate(paths)
}
```

---

## PART XXIII: SEARCH INFRASTRUCTURE DEEP IMPLEMENTATION (v1.5)

### 23.1 Full-Text Search Indexing

Index materialized views for full-text search.

**Search Indexer**:
```go
type SearchIndexer struct {
    client      *elasticsearch.Client
    indexPrefix string
}

type SearchDocument struct {
    ID          string
    Type        string
    Title       string
    Description string
    Content     string
    Tags        []string
    Metadata    map[string]any
    CreatedAt   time.Time
    UpdatedAt   time.Time
}

func (i *SearchIndexer) Index(doc *SearchDocument) error {
    indexName := i.indexPrefix + doc.Type
    
    body, err := json.Marshal(doc)
    if err != nil {
        return err
    }
    
    req := esapi.IndexRequest{
        Index:      indexName,
        DocumentID: doc.ID,
        Body:       bytes.NewReader(body),
        Refresh:    "true",
    }
    
    res, err := req.Do(context.Background(), i.client)
    if err != nil {
        return err
    }
    defer res.Body.Close()
    
    if res.IsError() {
        return fmt.Errorf("indexing failed: %s", res.String())
    }
    
    return nil
}
```

### 23.2 Faceted Search

Support filtering and faceting for search results.

**Faceted Search Query Builder**:
```go
type FacetedSearchQuery struct {
    Query   string
    Filters map[string][]string
    Facets  []string
    From    int
    Size    int
}

func (i *SearchIndexer) Search(query *FacetedSearchQuery) (*SearchResults, error) {
    // Build query
    must := []map[string]any{}
    
    // Add text query
    if query.Query != "" {
        must = append(must, map[string]any{
            "multi_match": map[string]any{
                "query":  query.Query,
                "fields": []string{"title^3", "description^2", "content"},
            },
        })
    }
    
    // Add filters
    for field, values := range query.Filters {
        must = append(must, map[string]any{
            "terms": map[string]any{
                field: values,
            },
        })
    }
    
    // Build aggregations for facets
    aggs := make(map[string]any)
    for _, facet := range query.Facets {
        aggs[facet] = map[string]any{
            "terms": map[string]any{
                "field": facet,
                "size":  20,
            },
        }
    }
    
    // Execute search
    searchBody := map[string]any{
        "query": map[string]any{
            "bool": map[string]any{
                "must": must,
            },
        },
        "aggs": aggs,
        "from": query.From,
        "size": query.Size,
    }
    
    body, _ := json.Marshal(searchBody)
    
    res, err := i.client.Search(
        i.client.Search.WithBody(bytes.NewReader(body)),
        i.client.Search.WithTrackTotalHits(true),
    )
    
    if err != nil {
        return nil, err
    }
    defer res.Body.Close()
    
    // Parse results
    return i.parseSearchResults(res)
}
```

### 23.3 Relevance Scoring

Implement custom relevance scoring with boost factors.

**Relevance Scorer**:
```go
type RelevanceScorer struct {
    boostFactors map[string]float64
}

func NewRelevanceScorer() *RelevanceScorer {
    return &RelevanceScorer{
        boostFactors: map[string]float64{
            "exact_title_match":   3.0,
            "recent_document":     1.5,
            "high_view_count":     1.2,
            "verified_source":     1.3,
        },
    }
}

func (s *RelevanceScorer) ApplyBoosts(
    baseQuery map[string]any,
    context map[string]any,
) map[string]any {
    
    shouldClauses := []map[string]any{}
    
    // Add base query
    shouldClauses = append(shouldClauses, baseQuery)
    
    // Add boost for exact title match
    if query, ok := context["query"].(string); ok {
        shouldClauses = append(shouldClauses, map[string]any{
            "match_phrase": map[string]any{
                "title": map[string]any{
                    "query": query,
                    "boost": s.boostFactors["exact_title_match"],
                },
            },
        })
    }
    
    // Add recency boost
    shouldClauses = append(shouldClauses, map[string]any{
        "function_score": map[string]any{
            "functions": []map[string]any{
                {
                    "gauss": map[string]any{
                        "created_at": map[string]any{
                            "origin": time.Now().Format(time.RFC3339),
                            "scale":  "30d",
                            "decay":  0.5,
                        },
                    },
                    "weight": s.boostFactors["recent_document"],
                },
            },
        },
    })
    
    return map[string]any{
        "bool": map[string]any{
            "should": shouldClauses,
        },
    }
}
```

---

## PART XXIV: EXTERNAL INTEGRATION DEEP PATTERNS (v1.5)

### 24.1 Circuit Breaker Pattern

Protect against cascading failures when calling external services.

**Circuit Breaker Implementation**:
```go
type CircuitBreaker struct {
    name            string
    maxFailures     int
    timeout         time.Duration
    resetTimeout    time.Duration
    state           CircuitState
    failures        int
    lastFailureTime time.Time
    mu              sync.RWMutex
}

type CircuitState int
const (
    StateClosed CircuitState = iota
    StateOpen
    StateHalfOpen
)

func (cb *CircuitBreaker) Call(fn func() error) error {
    cb.mu.Lock()
    
    switch cb.state {
    case StateOpen:
        // Check if reset timeout has passed
        if time.Since(cb.lastFailureTime) > cb.resetTimeout {
            cb.state = StateHalfOpen
            cb.mu.Unlock()
        } else {
            cb.mu.Unlock()
            return ErrCircuitOpen
        }
        
    case StateHalfOpen:
        cb.mu.Unlock()
        
    case StateClosed:
        cb.mu.Unlock()
    }
    
    // Execute function with timeout
    done := make(chan error, 1)
    go func() {
        done <- fn()
    }()
    
    select {
    case err := <-done:
        if err != nil {
            cb.recordFailure()
            return err
        }
        cb.recordSuccess()
        return nil
        
    case <-time.After(cb.timeout):
        cb.recordFailure()
        return ErrTimeout
    }
}

func (cb *CircuitBreaker) recordFailure() {
    cb.mu.Lock()
    defer cb.mu.Unlock()
    
    cb.failures++
    cb.lastFailureTime = time.Now()
    
    if cb.failures >= cb.maxFailures {
        cb.state = StateOpen
        log.Warn("Circuit breaker opened",
            "name", cb.name,
            "failures", cb.failures)
    }
}

func (cb *CircuitBreaker) recordSuccess() {
    cb.mu.Lock()
    defer cb.mu.Unlock()
    
    cb.failures = 0
    
    if cb.state == StateHalfOpen {
        cb.state = StateClosed
        log.Info("Circuit breaker closed", "name", cb.name)
    }
}
```

### 24.2 Rate Limiting

Implement token bucket or sliding window rate limiting.

**Token Bucket Rate Limiter**:
```go
type TokenBucketRateLimiter struct {
    capacity int
    refillRate int  // tokens per second
    tokens   float64
    lastRefill time.Time
    mu       sync.Mutex
}

func (r *TokenBucketRateLimiter) Allow() bool {
    r.mu.Lock()
    defer r.mu.Unlock()
    
    // Refill tokens
    now := time.Now()
    elapsed := now.Sub(r.lastRefill).Seconds()
    r.tokens += elapsed * float64(r.refillRate)
    
    if r.tokens > float64(r.capacity) {
        r.tokens = float64(r.capacity)
    }
    
    r.lastRefill = now
    
    // Check if token available
    if r.tokens >= 1.0 {
        r.tokens -= 1.0
        return true
    }
    
    return false
}

func (r *TokenBucketRateLimiter) Wait(ctx context.Context) error {
    for {
        if r.Allow() {
            return nil
        }
        
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-time.After(10 * time.Millisecond):
            // Retry
        }
    }
}
```

### 24.3 Credential Rotation

Automatically rotate API credentials without downtime.

**Credential Manager**:
```go
type CredentialManager struct {
    current  *Credential
    next     *Credential
    provider CredentialProvider
    rotation time.Duration
    mu       sync.RWMutex
}

type Credential struct {
    Key       string
    Secret    string
    ExpiresAt time.Time
}

func (m *CredentialManager) GetCredential() *Credential {
    m.mu.RLock()
    defer m.mu.RUnlock()
    
    if m.current != nil && time.Now().Before(m.current.ExpiresAt) {
        return m.current
    }
    
    if m.next != nil && time.Now().Before(m.next.ExpiresAt) {
        return m.next
    }
    
    return nil
}

func (m *CredentialManager) RotationLoop(ctx context.Context) {
    ticker := time.NewTicker(m.rotation)
    defer ticker.Stop()
    
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            m.rotate()
        }
    }
}

func (m *CredentialManager) rotate() {
    // Request new credential
    newCred, err := m.provider.RequestCredential()
    if err != nil {
        log.Error("Failed to rotate credential", "error", err)
        return
    }
    
    m.mu.Lock()
    defer m.mu.Unlock()
    
    // Promote next to current, set new as next
    m.current = m.next
    m.next = newCred
    
    log.Info("Credential rotated",
        "expires_at", newCred.ExpiresAt)
}
```

### 24.4 Webhook Delivery

Reliable webhook delivery with retries and dead-letter queue.

**Webhook Dispatcher**:
```go
type WebhookDispatcher struct {
    client      *http.Client
    maxRetries  int
    retryDelay  time.Duration
    dlq         DeadLetterQueue
}

type WebhookEvent struct {
    ID        string
    URL       string
    Event     string
    Payload   map[string]any
    Signature string
    Timestamp time.Time
}

func (d *WebhookDispatcher) Dispatch(event *WebhookEvent) error {
    var lastErr error
    
    for attempt := 0; attempt < d.maxRetries; attempt++ {
        if attempt > 0 {
            time.Sleep(d.retryDelay * time.Duration(1<<uint(attempt)))
        }
        
        err := d.sendWebhook(event)
        if err == nil {
            log.Info("Webhook delivered",
                "id", event.ID,
                "url", event.URL,
                "attempts", attempt+1)
            return nil
        }
        
        lastErr = err
        log.Warn("Webhook delivery failed",
            "id", event.ID,
            "attempt", attempt+1,
            "error", err)
    }
    
    // All retries failed, send to DLQ
    d.dlq.Add(event, lastErr)
    
    return fmt.Errorf("webhook delivery failed after %d attempts: %w",
        d.maxRetries, lastErr)
}

func (d *WebhookDispatcher) sendWebhook(event *WebhookEvent) error {
    payload, err := json.Marshal(event.Payload)
    if err != nil {
        return err
    }
    
    req, err := http.NewRequest("POST", event.URL, bytes.NewReader(payload))
    if err != nil {
        return err
    }
    
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("X-Webhook-ID", event.ID)
    req.Header.Set("X-Webhook-Event", event.Event)
    req.Header.Set("X-Webhook-Signature", event.Signature)
    req.Header.Set("X-Webhook-Timestamp", event.Timestamp.Format(time.RFC3339))
    
    resp, err := d.client.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    
    if resp.StatusCode < 200 || resp.StatusCode >= 300 {
        return fmt.Errorf("webhook returned status %d", resp.StatusCode)
    }
    
    return nil
}
```

---

## PART XXV: DATA GOVERNANCE DEEP IMPLEMENTATION (v1.5)

### 25.1 Retention Policy Enforcement

Automatically delete data based on retention policies.

**Retention Policy Engine**:
```go
type RetentionPolicyEngine struct {
    policies map[string]*RetentionPolicy
    storage  DataStorage
}

type RetentionPolicy struct {
    DataType    string
    Retention   time.Duration
    DeleteMode  string // "hard", "soft", "archive"
    Conditions  []*RetentionCondition
}

type RetentionCondition struct {
    Field    string
    Operator string
    Value    any
}

func (e *RetentionPolicyEngine) EnforceRetention(ctx context.Context) error {
    for dataType, policy := range e.policies {
        cutoff := time.Now().Add(-policy.Retention)
        
        log.Info("Enforcing retention policy",
            "dataType", dataType,
            "cutoff", cutoff,
            "mode", policy.DeleteMode)
        
        query := e.buildQuery(dataType, cutoff, policy.Conditions)
        
        switch policy.DeleteMode {
        case "hard":
            err := e.storage.Delete(query)
            if err != nil {
                return err
            }
            
        case "soft":
            err := e.storage.Update(query, map[string]any{
                "deleted_at": time.Now(),
            })
            if err != nil {
                return err
            }
            
        case "archive":
            err := e.archiveAndDelete(query)
            if err != nil {
                return err
            }
        }
    }
    
    return nil
}
```

### 25.2 Erasure Workflows

Implement GDPR/CCPA "right to be forgotten".

**Erasure Coordinator**:
```go
type ErasureCoordinator struct {
    dataMap  *DataEntityMap
    workers  []ErasureWorker
    audit    AuditLog
}

type ErasureRequest struct {
    RequestID  string
    SubjectID  string
    RequestedBy string
    RequestedAt time.Time
    Reason     string
}

func (c *ErasureCoordinator) ProcessErasure(
    req *ErasureRequest,
) (*ErasureResult, error) {
    
    log.Info("Processing erasure request",
        "requestID", req.RequestID,
        "subjectID", req.SubjectID)
    
    // Discover all data for subject
    entities := c.dataMap.FindEntitiesForSubject(req.SubjectID)
    
    result := &ErasureResult{
        RequestID:    req.RequestID,
        EntitiesFound: len(entities),
        Erased:       []string{},
        Failed:       []ErasureFailure{},
    }
    
    // Process each entity
    for _, entity := range entities {
        worker := c.findWorkerForEntity(entity)
        if worker == nil {
            result.Failed = append(result.Failed, ErasureFailure{
                Entity: entity.Name,
                Reason: "No worker available",
            })
            continue
        }
        
        err := worker.Erase(entity, req.SubjectID)
        if err != nil {
            result.Failed = append(result.Failed, ErasureFailure{
                Entity: entity.Name,
                Reason: err.Error(),
            })
        } else {
            result.Erased = append(result.Erased, entity.Name)
        }
    }
    
    // Audit the erasure
    c.audit.Record(AuditEntry{
        Type:      "data_erasure",
        RequestID: req.RequestID,
        SubjectID: req.SubjectID,
        Result:    result,
        Timestamp: time.Now(),
    })
    
    return result, nil
}
```

### 25.3 Data Residency Enforcement

Ensure data stays in specified regions.

**Residency Enforcer**:
```go
type ResidencyEnforcer struct {
    rules   map[string]*ResidencyRule
    regions map[string]*Region
}

type ResidencyRule struct {
    DataType       string
    AllowedRegions []string
    EnforceAt      string // "write", "read", "both"
}

func (e *ResidencyEnforcer) ValidateWrite(
    dataType string,
    targetRegion string,
) error {
    
    rule, ok := e.rules[dataType]
    if !ok {
        return nil  // No rule, allow
    }
    
    if rule.EnforceAt != "write" && rule.EnforceAt != "both" {
        return nil
    }
    
    for _, allowed := range rule.AllowedRegions {
        if allowed == targetRegion {
            return nil  // Allowed
        }
    }
    
    return fmt.Errorf("data residency violation: %s not allowed in %s",
        dataType, targetRegion)
}

func (e *ResidencyEnforcer) RouteRead(
    dataType string,
    currentRegion string,
) (string, error) {
    
    rule, ok := e.rules[dataType]
    if !ok {
        return currentRegion, nil
    }
    
    // Check if current region allowed
    for _, allowed := range rule.AllowedRegions {
        if allowed == currentRegion {
            return currentRegion, nil
        }
    }
    
    // Must route to allowed region
    if len(rule.AllowedRegions) > 0 {
        return rule.AllowedRegions[0], nil
    }
    
    return "", fmt.Errorf("no allowed region for %s", dataType)
}
```

---

## SUMMARY OF DEEP IMPLEMENTATION PATTERNS

### v1.3 Deep Patterns (Parts XVIII-XX)

1. **Authority Store**: JetStream KV-backed entity authority state
2. **Epoch Management**: Validate and increment authority epochs
3. **Lease Validation**: Time-bounded authority leases
4. **CAS Token Encoding**: HMAC-signed, base64-encoded tokens
5. **Control Storage**: Small, strongly consistent buckets
6. **Data Storage**: Append-only, rebuildable streams
7. **View Rebuild**: Replay from event streams
8. **Authority-Agnostic Consumption**: Multi-region event processing
9. **Epoch Ordering**: Buffer and order cross-epoch events
10. **Regional Merging**: Combine streams from multiple regions

### v1.5 Deep Patterns (Parts XXI-XXV)

1. **Distributed Sessions**: Redis-backed, multi-instance sessions
2. **Activity Tracking**: Batched activity event processing
3. **Session Timeout**: Idle and absolute timeout handling
4. **Asset Storage**: Multi-provider abstraction (S3, GCS, Azure)
5. **Presigned URLs**: Secure temporary access tokens
6. **Image Transformation**: On-demand resize, crop, format conversion
7. **CDN Integration**: Cache invalidation patterns
8. **Full-Text Search**: Elasticsearch indexing and querying
9. **Faceted Search**: Filter and aggregate results
10. **Relevance Scoring**: Custom boost factors
11. **Circuit Breaker**: Protect against external failures
12. **Token Bucket Rate Limiting**: Prevent API overload
13. **Credential Rotation**: Zero-downtime secret updates
14. **Webhook Delivery**: Reliable delivery with retries and DLQ
15. **Retention Enforcement**: Automatic data deletion by policy
16. **Erasure Workflows**: GDPR/CCPA compliance
17. **Residency Enforcement**: Geographic data constraints

### Critical Performance Numbers

- **Session Lookup**: < 1ms (Redis)
- **Authority Validation**: < 5ms (KV lookup)
- **CAS Token Encode/Decode**: < 100 microseconds
- **Asset Upload**: Streaming, no memory buffering
- **Search Query**: 50-200ms (depending on corpus size)
- **Circuit Breaker Check**: < 1 microsecond
- **Rate Limiter Check**: < 100 microseconds

---

## VERSION HISTORY & COMPLETENESS

**Implementation Coverage:**
- **v1.1**: Core runtime patterns, NATS integration, policy enforcement
- **v1.3**: Entity-scoped authority, multi-region, CAS tokens, authority migration
- **v1.4**: UI composition, experience routing, field permissions, navigation
- **v1.5**: Complete application development - forms, sessions, assets, search, schedules, workflows, integrations, governance

**Key Design Principles Preserved:**
1. Intent-first, runtime-decided execution
2. Local policy evaluation, no synchronous auth
3. Locality-first routing with graceful fallback
4. Entity-scoped authority for fine-grained control
5. View-only cross-region replication
6. Signal-driven scaling with emergent behavior
7. Graceful degradation under failure

**Total Coverage:** This implementation guide comprehensively covers all runtime components needed to build a conformant SMS v1.5 system.

