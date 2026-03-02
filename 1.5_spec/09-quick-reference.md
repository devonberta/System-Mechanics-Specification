# System Mechanics Specification - Quick Reference v1.5

## Overview

This document provides quick-reference cheat sheets, decision trees, and templates for common scenarios. Use this for rapid lookup during implementation.

### v1.5 Quick References (NEW)

- Form Intent Binding Cheat Sheet
- Asset Management Quick Reference
- Search & Discovery Patterns
- Session Contract Guide
- Scheduled Triggers & Notifications
- Workflow & Collaboration Patterns
- Advanced Data Types (Spatial, Structured Content)
- External Integration & Governance

### v1.4 Quick References

- UI Composition Cheat Sheet
- Experience Definition Guide
- Field Permission Patterns
- Navigation Hierarchy Builder
- Indicator Configuration

### v1.3 Quick References

- Authority Declaration Cheat Sheet
- Storage Role Decision Tree
- Invariant Placement Guide
- Authority Migration Triggers
- CAS Token Troubleshooting

---

## SPECIFICATION AUTHORING CHEAT SHEET

### Minimal Valid Specification

```yaml
system:
  name: MySystem
  version: v1

data:
  types:
    - name: MyData
      version: v1
      fields:
        id: uuid
        value: string

input:
  intents:
    - name: CreateMyData
      version: v1
      proposes:
        dataState: MyDataRecord
        version: v1
      supplies:
        required: [value]
      transitions:
        to: MyDataRecord

wts:
  workers:
    - name: MyWorker
      acceptsInputs: [CreateMyData v1]
      produces: [MyDataRecord v1]

sms:
  flows:
    - name: CreateFlow
      triggeredBy:
        inputIntent: CreateMyData
      steps:
        - persist: MyDataRecord
```

**Lines of code**: ~30
**Functionality**: Basic CRUD operation

---

## DECISION TREES

### When to Use Atomic Groups?

```
Do steps need to execute together?
├─ NO → Use regular flow steps
└─ YES → Are they in same boundary?
    ├─ NO → Use compensation pattern instead
    └─ YES → Use atomic group
        └─ Do they share local state?
            ├─ YES → Definitely use atomic group
            └─ NO → Consider if tight coupling justified
```

### Which Lifecycle for DataState?

```
Is data regenerable from other sources?
├─ YES → Use 'materialized'
└─ NO → Is data temporary during processing?
    ├─ YES → Use 'intermediate'
    └─ NO → Use 'persistent'
```

### Which Enforcement Level for Constraints?

```
Can system tolerate invalid data temporarily?
├─ YES → Is validation expensive?
│   ├─ YES → Use 'deferred'
│   └─ NO → Use 'best_effort'
└─ NO → Is data financially/legally critical?
    ├─ YES → Use 'strict'
    └─ NO → Use 'best_effort'
```

### When to Shadow a Change?

```
Is change to production system?
├─ NO → Skip shadow (dev/test)
└─ YES → Does change affect authorization/data/schema?
    ├─ NO → Maybe skip shadow
    └─ YES → ALWAYS shadow first
        └─ Monitor for: 24h minimum
```

### Which Completion Policy?

```
Must all steps complete?
├─ YES → Use 'all'
└─ NO → Is first response good enough?
    ├─ YES → Use 'any'
    └─ NO → Need N of M responses?
        ├─ YES → Use 'quorum'
        └─ NO → Reconsider design
```

---

## AUTHORITY & INVARIANT DECISION TREES (NEW in v1.3)

### Which Authority Scope?

```
Can entities be written independently?
├─ YES → Use 'entity' scope (recommended)
│   └─ Can authority move between regions?
│       ├─ YES → Enable migration with triggers
│       └─ NO → Set migration.allowed: false
└─ NO → Reconsider entity design
    └─ Can you decompose into independent entities?
        ├─ YES → Decompose, use entity scope
        └─ NO → Consider model scope (legacy)
```

### Should Authority Migrate?

```
Is data user-local or region-specific?
├─ NO → Usually no benefit to migration
└─ YES → Does user move between regions?
    ├─ NO → Stable authority
    └─ YES → Enable migration
        └─ Triggers:
            ├─ Follow-the-sun → time-based
            ├─ Latency optimization → latency-based
            └─ User request → manual
```

### Which Storage Role?

```
Is this coordination or domain data?
├─ Coordination (authority, leases, locks)
│   └─ Use 'control' role
│       └─ Properties:
│           ├─ Strongly consistent
│           ├─ Small payloads
│           └─ Not domain data source
└─ Domain data (events, state, views)
    └─ Use 'data' role
        └─ Properties:
            ├─ Append-only allowed
            ├─ Rebuildable
            └─ Can be partitioned
```

### Where Does Invariant Belong?

```
Does invariant span multiple entities?
├─ NO → Can enforce in single entity
│   └─ Consider entity-level constraint
└─ YES → How strong must enforcement be?
    ├─ Synchronous/blocking → Policy with compensation
    └─ Eventually consistent → View-level invariant
        └─ What on violation?
            ├─ Block writes → reject
            ├─ Alert operators → flag
            └─ Log only → log
```

---

## AUTHORITY DECLARATION CHEAT SHEET (NEW in v1.3)

### Minimal Authority Declaration

```yaml
authority:
  scope: entity
  resolution: entity.home_region
  migration:
    allowed: false
```

### Full Authority with Migration

```yaml
authority:
  scope: entity
  resolution: entity.active_region
  migration:
    allowed: true
    strategy: pause-and-cutover
    triggers:
      - type: latency
        condition: avg_write_latency > 100ms
      - type: manual
        condition: operator_request
    constraints:
      - type: region
        allowed: [us-east, eu-west, ap-southeast]
      - type: governance
        rule: entity.data_classification != "restricted"
    governed_by: MigrationPolicy
```

### CAS Token Structure

```
entity_id        : UUID of entity
entity_version   : Current version number
authority_epoch  : Monotonic epoch counter
authority_region : Current authoritative region
lease_id         : Current lease ID
```

### Authority State Values

| Status | Meaning | Writes Allowed |
|--------|---------|----------------|
| `ACTIVE` | Region is authoritative | Yes |
| `TRANSITIONING` | Authority transferring | No |

---

## INVARIANT PLACEMENT QUICK REFERENCE (NEW in v1.3)

| Invariant Type | Placement | Example |
|----------------|-----------|---------|
| Single-field constraint | Entity | `age >= 0` |
| Cross-field constraint | Entity | `end_date > start_date` |
| Cross-entity sum | View | `account.balance >= 0` |
| Business rule | Policy | `user.verified == true` |
| Audit requirement | View + Policy | Transaction history |

### View Invariant Template

```yaml
view:
  name: BalanceView
  inputs: [Account, Transaction]
  
  aggregation:
    type: sum
    expression: sum(credits) - sum(debits)
  
  invariant:
    expression: result >= 0
    on_violation: flag
  
  authority_agnostic: true
  epoch_tolerant: true
```

---

## TROUBLESHOOTING AUTHORITY (NEW in v1.3)

### CAS Failure Diagnosis

| Error | Cause | Fix |
|-------|-------|-----|
| `ErrEpochMismatch` | Authority migrated | Refresh CAS token |
| `ErrVersionMismatch` | Concurrent write | Reload entity, retry |
| `ErrWrongRegion` | Sent to wrong region | Route to authority_region |
| `ErrLeaseExpired` | Lease not renewed | Wait for new lease |
| `ErrTransitioning` | Authority moving | Wait, retry after transition |

### Authority Migration Issues

| Symptom | Possible Cause | Investigation |
|---------|---------------|---------------|
| Writes rejected everywhere | Transition stuck | Check `SMS.AUTHORITY` KV |
| Reads stale after migration | View not caught up | Check view worker lag |
| Epoch higher than expected | Rapid migrations | Review migration triggers |
| Region mismatch in token | Stale client cache | Force cache refresh |

### View Authority Issues

| Symptom | Possible Cause | Fix |
|---------|---------------|-----|
| View stops updating | Epoch boundary not received | Check epoch markers |
| Events from wrong region | Multi-region merge issue | Verify merger config |
| Duplicate events in view | Epoch rollback | Should not happen (investigate) |

---

## COMMON PATTERNS

### Pattern: Create Entity

```yaml
# 1. Define data
dataType:
  name: Entity
  version: v1
  fields:
    id: uuid
    name: string

dataState:
  name: EntityRecord
  type: Entity
  lifecycle: persistent

# 2. Define input
inputIntent:
  name: CreateEntity
  version: v1
  proposes: {dataState: EntityRecord, version: v1}
  supplies:
    required: [name]
  transitions:
    to: EntityRecord

# 3. Define flow
sms:
  flows:
    - name: CreateEntityFlow
      triggeredBy: {inputIntent: CreateEntity}
      steps:
        - validate: constraints
        - persist: EntityRecord

# 4. Define worker
wts:
  workers:
    - name: EntityService
      acceptsInputs: [CreateEntity v1]
      produces: [EntityRecord v1]
```

