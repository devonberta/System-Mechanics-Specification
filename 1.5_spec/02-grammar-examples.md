# System Mechanics Specification - Grammar Examples v1.5

## Overview

This document provides comprehensive examples for all grammar elements defined in the System Mechanics Specification. Examples progress from simple to complex, showing real-world usage patterns.

---

## DATA EXAMPLES

### Example 1: Simple DataType

```yaml
dataType:
  name: Order
  version: v1
  fields:
    id: uuid
    customer_id: uuid
    total: decimal
    status: enum[draft, paid, fulfilled]
```

**Use Case**: Basic e-commerce order

---

### Example 2: Evolved DataType (v2 → v3)

**Version 2:**
```yaml
dataType:
  name: Order
  version: v2
  fields:
    id: uuid
    customer_id: uuid
    subtotal: decimal
    tax: decimal
    discount: decimal
    total: decimal
    status: enum[draft, paid, fulfilled]
```

**Version 3 (with partial payments):**
```yaml
dataType:
  name: Order
  version: v3
  fields:
    id: uuid
    customer_id: uuid
    subtotal: decimal
    tax: decimal
    discount: decimal
    total: decimal
    paid_amount: decimal
    status: enum[draft, partially_paid, paid, fulfilled]
```

---

### Example 3: Constraint Across Versions

```yaml
constraint:
  name: order_total
  appliesTo: Order
  rules:
    v2: total == subtotal + tax - discount
    v3: total == subtotal + tax - discount
```

```yaml
constraint:
  name: payment_validity
  appliesTo: Order
  rules:
    v3: paid_amount <= total
```

**Key Point**: New constraint added in v3 for paid_amount

---

### Example 4: DataState with Lifecycle

**Persistent DataState:**
```yaml
dataState:
  name: OrderRecord
  type: Order
  lifecycle: persistent
  constraintPolicy:
    enforcement: strict
  evolutionPolicy:
    allowed:
      - additive
      - constraint_refinement
    forbidden:
      - destructive
      - semantic_break
```

**Intermediate DataState:**
```yaml
dataState:
  name: OrderDraft
  type: Order
  lifecycle: intermediate
  constraintPolicy:
    enforcement: deferred
  evolutionPolicy:
    allowed:
      - any
    forbidden: []
```

**Materialized DataState:**
```yaml
dataState:
  name: CustomerOrderSummary
  type: OrderSummary
  lifecycle: materialized
  constraintPolicy:
    enforcement: none
  evolutionPolicy:
    allowed:
      - rebuild
    forbidden: []
```

---

### Example 5: DataEvolution Declaration

```yaml
dataEvolution:
  name: Order_v2_to_v3
  from: { type: Order, version: v2 }
  to:   { type: Order, version: v3 }
  changes:
    - addField: 
        name: paid_amount
        default: 0
    - extendEnum:
        field: status
        values: [partially_paid]
  compatibility:
    read: backward
    write: forward
```

**Meaning**:
- Old workers can read new data (backward read)
- New workers can write, old workers forward to new (forward write)

---

### Example 6: Materialization

```yaml
dataState:
  name: CustomerOrderSummary
  version: v2
  lifecycle: materialized
  fields:
    customer_id: uuid
    order_id: uuid
    total: decimal
    paid_amount: decimal
    status: string

materialization:
  name: CustomerOrderSummaryView
  source: [OrderRecord, CustomerRecord]
  targetState: CustomerOrderSummary
  freshness:
    maxStaleness: 5m
  evolution:
    strategy: incremental
    blocking: false
```

**Use Case**: Real-time dashboard data

---

### Example 7: Complex Transition

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
```

**Key Features**:
- Self-transition (same DataState)
- Idempotency key includes both payment and order
- Global scope prevents duplicate charges across regions

---

## PRESENTATION (UI) EXAMPLES

### Example 8: PresentationView

```yaml
presentationView:
  name: CustomerOrderList
  version: v2
  consumes:
    dataState: CustomerOrderSummary
    version: v2
  guarantees:
    - show_total
    - show_payment_status
  interactionModel:
    badges:
      - if: paid_amount < total
        label: "Partially Paid"
        color: warning
      - if: status == "fulfilled"
        label: "Complete"
        color: success
    actions:
      - if: paid_amount < total
        action: add_payment
        label: "Add Payment"
```

**Key Features**:
- Binds to specific materialized view version
- Conditional UI affordances based on data state
- Declarative interaction model

---

### Example 9: PresentationBinding

```yaml
presentationBinding:
  name: CustomerOrderListBinding
  rules:
    - if: dataState(CustomerOrderSummary).version == v2
      use: CustomerOrderList v2
    - if: dataState(CustomerOrderSummary).version == v1
      use: CustomerOrderList v1
    - else:
      use: CustomerOrderListFallback
```

**Use Case**: Version-aware UI selection during migration

---

## INPUT (FORM) EXAMPLES

### Example 10: InputIntent (Create)

```yaml
inputIntent:
  name: CreateOrder
  version: v2
  proposes:
    dataState: OrderDraft
    version: v3
  supplies:
    required:
      - customer_id
      - items
      - subtotal
    optional:
      - discount
      - notes
  constraints:
    guarantees:
      - subtotal >= 0
      - items.length > 0
    defers:
      - order_total
      - tax
  transitions:
    to: OrderRecord
```

**Key Features**:
- Defers tax calculation to workflow
- Guarantees basic validation upfront
- Transitions to persistent state

---

### Example 11: InputIntent (Update)

```yaml
inputIntent:
  name: SubmitPayment
  version: v1
  proposes:
    dataState: OrderRecord
    version: v3
  supplies:
    required:
      - order_id
      - payment_amount
      - payment_method
    optional:
      - notes
  constraints:
    guarantees:
      - payment_amount > 0
      - payment_method in [card, bank, crypto]
    defers:
      - payment_processing
  transitions:
    to: OrderRecord
```

**Use Case**: Partial payment on existing order

---

## WORK UNIT CONTRACT EXAMPLES (NEW in v1.2)

### Example 41: Simple WorkUnitContract

```yaml
workUnitContract:
  name: validate_order
  version: v1
  description: "Validates order data against business rules"
  
  input:
    schema:
      order_id: uuid
      items: array<{sku: string, quantity: integer}>
      customer_id: uuid
    from: OrderDraft
  
  output:
    schema:
      valid: boolean
      errors: array<{field: string, message: string}>
    cardinality: one
  
  preconditions:
    - "order_id != null"
    - "len(items) > 0"
  
  postconditions:
    - "valid == true OR len(errors) > 0"
  
  errors:
    - code: VALIDATION_FAILED
      retryable: false
      message: "Order validation failed"
      category: validation
  
  timeout: 5s
```

**Use Case**: Basic order validation with explicit contracts

---

### Example 42: WorkUnitContract with Side Effects

```yaml
workUnitContract:
  name: reserve_inventory
  version: v1
  description: "Reserves inventory items for an order"
  
  input:
    schema:
      order_id: uuid
      items: array<{sku: string, quantity: integer}>
    from: OrderDraft
  
  output:
    schema:
      reservation_id: uuid
      reserved_items: array<{sku: string, quantity: integer, warehouse: string}>
      expires_at: timestamp
    to: InventoryReservation
  
  sideEffects:
    - type: database_write
      target: inventory_reservations
      description: "Creates reservation records"
      idempotency: reservation_id
      reversible: true
    - type: event_publish
      target: inventory.reserved
      description: "Notifies downstream systems of reservation"
  
  preconditions:
    - "all(items, i => i.quantity > 0)"
    - "order_exists(order_id)"
  
  postconditions:
    - "reservation_id != null"
    - "len(reserved_items) > 0"
    - "expires_at > now()"
  
  errors:
    - code: INSUFFICIENT_STOCK
      retryable: false
      message: "Not enough stock for {sku}"
      category: conflict
    - code: WAREHOUSE_UNAVAILABLE
      retryable: true
      backoff:
        strategy: exponential
        initial: 100ms
        multiplier: 2
        max: 10s
      maxRetries: 5
      category: unavailable
  
  dependencies:
    databases:
      - name: inventory_db
        required: true
        timeout: 2s
    services:
      - name: warehouse_service
        required: true
        timeout: 5s
        fallback: cache
    caches:
      - name: stock_levels_cache
        required: false
        fallback: skip
  
  timeout: 10s
  
  idempotency:
    strategy: deterministic_key
    key: [order_id, reservation_attempt_id]
    scope: global
    ttl: 24h
