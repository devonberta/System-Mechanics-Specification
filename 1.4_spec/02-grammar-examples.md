# System Mechanics Specification - Grammar Examples v1.4

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

All examples follow the principle: **Intent-first, runtime-decided**.