### Pattern: Cross-Boundary Transaction

```yaml
flow:
  name: OrderCheckout
  steps:
    - work_unit: reserve_inventory
      boundary: inventory
    - work_unit: charge_payment
      boundary: billing
    - work_unit: confirm_order
  failure:
    policy: compensate
    compensations:
      - on: charge_payment
        do: release_inventory
```

### Pattern: Data Evolution

```yaml
# Step 1: Define new version
dataType:
  name: Entity
  version: v2
  fields:
    id: uuid
    name: string
    description: string  # NEW

# Step 2: Define evolution
dataEvolution:
  name: Entity_v1_to_v2
  from: {type: Entity, version: v1}
  to: {type: Entity, version: v2}
  changes:
    - addField: {name: description, default: ""}
  compatibility:
    read: backward
    write: forward

# Step 3: Update workers to support both
wts:
  workers:
    - name: EntityService
      compatibleVersions: [v1, v2]  # Both!
      
# Step 4: Shadow write
# (Implementation handles dual writes)

# Step 5: Promote when safe
# (Routing preferences shift to v2)
```

### Pattern: Policy Rollout

```yaml
# Step 1: Deploy shadow
policyArtifact:
  policy: {name: NewAccessPolicy, version: v2}
  lifecycle:
    state: shadow
  scope: {domain: entity, component: worker.EntityService}

# Step 2: Monitor divergence
# (Watch signals for shadow vs enforced differences)

# Step 3: Promote
policyArtifact:
  policy: {name: NewAccessPolicy, version: v2}
  lifecycle:
    state: enforce
    supersedes: v1

# Step 4: Deprecate old
policyArtifact:
  policy: {name: NewAccessPolicy, version: v1}
  lifecycle:
    state: deprecated
```

### Pattern: Multi-Region Deployment

```yaml
topology:
  locations:
    - name: us_east
      scope: region
      provisioner: kubernetes
    - name: eu_west
      scope: region
      provisioner: kubernetes
  
  placement:
    worker_class: api_worker
    locations: [us_east, eu_west]
    min: 3
    max: 20
    distribution:
      strategy: weighted
      weights:
        us_east: 0.7
        eu_west: 0.3
  
  scaling_policy:
    signals:
      queue_depth > 100 => scale_up
      region.health == "degraded" => scale_up
  
  failure_domain:
    name: us_east_domain
    affected_locations: [us_east]
    failover:
      automatic: true
      target_domains: [eu_west_domain]
```

---

## COMMON EXPRESSIONS

### Constraint Expressions

```yaml
# Equality
total == subtotal + tax - discount

# Comparison
paid_amount <= total

# Logical AND
amount > 0 AND currency in ["USD", "EUR"]

# Logical OR
status == "approved" OR approver.role == "admin"

# Field access
order.customer.tier == "premium"

# Function call
len(items) > 0
```

### Policy Expressions

```yaml
# Role-based
subject.role == "admin"

# Attribute-based
subject.department == data.owner_department

# Complex
(subject.role == "customer" AND data.customer_id == subject.id) 
OR (subject.role == "admin")
OR (subject.role == "support" AND subject.tier >= 2)

# Time-based
data.created_at > now() - duration("7d")

# Data-dependent
data.amount < 1000 OR subject.approval_limit >= data.amount
```

### Signal Conditions

```yaml
# Simple threshold
queue_depth > 100

# Percentage
cpu_usage > 0.8

# Rate
error_rate > 0.05

# Complex
(latency_p95 > 500 AND error_rate > 0.01) OR queue_depth > 200

# Time-based
signal.age < duration("30s")
```

---

## TEMPLATE LIBRARY

### Template: Basic CRUD Service

```yaml
system:
  name: {ENTITY}Service
  
dataType:
  name: {ENTITY}
  version: v1
  fields:
    id: uuid
    created_at: timestamp
    updated_at: timestamp
    {custom_fields}: {types}

dataState:
  name: {ENTITY}Record
  type: {ENTITY}
  lifecycle: persistent
  constraintPolicy:
    enforcement: strict

inputIntent:
  name: Create{ENTITY}
  version: v1
  proposes: {dataState: {ENTITY}Record, version: v1}
  supplies:
    required: [{custom_fields}]
  transitions:
    to: {ENTITY}Record

inputIntent:
  name: Update{ENTITY}
  version: v1
  proposes: {dataState: {ENTITY}Record, version: v1}
  supplies:
    required: [id]
    optional: [{custom_fields}]
  transitions:
    to: {ENTITY}Record

wts:
  workers:
    - name: {ENTITY}Service
      acceptsInputs: 
        - Create{ENTITY} v1
        - Update{ENTITY} v1
      produces: [{ENTITY}Record v1]

sms:
  flows:
    - name: Create{ENTITY}Flow
      triggeredBy: {inputIntent: Create{ENTITY}}
      steps:
        - validate: constraints
        - persist: {ENTITY}Record
    
    - name: Update{ENTITY}Flow
      triggeredBy: {inputIntent: Update{ENTITY}}
      steps:
        - validate: constraints
        - persist: {ENTITY}Record
```

**Usage**: Replace `{ENTITY}` with your entity name, add custom fields

### Template: Policy for Resource

```yaml
policy:
  name: {ACTION}{RESOURCE}
  version: v1
  appliesTo:
    type: {target_type}
    name: {target_name}
  effect: allow
  when: |
    subject.role == "admin"
    OR (subject.role == "{resource_owner}" AND data.{owner_field} == subject.id)
```

**Usage**: Replace `{ACTION}`, `{RESOURCE}`, `{target_type}`, etc.

### Template: Scaling Policy

```yaml
scaling_policy:
  name: {worker}_autoscale
  target: {placement_name}
  evaluation_period: 10s
  cooldown: 30s
  signals:
    queue_depth > {scale_up_threshold} => scale_up
    latency_p95 > {latency_threshold}ms => scale_up
    queue_depth < {scale_down_threshold} => scale_down
    cpu_usage < 0.2 => scale_down
  limits:
    max_scale_up_rate: 10
    max_scale_down_rate: 5
```

**Usage**: Set thresholds based on your SLAs

---

## ERROR QUICK REFERENCE

### Common Validation Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `unknown_data_type` | Referenced DataType doesn't exist | Define DataType first |
| `version_skip` | Evolution skips version | Add intermediate version |
| `cross_boundary_atomic` | Atomic group spans boundaries | Use compensation instead |
| `missing_idempotency` | Transition without idempotency | Add idempotency spec |
| `duplicate_authority` | Multiple schedulers with authority=true in same scope | Set one to authority=false |
| `undefined_work_unit` | Flow references unknown work unit | Define work unit in worker |
| `invalid_compensation` | Compensation references undefined work unit | Define compensation work unit |

### Common Runtime Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `ErrBoundaryViolation` | Work unit executed in wrong boundary | Check routing logic |
| `ErrNoCompatibleVersion` | Version negotiation failed | Add version compatibility |
| `ErrPolicyDenied` | Authorization failed | Check policy conditions |
| `ErrIdempotencyViolation` | Duplicate processing detected | Idempotency key collision |
| `ErrNoWorkerAvailable` | No worker registered for work unit | Deploy worker |
| `ErrAtomicGroupFailed` | Step in atomic group failed | Check failure logs |
| `ErrCircuitOpen` | Circuit breaker tripped | Investigate downstream health |

---

## PERFORMANCE TARGETS

### Latency Budgets

| Operation | Target | Acceptable | Investigate |
|-----------|--------|------------|-------------|
| In-process invocation | <50μs | <100μs | >100μs |
| Local socket | <500μs | <1ms | >2ms |
| NATS invocation | <5ms | <10ms | >20ms |
| Policy evaluation | <100μs | <500μs | >1ms |
| Cache lookup | <10μs | <50μs | >100μs |
| Constraint validation | <1ms | <5ms | >10ms |
| Flow step transition | <50ms | <100ms | >200ms |

### Throughput Targets

| Component | Target | Scaling |
|-----------|--------|---------|
| Worker (per instance) | 100-1000 req/sec | Horizontal |
| Flow engine | 1000+ flows/sec | Horizontal |
| Policy evaluator | 10,000+ eval/sec | Vertical (CPU) |
| Signal aggregator | 100,000+ sig/sec | Horizontal |
| Routing cache | 1M+ lookups/sec | Vertical (RAM) |

### Resource Guidelines

