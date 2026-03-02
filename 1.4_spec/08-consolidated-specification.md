# System Mechanics Specification v1.4 - Consolidated Reference

## Document Purpose

This is a consolidated, self-contained reference specification that integrates all grammar elements, addendums, and clarifications. Use this as the authoritative source for implementation.

---

## EXECUTIVE SUMMARY

### What This Specification Defines

A universal, intent-first framework for describing distributed systems at the mechanics level, independent of architectural style, deployment topology, or technology choices.

### Core Abstractions

1. **Data**: Structure, lifecycle, evolution, constraints
2. **UI**: Presentation views bound to data versions
3. **Input**: User/system proposals with validation
4. **SMS**: Flow orchestration as finite state machines
5. **WTS**: Worker topology, placement, and scaling
6. **Policy**: Authorization and governance
7. **Signals**: Operational feedback for scheduling
8. **Execution**: Transport-agnostic invocation
9. **Authority**: Entity-scoped write ownership (NEW in v1.3)
10. **Relationships**: Semantic entity linkage (NEW in v1.3)
11. **Invariants**: View-level correctness enforcement (NEW in v1.3)
12. **UI Composition**: Semantic view grouping (NEW in v1.4)
13. **Experiences**: Application navigation contexts (NEW in v1.4)

### Key Principles

1. **Intent over mechanism**: Describe what, not how
2. **Local over global**: Avoid coordination overhead
3. **Eventual over immediate**: Embrace async for scale
4. **Explicit over implicit**: Make coupling visible
5. **Safe over fast**: Correctness first, optimize second
6. **Evolvable over static**: Change is constant
7. **Observable over opaque**: Debugging must be possible

---

## COMPLETE GRAMMAR (NORMATIVE)

### Document Structure

```yaml
specification:
  version: v1.4
  system:
    name: <string>
    description: <string>
  
  data:
    types: []
    states: []
    constraints: []
    evolutions: []
    materializations: []
    transitions: []
  
  # NEW in v1.3 - Authority and Relationships
  authority:
    models: []
    states: []
    
  relationships: []  # NEW in v1.3
  
  storage:  # NEW in v1.3
    roles: []
  
  ui:
    views: []
    bindings: []
    compositions: []  # NEW in v1.4
    experiences: []   # NEW in v1.4
    navigation:       # NEW in v1.4
      purposes: []
      scopes: []
  
  input:
    intents: []
  
  contracts:  # NEW in v1.2
    workUnitContracts: []
  
  sms:
    flows: []
  
  wts:
    locations: []
    worker_classes: []
    workers: []
    placements: []
    scaling_policies: []
    failure_domains: []
  
  policy:
    policies: []
    policy_sets: []
    data_policies: []
    artifacts: []
  
  signals:
    definitions: []
  
  realms: []
  
  invariants: []  # NEW in v1.3
```

---

## DATA LAYER (COMPLETE)

### DataType

```yaml
dataType:
  name: <string>
  version: <vN>
  description: <string>
  fields:
    <fieldName>:
      type: <type>
      required: <boolean>
      default: <value>
```

**Types**: `string`, `integer`, `decimal`, `boolean`, `uuid`, `timestamp`, `duration`, `enum`, `array`, `reference`

### DataState

```yaml
dataState:
  name: <string>
  type: <DataType>
  version: <vN>
  lifecycle: intermediate | persistent | materialized
  owner: <string>
  constraintPolicy:
    enforcement: none | best_effort | strict | deferred
    on_violation: reject | log | compensate
  evolutionPolicy:
    allowed: [additive, constraint_refinement, ...]
    forbidden: [destructive, semantic_break, ...]
  storage:
    consistency: strong | eventual
    durability: durable | transient
```

### Constraint

```yaml
constraint:
  name: <string>
  appliesTo: <DataType>
  description: <string>
  rules:
    <version>: <expression>
  severity: error | warning
```

### DataEvolution

```yaml
dataEvolution:
  name: <string>
  from:
    type: <DataType>
    version: <vN>
  to:
    type: <DataType>
    version: <vN+1>
  changes:
    - addField: {name: <string>, type: <type>, default: <value>}
    - removeField: <string>
    - renameField: {from: <string>, to: <string>}
    - changeType: {field: <string>, from: <type>, to: <type>}
    - extendEnum: {field: <string>, values: []}
    - refineConstraint: {constraint: <string>, tighter: <bool>}
  compatibility:
    read: backward | forward | none
    write: backward | forward | none
  migration:
    strategy: shadow | direct | batch
    validation_period: <duration>
```

### Materialization

```yaml
materialization:
  name: <string>
  description: <string>
  source: <DataState> | [<DataState>, ...]
  targetState: <DataState>
  transformation: <expression>
  freshness:
    maxStaleness: <duration>
    strategy: pull | push
  evolution:
    strategy: rebuild | incremental
    blocking: true | false
  triggers:
    - on_change: <DataState>
    - schedule: <cron>
```

### Transition

```yaml
transition:
  name: <string>
  description: <string>
  from: <DataState>
  to: <DataState>
  triggeredBy:
    type: inputIntent | smsFlow | event
    name: <string>
  preconditions: []
  postconditions: []
  guarantees: []
  idempotency:
    strategy: deterministic_key | content_hash
    key: <expression> | [<expression>, ...]
    scope: global | per_subject | per_realm
    ttl: <duration>
  authorization:
    policySet: <PolicySet>
```

