# System Mechanics Specification - Quick Reference v1.4

## Overview

This document provides quick-reference cheat sheets, decision trees, and templates for common scenarios. Use this for rapid lookup during implementation.

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

**Version**: 1.4
**Last Updated**: 2026-01-11
**Status**: Production Ready

This quick reference is maintained alongside the full specification and updated with each release.