```

**Key Features**:
- Explicit side effects with idempotency
- Retryable errors with exponential backoff
- Dependency declarations with fallbacks
- Preconditions and postconditions

---

### Example 43: WorkUnitContract for External API Call

```yaml
workUnitContract:
  name: charge_payment
  version: v2
  description: "Charges payment via external payment processor"
  
  input:
    schema:
      order_id: uuid
      amount: decimal
      currency: enum[USD, EUR, GBP]
      payment_method:
        type: enum[card, bank_transfer, wallet]
        token: string
    from: PaymentIntent
  
  output:
    schema:
      transaction_id: string
      status: enum[success, pending, failed]
      processor_reference: string
      charged_at: timestamp
    to: PaymentRecord
  
  sideEffects:
    - type: api_call
      target: payment_processor_api
      description: "Calls external payment processor"
    - type: database_write
      target: payment_transactions
      idempotency: idempotency_key
    - type: event_publish
      target: payments.charged
    - type: audit_log
      target: payment_audit
      description: "Records payment attempt for compliance"
  
  preconditions:
    - "amount > 0"
    - "payment_method.token != null"
  
  postconditions:
    - "transaction_id != null"
    - "status in [success, pending, failed]"
  
  errors:
    - code: PAYMENT_DECLINED
      retryable: false
      httpStatus: 402
      message: "Payment was declined by processor"
      category: validation
    - code: INSUFFICIENT_FUNDS
      retryable: false
      httpStatus: 402
      category: validation
    - code: PROCESSOR_TIMEOUT
      retryable: true
      httpStatus: 504
      backoff:
        strategy: jittered
        initial: 500ms
        max: 30s
      maxRetries: 3
      category: timeout
    - code: PROCESSOR_UNAVAILABLE
      retryable: true
      httpStatus: 503
      backoff:
        strategy: exponential
        initial: 1s
        max: 60s
      maxRetries: 5
      category: external_dependency
  
  dependencies:
    externalApis:
      - name: payment_processor
        required: true
        timeout: 30s
        fallback: error
    databases:
      - name: payments_db
        required: true
  
  timeout: 45s
  
  idempotency:
    strategy: deterministic_key
    key: [order_id, payment_attempt_id]
    scope: global
    ttl: 7d
```

**Key Features**:
- External API dependency with timeout
- Multiple error codes with HTTP status mapping
- Jittered backoff for distributed retry safety
- Audit logging for compliance
- Extended TTL for idempotency

---

### Example 44: WorkUnitContract with Compensation

```yaml
workUnitContract:
  name: release_inventory
  version: v1
  description: "Releases previously reserved inventory (compensation)"
  
  input:
    schema:
      reservation_id: uuid
      reason: enum[order_cancelled, payment_failed, expired, manual_release]
    from: InventoryReservation
  
  output:
    schema:
      released: boolean
      released_items: array<{sku: string, quantity: integer}>
      released_at: timestamp
  
  sideEffects:
    - type: database_write
      target: inventory_reservations
      description: "Marks reservation as released"
      idempotency: reservation_id
    - type: database_write
      target: inventory_levels
      description: "Restores available inventory"
      idempotency: release_transaction_id
    - type: event_publish
      target: inventory.released
  
  preconditions:
    - "reservation_exists(reservation_id)"
  
  postconditions:
    - "released == true"
  
  errors:
    - code: RESERVATION_NOT_FOUND
      retryable: false
      category: not_found
    - code: ALREADY_RELEASED
      retryable: false
      message: "Reservation was already released"
      category: conflict
    - code: DATABASE_ERROR
      retryable: true
      backoff: exponential
      category: internal
  
  idempotency:
    strategy: deterministic_key
    key: [reservation_id, reason]
    scope: global
    ttl: 30d
```

**Use Case**: Compensation work unit for saga pattern

---

### Example 45: WorkUnitContract Coverage in Flow

This example shows how contracts relate to SMS flows:

**Contracts:**
```yaml
workUnitContract:
  name: validate_order
  version: v1
  # ... as defined in Example 41

workUnitContract:
  name: reserve_inventory
  version: v1
  # ... as defined in Example 42

workUnitContract:
  name: charge_payment
  version: v2
  # ... as defined in Example 43
```

**Flow referencing contracts:**
```yaml
sms:
  flows:
    - name: CheckoutFlow
      triggeredBy:
        inputIntent: InitiateCheckout
      steps:
        - work_unit: validate_order    # Contract: validate_order v1
        - work_unit: reserve_inventory  # Contract: reserve_inventory v1
        - work_unit: charge_payment     # Contract: charge_payment v2
      failure:
        policy: compensate
        compensations:
          - on: charge_payment
            do: release_inventory      # Contract: release_inventory v1
```

**Coverage Report:**
```yaml
contractCoverage:
  flow: CheckoutFlow
  totalWorkUnits: 4
  contractedWorkUnits: 4
  coveragePercent: 100.0
  uncoveredWorkUnits: []
```

---

## SMS (FLOW) EXAMPLES

### Example 12: Simple Sequential Flow

```yaml
sms:
  flows:
    - name: SubmitCreateOrder
      triggeredBy:
        inputIntent: CreateOrder
      steps:
        - validate: constraints
        - enrich: tax
        - enrich: total
        - persist: OrderRecord
      authorization:
        policySet: OrderCreationPolicies
      onPolicyChange: revalidate
```

**Execution Order**: validate → enrich tax → enrich total → persist

---

### Example 13: Flow with Atomic Group

```yaml
sms:
  flows:
    - name: CheckoutFlow
      triggeredBy:
        inputIntent: InitiateCheckout
      steps:
        - type: atomic
          atomicGroup:
            name: ValidateAndReserve
            boundary: inventory
            steps:
              - type: work_unit
                workUnit: validate_order
              - type: work_unit
                workUnit: reserve_inventory
        - type: work_unit
          workUnit: charge_payment
          boundary: billing
        - type: work_unit
          workUnit: send_confirmation
      authorization:
        policySet: CheckoutPolicies
      onPolicyChange: drain
```

**Key Features**:
- Atomic group ensures validate + reserve happen together
- Cross-boundary flow (inventory → billing)
- Graceful degradation on policy change

---

### Example 14: Flow with Compensation

```yaml
sms:
  flows:
    - name: OrderCommitFlow
      triggeredBy:
        inputIntent: CommitOrder
      steps:
        - work_unit: reserve_inventory
          boundary: inventory
        - work_unit: charge_card
          boundary: billing
        - work_unit: emit_order_event
      failure:
        policy: compensate
        compensations:
          - on: charge_card
            do: release_inventory
      authorization:
        policySet: OrderCommitPolicies
```

**Failure Behavior**: If charge_card fails, automatically release_inventory

---

### Example 15: Evolution-Aware Flow

```yaml
sms:
  flows:
    - name: OrderEvolutionFlow
      purpose: zero_downtime_upgrade
      steps:
        - shadowWrite:
            old: OrderRecord v2
            new: OrderRecord v3
        - shadowValidate:
            constraints: order_total, payment_validity
        - promote:
            when: shadow_error_rate < 0.01
        - retire:
            version: v2
            drain_period: 24h
```

**Use Case**: Safe data migration with validation

---

## WORKER TOPOLOGY (WTS) EXAMPLES

### Example 16: Worker Declaration

```yaml
wts:
  workers:
    - name: OrderService
      acceptsInputs:
        - CreateOrder v1
        - CreateOrder v2
        - SubmitPayment v1
      produces:
        - OrderRecord v2
        - OrderRecord v3
      compatibleVersions: [v2, v3]
      capabilities:
        - order_management
        - payment_processing
```

**Key Features**: Multi-version support during migration

---

### Example 17: Location Definitions

```yaml
topology:
  locations:
    - location us_east:
        scope: region
        capabilities: [compute, storage, gpu]
        latency_to_user: low
        provisioner: kubernetes
        provisionable: true
    
    - location edge_browser:
        scope: runtime
        capabilities: [compute_light, ui]
        latency_to_user: minimal
        provisioner: browser
        provisionable: false
```

**Key Difference**: Browser location is not provisionable

---

### Example 18: Worker Class Definition

```yaml
topology:
  worker_class api_worker:
    runtime: sms-runtime
    supports: [order_management, inventory_check]
    transport: [nats, http, unix_socket]
    resources:
      cpu: 2
      memory: 4Gi
      connections: 1000
```

---

### Example 19: Placement with Constraints

```yaml
topology:
  placement order_workers:
    worker_class: api_worker
    locations: [us_east, us_west, eu_west]
    min: 3
    max: 50
    preferred: us_east
```

**Scheduler Interpretation**:
- Minimum 3 workers always
- Scale up to 50 based on signals
- Prefer us_east but can use others

---

### Example 20: Scaling Policy with Signals

```yaml
topology:
  scaling_policy order_scale:
    target: order_workers
    signals:
      queue_depth > 100 => scale_up
      latency_p95 > 500ms => scale_up
      error_rate > 5% => hold
      queue_depth < 10 => scale_down
      cpu_usage < 20% => scale_down
    cooldown: 30s
```

**Behavior**:
- Multiple conditions can trigger actions
- Cooldown prevents flapping
- Hold means maintain current count

---

### Example 21: Version-Aware Routing

```yaml
wts:
  routing:
    strategy: versioned
    rules:
      - if: Order.version == v3 AND worker.supports(v3)
        routeTo: OrderService_v3
      - if: Order.version == v2 AND worker.supports(v2)
        routeTo: OrderService_v2
      - else:
        routeTo: OrderService_legacy
```

**Use Case**: Gradual rollout of new worker version

---

## POLICY & AUTHORIZATION EXAMPLES

### Example 22: Simple Role-Based Policy

```yaml
policy:
  name: ViewOrders
  version: v1
  appliesTo:
    type: presentationView
    name: CustomerOrderList
  effect: allow
  when: subject.role in [customer, admin, support]
```

---

### Example 23: Attribute-Based Policy

```yaml
policy:
  name: ModifyOrder
  version: v2
  appliesTo:
    type: inputIntent
    name: UpdateOrder
  effect: allow
  when: |
    (subject.role == "customer" AND data.customer_id == subject.customer_id)
    OR (subject.role == "admin")
    OR (subject.role == "support" AND subject.department == "order_management")
```

**Key Features**:
- Multi-condition logic
- Data-aware authorization
- Role and attribute combination

---

### Example 24: Data Policy with Transition Control

```yaml
dataPolicy:
  name: OrderAccessPolicy
  version: v1
  appliesTo:
    dataState: OrderRecord
    version: v3
  permissions:
    read: subject.role in [customer, admin, support, finance]
    write: subject.role in [admin, system]
    transition:
      - name: ApplyPayment
        when: subject.role in [customer, admin, finance]
      - name: Cancel
        when: subject.role in [customer, admin]
      - name: Refund
        when: subject.role in [admin, finance]