---

## AUTHORITY & MUTABILITY LAYER (NEW in v1.3)

### Mutability

Defines write authority semantics for data models.

```yaml
mutability:
  scope: entity | append-only
  exclusive: true | false
  authority: <authority_ref>
```

**Scope Types**:
- `entity`: Each entity has independent write authority
- `append-only`: Data is immutable once written

### Authority

Defines write authority resolution and transition semantics.

```yaml
authority:
  scope: entity | model
  resolution: <expression>
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

### Authority State

Runtime state of authority for an entity.

```yaml
authorityState:
  entity_id: <uuid>
  authority_region: <string>
  authority_epoch: <integer>
  authority_lease: <uuid>
  authority_status: ACTIVE | TRANSITIONING
  entity_version: <integer>
```

### CAS Token

Compare-And-Set token structure for write safety.

```yaml
cas:
  token_fields:
    - entity_id
    - entity_version
    - authority_epoch
    - authority_region
    - lease_id
```

---

## STORAGE ROLES (NEW in v1.3)

### Storage Role Declaration

Distinguishes control state from data state.

```yaml
storage:
  role: data | control
  rebuildable: true | false
```

| Role | Characteristics | Examples |
|------|-----------------|----------|
| `data` | Append-only, rebuildable, partitionable | Event streams, entity state |
| `control` | Strongly consistent, small, coordination | Authority KV, leases |

---

## RELATIONSHIP LAYER (NEW in v1.3)

### Relationship

Defines semantic relationships between models.

```yaml
relationship:
  name: <string>
  from: <model_ref>
  to: <model_ref>
  type: <identifier>
  cardinality: one-to-one | one-to-many | many-to-one | many-to-many
  semantics:
    causal: true | false
    ordered: true | false
```

**Cardinality**: Constrains valid relationship instances

**Semantics**:
- `causal`: The "from" entity must exist before "to" can reference it
- `ordered`: Events must be processed in causal order

**Rules**:
- Relationships SHALL NOT imply synchronous write coordination
- Relationships SHALL NOT imply shared authority

---

## INVARIANT LAYER (NEW in v1.3)

### Invariant Policy

Declares how invariants are enforced for a model.

```yaml
invariants:
  enforced_by: view | policy
  description: <string>
```

### View Invariant

Views may enforce or validate invariants.

```yaml
view:
  invariant:
    expression: <expression>
    on_violation: reject | flag | log
  authority_agnostic: true | false
  epoch_tolerant: true | false
```

### Formal Invariants

| ID | Invariant |
|----|-----------|
| I1 | Authority Singularity: One write-authoritative region per entity |
| I2 | Epoch Monotonicity: Authority epochs never decrease |
| I3 | No Overlapping Authority: No concurrent writes from multiple regions |
| I4 | CAS Safety: Writes validated against epoch and version |
| I5 | View Continuity: Views readable during transitions |
| I6 | Rebuildability: All derived state reconstructible |
| I7 | Governance Compliance: Transitions respect policies |
| I8 | Failure Determinism: Unambiguous authority after failure |

---

## AVAILABILITY SEMANTICS (NEW in v1.3)

### Entity-Scoped Availability

```yaml
availability:
  write_scope: entity
  pause_allowed: true | false
  read_continuity: best-effort | guaranteed
```

### Write Failure Strategy

```yaml
write_failure:
  strategy: reject | client_queue | regional_buffer
```

| Strategy | Description |
|----------|-------------|
| `reject` | Immediately fail writes (recommended) |
| `client_queue` | Buffer at client (non-authoritative) |
| `regional_buffer` | Buffer in adjacent region (expires) |

---

## UI LAYER (COMPLETE)

### PresentationView

```yaml
presentationView:
  name: <string>
  version: <vN>
  description: <string>
  consumes:
    dataState: <DataState>
    version: <vN>
  guarantees: []
  interactionModel:
    badges:
      - if: <expression>
        label: <string>
        color: <string>
        severity: info | warning | error
    actions:
      - if: <expression>
        action: <identifier>
        label: <string>
        intent: <InputIntent>
    filters:
      - field: <string>
        type: search | date_range | enum
    sorts:
      - field: <string>
        default: asc | desc
  layout:
    type: table | card | detail | list
    responsive: true | false
```

### PresentationBinding

```yaml
presentationBinding:
  name: <string>
  description: <string>
  rules:
    - if: <expression>
      use: <PresentationView>
      reason: <string>
    - else:
      use: <PresentationView>
      reason: <string>
```

### Field Permissions (NEW in v1.4)

Field permissions control visibility and masking of individual fields within a view:

```yaml
presentationView:
  # ... other properties ...
  fieldPermissions:
    - field: <string>
      visible: true | false | { policy: <policy_ref> }
      mask:
        unless: <expression>
        type: full | partial | hash
        reveal: last4 | first3 | none