| Component | CPU | Memory | Notes |
|-----------|-----|--------|-------|
| Worker | 0.5-2 cores | 512MB-2GB | Depends on work units |
| Flow engine | 1-4 cores | 1-4GB | State machine overhead |
| Control plane | 1-2 cores | 2-8GB | Artifact storage |
| Scheduler | 0.5-1 core | 512MB-1GB | Signal aggregation |

---

## NATS SUBJECT QUICK REFERENCE

### Subject Patterns

| Purpose | Pattern | Example |
|---------|---------|---------|
| Spec artifact | `spec.{domain}.{type}.{name}.{version}` | `spec.order.dataType.Order.v3` |
| Policy artifact | `policy.{domain}.{component}.{name}.{version}` | `policy.order.worker.OrderService.v2` |
| Intent execution | `intent.{domain}.{action}` | `intent.order.create` |
| Work execution | `work.{boundary}.{workunit}` | `work.inventory.reserve` |
| Worker signal | `signal.worker.{name}.{type}` | `signal.worker.OrderService.capacity` |
| SMS signal | `signal.sms.{flow}.{type}` | `signal.sms.CheckoutFlow.backpressure` |
| Policy signal | `signal.policy.{name}.{type}` | `signal.policy.ModifyOrder.shadow` |
| Lifecycle event | `lifecycle.{artifact}.{name}.{version}.{action}` | `lifecycle.policy.ModifyOrder.v2.promote` |
| Discovery (KV) | `worker.{name}.{instance}` | `worker.OrderService.abc123` |

### Subject Subscription Patterns

| Component | Subscribes To | Why |
|-----------|---------------|-----|
| Worker | `spec.{domain}.>`, `policy.{domain}.worker.{name}.>`, `intent.{domain}.>` | Get specs, policies, work |
| Flow Engine | `spec.{domain}.>`, `policy.{domain}.sms.>` | Get flow definitions, policies |
| Scheduler | `signal.worker.>`, `topology.>` | Get signals, topology changes |
| UI | `spec.{domain}.>`, `policy.{domain}.ui.>` | Get view definitions, policies |
| Control Plane | (none - publishes only) | Distributes artifacts |

---

## COMMON FIELD TYPES

### Primitive Types

| Type | Go | TypeScript | Example |
|------|-----|------------|---------|
| `string` | `string` | `string` | `"hello"` |
| `integer` | `int64` | `number` | `42` |
| `decimal` | `decimal.Decimal` | `Decimal` | `99.99` |
| `boolean` | `bool` | `boolean` | `true` |
| `uuid` | `uuid.UUID` | `string` | `"550e8400-..."` |
| `timestamp` | `time.Time` | `Date` | `"2024-01-15T14:23:45Z"` |
| `duration` | `time.Duration` | `number` (ms) | `"5m"` |

### Complex Types

```yaml
# Enum
status: enum[draft, active, archived]

# Array
tags: array<string>

# Reference
customer: Customer  # References Customer DataType

# Nested
address:
  street: string
  city: string
  country: string
```

---

## VERSION COMPATIBILITY MATRIX

### Read Compatibility

| From (stored) | To (code) | Backward | Forward |
|---------------|-----------|----------|---------|
| v1 | v2 | New code reads old data | ✅ |
| v2 | v1 | Old code reads new data | ✅ |

**Backward Read**: New code + old data
**Forward Read**: Old code + new data

### Write Compatibility

| From (code) | To (stored) | Backward | Forward |
|-------------|-------------|----------|---------|
| v1 | v2 | Old code writes, new code reads | ✅ |
| v2 | v1 | New code writes, old code reads | ✅ |

**Backward Write**: Old code writes → new code reads
**Forward Write**: New code writes → old code reads

### Deployment Strategies by Compatibility

| Read | Write | Strategy |
|------|-------|----------|
| Backward | Forward | Deploy code first, migrate data later |
| Forward | Backward | Migrate data first, deploy code later |
| Both | Both | Deploy anytime (ideal) |
| None | None | Requires downtime (avoid) |

---

## SIGNAL SEVERITY GUIDELINES

### When to Emit Each Severity

| Severity | Condition | Example | Action |
|----------|-----------|---------|--------|
| `info` | Normal operation | `queue_depth: 15` | Log only |
| `warn` | Approaching limits | `cpu_usage: 0.75` | Alert on-call |
| `error` | Threshold exceeded | `error_rate: 0.08` | Page on-call |
| `critical` | Service degraded | `all_workers_unhealthy` | Wake everyone |

### Signal Types by Component

| Component | Emits | Frequency | Consumers |
|-----------|-------|-----------|-----------|
| Worker | capacity, health, latency | 5s | Scheduler, monitoring |
| SMS | backpressure, flow_completion | on event | Scheduler, audit |
| Policy Engine | shadow_divergence | on divergence | Ops, control plane |
| Infrastructure | region_health, node_status | 30s | Schedulers |

---

## POLICY TEMPLATES

### Template: Owner-Based Access

```yaml
policy:
  name: {ACTION}{RESOURCE}
  version: v1
  appliesTo:
    type: dataState
    name: {RESOURCE}Record
  effect: allow
  when: |
    subject.role == "admin"
    OR data.owner_id == subject.id
```

### Template: Role-Based Access

```yaml
policy:
  name: {ACTION}{RESOURCE}
  version: v1
  appliesTo:
    type: inputIntent
    name: {ACTION}{RESOURCE}
  effect: allow
  when: subject.role in ["admin", "{role1}", "{role2}"]
```

### Template: Time-Based Access

```yaml
policy:
  name: BusinessHoursOnly
  version: v1
  appliesTo:
    type: smsFlow
    name: {FLOW}
  effect: allow
  when: |
    environment.hour >= 9 AND environment.hour < 17
    AND environment.day_of_week in ["Mon", "Tue", "Wed", "Thu", "Fri"]
```

### Template: Data-Sensitive Access

```yaml
dataPolicy:
  name: {RESOURCE}Access
  version: v1
  appliesTo:
    dataState: {RESOURCE}Record
    version: v1
  permissions:
    read:
      allow: subject.clearance_level >= data.classification_level
      filters: [pii, ssn]  # Always filter these
    write:
      allow: subject.role == "admin" OR data.owner_id == subject.id
    transition:
      - name: Approve
        allow: subject.role in ["admin", "approver"]
      - name: Reject
        allow: subject.role in ["admin", "approver", "owner"]
```

---

## TROUBLESHOOTING GUIDE

### Symptom: High Latency

**Check**:
1. Cache hit rate (should be >95%)
2. Local invocation rate (should be >80%)
3. NATS latency (should be <5ms)
4. Worker queue depth (should be <50)
5. Policy evaluation time (should be <100μs)

**Common Causes**:
- Cache cold/invalidated frequently
- Forced remote invocation
- Slow policy expressions
- Worker overload
- Network issues

### Symptom: Duplicate Processing

**Check**:
1. Idempotency keys defined?
2. Deduplication cache working?
3. Key collisions?
4. TTL too short?

**Common Causes**:
- Missing idempotency specification
- Non-deterministic keys
- Dedup cache not shared across instances
- TTL shorter than retry window

### Symptom: Policy Not Enforcing

**Check**:
1. Policy artifact lifecycle state (should be 'enforce')
2. Component subscribed to correct scope?
3. Policy loaded in registry?
4. Expression syntax correct?
5. Policy stream replay enabled?

**Common Causes**:
- Still in 'shadow' state
- Wrong scope (policy not delivered to component)
- Expression always evaluates to false
- Cache not updated after policy change

### Symptom: Worker Not Receiving Work

**Check**:
1. Worker registered in KV?
2. Heartbeat loop running?
3. Subscription to correct subject?
4. Version compatibility?
5. Routing cache updated?

**Common Causes**:
- Registration expired (TTL)
- Subject mismatch
- Version incompatible
- Marked unhealthy
- Policy prevents routing

### Symptom: Zero-Downtime Deployment Failed

**Check**:
1. Version compatibility declared?
2. Shadow write enabled?
3. Graceful drain working?
4. Routing preference updated?
5. Old version deprecated properly?

**Common Causes**:
- Incompatible schema change
- Skipped shadow phase
- Force-killed workers (not drained)
- Routing not version-aware
- Too-fast rollout

---

## SPECIFICATION VALIDATION CHECKLIST

### Structural Validation

- [ ] All DataTypes have unique names
- [ ] All versions use format `v{integer}`
- [ ] All references point to existing elements
- [ ] All expressions use valid syntax
- [ ] All required fields present

### Semantic Validation

- [ ] Evolution chains are linear (no skips)
- [ ] Atomic groups single boundary
- [ ] Compensations reference existing work units
- [ ] Policies have explicit effects
- [ ] Idempotency keys deterministic
- [ ] One authority per scope
- [ ] Min ≤ max for all placements
- [ ] Quorum ≤ step count