```

**Key Features**:
- Read/write/transition separation
- Transition-specific authorization
- Version-specific policy

---

### Example 25: Policy Set with Conflict Resolution

```yaml
policySet:
  name: OrderManagementPolicies
  resolution: deny_overrides
  policies:
    - ViewOrders v1
    - ModifyOrder v2
    - OrderAccessPolicy v1
    - RegionalRestrictions v1
```

**Resolution**: Any deny trumps all allows

---

### Example 26: Policy Artifact with Lifecycle

```yaml
policyArtifact:
  policy:
    name: ModifyOrder
    version: v2
    definition: <policy from Example 23>
  lifecycle:
    state: shadow
    supersedes: v1
  scope:
    domain: order
    component: worker.OrderService
```

**State**: Shadow means evaluated but not enforced (testing phase)

---

### Example 27: Policy Evolution with Shadowing

**Step 1: Deploy v2 in shadow mode**
```yaml
policyArtifact:
  policy:
    name: ModifyOrder
    version: v2
  lifecycle:
    state: shadow
    supersedes: null
```

**Step 2: Monitor shadow divergence**
- Observe signal: `shadow_denies: 14`
- Analyze and refine

**Step 3: Promote v2**
```yaml
policyArtifact:
  policy:
    name: ModifyOrder
    version: v2
  lifecycle:
    state: enforce
    supersedes: v1
```

**Step 4: Deprecate v1**
```yaml
policyArtifact:
  policy:
    name: ModifyOrder
    version: v1
  lifecycle:
    state: deprecated
```

---

## SIGNALS & SCHEDULING EXAMPLES

### Example 28: Capacity Signal

```yaml
signal:
  type: capacity
  source:
    component: worker
    name: OrderService
    region: us_east
  subject:
    intent: CreateOrder
  metrics:
    inflight: 145
    max_capacity: 200
    queue_depth: 12
  severity: warn
  timestamp: 2024-01-15T14:23:45Z
```

**Scheduler Action**: Consider scaling up

---

### Example 29: Latency Signal

```yaml
signal:
  type: latency
  source:
    component: worker
    name: OrderService
    region: us_east
  subject:
    intent: CreateOrder
  metrics:
    p50_ms: 45
    p95_ms: 840
    p99_ms: 1250
  severity: critical
  timestamp: 2024-01-15T14:23:47Z
```

**Scheduler Action**: Scale up or investigate performance

---

### Example 30: Backpressure Signal

```yaml
signal:
  type: backpressure
  source:
    component: sms
    name: OrderCommitFlow
    region: us_east
  subject:
    dataState: OrderRecord
  metrics:
    blocked_transitions: 23
    wait_time_avg_ms: 450
  severity: warn
  timestamp: 2024-01-15T14:23:49Z
```

**Scheduler Action**: Increase downstream worker capacity

---

### Example 31: Policy Shadow Signal

```yaml
signal:
  type: policy
  source:
    component: worker
    name: OrderService
    region: us_east
  subject:
    policy: ModifyOrder v2
  metrics:
    shadow_allows: 1456
    shadow_denies: 14
    divergence_rate: 0.0095
  severity: info
  timestamp: 2024-01-15T14:23:50Z
```

**Use**: Validate policy changes before enforcement

---

### Example 32: Scheduler Configuration

```yaml
scheduler:
  name: regional_order_scheduler
  scope: region
  authority: true
  observes:
    - capacity
    - latency
    - backpressure
    - health
```

---

### Example 33: Failure Domain

```yaml
failure_domain:
  name: us_east_zone_a
  scope: zone
  affected_locations:
    - us_east_1a_cluster_1
    - us_east_1a_cluster_2
```

**Use**: Prevent cascading failures

---

## CROSS-CUTTING EXAMPLES

### Example 34: Execution Context (Correlation)

```yaml
executionContext:
  trace_id: 550e8400-e29b-41d4-a716-446655440000
  causation_id: 660e8400-e29b-41d4-a716-446655440001
  actor:
    subject: user-12345
    roles: [customer]
  metadata:
    origin: web_app
    session_id: sess-789
```

**Propagation**: Flows through SMS → Workers → Signals

---

### Example 35: Time Constraints

```yaml
timeConstraint:
  type: deadline
  value: 5s

timeConstraint:
  type: staleness
  value: 30m

timeConstraint:
  type: ttl
  value: 24h
```

**Usage**:
- Deadline: Request timeout
- Staleness: Materialized view refresh
- TTL: Cache expiry

---

### Example 36: Idempotency Configuration

```yaml
idempotency:
  strategy: deterministic_key
  key:
    - inputIntent.id
    - order.id
  scope: global
```

**Guarantees**: Same input never processed twice globally

---

### Example 37: Execution Outcome

**Success:**
```yaml
executionOutcome:
  type: success
  retryable: false
  reason: "Order created successfully"
```

**Rejection (Policy):**
```yaml
executionOutcome:
  type: rejection
  retryable: false
  reason: "Insufficient permissions"
```

**Failure (Infrastructure):**
```yaml
executionOutcome:
  type: failure
  retryable: true
  reason: "Database timeout"
```

---

### Example 38: Realm Isolation

```yaml
realm:
  name: customer_acme
  isolation:
    data: strict
    policy: strict
    signals: isolated

realm:
  name: customer_contoso
  isolation:
    data: strict
    policy: strict
    signals: shared
```

**Difference**: Acme gets isolated signals, Contoso shares signal infrastructure

---

## COMPLEX END-TO-END EXAMPLES

### Example 39: Complete Order System

**Data Layer:**
```yaml
dataType:
  name: Order
  version: v3
  fields:
    id: uuid
    customer_id: uuid
    items: array
    subtotal: decimal
    tax: decimal
    total: decimal
    paid_amount: decimal
    status: enum[draft, partially_paid, paid, fulfilled]

dataState:
  name: OrderRecord
  type: Order
  lifecycle: persistent
  constraintPolicy:
    enforcement: strict
```

**Input Layer:**
```yaml
inputIntent:
  name: CreateOrder
  version: v2
  proposes:
    dataState: OrderDraft
    version: v3
  supplies:
    required: [customer_id, items, subtotal]
  transitions:
    to: OrderRecord
```

**Flow Layer:**
```yaml
sms:
  flows:
    - name: OrderCreationFlow
      triggeredBy:
        inputIntent: CreateOrder
      steps:
        - validate: constraints
        - enrich: tax
        - persist: OrderRecord
      authorization:
        policySet: OrderPolicies
```

**Worker Layer:**
```yaml
wts:
  workers:
    - name: OrderService
      acceptsInputs: [CreateOrder v2]
      produces: [OrderRecord v3]
```

**Policy Layer:**
```yaml
policy:
  name: CreateOrderPolicy
  version: v1
  appliesTo:
    type: inputIntent
    name: CreateOrder
  effect: allow
  when: subject.role == "customer"
```

---

### Example 40: Multi-Region with Failover

**Topology:**
```yaml
topology:
  locations:
    - location us_east:
        scope: region
        provisioner: kubernetes
    - location us_west:
        scope: region
        provisioner: kubernetes
  
  placement order_workers:
    worker_class: api_worker
    locations: [us_east, us_west]
    min: 3
    max: 20
    preferred: us_east
  
  failure_domain us_east_domain:
    scope: region
    affected_locations: [us_east]
```

**Scaling Policy:**
```yaml
scaling_policy:
  target: order_workers
  signals:
    queue_depth > 100 => scale_up
    region.health == "degraded" => scale_up
  cooldown: 60s
```

**Behavior**: If us_east degrades, automatically scale us_west

---

## AUTHORITY & INVARIANT EXAMPLES (NEW in v1.3)

### Example 46: Entity-Scoped Authority Declaration

**Use Case**: User profiles with per-user write authority

```yaml
model:
  name: UserProfile
  version: v1
  
  mutability:
    scope: entity
    exclusive: true
    authority: user_authority
  
  authority:
    scope: entity
    resolution: entity.home_region
    migration:
      allowed: true
      strategy: pause-and-cutover
      governed_by: ProfileMigrationPolicy
```

**Key Points**:
- Each user entity has its own authoritative region
- Authority can migrate between regions
- Migration governed by policy

---

### Example 47: Authority Migration with Triggers

**Use Case**: Follow-the-sun workload optimization

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
      - type: load
        condition: entity.access_pattern.primary_region != current_region
      - type: manual
        condition: operator_request
    
    constraints:
      - type: region
        allowed: [us-east, us-west, eu-west, ap-southeast]
      - type: governance
        rule: entity.data_classification != "restricted"
```

**Behavior**: Authority moves to reduce latency, respecting region and classification constraints

---

### Example 48: Governance-Constrained Authority

**Use Case**: Financial data with regulatory constraints

```yaml
model:
  name: Account
  
  mutability:
    scope: entity
    exclusive: true
  
  authority:
    scope: entity
    resolution: entity.jurisdiction_region
    
    migration:
      allowed: false  # Regulatory requirement
    
    constraints:
      - type: governance
        rule: |
          entity.jurisdiction == authority_region.jurisdiction
          AND authority_region.compliance_certified == true
```

**Key Points**:
- Migration disabled for compliance
- Authority bound to regulatory jurisdiction

---

### Example 49: Control vs Data Storage Declaration

**Use Case**: Separating coordination from domain state