```

**Mask Types**:
- `full`: Completely hide the value
- `partial`: Show portion specified by `reveal`
- `hash`: Show hashed representation

**Visibility**:
- Boolean: Static visibility
- Policy reference: Dynamic visibility based on policy evaluation

---

## UI COMPOSITION LAYER (NEW in v1.4)

### Overview

UI Composition enables semantic grouping of views into workspaces without prescribing layout. It describes WHAT views belong together and WHY, not HOW they should render.

### PresentationComposition

```yaml
presentationComposition:
  name: <string>
  version: <vN>
  description: <string>
  primary: <view_ref>
  navigation:
    label: <string>
    purpose: home | accounts | transactions | customers | alerts | 
             compliance | audit | reports | settings | help | authentication
    parent: <composition_ref>
    param: <string>
    policy_set: <policy_set_ref>
    indicator:
      when: <expression>
      severity: info | warning | error | critical
  related:
    - view: <view_ref>
      relationship: derived | detail | contextual | action | supplementary | historical
      cardinality: one | many
      required: true | false
      basis: <relationship_ref>
      alias: <string>
      trigger: <action>
      policy: <policy_ref>
      policy_set: <policy_set_ref>
      data_scope: { <field>: <expression> }
      navigation:
        label: <string>
        scope: within_composition | always_visible | on_action
```

**View Relationship Types**:

| Type | Description | Example |
|------|-------------|---------|
| `derived` | Computed from primary | Balance from Account |
| `detail` | Child items | Transactions of Account |
| `contextual` | Related entity | Account owner (Customer) |
| `action` | User-triggered | Transfer form |
| `supplementary` | Additional info | Settings, preferences |
| `historical` | Audit/history | Activity log |

**Navigation Scopes**:

| Scope | Description |
|-------|-------------|
| `within_composition` | Visible only when parent composition active |
| `always_visible` | Always in primary navigation |
| `on_action` | Visible when triggered by action |

### Experience

```yaml
experience:
  name: <string>
  version: <vN>
  description: <string>
  entry_point: <composition_ref>
  includes:
    - <composition_ref>
  policy_set: <policy_set_ref>
  unauthenticated:
    redirect_to: <composition_ref>
  on_unauthorized: conceal | indicate | redirect | deny
```

**On_unauthorized Behaviors**:
- `conceal`: Hide navigation item entirely
- `indicate`: Show but indicate inaccessible
- `redirect`: Redirect to another composition
- `deny`: Show access denied message

### Navigation

```yaml
navigation:
  purposes:
    - name: <identifier>
      description: <string>
  scopes:
    - name: <identifier>
      description: <string>
```

**Purpose Categories** group navigation items semantically. Runtime may render these as primary nav, utility nav, or settings menus.

**Hierarchy** is inferred from `parent` relationships in compositions, forming an acyclic tree.

---

## INPUT LAYER (COMPLETE)

### InputIntent

```yaml
inputIntent:
  name: <string>
  version: <vN>
  description: <string>
  proposes:
    dataState: <DataState>
    version: <vN>
  supplies:
    required: [<field>, ...]
    optional: [<field>, ...]
  constraints:
    guarantees: [<expression>, ...]
    defers: [<constraint_name>, ...]
  validation:
    timing: immediate | deferred
    on_failure: reject | warn | accept_with_flag
  transitions:
    to: <DataState>
    via: <Transition>
  defaults: {}
  metadata:
    ui_hints: {}
```

---

## WORK UNIT CONTRACT LAYER (NEW in v1.2)

### Overview

Work Unit Contracts provide typed specifications for work units referenced in SMS flows. 
They bridge the gap between named work units and their executable implementations.

### WorkUnitContract

```yaml
workUnitContract:
  name: <string>
  version: <vN>
  description: <string>
  
  input:
    schema:
      <fieldName>:
        type: <type>
        required: <boolean>
        default: <value>
    from: <DataState>
    validation:
      timing: immediate | deferred | lazy
      on_failure: reject | warn | log
  
  output:
    schema:
      <fieldName>:
        type: <type>
    to: <DataState>
    cardinality: one | zero_or_one | many | stream
  
  sideEffects:
    - type: database_write | database_read | event_publish | event_consume |
            api_call | cache_write | cache_read | cache_invalidate |
            file_write | file_read | message_send | notification | audit_log
      target: <string>
      description: <string>
      idempotency: <field_name>
      reversible: true | false
  
  preconditions:
    - <expression>
  
  postconditions:
    - <expression>
  
  errors:
    - code: <ERROR_CODE>
      retryable: true | false
      message: <string>
      httpStatus: <integer>
      backoff:
        strategy: none | linear | exponential | fixed | jittered
        initial: <duration>
        multiplier: <float>
        max: <duration>
      maxRetries: <integer>
      category: validation | authorization | not_found | conflict |
                timeout | unavailable | resource_exhausted |
                internal | external_dependency
  
  dependencies:
    databases:
      - name: <string>
        required: <boolean>
        timeout: <duration>
        fallback: error | default | cache | skip
    services: []
    caches: []
    queues: []
    externalApis: []
  
  timeout: <duration>
  
  idempotency:
    strategy: deterministic_key | content_hash | timestamp | none
    key: <expression> | [<expression>, ...]
    scope: global | per_subject | per_realm | per_boundary
    ttl: <duration>
  
  authorization:
    policySet: <PolicySet>
  
  boundary:
    type: execution | trust | consistency | geographic | logical
    name: <string>
```

### Contract Coverage

Contracts enable coverage analysis for SMS flows:

```yaml
contractCoverage:
  flow: <flow_name>
  totalWorkUnits: <integer>
  contractedWorkUnits: <integer>
  coveragePercent: <float>
  uncoveredWorkUnits: [<work_unit_name>, ...]