### Completeness Validation

- [ ] All InputIntents have transitions
- [ ] All DataStates have owners
- [ ] All Flows have authorization
- [ ] All Workers declare versions
- [ ] All Policies have scopes
- [ ] All Signals have types
- [ ] All Materializations have sources

---

## IMPLEMENTATION CHECKLIST

### Week 1: Bootstrap
- [ ] Set up project structure
- [ ] Choose parser library (or hand-write)
- [ ] Define AST types
- [ ] Implement lexer
- [ ] Implement parser
- [ ] Add validation rules
- [ ] Write unit tests

### Week 2: Core Runtime
- [ ] Define runtime interfaces
- [ ] Implement WorkUnit registry
- [ ] Implement FlowEngine (FSM)
- [ ] Implement constraint validator
- [ ] Add execution context
- [ ] Add basic observability
- [ ] Create single-binary demo

### Week 3: Control Plane
- [ ] Set up NATS (embedded or external)
- [ ] Create JetStream streams
- [ ] Implement artifact publishing
- [ ] Implement artifact loading
- [ ] Add versioning support
- [ ] Test artifact distribution

### Week 4: Workers
- [ ] Implement Worker runtime
- [ ] Add KV registration
- [ ] Implement heartbeat loop
- [ ] Add subscription management
- [ ] Implement work unit execution
- [ ] Add graceful shutdown

### Week 5: Policy
- [ ] Implement PolicyRegistry
- [ ] Add policy evaluation engine
- [ ] Implement shadow evaluation
- [ ] Add lifecycle handling
- [ ] Integrate with Workers/SMS/UI
- [ ] Add audit logging

### Week 6: Optimization
- [ ] Implement routing cache
- [ ] Add locality-first logic
- [ ] Optimize hot paths
- [ ] Add connection pooling
- [ ] Implement batching
- [ ] Performance testing

### Week 7: Signals
- [ ] Define signal types
- [ ] Implement signal emission
- [ ] Add signal aggregation
- [ ] Create scheduler integration
- [ ] Test autoscaling
- [ ] Add dashboards

### Week 8: Evolution
- [ ] Implement version negotiation
- [ ] Add shadow write support
- [ ] Implement gradual rollout
- [ ] Add rollback support
- [ ] Test zero-downtime deploy
- [ ] Document procedures

### Week 9: Multi-Region
- [ ] Regional deployment
- [ ] Cross-region policy sync
- [ ] Regional schedulers
- [ ] Failover testing
- [ ] Partition handling
- [ ] Chaos testing

### Week 10: Production Hardening
- [ ] Security audit
- [ ] Performance tuning
- [ ] Observability polish
- [ ] Runbook creation
- [ ] Incident procedures
- [ ] Documentation

---

## QUICK WINS

### Easy Optimizations

1. **Enable Routing Cache**: 100x speedup for lookups
2. **Use Unix Sockets**: 10x faster than HTTP for local calls
3. **Batch Cache Updates**: Reduce lock contention
4. **Compile Policy Expressions**: 50x faster evaluation
5. **Connection Pooling**: Reuse connections

### Common Mistakes to Avoid

1. ❌ Synchronous NATS requests in hot path
2. ❌ No routing cache
3. ❌ Storing state in workers
4. ❌ Skipping shadow validation
5. ❌ Manual cleanup processes
6. ❌ Ignoring graceful shutdown
7. ❌ Inferring missing semantics
8. ❌ Coupling workers directly

---

## ONE-PAGERS

### For Developers

**What**: System Mechanics Spec defines what your system does, not how it does it

**Benefits**:
- Zero-downtime upgrades
- Automatic scaling
- Built-in authorization
- End-to-end tracing
- Multi-region by default

**Cost**:
- Learn new grammar
- More upfront design
- Runtime complexity

**Start**: Use templates, evolve incrementally

### For Architects

**What**: Universal mechanics-level specification for distributed systems

**Replaces**:
- Architecture diagrams
- Service contracts
- Policy documents
- Scaling configurations

**Enables**:
- Code generation
- Automated validation
- Safe evolution
- Policy enforcement
- Dynamic optimization

**Decision**: Use for new systems or modernization efforts

### For Operators

**What**: Declarative specification of system behavior

**Operations Benefits**:
- No manual scaling
- Automatic failover
- Graceful deploys
- Self-healing
- Observable by default

**Requirements**:
- NATS cluster
- JetStream storage
- Scheduler integration (K8s/Nomad)
- Monitoring setup

**Maintenance**: GitOps-friendly, version controlled

---

## GLOSSARY OF ABBREVIATIONS

- **SMS**: State Machine System (flow orchestration)
- **WTS**: Worker Topology Specification (placement/scaling)
- **RBAC**: Role-Based Access Control
- **ABAC**: Attribute-Based Access Control
- **KV**: Key-Value (NATS KV bucket)
- **TTL**: Time To Live
- **FSM**: Finite State Machine
- **AST**: Abstract Syntax Tree
- **EBNF**: Extended Backus-Naur Form
- **SLA**: Service Level Agreement
- **P50/P95/P99**: Latency percentiles

---

## USEFUL COMMANDS

### Validation
```bash
# Validate specification
mechanics-validate system.yaml

# Check cross-references
mechanics-validate --check-refs system.yaml

# Validate against schema
mechanics-validate --schema v1.1 system.yaml
```

### Code Generation
```bash
# Generate Go types
mechanics-gen --lang go --out ./generated system.yaml

# Generate TypeScript
mechanics-gen --lang ts --out ./generated system.yaml

# Generate documentation
mechanics-gen --doc --format html system.yaml
```

### Runtime
```bash
# Run single binary
mechanics-runtime --spec system.yaml

# Run with NATS
mechanics-runtime --spec system.yaml --nats nats://localhost:4222

# Run in debug mode
mechanics-runtime --spec system.yaml --log-level debug
```

### Operations
```bash
# Promote policy
mechanics-ctl policy promote ModifyOrder v2

# Deprecate version
mechanics-ctl version deprecate Order v1

# Check worker health
mechanics-ctl workers list --health

# View signals
mechanics-ctl signals watch --worker OrderService
```

---

## ADDITIONAL RESOURCES

### Documentation
- Grammar Reference: Formal grammar definitions
- Grammar Examples: 40+ concrete examples
- Implementation Spec: Detailed implementation guidance
- Design Rationale: Why decisions were made
- Addendums: Critical missing pieces
- This Document: Quick reference

### Tools (Conceptual)
- mechanics-validate: Specification validator
- mechanics-gen: Code generator
- mechanics-runtime: Reference runtime
- mechanics-ctl: Operational CLI
- mechanics-viz: Visualization tool

### Community
- GitHub: (repository TBD)
- Discussions: Implementation questions
- RFCs: Specification evolution
- Examples: Real-world systems

---

## GETTING UNSTUCK

### "I don't know where to start"
→ Use Template: Basic CRUD Service
→ Follow Implementation Phases
→ Start with single binary

### "My specification won't validate"
→ Check Common Validation Errors table
→ Validate incrementally
→ Use examples as reference

### "Performance is poor"
→ Check Performance Targets
→ Enable routing cache
→ Use local invocation
→ Profile hot paths

### "Deployment failed"
→ Check Troubleshooting Guide
→ Verify shadow phase completed
→ Check compatibility declarations
→ Review rollback procedure

### "I need a feature not in spec"
→ Check if it's runtime concern (probably is)
→ Review Design Rationale for "why not"
→ Propose extension via RFC process

---

## UI COMPOSITION CHEAT SHEET (NEW in v1.4)

### Composition Structure

```yaml
compositions:
  - name: <WorkspaceName>
    version: v1
    primary: <PrimaryView>       # Required: main view
    navigation:                  # Navigation properties
      label: "{{entity.name}}"   # Dynamic label
      purpose: accounts          # Semantic category
      parent: ParentComposition  # Hierarchy parent
      param: entity_id           # URL parameter
      policy_set: AccessPolicy   # Authorization
      indicator:                 # Dynamic badge
        when: condition
        severity: warning
    related:                     # Related views
      - view: DetailView
        relationship: detail
        cardinality: many
```

### View Relationship Types Quick Reference

| Type | When to Use | Example |
|------|-------------|---------|
| `derived` | Computed from primary | Balance from Account |
| `detail` | Child collection | Order items |
| `contextual` | Related entity | Customer of Account |
| `action` | User-triggered form | Transfer form |
| `supplementary` | Additional info | Settings panel |
| `historical` | Audit/history | Activity log |

### Navigation Scope Quick Reference

| Scope | Where Visible |
|-------|---------------|
| `within_composition` | Only in parent composition's context |
| `always_visible` | Always in primary nav |
| `on_action` | Only when action triggered |