```yaml
# Control storage - authority state
storage:
  name: AuthorityKV
  role: control
  rebuildable: false

# Data storage - event stream
storage:
  name: UserEvents
  role: data
  rebuildable: true

# Data storage - materialized view
storage:
  name: UserDashboardView
  role: data
  rebuildable: true
```

**Rules**:
- Control storage is strongly consistent
- Data storage is rebuildable from events

---

### Example 50: Authority-Agnostic View

**Use Case**: Dashboard that tolerates authority changes

```yaml
materialization:
  name: UserDashboard
  
  source:
    - UserProfile
    - UserActivity
    - AccountStatus
  
  targetState: UserDashboardView
  
  authority_agnostic: true
  epoch_tolerant: true
  
  freshness:
    maxStaleness: 30s
  
  evolution:
    strategy: incremental
    blocking: false
```

**Behavior**: View continues working during authority transitions, processing events from any region/epoch

---

### Example 51: Relationship with Causality Semantics

**Use Case**: Transaction → Account relationships for ledger

```yaml
relationship:
  name: debit
  from: Transaction
  to: Account
  type: affects
  cardinality: many-to-one
  semantics:
    causal: true
    ordered: true

relationship:
  name: credit
  from: Transaction
  to: Account
  type: affects
  cardinality: many-to-one
  semantics:
    causal: true
    ordered: true
```

**Key Points**:
- Transaction must exist before it can affect Account
- Events processed in causal order
- No synchronous write coordination implied

---

### Example 52: View-Level Invariant (Balance Constraint)

**Use Case**: Non-negative balance enforced by view

```yaml
view:
  name: AccountBalance
  
  inputs:
    - Account
    - Transaction
  
  aggregation:
    type: sum
    expression: credits - debits
  
  invariant:
    expression: result >= 0
    on_violation: flag
  
  authority_agnostic: true
```

**Behavior**: View calculates balance from transactions, flags violations rather than blocking writes

---

### Example 53: Banking-Style Entity Decomposition

**Use Case**: Proper invariant-oriented model for financial system

```yaml
# Entity: Identity + constraints (NOT balance)
model:
  name: Account
  mutability:
    scope: entity
    exclusive: true
  authority:
    migration:
      allowed: false

# Entity: Immutable intent
model:
  name: Transaction
  mutability:
    scope: append-only

# Relationships
relationships:
  - name: debit
    from: Transaction
    to: Account
    semantics:
      causal: true
      ordered: true
  
  - name: credit
    from: Transaction
    to: Account
    semantics:
      causal: true
      ordered: true

# Invariant enforced by view
view:
  name: AccountBalance
  inputs: [Account, Transaction]
  invariant:
    expression: sum(credits) - sum(debits) >= 0
```

**Anti-Pattern Avoided**: Balance as mutable field requiring global transactions

---

### Example 54: CAS Token Usage Pattern

**Use Case**: Safe writes across authority migrations

```yaml
# CAS token structure
cas:
  token_fields:
    - entity_id
    - entity_version
    - authority_epoch
    - authority_region
    - lease_id

# Write flow with CAS
sms:
  flows:
    - name: UpdateProfile
      steps:
        - work_unit: get_current_cas_token
        - work_unit: validate_authority
        - work_unit: apply_update_with_cas
          requires:
            cas_valid: true
      
      failure:
        policy: abort
        on: cas_mismatch
```

**Behavior**: Write fails if authority migrated since token was issued

---

### Example 55: Multi-Region Failover Configuration

**Use Case**: Explicit failure semantics per entity

```yaml
availability:
  write_scope: entity
  pause_allowed: true
  read_continuity: best-effort

write_failure:
  strategy: reject

# Regional view replication
materialization:
  name: GlobalUserView
  
  source:
    - us-east:UserDashboardView
    - eu-west:UserDashboardView
  
  authority_agnostic: true
  
  # Global views replicate regional views, not raw data
  scope: global
  contains_pii: false
  replication:
    allowed: true
```

**Behavior**: 
- Entity writes fail if authoritative region unavailable
- Reads continue via replicated views
- No raw data crosses regions

---

## ANTI-PATTERNS (What NOT to do)

### Anti-Pattern 1: Mixing Versions Without Evolution

❌ **Wrong:**
```yaml
dataState:
  name: OrderRecord
  type: Order
  lifecycle: persistent
  # No version specified - ambiguous
```

✅ **Correct:**
```yaml
dataState:
  name: OrderRecord
  type: Order
  version: v3
  lifecycle: persistent
```

---

### Anti-Pattern 2: Hardcoded Transport in Flow

❌ **Wrong:**
```yaml
sms:
  flows:
    - name: OrderFlow
      steps:
        - call_http: "https://inventory-service/reserve"  # DON'T
```

✅ **Correct:**
```yaml
sms:
  flows:
    - name: OrderFlow
      steps:
        - work_unit: reserve_inventory  # Intent, not mechanism
```

---

### Anti-Pattern 3: Policy as Code

❌ **Wrong:**
```python
# Policy embedded in application code
if user.role == "admin" or (user.role == "customer" and order.customer_id == user.id):
    allow()
```

✅ **Correct:**
```yaml
policy:
  name: ModifyOrder
  when: |
    subject.role == "admin" 
    OR (subject.role == "customer" AND data.customer_id == subject.customer_id)
```

---

### Anti-Pattern 4: Implicit Coupling

❌ **Wrong:**
```yaml
# Flow assumes inventory service exists at specific location
sms:
  flows:
    - name: OrderFlow
      steps:
        - work_unit: check_inventory
          # No boundary declared - implicit coupling
```

✅ **Correct:**
```yaml
sms:
  flows:
    - name: OrderFlow
      steps:
        - work_unit: check_inventory
          boundary: inventory  # Explicit boundary
```

---

## UI COMPOSITION EXAMPLES (NEW in v1.4)

### Example 56: Simple Composition

A basic workspace grouping related views.

```yaml
compositions:
  - name: OrderDetailsWorkspace
    version: v1
    description: "Order details with line items"
    primary: OrderDetailsView
    navigation:
      label: "Order #{{order.order_number}}"
      purpose: orders
      parent: OrderListWorkspace
      param: order_id
    related:
      - view: OrderItemsView
        relationship: detail
        cardinality: many
        navigation:
          label: "Items"
          scope: within_composition
```

**Use Case**: Standard master-detail pattern

---

### Example 57: Composition with Navigation and Indicators

A dashboard composition with dynamic indicators.

```yaml
compositions:
  - name: AdminDashboard
    version: v1
    description: "Admin home with system metrics"
    primary: SystemMetricsView
    navigation:
      label: "Dashboard"
      purpose: home
      policy_set: AdminAccess
      indicator:
        when: critical_alerts > 0
        severity: critical
    related:
      - view: AlertSummaryView
        relationship: supplementary
        navigation:
          label: "Alerts"
          scope: within_composition
      - view: PendingApprovalsView
        relationship: supplementary
        navigation:
          label: "Approvals"
          scope: within_composition
```

**Use Case**: Admin dashboard with dynamic alert indicator

---

### Example 58: Composition with Field Permissions

A view with policy-controlled field visibility and masking.

```yaml
views:
  - name: CustomerDetailsView
    version: v1
    consumes:
      dataState: CustomerRecord
      version: v1
    fieldPermissions:
      - field: full_name
        visible: true
      - field: email
        visible: true
      - field: phone
        visible: true
      - field: ssn
        visible:
          policy: ViewSensitiveData
        mask:
          unless: subject.clearance_level >= 4
          type: partial
          reveal: last4
      - field: date_of_birth
        visible:
          policy: ViewSensitiveData

compositions:
  - name: CustomerWorkspace
    version: v1
    primary: CustomerDetailsView
    navigation:
      label: "{{customer.full_name}}"
      purpose: customers
      policy_set: CustomerManagement
```

**Use Case**: Admin view with PII protection and masking

---

### Example 59: Experience Definition

A complete customer-facing experience.

```yaml
experiences:
  - name: CustomerPortal
    version: v1
    description: "Customer-facing banking experience"
    entry_point: CustomerDashboard
    includes:
      - CustomerDashboard
      - AccountListWorkspace
      - AccountDetailsWorkspace
      - TransactionHistoryWorkspace
      - ProfileWorkspace
      - HelpWorkspace
    policy_set: CustomerAccess
    unauthenticated:
      redirect_to: LoginWorkspace
    on_unauthorized: conceal
```

**Use Case**: Complete customer application with login redirect

---

### Example 60: Full Admin Experience with Navigation Hierarchy

A comprehensive admin experience with full navigation structure.