```

### Best Practices

| Practice | Reason |
|----------|--------|
| Always define idempotency for writes | Prevents duplicate operations |
| Specify backoff for retryable errors | Enables graceful degradation |
| Document all side effects | Operational visibility |
| Reference DataStates in from/to | Type safety and validation |
| Include timeout for external calls | Prevents resource exhaustion |
| Categorize errors consistently | Uniform error handling |

---

## SMS (FLOW) LAYER (COMPLETE)

### Flow Definition

```yaml
sms:
  flows:
    - name: <string>
      version: <vN>
      description: <string>
      triggeredBy:
        type: inputIntent | event | schedule
        name: <string>
      input:
        schema: <DataType>
        version: <vN>
      output:
        schema: <DataType>
        version: <vN>
      steps:
        - <step_definition>
      completion:
        policy: all | any | quorum
        quorum: <integer>
      failure:
        policy: abort | retry | compensate | continue
        max_attempts: <integer>
        backoff:
          strategy: exponential | linear | fixed
          initial: <duration>
          max: <duration>
        compensations:
          - on: <step>
            do: <work_unit>
        on_exhaustion: abort | continue | escalate
      authorization:
        policySet: <PolicySet>
      onPolicyChange: revalidate | drain | fail
      timeout: <duration>
      idempotency:
        scope: global | per_subject
        key: [<expression>, ...]
```

### Step Types

#### Simple Work Unit
```yaml
- work_unit: <string>
  boundary: <string>
  timeout: <duration>
```

#### Atomic Group
```yaml
- atomic <name>:
    boundary: <string>
    steps:
      - work_unit: <string>
      - work_unit: <string>
```

#### Parallel Group
```yaml
- parallel:
    steps:
      - work_unit: <string>
      - work_unit: <string>
    completion:
      policy: all | any | quorum
      quorum: <integer>
```

#### Conditional
```yaml
- if: <expression>
  then:
    - work_unit: <string>
  else:
    - work_unit: <string>
```

---

## WTS (WORKER TOPOLOGY) LAYER (COMPLETE)

### Location

```yaml
location:
  name: <string>
  scope: global | region | zone | cluster | runtime
  capabilities: [<string>, ...]
  properties:
    latency_to_user: minimal | low | medium | high
    provisioner: kubernetes | nomad | ecs | manual | browser
    provisionable: true | false
    region: <string>
    availability_zone: <string>
  constraints:
    data_residency: [<region>, ...]
    compliance: [<standard>, ...]
```

### Worker Class

```yaml
worker_class:
  name: <string>
  description: <string>
  runtime: <runtime_type>
  supports: [<work_unit>, ...]
  transport: [nats, http, grpc, unix_socket, in_process, ...]
  resources:
    cpu: <number>
    memory: <bytes>
    gpu: <number>
    storage: <bytes>
    network_bandwidth: <bytes_per_sec>
  healthcheck:
    type: http | tcp | exec
    endpoint: <string>
    interval: <duration>
    timeout: <duration>
    threshold: <integer>
```

### Worker

```yaml
worker:
  name: <string>
  worker_class: <string>
  version: <vN>
  acceptsInputs: [<InputIntent> <version>, ...]
  produces: [<DataState> <version>, ...]
  compatibleVersions: [<vN>, ...]
  capabilities: [<string>, ...]
  boundaries: [<string>, ...]
```

### Placement

```yaml
placement:
  name: <string>
  description: <string>
  worker_class: <string>
  locations: [<location>, ...]
  distribution:
    strategy: uniform | weighted | affinity
    weights: {}  # If weighted
    affinity_rules: []  # If affinity
  capacity:
    min: <integer>
    max: <integer>
    initial: <integer>
    preferred: <location>
  constraints:
    colocation: [<worker_class>, ...]  # Must be near these
    antiaffinity: [<worker_class>, ...]  # Must NOT be near these
```

### Scaling Policy

```yaml
scaling_policy:
  name: <string>
  target: <placement>
  evaluation_period: <duration>
  cooldown: <duration>
  signals:
    <signal_name> <operator> <value> => <action>
  rules:
    - condition: <expression>
      action: scale_up | scale_down | hold
      delta: <integer>
      reason: <string>
  limits:
    max_scale_up_rate: <integer_per_minute>
    max_scale_down_rate: <integer_per_minute>
  safety:
    min_stable_period: <duration>
    require_consecutive: <integer>
```

### Scheduler

```yaml
scheduler:
  name: <string>
  scope: <scope_type>
  authority: true | false
  observes: [<signal_type>, ...]
  strategy:
    algorithm: reactive | predictive | ml_based
    optimization_goal: latency | cost | utilization | balance
  boundaries:
    manages: [<location>, ...]
    coordinates_with: [<scheduler>, ...]
```

### Failure Domain

```yaml
failure_domain:
  name: <string>
  scope: <scope_type>
  affected_locations: [<location>, ...]
  isolation_policy: strict | best_effort
  failover:
    automatic: true | false
    target_domains: [<failure_domain>, ...]
```

---

## POLICY LAYER (COMPLETE)

### Policy

```yaml
policy:
  name: <string>
  version: <vN>
  description: <string>
  appliesTo:
    type: inputIntent | dataState | transition | materialization | presentationView | smsFlow
    name: <string>
    version: <vN>
  effect: allow | deny
  when: <expression>
  metadata:
    owner: <string>
    compliance: [<standard>, ...]
    audit_level: low | medium | high