### Common Composition Patterns

**Master-Detail:**
```yaml
- name: OrderWorkspace
  primary: OrderDetailsView
  related:
    - view: OrderItemsView
      relationship: detail
      cardinality: many
```

**Dashboard with Widgets:**
```yaml
- name: Dashboard
  primary: MetricsView
  related:
    - view: AlertsWidget
      relationship: supplementary
    - view: RecentActivity
      relationship: supplementary
```

**Form with Context:**
```yaml
- name: TransferWorkspace
  primary: TransferFormView
  related:
    - view: AccountBalanceView
      relationship: derived
      required: true
```

---

## EXPERIENCE CHEAT SHEET (NEW in v1.4)

### Experience Structure

```yaml
experiences:
  - name: CustomerPortal
    version: v1
    entry_point: Dashboard          # First composition
    includes:                       # All included compositions
      - Dashboard
      - AccountList
      - AccountDetails
    policy_set: CustomerAccess      # Default policy
    unauthenticated:
      redirect_to: LoginPage        # Login redirect
    on_unauthorized: conceal        # Behavior on denied
```

### On_unauthorized Behaviors

| Behavior | Navigation | Content | Use Case |
|----------|------------|---------|----------|
| `conceal` | Hidden | N/A | Customer-facing |
| `indicate` | Visible, disabled | Hint | Admin tools |
| `redirect` | Visible, redirects | Target | SSO flows |
| `deny` | Visible | Error | Debug mode |

### Experience vs Composition

| Question | Answer |
|----------|--------|
| Groups views? | Composition |
| Groups compositions? | Experience |
| Has entry point? | Experience |
| Has primary view? | Composition |
| Defines user journey? | Experience |
| Defines workspace? | Composition |

### Common Experience Patterns

**Customer App:**
```yaml
- name: CustomerApp
  entry_point: Dashboard
  includes: [Dashboard, Accounts, Transactions, Profile]
  policy_set: CustomerAccess
  unauthenticated: {redirect_to: Login}
  on_unauthorized: conceal
```

**Admin Console:**
```yaml
- name: AdminConsole
  entry_point: AdminDashboard
  includes: [AdminDashboard, Users, Audit, Settings]
  policy_set: AdminAccess
  unauthenticated: {redirect_to: AdminLogin}
  on_unauthorized: indicate
```

---

## FIELD PERMISSIONS CHEAT SHEET (NEW in v1.4)

### Field Permission Structure

```yaml
fieldPermissions:
  - field: ssn
    visible:
      policy: ViewSensitiveData    # Policy-controlled
    mask:
      unless: subject.clearance >= 4
      type: partial
      reveal: last4
```

### Visibility Options

| Syntax | Meaning |
|--------|---------|
| `visible: true` | Always visible |
| `visible: false` | Never visible |
| `visible: {policy: PolicyName}` | Policy-controlled |

### Mask Types

| Type | Example Input | Example Output |
|------|---------------|----------------|
| `full` | `123-45-6789` | `***-**-****` |
| `partial` + `last4` | `123-45-6789` | `***-**-6789` |
| `partial` + `first3` | `123-45-6789` | `123-**-****` |
| `hash` | `123-45-6789` | `a1b2c3d4` |

### Common Permission Patterns

**PII Protection:**
```yaml
- field: ssn
  visible: {policy: ViewPII}
  mask: {unless: subject.role == "compliance", type: partial, reveal: last4}
```

**Admin-Only Field:**
```yaml
- field: internal_notes
  visible: {policy: AdminOnly}
```

**Always Masked Unless Authorized:**
```yaml
- field: account_number
  visible: true
  mask: {unless: subject.owns_account, type: partial, reveal: last4}
```

---

## NAVIGATION HIERARCHY CHEAT SHEET (NEW in v1.4)

### Building Hierarchy

Hierarchy is built from `parent` references:

```
Dashboard (no parent = root)
├── AccountList (parent: Dashboard)
│   └── AccountDetails (parent: AccountList)
│       └── TransactionDetails (parent: AccountDetails)
└── Settings (parent: Dashboard)
```

### Navigation Purpose Categories

| Purpose | Description | Typical Placement |
|---------|-------------|-------------------|
| `home` | Entry point | Primary nav, first |
| `accounts` | Account-related | Primary nav |
| `transactions` | Transaction-related | Primary nav |
| `customers` | Customer mgmt | Admin primary |
| `alerts` | Alerts/notifications | Admin primary |
| `compliance` | Compliance/KYC | Admin secondary |
| `audit` | Audit logs | Admin secondary |
| `reports` | Reporting | Admin secondary |
| `settings` | User settings | Utility nav |
| `help` | Help/support | Utility nav |
| `authentication` | Login/auth | Hidden |

### Indicator Severity Levels

| Severity | Use Case | Typical Style |
|----------|----------|---------------|
| `info` | Informational (count) | Blue badge |
| `warning` | Needs attention | Yellow badge |
| `error` | Problem exists | Red badge |
| `critical` | Urgent action | Red pulsing |

### Dynamic Label Templates

```yaml
navigation:
  label: "{{entity.name}}"              # From entity
  label: "Order #{{order.number}}"      # Formatted
  label: "{{account.type}} ****{{account.last4}}"  # Composite
```

---

## UI TROUBLESHOOTING (NEW in v1.4)

### "Composition not rendering"
→ Check primary view exists
→ Verify policy_set grants access
→ Check data source returns data
→ Verify view version matches

### "Navigation item missing"
→ Check composition in experience.includes
→ Verify on_unauthorized is not `conceal` for denied user
→ Check parent composition is accessible
→ Verify purpose is in rendered nav group

### "Field still visible when shouldn't be"
→ Check policy evaluation
→ Verify field name matches exactly
→ Check mask.unless condition
→ Clear permission cache

### "Indicator not updating"
→ Check indicator.when expression
→ Verify data context available
→ Check WebSocket/SSE connection
→ Verify polling interval

### "Related view not loading"
→ Check relationship basis exists
→ Verify data_scope expression
→ Check required vs optional
→ Verify cardinality matches data

---

## FORM INTENT BINDING CHEAT SHEET (NEW in v1.5)

### Form Binding Structure

```yaml
presentationView:
  name: <FormName>
  version: v1
  consumes:
    dataState: <ContextData>
  
  triggers:                           # Form intent binding
    intent: <InputIntent>
    binding: form | action | link
    
    field_mapping:
      - view_field: <field>
        supplies: <intent_field>
        source: context | input | computed
        required: true | false
        validation:
          constraint: <expression>
          message: <error_message>
    
    confirmation:
      required: true | false
      message: <template>
    
    on_success:
      action: navigate_back | navigate_to | stay | refresh
      target: <composition>            # If navigate_to
      notification:
        type: toast | inline | none
        message: <template>
    
    on_error:
      action: display_inline | display_modal
      show_field_errors: true
```

### Binding Types Quick Reference

|| Binding | When to Use | Example |
||---------|-------------|---------|
|| `form` | Multi-field user input | Transfer form, account creation |
|| `action` | Single-click operation | Confirm payment, mark read |
|| `link` | Navigation with data | View details, open document |

### Field Source Types

|| Source | Description | Example |
||--------|-------------|---------|
|| `context` | From session/composition | `session.account_id` |
|| `input` | User provides | Text field, dropdown, file upload |
|| `computed` | Derived from other fields | `source.balance - amount` |

### Success/Error Actions

|| Action | Behavior |
||--------|----------|
|| `navigate_back` | Return to previous screen |
|| `navigate_to` | Go to specific composition |
|| `stay` | Remain on current view |
|| `refresh` | Reload current data |
|| `close` | Close modal/drawer |
|| `display_inline` | Show errors in form |
|| `display_modal` | Show errors in popup |

### Common Form Patterns

**Simple Action Button:**
```yaml
triggers:
  intent: AcknowledgeAlert
  binding: action
  field_mapping:
    - view_field: alert_id
      supplies: alert_id
      source: context
  on_success:
    action: navigate_back
```

**Multi-Field Form:**
```yaml
triggers:
  intent: TransferFunds
  binding: form
  field_mapping:
    - view_field: from_account
      supplies: from_account_id
      source: context
      required: true
    - view_field: to_account
      supplies: to_account_id
      source: input
      required: true
    - view_field: amount
      supplies: amount
      source: input
      required: true
      validation:
        constraint: amount > 0 AND amount <= from_account.balance
        message: "Amount must be positive and not exceed balance"
  confirmation:
    required: true
    message: "Transfer ${{amount}} to {{to_account.name}}?"
  on_success:
    action: navigate_to
    target: TransactionConfirmation
```