```yaml
experiences:
  - name: AdminConsole
    version: v1
    description: "Administrative console"
    entry_point: AdminDashboard
    includes:
      - AdminDashboard
      - UserManagementWorkspace
      - UserDetailsWorkspace
      - RoleManagementWorkspace
      - AuditLogWorkspace
      - SystemConfigWorkspace
      - HelpWorkspace
    policy_set: AdminAccess
    unauthenticated:
      redirect_to: AdminLoginWorkspace
    on_unauthorized: indicate

compositions:
  - name: AdminDashboard
    version: v1
    primary: AdminMetricsView
    navigation:
      label: "Dashboard"
      purpose: home
      policy_set: AdminAccess
      indicator:
        when: pending_approvals > 0
        severity: warning

  - name: UserManagementWorkspace
    version: v1
    primary: UserListView
    navigation:
      label: "Users"
      purpose: users
      parent: AdminDashboard
      policy_set: UserManagement

  - name: UserDetailsWorkspace
    version: v1
    primary: UserDetailsView
    navigation:
      label: "{{user.email}}"
      purpose: users
      parent: UserManagementWorkspace
      param: user_id
      policy_set: UserManagement
      indicator:
        when: user.status == "suspended"
        severity: error
    related:
      - view: UserRolesView
        relationship: detail
        cardinality: many
        navigation:
          label: "Roles"
          scope: within_composition
      - view: UserActivityView
        relationship: historical
        cardinality: many
        navigation:
          label: "Activity"
          scope: within_composition
      - view: UserAuditLogView
        relationship: historical
        cardinality: many
        basis: audit_for_user
        navigation:
          label: "Audit Log"
          scope: within_composition

  - name: AuditLogWorkspace
    version: v1
    primary: AuditLogListView
    navigation:
      label: "Audit Logs"
      purpose: audit
      parent: AdminDashboard
      policy_set: AuditAccess

navigation:
  purposes:
    - name: home
      description: "Primary dashboard"
    - name: users
      description: "User management"
    - name: audit
      description: "Audit and compliance"
```

**Use Case**: Complete admin console with user management and audit

---

## V1.5 ADDENDUM EXAMPLES (NEW in v1.5)

### Example 61: Form Intent Binding (ADDENDUM X)

**Use Case**: Transfer form that submits to intent

```yaml
presentationView:
  name: TransferFormView
  version: v1
  consumes:
    dataState: AccountBalance
    version: v1
  
  triggers:
    intent: InitiateTransfer
    binding: form
    
    field_mapping:
      - view_field: from_account_id
        supplies: source_account_id
        source: context
        required: true
      - view_field: to_account_id
        supplies: destination_account_id
        source: input
        required: true
      - view_field: amount
        supplies: amount
        source: input
        required: true
        validation:
          constraint: amount > 0 AND amount <= from_account.balance
          message: "Amount must be positive and not exceed balance"
      - view_field: memo
        supplies: memo
        source: input
        required: false
    
    confirmation:
      required: true
      message: "Transfer ${{amount}} from {{from_account.name}} to {{to_account.name}}?"
    
    on_success:
      action: navigate_back
      notification:
        type: toast
        message: "Transfer initiated successfully"
    
    on_error:
      action: display_inline
      show_field_errors: true
      notification:
        type: inline
        message: "Transfer failed: {{error.message}}"
```

**Key Features**:
- Multi-field form with context and input sources
- Client-side validation before submission
- Confirmation dialog with interpolation
- Success/error handling with navigation

---

### Example 62: View Materialization with Retrieval Contract (ADDENDUM Y)

**Use Case**: Account balance view with key-based retrieval

```yaml
materialization:
  name: AccountBalanceView
  version: v1
  source:
    - Account
    - Transaction
  targetState: AccountBalance
  
  retrieval:
    mode: by_entity
    entity_key: account_id
    
    freshness:
      max_staleness: 5s
    
    on_unavailable:
      strategy: use_cached
      cache_ttl: 30s
  
  evolution:
    strategy: incremental
    blocking: false
```

**Key Features**:
- Entity-keyed retrieval pattern
- Freshness guarantee with cached fallback
- Incremental materialization

---

### Example 63: Streaming View with Temporal Windows (ADDENDUM Y)

**Use Case**: Real-time transaction monitoring with windowed aggregations

```yaml
materialization:
  name: TransactionActivityStream
  version: v1
  source: Transaction
  targetState: TransactionActivity
  
  retrieval:
    mode: singleton
    list:
      max_items: 1000
      default_order: timestamp
      order_direction: desc
  
  temporal:
    window:
      type: tumbling
      size: 1m
    
    aggregation:
      - field: amount
        function: sum
        as: total_volume
      - field: transaction_id
        function: count
        as: transaction_count
      - field: amount
        function: avg
        as: avg_transaction_size
    
    timestamp_field: created_at
    watermark: 10s
    
    overflow:
      strategy: drop_oldest
      buffer_size: 10000
    
    emit:
      trigger: on_window_close
      include_partial: false
```

**Key Features**:
- Tumbling window for fixed time intervals
- Multiple aggregation functions
- Late event watermark tolerance
- Overflow backpressure handling

---

### Example 64: Intent Delivery Contract (ADDENDUM Z)

**Use Case**: Transfer intent with retry and timeout semantics

```yaml
inputIntent:
  name: InitiateTransfer
  version: v1
  proposes:
    dataState: TransferRecord
    version: v1
  supplies:
    required:
      - source_account_id
      - destination_account_id
      - amount
  
  delivery:
    timeout: 30s
    retry:
      max_attempts: 3
      backoff:
        strategy: exponential
        initial: 100ms
        multiplier: 2
        max: 5s
    
    idempotency:
      strategy: deterministic_key
      key: [intent_id, source_account_id]
      scope: global
      ttl: 24h
    
    result:
      success:
        - dataState: TransferRecord
          field: transfer_id
      error:
        - code: INSUFFICIENT_FUNDS
          retryable: false
          message: "Insufficient balance"
        - code: ACCOUNT_LOCKED
          retryable: false
          message: "Account is locked"
        - code: TIMEOUT
          retryable: true
          message: "Processing timeout, safe to retry"
```

**Key Features**:
- Explicit timeout and retry semantics
- Idempotency with deterministic keys
- Typed error responses with retryability

---

### Example 65: Session Contract with Multi-Device Sync (ADDENDUM AA)

**Use Case**: Banking session with cross-device state

```yaml
session:
  name: BankingSession
  version: v1
  
  scope: subject
  isolation: strict
  
  state:
    - name: current_account_id
      type: uuid
      persistence: session
      sync: true
    - name: active_composition
      type: string
      persistence: session
      sync: true
    - name: scroll_position
      type: integer
      persistence: device
      sync: false
    - name: ui_preferences
      type: object
      persistence: persistent
      sync: true
  
  lifecycle:
    idle_timeout: 15m
    absolute_timeout: 8h
    extend_on_activity: true
  
  multi_device:
    sync_enabled: true
    conflict_resolution: last_write_wins
    sync_delay: 2s
  
  security:
    require_reauth_for:
      - transfers
      - settings_change
    reauth_timeout: 5m
```

**Key Features**:
- Multi-device state synchronization
- Device-local vs synced state
- Session lifecycle with timeouts
- Security reauth requirements

---

### Example 66: Offline-First Session (ADDENDUM AA)

**Use Case**: Mobile app with offline capability

```yaml
session:
  name: MobileAppSession
  version: v1
  
  scope: subject
  
  offline:
    enabled: true
    
    local_storage:
      strategy: progressive_cache
      max_size: 50MB
      
    sync:
      strategy: optimistic
      conflict_resolution: merge
      
      on_reconnect:
        action: sync_all
        priority:
          - intents_pending
          - view_updates
          - assets_pending
    
    intents:
      queue_locally: true
      max_queue_size: 100
      retention: 7d
  
  multi_device:
    sync_enabled: true
    offline_changes_merge: true
```

**Key Features**:
- Offline intent queuing
- Progressive cache strategy
- Reconnection sync priority
- Optimistic conflict resolution

---

### Example 67: Indicator Source Binding (ADDENDUM AB)

**Use Case**: Alert indicators from materialized views

```yaml
compositions:
  - name: AccountDashboard
    version: v1
    primary: AccountOverviewView
    navigation:
      label: "Accounts"
      purpose: home
      
      indicator:
        source: AlertSummaryView
        binding:
          field: critical_alert_count
        when: critical_alert_count > 0
        severity: critical
        badge:
          text: "{{critical_alert_count}}"
          style: numeric
    
    related:
      - view: AccountListView
        relationship: detail
        navigation:
          label: "All Accounts"
          scope: within_composition
          
          indicator:
            source: AccountListView
            binding:
              field: low_balance_count
            when: low_balance_count > 0
            severity: warning
            badge:
              text: "{{low_balance_count}} low"
              style: text
```

**Key Features**:
- Indicators driven by materialized views
- Dynamic badge content from data
- Multiple severity levels
- Declarative condition expressions

---

### Example 68: Composition Fetch Semantics (ADDENDUM AC)

**Use Case**: Dashboard with optimized data fetching

```yaml
compositions:
  - name: CustomerDashboard
    version: v1
    primary: CustomerSummaryView
    
    fetch:
      strategy: parallel
      
      primary:
        required: true
        timeout: 2s
        on_timeout: fail
      
      related:
        required: false
        timeout: 5s
        on_timeout: degrade
        
      prefetch:
        - view: RecentTransactionsView
          condition: always
        - view: RecommendationsView
          condition: on_idle
        - view: DocumentLibraryView
          condition: on_demand
      
      cache:
        strategy: stale_while_revalidate
        ttl: 30s
        revalidate_in_background: true
    
    related:
      - view: RecentTransactionsView
        relationship: detail
        fetch:
          lazy: false
      - view: AccountListView
        relationship: detail
        fetch:
          lazy: false
      - view: DocumentLibraryView
        relationship: supplementary
        fetch:
          lazy: true
```

**Key Features**:
- Parallel vs sequential fetch strategies
- Lazy loading for supplementary views
- Prefetch strategies (always, on_idle, on_demand)
- Cache revalidation in background

---

### Example 69: Mask Expression Grammar (ADDENDUM AD)

**Use Case**: PII masking with policy-based control