```

### Policy Set

```yaml
policySet:
  name: <string>
  version: <vN>
  description: <string>
  resolution: deny_overrides | allow_overrides | first_match
  policies: [<policy_ref>, ...]
  conflict_resolution:
    strategy: explicit | implicit
    on_conflict: deny | log_and_deny | escalate
```

### Data Policy

```yaml
dataPolicy:
  name: <string>
  version: <vN>
  appliesTo:
    dataState: <name>
    version: <vN>
  permissions:
    read:
      allow: <expression>
      deny: <expression>
      filters: [<field>, ...]  # Field-level filtering
    write:
      allow: <expression>
      deny: <expression>
      fields: [<field>, ...]  # Field-level control
    transition:
      - name: <transition>
        allow: <expression>
        deny: <expression>
  row_level_security:
    filter: <expression>
```

### Policy Artifact (Distribution)

```yaml
policyArtifact:
  policy:
    name: <string>
    version: <vN>
    definition: <policy | policySet | dataPolicy>
  lifecycle:
    state: shadow | enforce | deprecated | revoked
    entered_at: <timestamp>
    supersedes: <vN | null>
    shadow_period: <duration>
    deprecation_period: <duration>
  scope:
    domain: <string>
    component: sms | wts | worker.<name> | ui.<name> | global
    realm: <string>
  distribution:
    priority: low | normal | high | critical
    channels: [<channel>, ...]
```

### Subject, Role, Attribute (RBAC/ABAC)

```yaml
subject:
  name: <string>
  type: user | service | system
  attributes:
    <key>: <type>

role:
  name: <string>
  description: <string>
  grants: [<permission>, ...]
  inherits: [<role>, ...]

attribute:
  name: <string>
  source: subject | data | environment | runtime
  type: <type>
  derivation: <expression>
```

---

## SIGNALS LAYER (COMPLETE)

### Signal Definition

```yaml
signal:
  type: capacity | health | latency | saturation | policy | backpressure | error | cost
  source:
    component: worker | sms | ui | infra | policy_engine
    name: <string>
    instance: <string>
    region: <string>
    realm: <string>
  subject:
    intent: <InputIntent>
    dataState: <DataState>
    flow: <SMS Flow>
    policy: <Policy>
  metrics:
    <key>: <number | string>
  dimensions:
    <key>: <string>
  severity: info | warn | error | critical
  timestamp: <iso8601>
  correlation:
    trace_id: <uuid>
    causation_id: <uuid>
```

### Signal Stream Configuration

```yaml
signalStream:
  name: <string>
  subjects: [<pattern>, ...]
  retention:
    policy: age | count | interest
    duration: <duration>
    max_bytes: <bytes>
  sampling:
    rate: <float>  # 0.0 - 1.0
    strategy: random | deterministic | adaptive
```

---

## EXECUTION SEMANTICS (COMPLETE)

### Execution Context

```yaml
executionContext:
  trace_id: <uuid>
  causation_id: <uuid>
  correlation_id: <uuid>
  parent_id: <uuid>
  actor:
    subject: <string>
    roles: [<string>, ...]
    attributes: {}
  environment:
    region: <string>
    realm: <string>
    execution_domain: <string>
  timing:
    started_at: <timestamp>
    deadline: <timestamp>
  metadata: {}
```

### Execution Outcome

```yaml
executionOutcome:
  type: success | rejection | failure
  retryable: true | false
  reason: <string>
  details:
    error_code: <string>
    error_class: <error_type>
    retry_after: <duration>
  context:
    trace_id: <uuid>
    step: <string>
    attempts: <integer>
```

### Idempotency Configuration

```yaml
idempotency:
  strategy: deterministic_key | content_hash | timestamp | none
  key: <expression> | [<expression>, ...]
  scope: global | per_subject | per_realm | per_boundary
  ttl: <duration>
  storage:
    backend: memory | redis | database
    max_entries: <integer>
  collision_handling: log | reject | override
```

### Time Constraint

```yaml
timeConstraint:
  type: ttl | deadline | staleness | window | rate_limit
  value: <duration>
  enforcement: strict | advisory
  on_violation: fail | warn | continue
```

---

## BOUNDARY SEMANTICS (COMPLETE)

### Boundary Definition

```yaml
boundary:
  name: <string>
  type: execution | trust | consistency | geographic | logical
  properties:
    consistency: strong | eventual | none
    trust_level: trusted | untrusted | restricted
    region: <string>
    zone: <string>
    execution_domain: <string>
  guarantees:
    - <guarantee_name>
  guarantees_lost:
    - <guarantee_name>
  crossing_cost:
    latency: <duration>
    reliability: <float>
```

### Boundary Crossing Rules

**Atomic Groups**:
- MUST NOT cross consistency boundaries
- MUST NOT cross trust boundaries
- MAY cross execution domain boundaries if consistency preserved
- MAY cross geographic boundaries if latency acceptable

**Work Units**:
- MAY cross any boundary
- MUST declare boundary requirements
- MUST handle crossing failures

**Data**:
- MUST respect trust boundaries (encryption/filtering)
- MUST respect geographic boundaries (data residency)
- MAY cache across boundaries if consistency allows

---

## REALM LAYER (COMPLETE)

### Realm Definition

```yaml
realm:
  name: <string>
  description: <string>
  isolation:
    data: strict | shared
    policy: strict | shared
    signals: isolated | shared
    workers: dedicated | shared
  quota:
    workers: <integer>
    storage: <bytes>
    compute: <cpu_units>
    network: <bytes_per_sec>
  residency:
    allowed_regions: [<region>, ...]
    data_sovereignty: <country_code>
  tenant:
    id: <string>
    tier: free | pro | enterprise