**Computed Fields:**
```yaml
field_mapping:
  - view_field: subtotal
    supplies: line_items_total
    source: computed
    required: true
  - view_field: tax
    supplies: tax_amount
    source: computed
    required: true
  - view_field: total
    supplies: grand_total
    source: computed
    required: true
```

---

## ASSET MANAGEMENT QUICK REFERENCE (NEW in v1.5)

### Asset Field Declaration

```yaml
dataType:
  name: Document
  version: v1
  fields:
    id: uuid
    title: string
    
    file:
      type: asset
      asset:
        category: document | image | media | archive | generic
        
        constraints:
          max_size: "10MB"
          mime_types: ["application/pdf", "image/png"]
          extensions: [".pdf", ".png"]
        
        lifecycle:
          type: permanent | temporary | expiring
          expires_after: "7d"          # For expiring
        
        reference:
          style: by_reference          # or embedded
        
        access:
          read: public | signed | authenticated | policy
          signed_url_ttl: "1h"         # For signed
          policy: ViewDocumentPolicy   # For policy
        
        variants:                      # Optional transforms
          - name: thumbnail
            transform: resize
            params: { width: 200, height: 200, fit: cover }
```

### Asset Categories

|| Category | Use Case | Typical Mime Types |
||----------|----------|-------------------|
|| `image` | Photos, diagrams | image/png, image/jpeg, image/webp |
|| `document` | PDFs, Office docs | application/pdf, application/msword |
|| `media` | Audio, video | video/mp4, audio/mpeg |
|| `archive` | ZIP files, backups | application/zip, application/tar |
|| `generic` | Any binary file | application/octet-stream |

### Access Modes

|| Mode | Description | When to Use |
||------|-------------|-------------|
|| `public` | Anyone can access | Public images, marketing materials |
|| `signed` | Time-limited URL | Temporary access, downloads |
|| `authenticated` | Requires login | User documents, private files |
|| `policy` | Policy-evaluated | Sensitive data, permission-based |

### Asset Upload Pattern

```yaml
inputIntent:
  name: UploadDocument
  version: v1
  supplies:
    required: [title, file]
  
  asset_upload:
    field: file
    stage: pre_submit | inline | post_submit
    validation:
      scan_virus: true
      validate_mime: true
    
presentationView:
  name: DocumentUploadForm
  triggers:
    intent: UploadDocument
    binding: form
    field_mapping:
      - view_field: title
        supplies: title
        source: input
      - view_field: file_input
        supplies: file
        source: input
        required: true
```

### Asset Variants (Image Processing)

```yaml
variants:
  - name: thumbnail
    transform: resize
    params: { width: 128, height: 128, fit: cover }
  
  - name: large
    transform: resize
    params: { width: 1920, height: 1080, fit: inside }
  
  - name: webp
    transform: convert
    params: { target_format: "webp", quality: 85 }
```

### Progressive Loading Pattern

```yaml
presentationView:
  name: ImageGallery
  fields:
    - name: photos
      asset_loading:
        strategy: progressive
        initial: thumbnail          # Load first
        full: large                 # Load on demand
        preload: next_3             # Preload adjacent
```

---

## SEARCH & DISCOVERY CHEAT SHEET (NEW in v1.5)

### Searchable Field Declaration

```yaml
dataType:
  name: Product
  version: v1
  fields:
    title:
      type: string
      search:
        indexed: true
        strategy: full_text
        full_text:
          analyzer: standard
          stemming: enabled
          stop_words: enabled
        weight: 2.0                  # Higher relevance
        suggest:
          enabled: true
          type: completion
        highlight:
          enabled: true
          fragment_size: 150
    
    sku:
      type: string
      search:
        indexed: true
        strategy: exact              # Exact match only
        weight: 3.0                  # Highest relevance
    
    description:
      type: string
      search:
        indexed: true
        strategy: full_text
        weight: 1.0
    
    category:
      type: string
      search:
        indexed: true
        strategy: exact
        facet:
          enabled: true
          type: terms                # Category filter
    
    price:
      type: decimal
      search:
        indexed: true
        facet:
          enabled: true
          type: range
          ranges:
            - { from: 0, to: 25, label: "Under $25" }
            - { from: 25, to: 100, label: "$25-$100" }
            - { from: 100, label: "Over $100" }
```

### Search Strategies

|| Strategy | Description | Use Case |
||----------|-------------|----------|
|| `full_text` | Natural language search | Product descriptions, articles |
|| `exact` | Exact match only | SKU, email, order ID |
|| `prefix` | Prefix matching | Autocomplete, type-ahead |
|| `fuzzy` | Typo-tolerant | User-entered search terms |
|| `semantic` | Vector similarity | AI-powered recommendations |

### Search Index Materialization

```yaml
materialization:
  name: ProductSearchIndex
  source: ProductRecord
  targetState: ProductSearchState
  
  search_index:
    enabled: true
    
    document:
      root: Product
      include_related:
        - relationship: category
          fields: [name, path]
        - relationship: brand
          fields: [name]
    
    relevance:
      default_boost: 1.0
      boost_recent:
        enabled: true
        decay_function: exponential
        scale: "30d"
      boost_popular:
        enabled: true
        signal: view_count
        scale: linear
    
    settings:
      shards: 3
      replicas: 2
      refresh_interval: "5s"
```

### Faceted Search Pattern

```yaml
presentationView:
  name: ProductSearchResults
  version: v1
  
  search:
    query_field: search_text
    target_index: ProductSearchIndex
    
    facets:
      - field: category
        display_name: "Category"
        type: terms
        size: 10
      
      - field: price
        display_name: "Price Range"
        type: range
      
      - field: brand
        display_name: "Brand"
        type: terms
        size: 20
    
    sorting:
      default: relevance
      options:
        - name: relevance
          label: "Best Match"
        - name: price_asc
          label: "Price: Low to High"
          field: price
          order: asc
        - name: price_desc
          label: "Price: High to Low"
          field: price
          order: desc
        - name: newest
          label: "Newest First"
          field: created_at
          order: desc
```

### Autocomplete Pattern

```yaml
presentationView:
  name: SearchBar
  
  autocomplete:
    source: ProductSearchIndex
    field: title
    suggest_type: completion
    min_chars: 2
    max_suggestions: 5
    highlight_match: true
```

---

## SESSION CONTRACT CHEAT SHEET (NEW in v1.5)

### Session Declaration

```yaml
session:
  name: UserSession
  version: v1
  
  scope: global | realm | entity
  
  lifecycle:
    creation: explicit | implicit
    duration:
      idle_timeout: "30m"
      absolute_timeout: "8h"
    refresh:
      strategy: sliding | fixed
      extend_on_activity: true
  
  bindings:
    context:
      subject_id: session.actor.id
      account_id: session.data.active_account_id
      preferences: session.data.preferences
    
    storage:
      backend: stateful | stateless_jwt | hybrid
      encryption: required | optional
  
  data_schema:
    active_account_id: uuid
    preferences:
      theme: string
      language: string
    cart_items: array<uuid>
```

### Session Scope Types

|| Scope | Lifetime | Use Case |
||-------|----------|----------|
|| `global` | Across entire application | User authentication |
|| `realm` | Within single tenant/realm | Multi-tenant isolation |
|| `entity` | Bound to specific entity | Document editing session |

### Session Lifecycle Patterns

**Standard Web Session:**
```yaml
session:
  name: WebSession
  lifecycle:
    creation: explicit              # Login required
    duration:
      idle_timeout: "30m"
      absolute_timeout: "24h"
    refresh:
      strategy: sliding             # Extends on activity
      extend_on_activity: true
```

**Long-Lived API Session:**
```yaml
session:
  name: APISession
  lifecycle:
    creation: explicit
    duration:
      idle_timeout: "90d"           # Long-lived
      absolute_timeout: "365d"
    refresh:
      strategy: fixed               # No auto-extension
      extend_on_activity: false
```

**Temporary Guest Session:**
```yaml
session:
  name: GuestSession
  lifecycle:
    creation: implicit              # Auto-created
    duration:
      idle_timeout: "15m"
      absolute_timeout: "2h"
    refresh:
      strategy: sliding
```

### Session Storage Backends

|| Backend | Characteristics | Use Case |
||---------|----------------|----------|
|| `stateful` | Server-side storage | Traditional web apps |
|| `stateless_jwt` | Client-side JWT | Stateless APIs, microservices |
|| `hybrid` | Session ID + JWT claims | Scalable web apps |

### Session Context Binding

```yaml
composition:
  name: AccountDashboard
  
  session_context:
    required:
      - subject_id
      - active_account_id
    optional:
      - preferences
  
  bindings:
    - composition_param: account_id
      session_field: session.data.active_account_id
```

---

## SCHEDULED TRIGGERS & NOTIFICATIONS (NEW in v1.5)

### Scheduled Trigger Declaration