```yaml
views:
  - name: CustomerDetailsView
    version: v1
    consumes:
      dataState: CustomerRecord
      version: v1
    
    fieldPermissions:
      - field: full_name
        visible: true
      
      - field: ssn
        visible:
          policy: ViewSensitiveData
        mask:
          unless: subject.clearance_level >= 4
          type: partial
          reveal: last4
          pattern: "***-**-{{last4}}"
      
      - field: email
        visible: true
        mask:
          unless: subject.role in [admin, support]
          type: partial
          reveal: domain
          pattern: "***@{{domain}}"
      
      - field: phone
        visible:
          policy: ViewContactInfo
        mask:
          unless: subject.department == "customer_service"
          type: full
          replacement: "[REDACTED]"
      
      - field: account_balance
        visible: true
        mask:
          unless: data.customer_id == subject.customer_id OR subject.role == "admin"
          type: approximate
          precision: 1000
          pattern: "~${{rounded}}"
```

**Key Features**:
- Multiple masking strategies (partial, full, approximate)
- Policy-based visibility control
- Pattern-based reveal with interpolation
- Role and attribute-based unmasking

---

### Example 70: Navigation Semantics with Hierarchy (ADDENDUM AE)

**Use Case**: Multi-level navigation tree

```yaml
navigation:
  purposes:
    - name: home
      description: "Main dashboard"
    - name: accounts
      description: "Account management"
    - name: transactions
      description: "Transaction history"
    - name: settings
      description: "User settings"

compositions:
  - name: Dashboard
    version: v1
    primary: DashboardView
    navigation:
      label: "Home"
      purpose: home
      policy_set: UserAccess
  
  - name: AccountList
    version: v1
    primary: AccountListView
    navigation:
      label: "Accounts"
      purpose: accounts
      parent: Dashboard
      policy_set: UserAccess
      
  - name: AccountDetails
    version: v1
    primary: AccountDetailsView
    navigation:
      label: "{{account.name}}"
      purpose: accounts
      parent: AccountList
      param: account_id
      policy_set: UserAccess
      
      breadcrumb:
        - label: "Home"
          target: Dashboard
        - label: "Accounts"
          target: AccountList
        - label: "{{account.name}}"
          current: true
  
  - name: TransactionHistory
    version: v1
    primary: TransactionListView
    navigation:
      label: "Transactions"
      purpose: transactions
      parent: AccountDetails
      param: account_id
      policy_set: UserAccess
```

**Key Features**:
- Purpose-based navigation grouping
- Hierarchical parent-child relationships
- Dynamic labels with interpolation
- Automatic breadcrumb generation
- Per-composition policy sets

---

### Example 71: View Availability Policy (ADDENDUM AF)

**Use Case**: Graceful degradation during partial outages

```yaml
materialization:
  name: EnrichedAccountView
  version: v1
  source:
    - AccountRecord
    - ExternalCreditScore
    - MarketingPreferences
  targetState: EnrichedAccount
  
  availability:
    primary_sources:
      - AccountRecord
    optional_sources:
      - ExternalCreditScore
      - MarketingPreferences
    
    degradation:
      - condition: ExternalCreditScore.unavailable
        action: omit_field
        fields: [credit_score, credit_rating]
        indicate: true
        message: "Credit information temporarily unavailable"
      
      - condition: MarketingPreferences.unavailable
        action: use_default
        defaults:
          marketing_opt_in: false
          communication_preference: "email"
        indicate: false
      
      - condition: AccountRecord.unavailable
        action: fail
        message: "Account data required"
    
    freshness:
      max_staleness: 10s
      on_stale: use_cached_and_indicate
```

**Key Features**:
- Primary vs optional source classification
- Multiple degradation strategies (omit, default, fail)
- User indication of degraded state
- Stale data handling with indication

---

### Example 72: Presentation Hints (ADDENDUM AG)

**Use Case**: Rendering hints for various screen sizes

```yaml
presentationView:
  name: AccountListView
  version: v1
  consumes:
    dataState: AccountSummary
    version: v1
  
  hints:
    responsive:
      mobile:
        layout: list
        fields: [account_name, balance]
        collapse: [account_type, last_activity]
      
      tablet:
        layout: grid
        columns: 2
        fields: [account_name, account_type, balance, last_activity]
      
      desktop:
        layout: table
        fields: [account_name, account_number, account_type, balance, last_activity, actions]
    
    formatting:
      - field: balance
        type: currency
        params:
          symbol: "$"
          decimals: 2
      
      - field: last_activity
        type: relative_time
        params:
          format: "short"
      
      - field: account_number
        type: masked
        params:
          reveal: last4
    
    interaction:
      - field: account_name
        action: navigate
        target: AccountDetailsWorkspace
        params:
          account_id: "{{account_id}}"
      
      - field: actions
        type: menu
        options:
          - label: "View Details"
            action: navigate
            target: AccountDetailsWorkspace
          - label: "Transfer"
            action: trigger_intent
            intent: InitiateTransfer
          - label: "Download Statement"
            action: trigger_intent
            intent: RequestStatement
```

**Key Features**:
- Responsive layout hints (mobile, tablet, desktop)
- Field formatting hints (currency, time, masking)
- Interaction hints (navigation, menus, intents)
- Runtime decides actual rendering

---

### Example 73: Surface Container Semantics (ADDENDUM AH)

**Use Case**: Modal dialogs and overlays

```yaml
experiences:
  - name: CustomerPortal
    version: v1
    entry_point: Dashboard
    
    surfaces:
      - name: main
        type: primary
        compositions: [Dashboard, AccountList, AccountDetails, TransactionHistory]
      
      - name: transfer_modal
        type: modal
        trigger: intent_form
        intent: InitiateTransfer
        container:
          dismissible: true
          backdrop: true
          size: medium
          position: center
        on_success:
          action: close
        on_cancel:
          action: close
      
      - name: alert_panel
        type: sidebar
        trigger: indicator_click
        compositions: [AlertDetailsPanel]
        container:
          dismissible: true
          position: right
          width: 400px
          overlay: false
      
      - name: quick_actions
        type: popover
        trigger: button_click
        container:
          anchor: trigger_element
          position: below
          arrow: true
          dismissible: true
      
      - name: notification_toast
        type: toast
        trigger: notification
        container:
          position: top_right
          duration: 5s
          dismissible: true
          stack: true
```

**Key Features**:
- Multiple surface types (modal, sidebar, popover, toast)
- Container properties (dismissible, backdrop, size)
- Trigger semantics (intent, click, notification)
- Success/cancel actions

---

### Example 74: Accessibility Preferences (ADDENDUM AI)

**Use Case**: Accessible experience with user preferences

```yaml
experiences:
  - name: CustomerPortal
    version: v1
    entry_point: Dashboard
    
    accessibility:
      compliant_with: [WCAG_2.1_AA, ADA, Section_508]
      
      preferences:
        - name: high_contrast
          type: boolean
          default: false
          affects: [colors, borders, focus_indicators]
        
        - name: font_size
          type: enum
          values: [small, medium, large, x_large]
          default: medium
          affects: [text, spacing]
        
        - name: reduce_motion
          type: boolean
          default: false
          affects: [animations, transitions]
        
        - name: screen_reader
          type: boolean
          default: false
          affects: [aria_labels, live_regions, focus_management]
      
      keyboard_navigation:
        enabled: true
        shortcuts:
          - key: "h"
            action: navigate_home
          - key: "a"
            action: navigate_accounts
          - key: "/"
            action: focus_search
          - key: "?"
            action: show_help
      
      aria:
        landmarks: true
        live_regions: true
        labels: required
        descriptions: encouraged

compositions:
  - name: AccountList
    version: v1
    primary: AccountListView
    
    accessibility:
      label: "List of your accounts"
      description: "View and manage your bank accounts"
      role: region
      
      keyboard:
        focusable: true
        tab_order: 1
      
      screen_reader:
        announce_count: true
        announce_loading: true
        announce_updates: true
```

**Key Features**:
- WCAG/ADA compliance declaration
- User preference system
- Keyboard navigation with shortcuts
- ARIA semantics (landmarks, live regions, labels)
- Per-composition accessibility metadata

---

### Example 75: Asset Semantics (ADDENDUM AK)

**Use Case**: Profile photo with variants

```yaml
dataType:
  name: UserProfile
  version: v1
  fields:
    id: uuid
    name: string
    email: string
    
    profile_photo:
      type: asset
      asset:
        category: image
        
        constraints:
          max_size: 5MB
          mime_types: [image/jpeg, image/png, image/webp]
          extensions: [jpg, jpeg, png, webp]
          dimensions:
            min_width: 100
            min_height: 100
            max_width: 4000
            max_height: 4000
        
        lifecycle:
          type: permanent
        
        reference:
          style: by_reference
        
        access:
          read: authenticated
          signed_url_ttl: 1h
        
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
          - name: medium
            transform: resize
            params:
              width: 512
              height: 512
              fit: contain
    
    documents:
      type: array<asset>
      asset:
        category: document
        
        constraints:
          max_size: 10MB
          mime_types: [application/pdf, application/msword, image/jpeg]
        
        lifecycle:
          type: expiring
          expires_after: 90d
        
        access:
          read: policy
          policy: DocumentAccessPolicy

inputIntent:
  name: UpdateProfilePhoto
  version: v1
  supplies:
    required:
      - profile_photo
  
  upload:
    field: profile_photo
    strategy: direct
    preprocessing:
      - validate_mime_type
      - validate_size
      - generate_variants
```

**Key Features**:
- Asset field type with constraints
- Variant generation (thumbnail, small, medium)
- Multiple lifecycle types
- Access control (authenticated, policy-based)
- Upload integration with intents

---

### Example 76: Search Semantics (ADDENDUM AL)