```

---

## FAILURE HANDLING (COMPLETE)

### Failure Classifications

```yaml
errorClassification:
  categories:
    - name: client_error
      retryable: false
      includes: [validation, authorization, not_found, conflict]
    
    - name: infrastructure_error
      retryable: true
      includes: [timeout, network, unavailable, resource_exhausted]
    
    - name: system_error
      retryable: conditional
      includes: [internal, data_corruption, invariant_violation]
```

### Retry Configuration

```yaml
retryPolicy:
  enabled: true | false
  max_attempts: <integer>
  backoff:
    strategy: exponential | linear | fixed | jittered
    initial: <duration>
    multiplier: <float>
    max: <duration>
    jitter: <float>
  conditions:
    retry_on: [<error_type>, ...]
    dont_retry_on: [<error_type>, ...]
  circuit_breaker:
    enabled: true | false
    threshold: <integer>
    timeout: <duration>
    half_open_requests: <integer>
```

---

## CONTROL PLANE ARCHITECTURE (COMPLETE)

### Control Plane Components

```yaml
controlPlane:
  components:
    - artifact_manager:
        manages: [spec, policy, topology]
        storage: git | object_store
        distribution: jetstream
    
    - lifecycle_manager:
        handles: [shadow, promote, deprecate, revoke]
        coordination: leader_election
    
    - validation_service:
        validates: [grammar, semantics, cross_refs]
        on_invalid: reject | warn | log
    
    - distribution_layer:
        transport: nats
        streams: [SPEC_DEFS, POLICY_DEFS, LIFECYCLE_EVENTS]
        replication: 3
```

### Distribution Configuration

```yaml
distribution:
  transport: nats
  streams:
    - name: SPEC_DEFS
      subjects: [spec.>]
      retention: limits
      max_msgs_per_subject: 1
      storage: file
      replicas: 3
    
    - name: POLICY_DEFS
      subjects: [policy.>]
      retention: limits
      max_msgs_per_subject: 1
      storage: file
      replicas: 3
    
    - name: LIFECYCLE_EVENTS
      subjects: [lifecycle.>]
      retention: interest
      max_age: 7d
      replicas: 3
  
  kv_buckets:
    - name: WORKER_REGISTRY
      ttl: 15s
      replicas: 3
```

---

## OBSERVABILITY (COMPLETE)

### Observability Requirements

```yaml
observability:
  tracing:
    enabled: true
    sampling_rate: 1.0
    propagation: [trace_id, causation_id, correlation_id]
    backends: [jaeger, zipkin, honeycomb]
  
  metrics:
    enabled: true
    export_interval: 10s
    metrics:
      - flow_duration
      - step_duration
      - policy_evaluations
      - cache_hit_rate
      - worker_capacity
      - signal_latency
  
  logging:
    enabled: true
    level: info | debug
    structured: true
    includes: [trace_id, causation_id, actor]
  
  audit:
    enabled: true
    targets: [policy_decisions, data_transitions, authorization]
    retention: 90d
    immutable: true