```yaml
scheduledTrigger:
  name: DailyReportGeneration
  version: v1
  
  schedule:
    type: cron | interval | calendar
    expression: "0 2 * * *"         # Daily at 2 AM
    timezone: "America/New_York"
  
  triggers:
    intent: GenerateDailyReport
    version: v1
  
  supplies:
    date: "{{ execution_date }}"
    type: "daily"
  
  idempotency:
    key: ["date", "type"]
    scope: global
  
  execution:
    concurrency: forbid | allow | replace
    timeout: "30m"
    retry:
      max_attempts: 3
      backoff: exponential
```

### Schedule Types

|| Type | Expression Format | Example |
||------|------------------|---------|
|| `cron` | Standard cron syntax | `0 2 * * *` (2 AM daily) |
|| `interval` | Duration between runs | `every 15m` |
|| `calendar` | Business calendar | `weekdays at 9am` |

### Concurrency Control

|| Mode | Behavior |
||------|----------|
|| `forbid` | Skip if previous still running |
|| `allow` | Run concurrently |
|| `replace` | Cancel previous, start new |

### Notification Channel Declaration

```yaml
notificationChannel:
  name: UserAlerts
  version: v1
  
  delivery:
    type: email | sms | push | webhook | in_app
    
    email:
      from: "alerts@example.com"
      template: AlertEmailTemplate
    
    routing:
      strategy: preference_based
      fallback: [email, in_app]
  
  batching:
    enabled: true
    window: "5m"
    max_batch_size: 10
  
  rate_limiting:
    per_user: "10/hour"
    per_channel: "1000/minute"
  
  retry:
    max_attempts: 3
    backoff: exponential
```

### Notification Pattern

```yaml
# Trigger notification from flow
sms:
  flows:
    - name: ProcessPayment
      steps:
        - work_unit: charge_card
        - notify:
            channel: UserAlerts
            template: PaymentSuccess
            recipients:
              - "{{ payment.user_id }}"
            data:
              amount: "{{ payment.amount }}"
              card_last4: "{{ payment.card_last4 }}"
```

### Common Scheduled Patterns

**Daily Batch Job:**
```yaml
scheduledTrigger:
  name: NightlyBatch
  schedule:
    type: cron
    expression: "0 1 * * *"
  triggers:
    intent: RunNightlyBatch
  execution:
    concurrency: forbid
    timeout: "2h"
```

**Periodic Cleanup:**
```yaml
scheduledTrigger:
  name: CleanupExpiredSessions
  schedule:
    type: interval
    expression: "every 1h"
  triggers:
    intent: CleanupSessions
  execution:
    concurrency: allow
```

**Business Hours Alert:**
```yaml
scheduledTrigger:
  name: BusinessHoursCheck
  schedule:
    type: calendar
    expression: "weekdays at 9am, 2pm, 5pm"
    timezone: "America/New_York"
  triggers:
    intent: CheckSystemHealth
```

---

## WORKFLOW & COLLABORATION PATTERNS (NEW in v1.5)

### Durable Workflow Declaration

```yaml
durableWorkflow:
  name: OrderFulfillment
  version: v1
  
  state_machine:
    initial: pending_payment
    
    states:
      pending_payment:
        on_enter: [notify_customer]
        transitions:
          - event: payment_confirmed
            to: preparing_shipment
          - event: payment_failed
            to: payment_failed
      
      preparing_shipment:
        on_enter: [allocate_inventory]
        transitions:
          - event: shipment_ready
            to: shipped
          - event: out_of_stock
            to: awaiting_restock
      
      shipped:
        on_enter: [send_tracking]
        transitions:
          - event: delivered
            to: completed
      
      completed:
        terminal: true
      
      payment_failed:
        terminal: true
  
  compensation:
    on_failure:
      - state: preparing_shipment
        compensate: release_inventory
      - state: shipped
        compensate: initiate_return
  
  persistence:
    checkpoint_strategy: after_each_state
    retention: "90d"
  
  timeout:
    per_state:
      pending_payment: "24h"
      preparing_shipment: "48h"
      shipped: "14d"
```

### Collaborative Session Pattern

```yaml
collaborativeSession:
  name: DocumentEditing
  version: v1
  
  scope:
    entity_type: Document
    entity_id_param: document_id
  
  participants:
    max_concurrent: 10
    join_policy: PolicyName
    roles: [editor, viewer, commenter]
  
  synchronization:
    strategy: operational_transform | crdt | lock_based
    
    conflict_resolution:
      strategy: last_write_wins | merge | manual
  
  awareness:
    share_cursor: true
    share_selection: true
    share_presence: true
  
  persistence:
    save_strategy: periodic | on_idle | manual
    save_interval: "30s"
    checkpoint_interval: "5m"
```

### Workflow State Pattern

```yaml
# Simple approval workflow
durableWorkflow:
  name: ExpenseApproval
  state_machine:
    initial: pending_review
    
    states:
      pending_review:
        transitions:
          - event: manager_approved
            to: pending_finance
            condition: amount < 1000
          - event: manager_approved
            to: pending_director
            condition: amount >= 1000
          - event: rejected
            to: rejected
      
      pending_director:
        transitions:
          - event: director_approved
            to: pending_finance
          - event: rejected
            to: rejected
      
      pending_finance:
        transitions:
          - event: finance_approved
            to: approved
          - event: rejected
            to: rejected
      
      approved:
        terminal: true
        on_enter: [process_payment, notify_requester]
      
      rejected:
        terminal: true
        on_enter: [notify_requester]
```

### Collaborative Editing Pattern

```yaml
composition:
  name: DocumentEditor
  
  collaborative_session:
    type: DocumentEditing
    entity_binding: document_id
    
    presence:
      show_avatars: true
      show_cursors: true
      timeout: "30s"
    
    actions:
      - name: save
        triggers:
          intent: SaveDocument
        debounce: "2s"
      
      - name: comment
        triggers:
          intent: AddComment
        scope: selection
```

---

## ADVANCED DATA TYPES CHEAT SHEET (NEW in v1.5)

### Spatial Type Declaration

```yaml
dataType:
  name: Location
  version: v1
  fields:
    id: uuid
    name: string
    
    coordinates:
      type: spatial
      spatial:
        geometry: point | linestring | polygon | multipoint
        srid: 4326                    # Coordinate system
        
        constraints:
          within: BoundaryPolygon     # Geographic constraint
          max_distance_from: CenterPoint
          distance: "100km"
        
        indexing:
          type: rtree | geohash | s2
          precision: 10
```

### Spatial Geometry Types

|| Geometry | Description | Use Case |
||----------|-------------|----------|
|| `point` | Single coordinate | Store location, POI |
|| `linestring` | Sequence of points | Route, path, boundary |
|| `polygon` | Closed area | Delivery zone, region |
|| `multipoint` | Multiple points | Store locations, stops |

### Spatial Query Pattern

```yaml
materialization:
  name: NearbyStores
  source: StoreLocation
  
  spatial_query:
    type: proximity
    
    proximity:
      reference_point: "{{ session.user_location }}"
      radius: "10km"
      sort_by: distance
      limit: 20
```

### Structured Content Type

```yaml
dataType:
  name: Article
  version: v1
  fields:
    id: uuid
    title: string
    
    content:
      type: structured_content
      structured_content:
        format: prosemirror | slate | markdown | html
        
        allowed_blocks:
          - heading
          - paragraph
          - list
          - blockquote
          - code_block
          - image
          - table
        
        allowed_marks:
          - bold
          - italic
          - link
          - code
        
        validation:
          max_depth: 10
          max_length: 50000
        
        asset_handling:
          inline_images: true
          max_image_size: "5MB"
```

### Inference-Derived Fields

```yaml
dataType:
  name: CustomerProfile
  version: v1
  fields:
    id: uuid
    email: string
    
    predicted_churn_risk:
      type: decimal
      inference:
        model: ChurnPredictionModel
        version: v3
        inputs: [activity_score, tenure_days, support_tickets]
        refresh_policy:
          strategy: on_read | scheduled | on_write
          max_staleness: "24h"
        confidence_threshold: 0.75
```

### Structured Content Pattern

```yaml
presentationView:
  name: ArticleEditor
  
  fields:
    - name: content
      type: rich_text_editor
      
      editor_config:
        format: prosemirror
        toolbar: [bold, italic, link, heading, list]
        
        plugins:
          - name: mentions
            trigger: "@"
            source: UserSearchIndex
          
          - name: slash_commands
            trigger: "/"
            commands: [heading, list, quote, code]
        
        validation:
          real_time: true
          constraint: content.length <= 50000
```

---

## EXTERNAL INTEGRATION & GOVERNANCE (NEW in v1.5)

### External Integration Declaration