**Use Case**: Full-text search with faceting

```yaml
search:
  name: TransactionSearch
  version: v1
  
  indexes:
    - name: transaction_fulltext
      type: fulltext
      source: TransactionRecord
      fields:
        - field: description
          weight: 10
          analyzer: standard
        - field: merchant_name
          weight: 5
          analyzer: standard
        - field: category
          weight: 3
          analyzer: keyword
      
      filters:
        - field: account_id
          type: term
        - field: amount
          type: range
        - field: transaction_date
          type: date_range
        - field: category
          type: term
        - field: status
          type: term
      
      facets:
        - field: category
          type: terms
          size: 20
        - field: merchant_name
          type: terms
          size: 10
        - field: amount
          type: range
          ranges:
            - to: 10
              label: "Under $10"
            - from: 10
              to: 50
              label: "$10-$50"
            - from: 50
              to: 100
              label: "$50-$100"
            - from: 100
              label: "Over $100"
      
      ranking:
        - field: transaction_date
          direction: desc
          weight: 1.0
        - field: _score
          weight: 2.0
      
      highlighting:
        fields: [description, merchant_name]
        fragment_size: 150
        num_fragments: 3

inputIntent:
  name: SearchTransactions
  version: v1
  proposes:
    dataState: SearchResults
    version: v1
  supplies:
    required:
      - query
    optional:
      - filters
      - facets
      - page
      - page_size
  
  search:
    index: transaction_fulltext
    
    pagination:
      default_size: 20
      max_size: 100
    
    timeout: 2s
```

**Key Features**:
- Full-text indexing with field weights
- Multiple filter types (term, range, date_range)
- Faceted search for aggregations
- Custom ranking with multiple factors
- Result highlighting
- Pagination support

---

### Example 77: Scheduled Triggers & Notifications (ADDENDUMS AM, AN)

**Use Case**: Statement generation and delivery

```yaml
scheduledTrigger:
  name: MonthlyStatementGeneration
  version: v1
  
  schedule:
    type: cron
    expression: "0 0 1 * *"
    timezone: "America/New_York"
  
  intent: GenerateStatement
  
  scope:
    type: per_entity
    entity_type: Account
    filter:
      expression: account.status == "active" AND account.statement_delivery == "monthly"
  
  parameters:
    statement_period: previous_month
    format: pdf
  
  retry:
    max_attempts: 3
    backoff: exponential

inputIntent:
  name: GenerateStatement
  version: v1
  supplies:
    required:
      - account_id
      - statement_period
      - format
  
  notifications:
    - channel: email
      trigger: on_success
      template: statement_ready
      
      recipients:
        primary: "{{account.owner_email}}"
        cc: []
      
      attachments:
        - asset_ref: "{{result.statement_pdf}}"
          filename: "statement_{{statement_period}}.pdf"
      
      content:
        subject: "Your {{statement_period}} statement is ready"
        body_template: |
          Hello {{account.owner_name}},
          
          Your account statement for {{statement_period}} is now available.
          
          Summary:
          - Beginning Balance: {{statement.beginning_balance}}
          - Ending Balance: {{statement.ending_balance}}
          - Total Transactions: {{statement.transaction_count}}
          
          View online: {{statement.view_url}}
    
    - channel: push
      trigger: on_success
      
      recipients:
        devices: "{{account.owner_devices}}"
      
      content:
        title: "Statement Ready"
        body: "Your {{statement_period}} statement is ready to view"
        action:
          type: navigate
          target: StatementDetailsWorkspace
          params:
            statement_id: "{{result.statement_id}}"
    
    - channel: sms
      trigger: on_success
      condition: account.sms_notifications_enabled == true
      
      recipients:
        primary: "{{account.owner_phone}}"
      
      content:
        body: "Your {{statement_period}} bank statement is ready. View: {{statement.short_url}}"

notificationChannel:
  name: email
  provider: smtp
  config:
    from: "statements@bank.example.com"
    reply_to: "support@bank.example.com"

notificationChannel:
  name: push
  provider: fcm
  config:
    priority: normal

notificationChannel:
  name: sms
  provider: twilio
  config:
    from_number: "+15551234567"
```

**Key Features**:
- Cron-based scheduled triggers
- Per-entity trigger scoping
- Multiple notification channels (email, push, SMS)
- Conditional notifications
- Template-based content with interpolation
- Attachment support for assets
- Deep linking for mobile notifications

---

### Example 78: Collaborative Session & Structured Content (ADDENDUMS AO, AP)

**Use Case**: Real-time document collaboration

```yaml
collaborativeSession:
  name: DocumentEditSession
  version: v1
  
  scope: document
  participants:
    max: 10
    roles: [editor, viewer, commenter]
  
  state:
    - name: document_content
      type: structured_content
      persistence: persistent
      sync: realtime
      conflict_resolution: operational_transform
    
    - name: active_cursors
      type: map<user_id, cursor_position>
      persistence: ephemeral
      sync: realtime
    
    - name: active_selections
      type: map<user_id, selection_range>
      persistence: ephemeral
      sync: realtime
  
  awareness:
    presence: true
    typing_indicators: true
    cursor_tracking: true
    selection_tracking: true
  
  lifecycle:
    idle_timeout: 30m
    disconnect_grace_period: 2m
  
  operations:
    - type: insert
      permissions: [editor]
    - type: delete
      permissions: [editor]
    - type: format
      permissions: [editor]
    - type: comment
      permissions: [editor, commenter]

dataType:
  name: Document
  version: v1
  fields:
    id: uuid
    title: string
    owner_id: uuid
    
    content:
      type: structured_content
      schema:
        type: prosemirror
        version: "1.0"
        
        nodes:
          - type: doc
            content: [block+]
          - type: paragraph
            content: [inline*]
            attrs: [align]
          - type: heading
            content: [inline*]
            attrs: [level, align]
          - type: blockquote
            content: [block+]
          - type: code_block
            content: [text*]
            attrs: [language]
          - type: image
            attrs: [src, alt, width, height, caption]
          - type: table
            content: [table_row+]
        
        marks:
          - type: bold
          - type: italic
          - type: underline
          - type: link
            attrs: [href, title]
          - type: code
          - type: highlight
            attrs: [color]
        
        validation:
          max_depth: 10
          max_nodes: 50000
      
      operations:
        transform: operational_transform
        conflict_resolution: merge
        
      rendering:
        output_formats: [html, markdown, pdf, plain_text]
    
    comments:
      type: array<comment>
      comment:
        id: uuid
        author_id: uuid
        content: string
        position: content_position
        created_at: timestamp
        resolved: boolean

collaborativeOperation:
  name: InsertText
  session: DocumentEditSession
  
  operation:
    type: insert
    position: content_position
    content: structured_content
    author: user_id
  
  transform:
    strategy: operational_transform
    conflict_resolution: last_write_wins_with_rebase
  
  broadcast:
    to: all_participants
    immediate: true
```

**Key Features**:
- Real-time collaborative sessions
- Operational transform for conflict resolution
- Presence awareness (cursors, selections, typing)
- Role-based operation permissions
- Structured content with schema (ProseMirror-style)
- Support for rich content (headings, images, tables)
- Comment anchoring to content positions
- Multiple output formats

---

### Example 79: Durable Workflows & Delegation (ADDENDUMS AT, AU)

**Use Case**: Loan approval workflow with delegation

```yaml
durableWorkflow:
  name: LoanApprovalProcess
  version: v1
  
  triggeredBy:
    inputIntent: SubmitLoanApplication
  
  state:
    persistence: durable
    max_duration: 30d
  
  steps:
    - name: initial_review
      type: human_task
      
      task:
        role: loan_officer
        title: "Review loan application for {{applicant.name}}"
        description: "Review loan application #{{application.id}}"
        
        data:
          - application
          - credit_report
          - income_verification
        
        actions:
          - name: approve_for_underwriting
            label: "Approve for Underwriting"
            next: underwriting_review
          - name: request_more_info
            label: "Request More Information"
            next: await_applicant_response
          - name: reject
            label: "Reject Application"
            next: notify_rejection
        
        assignment:
          strategy: round_robin
          pool: loan_officers
          
        delegation:
          allowed: true
          delegate_to: [loan_officers, senior_loan_officers]
          retain_visibility: true
        
        timeout: 3d
        on_timeout: escalate
        escalate_to: senior_loan_officers
    
    - name: await_applicant_response
      type: wait_for_event
      event: ApplicantDocumentSubmitted
      timeout: 14d
      on_timeout: notify_expiration
      next: initial_review
    
    - name: underwriting_review
      type: human_task
      
      task:
        role: underwriter
        title: "Underwrite loan for {{applicant.name}}"
        
        actions:
          - name: approve
            label: "Approve Loan"
            next: generate_documents
          - name: reject
            label: "Reject Application"
            next: notify_rejection
          - name: request_review
            label: "Request Senior Review"
            next: senior_review
        
        delegation:
          allowed: true
          require_consent: true
          consent_request:
            message: "{{delegator.name}} would like to delegate loan underwriting to you"
            expires_after: 48h
        
        timeout: 5d
        on_timeout: escalate
    
    - name: senior_review
      type: human_task
      
      task:
        role: senior_underwriter
        title: "Senior review for loan {{application.id}}"
        
        actions:
          - name: approve
            next: generate_documents
          - name: reject
            next: notify_rejection
        
        timeout: 2d
    
    - name: generate_documents
      type: work_unit
      workUnit: GenerateLoanDocuments
      next: await_signature
    
    - name: await_signature
      type: wait_for_event
      event: DocumentsSigned
      timeout: 7d
      on_timeout: notify_expiration
      next: finalize_loan
    
    - name: finalize_loan
      type: work_unit
      workUnit: FinalizeLoan
      next: complete
    
    - name: notify_rejection
      type: work_unit
      workUnit: SendRejectionNotice
      next: complete
    
    - name: notify_expiration
      type: work_unit
      workUnit: SendExpirationNotice
      next: complete
    
    - name: complete
      type: terminal
  
  compensation:
    on_cancel:
      - reverse: generate_documents
        using: CancelLoanDocuments

delegationConsent:
  name: UnderwritingDelegation
  version: v1
  
  delegation:
    from_role: underwriter
    to_role: underwriter
    action: underwriting_review
    
    consent:
      required: true
      
      request:
        title: "Underwriting Delegation Request"
        message: "{{delegator.name}} would like to delegate underwriting for application #{{application.id}}"
        
        details:
          - label: "Applicant"
            value: "{{applicant.name}}"
          - label: "Loan Amount"
            value: "{{application.amount}}"
          - label: "Property"
            value: "{{application.property_address}}"
        
        actions:
          - name: accept
            label: "Accept"
          - name: decline
            label: "Decline"
        
        expires_after: 48h
      
      audit:
        log_request: true
        log_response: true
        retention: 7y
    
    visibility:
      delegator_retains: true
      original_assignee_notified: true
```