```

---

## IMPLEMENTATION PHASES

### Phase 1: Core Runtime (Weeks 1-2)
- [ ] Parse specification files
- [ ] Validate grammar and semantics
- [ ] Generate types from DataTypes
- [ ] Implement flow engine (FSM)
- [ ] Execute work units in-process
- [ ] Validate constraints (strict mode)
- [ ] Basic logging and errors

**Deliverable**: Single-binary demo

### Phase 2: Control Plane (Weeks 3-4)
- [ ] NATS integration
- [ ] Spec distribution via JetStream
- [ ] Worker registration via KV
- [ ] Heartbeat protocol
- [ ] Policy loading and caching
- [ ] Signal emission

**Deliverable**: Multi-worker system

### Phase 3: Policy Enforcement (Weeks 5-6)
- [ ] Policy evaluation engine
- [ ] Local policy registry
- [ ] Shadow policy support
- [ ] Lifecycle management
- [ ] Authorization gates (UI, SMS, Workers)
- [ ] Audit logging

**Deliverable**: Secure system with RBAC/ABAC

### Phase 4: Advanced Execution (Weeks 7-8)
- [ ] Atomic group enforcement
- [ ] Compensation patterns
- [ ] Version-aware routing
- [ ] Locality-first invocation
- [ ] Routing cache optimization
- [ ] Graceful draining

**Deliverable**: Production-grade execution

### Phase 5: Observability (Weeks 9-10)
- [ ] Distributed tracing
- [ ] Correlation propagation
- [ ] Signal aggregation
- [ ] Metrics collection
- [ ] Dashboard/visualization
- [ ] Alerting

**Deliverable**: Observable system

### Phase 6: Multi-Region (Weeks 11-12)
- [ ] Regional deployment
- [ ] Cross-region policy distribution
- [ ] Regional schedulers
- [ ] Automatic failover
- [ ] Partition handling
- [ ] Chaos testing

**Deliverable**: Globally distributed system

---

## VALIDATION CHECKLIST

### Specification Validation

- [ ] All DataTypes have versions
- [ ] All constraints reference existing DataTypes
- [ ] All evolutions form linear chains (no skips)
- [ ] All InputIntents specify transitions
- [ ] All PresentationViews bind to materialized DataStates
- [ ] All SMS flows reference existing work units
- [ ] All atomic groups have single boundary
- [ ] All policies have explicit effects
- [ ] All workers declare compatible versions
- [ ] All placements have min ≤ max
- [ ] All schedulers have unique authority per scope
- [ ] All compensations reference existing work units
- [ ] All idempotency keys are deterministic
- [ ] All transitions have idempotency specifications

### WorkUnitContract Validation (NEW in v1.2)

- [ ] All WorkUnitContracts have name and version
- [ ] All WorkUnitContracts have input and output schemas
- [ ] All input.from references valid DataStates
- [ ] All output.to references valid DataStates
- [ ] All retryable errors have backoff strategies
- [ ] All database_write side effects have idempotency keys
- [ ] All flow work units have corresponding contracts
- [ ] Contract versions match flow references
- [ ] All dependencies reference valid resources
- [ ] All preconditions use valid expressions

### Runtime Validation

- [ ] Flow execution follows FSM semantics
- [ ] Atomic groups enforce co-location
- [ ] Policies evaluated locally
- [ ] Idempotency prevents duplicates
- [ ] Correlation propagates end-to-end
- [ ] Signals emitted correctly
- [ ] Boundaries enforced at runtime
- [ ] Graceful shutdown works
- [ ] Version negotiation succeeds
- [ ] Failure handling follows specifications

### Operational Validation

- [ ] Zero-downtime deployment works
- [ ] Rollback completes without errors
- [ ] Policy changes propagate within SLA
- [ ] Workers auto-recover from crashes
- [ ] Regional failover is automatic
- [ ] Control plane outage handled gracefully
- [ ] Chaos testing passes
- [ ] Performance SLAs met
- [ ] Security audit passes
- [ ] Compliance requirements met

---

## CONFORMANCE LEVELS

### Level 0: Grammar Conformance
- Parse specifications correctly
- Validate structural rules
- Validate semantic rules
- Generate types from specifications

### Level 1: Execution Conformance
- Level 0 +
- Execute flows as FSMs
- Enforce atomic groups
- Validate constraints
- Implement idempotency

### Level 2: Distribution Conformance
- Level 1 +
- Control plane integration
- Worker discovery
- Policy distribution
- Signal emission
- Routing optimization

### Level 3: Production Conformance
- Level 2 +
- Multi-region support
- Zero-downtime evolution
- Chaos resilience
- Full observability
- Security audit ready

### Level 4: Authority Conformance (NEW in v1.3)
- Level 3 +
- Entity-scoped authority enforcement
- Authority transition protocol
- CAS token validation
- View authority independence
- Epoch monotonicity enforcement
- Governance-bound migration

---

## GLOSSARY

### Core Terms

**Work Unit**: Smallest executable operation with single responsibility

**Work Unit Contract**: Typed specification for a work unit defining input/output schemas, side effects, errors, and dependencies (NEW in v1.2)

**Flow**: Directed graph of work units describing causal structure

**Atomic Group**: Set of work units that must execute together

**Boundary**: Point where guarantees, trust, or semantics change

**DataState**: Data with lifecycle, constraints, and evolution rules

**InputIntent**: What user/system proposes before validation

**PresentationView**: UI contract bound to data version

**Policy**: Declarative authorization rule

**Signal**: Operational observation about system state

**Scheduler**: Determines desired worker topology

**Worker**: Stateless execution instance

**Realm**: Isolation boundary for multi-tenancy

**Transition**: Typed edge between data states

**Materialization**: Derived, regenerable state

**Execution Context**: Correlation and causation metadata

### v1.3 Terms (NEW)

**Authority**: Exclusive write ownership for an entity

**Authority Epoch**: Monotonic counter incremented on authority transfer

**Authority Lease**: Time-bound grant of write authority

**CAS Token**: Compare-And-Set token containing entity, version, and epoch

**Control Storage**: Strongly consistent coordination state (authority, leases)

**Data Storage**: Append-only or rebuildable domain state

**Entity-Scoped Authority**: Write authority resolved per entity, not per model

**Invariant**: Correctness condition that must always hold

**Mutability**: Declaration of write semantics for a model

**Relationship**: Semantic linkage between entities with cardinality

**View Authority Independence**: View tolerates events from multiple regions/epochs

### Relationship Terms

**Trigger**: What causes work to begin

**Transition**: How state changes

**Compensation**: Reverse operation for failure handling

**Evolution**: How schemas change over time

**Binding**: How UI selects appropriate view

**Scope**: Authority boundary for decisions

### Operational Terms

**Shadow**: Parallel evaluation without enforcement

**Drain**: Graceful shutdown allowing work completion

**Promotion**: Shadow → enforced state transition

**Deprecation**: Enforced → deprecated state transition

**Revocation**: Immediate removal from enforcement

**Heartbeat**: Periodic liveness and health signal

**TTL**: Time-to-live, automatic expiry

**Idempotency Key**: Deterministic deduplication identifier

---

## IMPLEMENTATION PRIORITIES

### Must Have (P0)
1. Flow execution as FSM
2. Constraint validation
3. Idempotency enforcement
4. Policy evaluation (local)
5. Correlation propagation
6. Atomic group enforcement
7. Version compatibility checking

### Should Have (P1)
8. Control plane distribution
9. Worker discovery
10. Routing optimization
11. Signal emission
12. Graceful draining
13. Shadow policies
14. Multi-version support

### Nice to Have (P2)
15. Scheduler integration
16. Chaos testing framework
17. Auto-scaling
18. Multi-region deployment
19. Visualization/dashboards
20. Performance optimization

---

## CRITICAL SUCCESS FACTORS

An implementation succeeds when:

1. **Correctness**: No data corruption or duplication
2. **Performance**: <10ms p99 latency for local execution
3. **Resilience**: Survives 20% random failures
4. **Evolution**: Zero-downtime version upgrades
5. **Security**: No authorization bypasses
6. **Scale**: Linear scaling to 10,000+ workers
7. **Observability**: End-to-end request tracing
8. **Operability**: Simple deployment and debugging

---

## ANTI-PATTERNS TO AVOID

### In Specification Authoring
1. ❌ Embedding transport details in flows
2. ❌ Skipping versions in evolution
3. ❌ Implicit coupling between components
4. ❌ Undefined idempotency keys
5. ❌ Cross-boundary atomic groups
6. ❌ Unversioned data types
7. ❌ Missing policy effects

### In Implementation
1. ❌ Synchronous central coordinator
2. ❌ Coupling workers to each other
3. ❌ Storing state in workers
4. ❌ Skipping shadow validation
5. ❌ Manual cleanup processes
6. ❌ Global locks or transactions
7. ❌ Hardcoding transport mechanisms
8. ❌ Inferring missing semantics

---

## TESTING REQUIREMENTS

### Unit Testing
- Grammar parsing
- Constraint validation
- Policy evaluation
- Idempotency checking
- Version compatibility

### Integration Testing
- Multi-component flows
- Policy distribution
- Worker discovery
- Signal propagation
- Routing decisions

### Chaos Testing
- Random worker kills
- Network partitions
- Policy corruption
- Signal delays
- Control plane outage

### Performance Testing
- Throughput (requests/sec)
- Latency (p50, p95, p99)
- Scaling (workers, regions)
- Cache efficiency
- Signal overhead

---

## DEPLOYMENT MODELS

### Single Binary (Development/Small Scale)
- All components in one process
- In-memory communication
- No NATS required
- Simple deployment

### Distributed (Production)
- Separate flow engines and workers
- NATS for coordination
- Multi-region deployment
- Horizontal scaling

### Edge (IoT/Browser)
- Lightweight workers in edge locations
- NATS for global coordination
- Opportunistic execution
- Graceful degradation

### Hybrid (Common)
- Core services in cloud
- Edge workers for latency
- Browser workers for interaction
- Unified coordination

---

## SPECIFICATION COMPLETENESS STATEMENT

This consolidated specification defines:

✅ **Data modeling** with versioning and evolution
✅ **UI contracts** bound to data versions
✅ **Input validation** with deferred constraints
✅ **Work Unit Contracts** with typed specifications (NEW in v1.2)
✅ **Flow orchestration** with atomic groups and compensation
✅ **Worker topology** with multi-region support
✅ **Policy enforcement** with RBAC/ABAC
✅ **Signal-driven scheduling** with operational feedback
✅ **Transport-agnostic execution** with locality optimization
✅ **Correlation & tracing** for observability
✅ **Idempotency** for correctness
✅ **Time constraints** for SLAs
✅ **Realm isolation** for multi-tenancy
✅ **Failure handling** with classification
✅ **Boundary semantics** for correctness
✅ **Contract coverage** for validation completeness (NEW in v1.2)
✅ **Entity-scoped authority** with migration support (NEW in v1.3)
✅ **Relationship grammar** with cardinality and causality (NEW in v1.3)
✅ **Invariant-oriented modeling** via views (NEW in v1.3)
✅ **UI Composition** with semantic view grouping (NEW in v1.4)
✅ **Experiences** for application-level navigation (NEW in v1.4)
✅ **Field Permissions** with visibility and masking (NEW in v1.4)
✅ **Navigation Grammar** with purposes and scopes (NEW in v1.4)

The specification is:
- **Complete**: No semantic gaps
- **Consistent**: No contradictions
- **Implementable**: Sufficient detail
- **Testable**: Clear success criteria
- **Evolvable**: Supports gradual enhancement

This is the authoritative reference for System Mechanics Specification v1.4.

---

## NEXT STEPS FOR IMPLEMENTERS

1. **Read Design Rationale** to understand the "why"
2. **Study Grammar Examples** to see patterns
3. **Reference Grammar Reference** for formal rules
4. **Follow Implementation Specification** for building
5. **Use this Consolidated Specification** as single source of truth
6. **Validate against Conformance Levels** to measure progress
7. **Test with Chaos Testing** to prove resilience

**Start simple** (single binary, in-process)
**Evolve incrementally** (add NATS, distribution, regions)
**Validate continuously** (grammar, execution, operations)

---

**Version**: 1.4
**Status**: Production Ready
**Last Updated**: 2026-01-11
**Changelog v1.4**: Added UI Composition, Experiences, Field Permissions, and Navigation grammar
**Changelog v1.3**: Added Entity-scoped Authority, Relationships, Invariants, Storage Roles
**Changelog v1.2**: Added WorkUnitContract grammar for typed work unit specifications