```yaml
externalIntegration:
  name: StripePayments
  version: v1
  
  provider:
    type: rest_api | graphql | grpc | soap | custom
    base_url: "https://api.stripe.com/v1"
  
  authentication:
    type: api_key | oauth2 | jwt | mutual_tls
    credentials_source: vault | env | kv
    credential_ref: "stripe.api_key"
  
  operations:
    - name: create_payment
      method: POST
      path: "/charges"
      
      mapping:
        request:
          - source_field: amount
            target_field: amount
            transform: cents_conversion
          - source_field: currency
            target_field: currency
        
        response:
          - source_field: id
            target_field: transaction_id
          - source_field: status
            target_field: payment_status
      
      retry:
        max_attempts: 3
        backoff: exponential
        idempotency_key: "{{ payment_id }}"
      
      circuit_breaker:
        enabled: true
        failure_threshold: 5
        timeout: "30s"
```

### Integration Patterns

**REST API Integration:**
```yaml
externalIntegration:
  name: WeatherAPI
  provider:
    type: rest_api
    base_url: "https://api.weather.com"
  
  operations:
    - name: get_forecast
      method: GET
      path: "/forecast/{{location}}"
      
      caching:
        enabled: true
        ttl: "1h"
        key: ["location", "date"]
```

**Event-Driven Integration:**
```yaml
externalIntegration:
  name: KafkaEvents
  provider:
    type: custom
    protocol: kafka
  
  events:
    - name: order_created
      topic: "orders.created"
      triggers:
        intent: ProcessExternalOrder
      
      schema_validation:
        enabled: true
        schema_registry: ConfluentSchemaRegistry
```

### Data Governance Declaration

```yaml
dataGovernance:
  name: CustomerDataGovernance
  version: v1
  
  classification:
    levels:
      - name: public
        label: "Public"
      - name: internal
        label: "Internal Use Only"
      - name: confidential
        label: "Confidential"
      - name: restricted
        label: "Restricted - PII"
  
  policies:
    - classification: restricted
      
      retention:
        duration: "7 years"
        action: delete | anonymize | archive
      
      access_controls:
        require_mfa: true
        require_justification: true
        audit_all_access: true
      
      encryption:
        at_rest: required
        in_transit: required
        key_rotation: "90d"
      
      data_residency:
        allowed_regions: [us-east, us-west]
        prohibited_regions: [*]
        cross_border_transfer: prohibited
```

### Field-Level Classification

```yaml
dataType:
  name: Customer
  version: v1
  fields:
    id: uuid
    
    email:
      type: string
      governance:
        classification: restricted
        pii: true
        retention: "7 years"
    
    ssn:
      type: string
      governance:
        classification: restricted
        pii: true
        sensitive: true
        encryption: required
        masking: required
```

### Compliance Pattern

```yaml
dataGovernance:
  name: GDPRCompliance
  
  subject_rights:
    - right: access
      intent: RequestDataAccess
      response_time: "30d"
    
    - right: rectification
      intent: RequestDataCorrection
      response_time: "30d"
    
    - right: erasure
      intent: RequestDataDeletion
      response_time: "30d"
      
    - right: portability
      intent: RequestDataExport
      response_time: "30d"
      format: json | csv | xml
  
  consent_management:
    required_for: [marketing, analytics, third_party_sharing]
    
    consent_tracking:
      store_proof: true
      retention: "7 years"
```

---

## v1.5 DECISION TREES

### Which Asset Lifecycle?

```
Is the asset permanent or temporary?
├─ Permanent → Use 'permanent'
│   └─ Follows entity lifecycle
└─ Temporary → How long needed?
    ├─ Until upload completes → Use 'temporary'
    └─ Fixed duration → Use 'expiring'
        └─ Set expires_after duration
```

### Which Search Strategy?

```
What are you searching?
├─ Natural language text → Use 'full_text'
│   └─ Enable stemming, stop words
├─ Exact identifiers (SKU, ID) → Use 'exact'
├─ Autocomplete/type-ahead → Use 'prefix'
├─ User input with typos → Use 'fuzzy'
│   └─ Set max_edits: 1 or 2
└─ AI/semantic similarity → Use 'semantic'
    └─ Requires embeddings
```

### Which Session Storage?

```
What's your scaling strategy?
├─ Traditional monolith → Use 'stateful'
│   └─ Redis/Memcached session store
├─ Stateless microservices → Use 'stateless_jwt'
│   └─ Everything in JWT claims
└─ Hybrid/scalable web app → Use 'hybrid'
    └─ Session ID + essential claims in JWT
```

### Which Notification Channel?

```
What's the message urgency?
├─ Critical/immediate → SMS or Push
├─ Important but not urgent → Email
├─ Informational → In-app notification
└─ Depends on user preference?
    └─ Use preference-based routing
        └─ Define fallback chain
```

---

## v1.5 COMMON PATTERNS

### Pattern: File Upload with Validation

```yaml
# 1. Define asset field
dataType:
  name: Document
  fields:
    file:
      type: asset
      asset:
        category: document
        constraints:
          max_size: "10MB"
          mime_types: ["application/pdf"]

# 2. Define upload intent
inputIntent:
  name: UploadDocument
  supplies:
    required: [title, file]
  
  asset_upload:
    field: file
    validation:
      scan_virus: true
      validate_mime: true

# 3. Define form
presentationView:
  name: DocumentUploadForm
  triggers:
    intent: UploadDocument
    binding: form
    field_mapping:
      - view_field: file_input
        supplies: file
        source: input
```

### Pattern: Searchable Product Catalog

```yaml
# 1. Define searchable fields
dataType:
  name: Product
  fields:
    title:
      type: string
      search:
        indexed: true
        strategy: full_text
        weight: 2.0
    
    category:
      type: string
      search:
        indexed: true
        facet:
          enabled: true
          type: terms

# 2. Create search index
materialization:
  name: ProductSearchIndex
  source: ProductRecord
  search_index:
    enabled: true

# 3. Search results view
presentationView:
  name: ProductSearch
  search:
    target_index: ProductSearchIndex
    facets:
      - field: category
      - field: price
```

### Pattern: Scheduled Report Generation

```yaml
# 1. Define trigger
scheduledTrigger:
  name: DailyReport
  schedule:
    type: cron
    expression: "0 6 * * *"
  triggers:
    intent: GenerateReport

# 2. Define notification
notificationChannel:
  name: ReportDelivery
  delivery:
    type: email

# 3. Link in flow
sms:
  flows:
    - name: GenerateReportFlow
      steps:
        - work_unit: generate_report
        - notify:
            channel: ReportDelivery
            template: ReportReady
```

### Pattern: Collaborative Document Editing

```yaml
# 1. Define session
collaborativeSession:
  name: DocEditing
  scope:
    entity_type: Document
  participants:
    max_concurrent: 10
  synchronization:
    strategy: operational_transform

# 2. Define composition
composition:
  name: DocumentEditor
  collaborative_session:
    type: DocEditing
    entity_binding: document_id

# 3. Define structured content
dataType:
  name: Document
  fields:
    content:
      type: structured_content
      structured_content:
        format: prosemirror
```

### Pattern: External Payment Integration

```yaml
# 1. Define integration
externalIntegration:
  name: PaymentGateway
  operations:
    - name: process_payment
      method: POST
      retry:
        max_attempts: 3
        idempotency_key: "{{ payment_id }}"

# 2. Use in flow
sms:
  flows:
    - name: CheckoutFlow
      steps:
        - external_call:
            integration: PaymentGateway
            operation: process_payment
        - persist: OrderRecord
```

---

## CONTACT & SUPPORT

### Questions About
- **Grammar**: See Grammar Reference + Examples
- **Implementation**: See Implementation Spec
- **Why**: See Design Rationale
- **How**: See this Quick Reference

### Reporting Issues
- Grammar ambiguities
- Validation false positives
- Missing examples
- Documentation errors

### Contributing
- Additional examples
- Implementation patterns
- Tool integrations
- Case studies

---

**Version**: 1.5
**Last Updated**: 2026-01-18
**Status**: Production Ready

This quick reference is maintained alongside the full specification and updated with each release.

## v1.5 Feature Summary

**Forms & Interaction**: Form intent binding with field mapping, validation, and submission flows  
**Assets**: Binary file handling with lifecycle, variants, and access control  
**Search**: Full-text search, faceting, autocomplete, and relevance tuning  
**Sessions**: Stateful/stateless session contracts with lifecycle management  
**Scheduling**: Cron triggers, scheduled jobs, and notification channels  
**Workflows**: Durable workflows with state machines and compensation  
**Collaboration**: Real-time collaborative sessions with presence and synchronization  
**Advanced Types**: Spatial data, structured content, inference-derived fields  
**Integration**: External system integration with retry, circuit breaker, and mapping  
**Governance**: Data classification, retention, encryption, and compliance