**Key Features**:
- Durable workflow with persistent state
- Human task steps with role-based assignment
- Delegation with consent requirements
- Multiple routing options per task
- Wait-for-event steps with timeouts
- Escalation on timeout
- Compensation/rollback semantics
- Audit trail for delegations

---

### Example 80: External Integration, Governance & Edge (ADDENDUMS AW, AX, AY)

**Use Case**: IoT device integration with governance constraints

```yaml
externalIntegration:
  name: WeatherDataProvider
  version: v1
  
  integration_type: api
  
  endpoint:
    base_url: "https://api.weather.example.com"
    authentication:
      type: api_key
      credential_ref: weather_api_key
  
  operations:
    - name: get_current_weather
      method: GET
      path: "/v1/weather/current"
      
      params:
        - name: lat
          type: decimal
          required: true
        - name: lon
          type: decimal
          required: true
      
      response:
        schema:
          temperature: decimal
          humidity: decimal
          conditions: string
          timestamp: timestamp
      
      caching:
        enabled: true
        ttl: 5m
        key: [lat, lon]
      
      retry:
        max_attempts: 3
        backoff: exponential
      
      timeout: 5s

edgeDevice:
  name: WarehouseEnvironmentSensor
  version: v1
  
  device_type: sensor
  
  capabilities:
    - temperature_sensing
    - humidity_sensing
    - local_compute
  
  data:
    - name: temperature_reading
      type: decimal
      unit: celsius
      sample_rate: 30s
    
    - name: humidity_reading
      type: decimal
      unit: percent
      sample_rate: 30s
  
  location:
    type: fixed
    tracking: facility_zone
  
  connectivity:
    online:
      protocol: mqtt
      sync: realtime
    
    offline:
      buffer_size: 1000
      buffer_strategy: ring
      sync_on_reconnect: true
  
  local_processing:
    enabled: true
    
    rules:
      - name: temperature_alert
        condition: temperature_reading > 30 OR temperature_reading < 0
        action: publish_event
        event: TemperatureAlert
        priority: high
      
      - name: aggregate_hourly
        window: 1h
        aggregation: avg
        fields: [temperature_reading, humidity_reading]
        action: publish_event
        event: HourlyEnvironmentSummary

dataGovernance:
  name: CustomerDataGovernance
  version: v1
  
  scope:
    entity_types: [Customer, CustomerProfile, CustomerPreferences]
  
  classification:
    - field_pattern: "*.ssn"
      classification: pii_sensitive
      regulations: [GDPR, CCPA, HIPAA]
    
    - field_pattern: "*.email"
      classification: pii_standard
      regulations: [GDPR, CCPA]
    
    - field_pattern: "*.phone"
      classification: pii_standard
      regulations: [GDPR, CCPA, TCPA]
  
  retention:
    - classification: pii_sensitive
      retain_for: 7y
      delete_after: 7y
      justification: "Financial services regulatory requirement"
    
    - classification: pii_standard
      retain_for: 3y
      delete_after: 5y
      justification: "Business need and GDPR compliance"
  
  access_controls:
    - classification: pii_sensitive
      required_clearance: 4
      audit_access: true
      masking_required: true
    
    - classification: pii_standard
      required_clearance: 2
      audit_access: true
  
  subject_rights:
    - right: access
      enabled: true
      response_time: 30d
      export_formats: [json, pdf]
    
    - right: rectification
      enabled: true
      response_time: 14d
    
    - right: erasure
      enabled: true
      response_time: 30d
      exceptions: [legal_hold, regulatory_requirement]
    
    - right: portability
      enabled: true
      response_time: 30d
      export_formats: [json, csv]
  
  cross_border:
    restrictions:
      - from_region: eu
        to_region: us
        allowed: false
        justification: "GDPR Article 45"
      
      - from_region: eu
        to_region: uk
        allowed: true
        basis: adequacy_decision
    
    data_residency:
      - entity_type: Customer
        field_pattern: "*.pii*"
        must_remain_in: entity.jurisdiction_region
  
  audit:
    log_access: true
    log_modifications: true
    log_deletions: true
    retention: 7y

inputIntent:
  name: ExerciseDataSubjectRight
  version: v1
  proposes:
    dataState: DataSubjectRequest
    version: v1
  supplies:
    required:
      - subject_id
      - right_type
      - verification_token
  
  governance:
    applies_to: CustomerDataGovernance
    
    actions:
      - right: access
        workflow: GenerateDataExport
      - right: rectification
        workflow: UpdateSubjectData
      - right: erasure
        workflow: EraseSubjectData
      - right: portability
        workflow: ExportSubjectData
  
  verification:
    required: true
    method: multi_factor
    
  audit:
    required: true
    category: data_subject_request
```

**Key Features**:
- External API integration with credentials
- Caching and retry for external calls
- Edge device with offline buffering
- Local processing rules on edge
- Data classification (PII levels)
- Retention policies with justifications
- Subject rights (access, rectification, erasure, portability)
- Cross-border transfer restrictions
- Data residency requirements
- Audit trail requirements
- Regulatory compliance (GDPR, CCPA, HIPAA)

---

## SUMMARY

These examples demonstrate:

1. **Data modeling** with evolution and constraints
2. **UI binding** to data versions
3. **Input validation** and deferred constraints
4. **Work Unit Contracts** with typed specifications (NEW in v1.2)
5. **Flow orchestration** with atomic groups and compensation
6. **Worker topology** with multi-region support
7. **Policy enforcement** with RBAC/ABAC
8. **Signal-driven scheduling** with operational feedback
9. **Cross-cutting concerns** like correlation and idempotency
10. **End-to-end integration** of all layers
11. **Contract coverage** for validation completeness (NEW in v1.2)
12. **Entity-scoped authority** with migration and governance (NEW in v1.3)
13. **Storage roles** distinguishing control from data (NEW in v1.3)
14. **Relationships** with cardinality and causality (NEW in v1.3)
15. **View-level invariants** for correctness enforcement (NEW in v1.3)
16. **CAS tokens** for write safety across migrations (NEW in v1.3)
17. **UI Composition** with semantic view grouping (NEW in v1.4)
18. **Experiences** for application-level navigation (NEW in v1.4)
19. **Field Permissions** with visibility and masking (NEW in v1.4)
20. **Navigation** with purposes, hierarchy, and indicators (NEW in v1.4)
21. **Form Intent Binding** for UI-to-intent integration (NEW in v1.5)
22. **View Materialization Contracts** with retrieval and temporal semantics (NEW in v1.5)
23. **Intent Delivery Contracts** with retry and timeout (NEW in v1.5)
24. **Session Contracts** with multi-device sync and offline support (NEW in v1.5)
25. **Indicator Source Binding** for dynamic UI indicators (NEW in v1.5)
26. **Composition Fetch Semantics** with prefetch and caching (NEW in v1.5)
27. **Mask Expression Grammar** for policy-based PII masking (NEW in v1.5)
28. **Navigation Semantics** with hierarchical breadcrumbs (NEW in v1.5)
29. **View Availability Policy** for graceful degradation (NEW in v1.5)
30. **Presentation Hints** for responsive and accessible rendering (NEW in v1.5)
31. **Surface Container Semantics** for modals, sidebars, and overlays (NEW in v1.5)
32. **Accessibility Preferences** with WCAG compliance (NEW in v1.5)
33. **Asset Semantics** for file uploads and media handling (NEW in v1.5)
34. **Search Semantics** with full-text indexing and faceting (NEW in v1.5)
35. **Scheduled Triggers** for cron-based automation (NEW in v1.5)
36. **Notification Channels** for email, push, and SMS (NEW in v1.5)
37. **Collaborative Sessions** for real-time multi-user interactions (NEW in v1.5)
38. **Structured Content Types** for rich document editing (NEW in v1.5)
39. **Durable Workflows** with human tasks and delegation (NEW in v1.5)
40. **Delegation Consent** for task reassignment with approval (NEW in v1.5)
41. **External Integration** for third-party API composition (NEW in v1.5)
42. **Data Governance** for GDPR/CCPA compliance (NEW in v1.5)
43. **Edge Device Semantics** for IoT and offline processing (NEW in v1.5)

All examples follow the principle: **Intent-first, runtime-decided**.

