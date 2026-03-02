# System Mechanics Specification - Grammar Reference v1.5

## Overview

This document provides the complete formal grammar for the Unified System Specification, covering Data, Evolution, UI, Input, Flow (SMS), Worker Topology (WTS), Policy, Signals, Scheduling, Authority, Multi-Region Survivability, UI Composition, Experiences, and v1.5 Extensions (Forms, Search, Assets, Sessions, and more).

---

## Core Principles

### Normative Principles

1. **Intent-First, Runtime-Decided**
   - The specification declares intent, guarantees, and prohibitions
   - Execution strategy, transport, locality, and optimization are runtime responsibilities

2. **Versioned Truth**
   - All persisted data, materialized data, inputs, UI, and policies MUST be versioned
   - Semantic change ⇒ new version, even if structure is unchanged

3. **Linear Evolution for Persisted Data**
   - Persisted data types evolve sequentially
   - Versions may not be skipped

4. **UI and Input Bind to Data, Not Services**
   - UI binds to materialized data versions
   - Input binds to intermediate or target data states

5. **Zero Downtime by Construction**
   - Shadowing, promotion, and retirement are first-class and mandatory
   - Runtime may run multiple versions concurrently

6. **Declarative Authorization**
   - Authorization is expressed declaratively
   - Enforcement mechanics are runtime-defined but locally evaluated

---

## PART I: DATA GRAMMAR

### DataType

Defines structure only.

```yaml
dataType:
  name: <string>
  version: <vN>
  fields:
    <fieldName>: <type>
```

**Purpose**: Structural definition without behavior or lifecycle

---

### Constraint

Defines semantic truth, version-aware.

```yaml
constraint:
  name: <string>
  appliesTo: <DataType>
  rules:
    <version>: <expression>
```

**Purpose**: Semantic invariants that must hold true for specific data versions

---

### DataState

Defines lifecycle, enforcement, and evolution boundaries.

```yaml
dataState:
  name: <string>
  type: <DataType>
  lifecycle: intermediate | persistent | materialized
  constraintPolicy:
    enforcement: none | best_effort | strict | deferred
  evolutionPolicy:
    allowed: []
    forbidden: []
```

**Lifecycle Types**:
- `intermediate`: Transient state during processing
- `persistent`: Durable, authoritative state
- `materialized`: Derived, regenerable state

**Enforcement Levels**:
- `none`: No validation
- `best_effort`: Validate if possible
- `strict`: Block on violation
- `deferred`: Validate asynchronously

---

### Mutability (NEW in v1.3)

Defines write authority semantics for data.

```yaml
mutability:
  scope: entity | append-only
  exclusive: true | false
  authority: <authority_ref>
```

**Scope Types**:
- `entity`: Each entity has independent write authority
- `append-only`: Data is immutable once written

**Rules**:
- When `exclusive: true`, exactly one authority may accept writes at any time
- Entities SHOULD NOT require multi-entity CAS operations
- Authority reference links to authority declaration

---

### Authority (NEW in v1.3)

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

**Scope**:
- `entity`: Authority resolved per entity (recommended)
- `model`: Authority resolved for entire model (legacy pattern)

**Resolution**: Expression determining authoritative region for an entity

**Migration Strategy**:
- `pause-and-cutover`: Writes pause during transition, cutover is atomic

**Purpose**: Enables safe, observable transfer of write authority between regions

---

### Authority State (NEW in v1.3)

Defines the runtime state of authority for an entity.

```yaml
authorityState:
  entity_id: <uuid>
  authority_region: <string>
  authority_epoch: <integer>
  authority_lease: <uuid>
  authority_status: ACTIVE | TRANSITIONING
  entity_version: <integer>
```

**States**:
- `ACTIVE(region, epoch)`: Region is accepting writes
- `TRANSITIONING(from_region, to_region, epoch)`: Authority is being transferred

**Invariants**:
- Authority epoch MUST be monotonic
- Only one region may be ACTIVE at any time
- During TRANSITIONING, no new writes are accepted

---

### Storage Role (NEW in v1.3)

Distinguishes control state from data state.

```yaml
storage:
  role: data | control
  rebuildable: true | false
```

**Roles**:
- `data`: Append-only or rebuildable event/state data
- `control`: Coordination, authority, or governance state

**Rules**:
- Control storage MUST be strongly consistent
- Control storage SHALL be modified only by designated control components
- Data storage MAY be partitioned, replayed, or materialized
- Implementations SHALL NOT treat control storage as domain data source

---

### Relationship (NEW in v1.3)

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

**Purpose**: Express semantic linkage, causality, and constraints between entities

**Cardinality**: Constrains valid relationship instances

**Semantics**:
- `causal`: The "from" entity must exist before "to" can reference it
- `ordered`: Events must be processed in causal order

**Rules**:
- Relationships SHALL NOT imply synchronous write coordination
- Relationships SHALL NOT imply shared authority
- Relationships SHALL NOT imply atomic multi-entity mutation

---

### Invariant Policy (NEW in v1.3)

Declares how invariants are enforced for a model.

```yaml
invariants:
  enforced_by: view | policy
  description: <string>
```

**Enforcement Options**:
- `view`: Invariants validated by derived views
- `policy`: Invariants enforced by policy evaluation

**Purpose**: Prevents embedding cross-entity invariants inside entities

---

### CAS Token (NEW in v1.3)

Defines Compare-And-Set token structure for write safety.

```yaml
cas:
  token_fields:
    - entity_id
    - entity_version
    - authority_epoch
    - authority_region
    - lease_id
```

**Validation Rules**:
- Request MUST be sent to authority_region
- authority_epoch MUST match current epoch
- entity_version MUST match current version
- lease_id MUST be valid

**Purpose**: Preserves linearizability across authority migrations

---

### DataEvolution

Declares allowed change (not migration steps).

```yaml
dataEvolution:
  name: <string>
  from:
    type: <DataType>
    version: <vN>
  to:
    type: <DataType>
    version: <vN+1>
  changes: []
  compatibility:
    read: backward | forward | none
    write: backward | forward | none
```

**Change Types**:
- `addField`: Add new field with default
- `removeField`: Remove field (requires backward compatibility)
- `extendEnum`: Add enum values
- `refineConstraint`: Tighten constraints

---

### Materialization

Defines derived, regenerable state.

```yaml
materialization:
  name: <string>
  source: <DataState | list>
  targetState: <DataState>
  freshness:
    maxStaleness: <duration>
  evolution:
    strategy: rebuild | incremental
    blocking: true | false
  authority_agnostic: true | false  # NEW in v1.3
  epoch_tolerant: true | false      # NEW in v1.3
```

**Strategies**:
- `rebuild`: Full regeneration on schema change
- `incremental`: Progressive update

**Authority Properties (NEW in v1.3)**:
- `authority_agnostic`: View tolerates events from multiple authoritative regions
- `epoch_tolerant`: View tolerates events from different authority epochs

**Rules**:
- Views derived from entity events SHALL be authority-agnostic unless explicitly stated
- Views MUST tolerate events originating from multiple regions
- Views MUST tolerate events written under different authority epochs
- Views SHALL NOT assume a fixed authoritative region for inputs

---

### Transition

Defines state transitions as first-class objects.

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
```

**Purpose**: Typed edges between states with explicit guarantees

---

## PART II: PRESENTATION (UI) GRAMMAR

### PresentationView

Defines UI contract bound to data.

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
        severity: info | warning | error | success
    actions:
      - if: <expression>
        action: <identifier>
        label: <string>
        intent: <InputIntent>
    filters:
      - field: <string>
        type: search | date_range | enum | range
    sorts:
      - field: <string>
        default: asc | desc
  fieldPermissions:  # NEW in v1.4
    - field: <string>
      visible: true | false | { policy: <policy_ref> }
      mask:
        unless: <expression>
        type: full | partial | hash
        reveal: last4 | first3 | none
```

**Guarantees**: What the UI promises to display or enable

**Field Permissions (NEW in v1.4)**:
- `visible`: Controls field visibility (boolean or policy reference)
- `mask.type`: How to mask the field (full, partial, hash)
- `mask.reveal`: What portion to reveal when partial masking
- `mask.unless`: Expression that allows full visibility

---

### PresentationBinding

Defines automatic UI selection based on context.

```yaml
presentationBinding:
  name: <string>
  rules:
    - if: <expression>
      use: <PresentationView>
```

**Purpose**: Version-aware UI routing

---

## PART III: INPUT (FORM) GRAMMAR

### InputIntent

Defines what a user or system may propose.

```yaml
inputIntent:
  name: <string>
  version: <vN>
  proposes:
    dataState: <DataState>
    version: <vN>
  supplies:
    required: []
    optional: []
  constraints:
    guarantees: []
    defers: []
  transitions:
    to: <DataState>
```

**Purpose**: Inverse data intent - what can be proposed before validation

---

## PART IV: WORK UNIT CONTRACTS (NEW in v1.2)

### Overview

Work Unit Contracts provide typed specifications for work units referenced in SMS flows. They define explicit input/output schemas, side effects, pre/postconditions, error handling, and dependencies.

**Purpose**: Bridge the gap between named work units in flows and their executable specifications.

---

### WorkUnitContract

Defines a typed contract for a work unit.

```yaml
workUnitContract:
  name: <string>
  version: <vN>
  description: <string>
  
  input:
    schema:
      <fieldName>: <type>
    from: <DataState>  # Optional
    validation:
      timing: immediate | deferred | lazy
      on_failure: reject | warn | log
  
  output:
    schema:
      <fieldName>: <type>
    to: <DataState>  # Optional
    cardinality: one | zero_or_one | many | stream
  
  sideEffects:
    - type: <side_effect_type>
      target: <string>
      description: <string>
      idempotency: <field_name>
      reversible: true | false
  
  preconditions: []
  postconditions: []
  
  errors:
    - code: <ERROR_CODE>
      retryable: true | false
      message: <string>
      httpStatus: <integer>
      backoff: none | linear | exponential | fixed | jittered
      maxRetries: <integer>
      category: <error_category>
  
  dependencies:
    databases: []
    services: []
    caches: []
    queues: []
    externalApis: []
  
  timeout: <duration>
  
  idempotency:
    strategy: deterministic_key | content_hash
    key: <expression>
    scope: global | per_subject | per_realm
    ttl: <duration>
```

**Required Fields**: `name`, `version`, `input.schema`, `output.schema`

---

### Side Effect Types

| Type | Description |
|------|-------------|
| `database_write` | Write to database (insert, update, delete) |
| `database_read` | Read from database |
| `event_publish` | Publish event to message broker |
| `event_consume` | Consume event from message broker |
| `api_call` | Call external or internal API |
| `cache_write` | Write to cache |
| `cache_read` | Read from cache |
| `cache_invalidate` | Invalidate cache entry |
| `file_write` | Write to file system or object storage |
| `file_read` | Read from file system or object storage |
| `message_send` | Send message (email, SMS, push notification) |
| `notification` | Send notification |
| `audit_log` | Write audit log entry |

---

### Error Categories

| Category | Description | Typically Retryable |
|----------|-------------|---------------------|
| `validation` | Input validation failure | No |
| `authorization` | Permission denied | No |
| `not_found` | Resource not found | No |
| `conflict` | Optimistic locking / conflict | Maybe |
| `timeout` | Operation timed out | Yes |
| `unavailable` | Service unavailable | Yes |
| `resource_exhausted` | Rate limit, quota exceeded | Yes (with backoff) |
| `internal` | Internal error | Maybe |
| `external_dependency` | External service failure | Yes |

---

### Best Practices

1. **Always specify idempotency** for database writes
2. **Define backoff strategies** for retryable errors
3. **Document all side effects** for operational awareness
4. **Reference DataStates** in input.from and output.to for type safety
5. **Include timeout** for work units calling external services
6. **Group related errors** by category for consistent handling

---

## PART V: SMS (FLOW) GRAMMAR

### SMS Flow

State Machine System flow definition.

```yaml
sms:
  flows:
    - name: <string>
      triggeredBy:
        inputIntent: <InputIntent>
      steps:
        - <operation>
      authorization:
        policySet: <PolicySet>
      onPolicyChange: revalidate | drain | fail
```

**Operations**:
- `validate`: constraint checking
- `enrich`: add derived data
- `persist`: commit to DataState
- `atomic`: atomic group execution
- `parallel`: parallel execution

---

### Atomic Group

Defines tightly coupled steps that must execute together.

```yaml
atomic_group:
  boundary: <boundary_name>
  steps:
    - work_unit: <name>
```

**Rules**:
- All steps MUST execute in the same boundary
- All steps MUST execute sequentially
- Failure of any step fails the group
- No partial success is permitted

---

## PART VI: WTS (WORKER TOPOLOGY SPEC) GRAMMAR

### Worker Declaration

```yaml
wts:
  workers:
    - name: <string>
      acceptsInputs: []
      produces: []
      compatibleVersions: []
      capabilities: []
```

**Purpose**: Declare what work a worker can perform

---

### Location Definition

```yaml
location:
  name: <string>
  scope: global | region | zone | cluster | runtime
  capabilities: []
  latency_to_user: low | medium | high
  provisioner: <string>
  provisionable: true | false
```

**Scope Hierarchy**:
- `global`: Worldwide
- `region`: Geographic region
- `zone`: Availability zone
- `cluster`: Compute cluster
- `runtime`: Execution domain

---

### Worker Class

```yaml
worker_class:
  name: <string>
  runtime: <runtime_type>
  supports: []
  transport: []
  resources:
    cpu: <number>
    memory: <string>
    gpu: <number>
```

**Purpose**: Define worker type and resource requirements

---

### Placement Declaration

```yaml
placement:
  name: <string>
  worker_class: <name>
  locations: []
  min: <integer>
  max: <integer>
  preferred: <location>
```

**Purpose**: Define where and how many workers should exist

---

### Scaling Policy

```yaml
scaling_policy:
  name: <string>
  target: <placement_name>
  signals:
    <signal_expression> => <scale_action>
  cooldown: <duration>
```

**Scale Actions**:
- `scale_up`: Increase worker count
- `scale_down`: Decrease worker count
- `hold`: Maintain current count

---

### Version-Aware Routing

```yaml
wts:
  routing:
    strategy: versioned
    rules:
      - if: <expression>
        routeTo: <worker>
```

**Purpose**: Route intents to version-compatible workers

---

## PART VII: POLICY & AUTHORIZATION GRAMMAR

### Subject, Role, Attribute

```yaml
subject:
  name: <string>
  attributes:
    <key>: <type>

role:
  name: <string>
  grants: []

attribute:
  name: <string>
  source: subject | data | environment | runtime
```

**Purpose**: RBAC and ABAC primitives

---

### Policy Definition

```yaml
policy:
  name: <string>
  version: <vN>
  appliesTo:
    type: inputIntent | dataState | transition | materialization | presentationView | smsFlow | presentationComposition | experience
    name: <string>
  effect: allow | deny
  when: <expression>
```

**Policy Target Types**:
- `inputIntent`: User/system input proposals
- `dataState`: Data access control
- `transition`: State change authorization
- `materialization`: View access
- `presentationView`: UI view access
- `smsFlow`: Flow execution authorization
- `presentationComposition`: Composition access (NEW in v1.4)
- `experience`: Experience access (NEW in v1.4)

**Effect Resolution**: Deny MUST override allow by default

---

### Policy Set

```yaml
policySet:
  name: <string>
  resolution: deny_overrides | allow_overrides
  policies: []
```

**Purpose**: Group policies with conflict resolution strategy

---

### Data Policy

```yaml
dataPolicy:
  name: <string>
  appliesTo:
    dataState: <name>
    version: <vN>
  permissions:
    read: <expression>
    write: <expression>
    transition: <expression>
```

**Purpose**: Fine-grained data access control

---

### Policy Artifact (Distribution Unit)

```yaml
policyArtifact:
  policy:
    name: <string>
    version: <vN>
    definition: <policy | policySet | dataPolicy>
  lifecycle:
    state: shadow | enforce | deprecated | revoked
    supersedes: <vN | null>
  scope:
    domain: <string>
    component: <string>
```

**Lifecycle States**:
- `shadow`: Evaluated but not enforced (testing)
- `enforce`: Active enforcement
- `deprecated`: Warning phase before removal
- `revoked`: Disabled immediately

---

## PART VIII: SIGNALS & SCHEDULING GRAMMAR

### Signal Definition

```yaml
signal:
  type: capacity | health | latency | saturation | policy | backpressure
  source:
    component: worker | sms | ui | infra
    name: <string>
    region: <string>
  subject:
    intent: <optional>
    dataState: <optional>
  metrics:
    <key>: <number | string>
  severity: info | warn | critical
  timestamp: <iso8601>
```

**Purpose**: Operational observations that inform scheduling

---

### Scheduler Definition

```yaml
scheduler:
  name: <string>
  scope: <scope_type>
  authority: true | false
  observes: []
```

**Authority**: Only one authoritative scheduler per scope

---

### Failure Domain

```yaml
failure_domain:
  name: <string>
  scope: <scope_type>
  affected_locations: []
```

**Purpose**: Define blast radius for failures

---

## PART IX: CROSS-CUTTING CONCERNS

### Execution Context (Correlation)

```yaml
executionContext:
  trace_id: <uuid>
  causation_id: <uuid>
  actor:
    subject: <subject>
    roles: []
```

**Purpose**: End-to-end correlation and audit trail

---

### Time Constraint

```yaml
timeConstraint:
  type: ttl | deadline | staleness
  value: <duration>
```

**Types**:
- `ttl`: Time to live
- `deadline`: Hard deadline
- `staleness`: Acceptable age

---

### Idempotency Configuration

```yaml
idempotency:
  strategy: deterministic_key
  key:
    - inputIntent.id
    - dataState.primary_key
  scope: global | per_subject
```

**Rules**:
- All transitions MUST be idempotent
- Workers MUST tolerate duplicate delivery
- Deduplication keys MUST be deterministic

---

### Execution Outcome

```yaml
executionOutcome:
  type: success | rejection | failure
  retryable: true | false
  reason: <string>
```

**Definitions**:
- `rejection`: Policy or constraint violation
- `failure`: Infrastructure or execution error
- `success`: Committed transition

---

### Realm Isolation (Multi-Tenancy)

```yaml
realm:
  name: <string>
  isolation:
    data: strict
    policy: strict
    signals: shared | isolated
```

**Rules**:
- Policies scoped per realm
- DataStates never cross realms
- SMS flows are realm-aware

---

## PART X: TRANSPORT & EXECUTION SEMANTICS

### Normative Transport Rules

1. **Flow is a logical execution graph, not a transport graph**
   - Flows describe ordering, dependency, and atomicity
   - Flows do not define addressing, routing, or network hops

2. **Runtime MUST choose execution paths that:**
   - Satisfy declared constraints
   - Avoid unnecessary execution-domain boundaries when possible
   - Preserve execution semantics regardless of transport

3. **Transport is not encoded in the grammar**
   - Local-first execution is preferred when valid
   - Brokered execution is used when required

4. **Applies equally to:**
   - In-process calls
   - Sockets
   - HTTP
   - gRPC
   - NATS
   - Browser execution
   - WASM

---

## PART XI: POLICY STREAM SEMANTICS

### Policy Stream

```yaml
policyStream:
  name: POLICY_DEFS
  retention: latest_per_subject
  replay: required
```

**Guarantees**:
- Replay for late joiners
- Recovery after component restart
- Immutable version delivery

---

## GRAMMAR VALIDATION RULES

### Structural Violations (MUST reject)

1. Flow without steps
2. Duplicate work units in atomic group
3. Boundary declared on step inside atomic group
4. Atomic group without boundary
5. Compensation referencing unknown work unit
6. Retry on non-idempotent work unit
7. Cross-boundary atomic group
8. Multiple authoritative schedulers in same scope

### Semantic Violations (MUST reject)

1. Multiple steps mutating same strong-consistency state outside atomic group
2. Workers belonging to multiple authoritative scopes
3. DataState evolution skipping versions
4. Policy without defined effect
5. InputIntent without transition target
6. Materialization without source

---

## PART XII: AUTHORITY & AVAILABILITY SEMANTICS (NEW in v1.3)

### Entity-Scoped Availability

SMS permits temporary write unavailability at the entity scope as part of governed authority transitions.

```yaml
availability:
  write_scope: entity
  pause_allowed: true | false
  read_continuity: best-effort | guaranteed
```

**Rules**:
- Write unavailability SHALL be bounded in duration
- Write unavailability SHALL NOT affect unrelated entities
- Write unavailability SHALL NOT interrupt read access to existing views
- This behavior SHALL be considered compliant with SMS availability guarantees

---

### Failure Semantics by Entity

When a region becomes unavailable:

| Entity Authority | Impact |
|------------------|--------|
| Entity owned by failed region | Writes unavailable, reads degraded |
| Entity owned by healthy region | Fully operational |

**Write Failure Strategies**:

```yaml
write_failure:
  strategy: reject | client_queue | regional_buffer
```

- `reject`: Immediately fail writes (recommended)
- `client_queue`: Buffer at client (non-authoritative)
- `regional_buffer`: Buffer in adjacent region (expires on timeout)

**Rules**:
- Buffered writes SHALL be non-authoritative
- Buffered writes SHALL NOT replicate
- Buffered writes SHALL expire if authority is not restored

---

### Formal Invariants (Normative)

**I1. Authority Singularity**
At any point in time, an entity SHALL have at most one write-authoritative region.

**I2. Epoch Monotonicity**
Authority epochs SHALL increase monotonically and SHALL NOT be reused.

**I3. No Overlapping Authority**
No two regions SHALL simultaneously accept writes for the same entity.

**I4. CAS Safety**
A write SHALL be accepted if and only if:
- The authority region is current
- The authority epoch matches
- The entity version matches

**I5. View Continuity**
Derived views SHALL remain readable across authority transitions.

**I6. Rebuildability**
All derived state SHALL be reconstructible from authoritative event history.

**I7. Governance Compliance**
Authority transitions SHALL NOT violate declared governance constraints.

**I8. Failure Determinism**
On failure during authority transition, the system SHALL resolve to a single authoritative region without ambiguity.

---

## PART XIII: PRESENTATION COMPOSITION GRAMMAR (NEW in v1.4)

### Overview

Presentation Composition provides semantic grouping of views into workspaces. It enables intent-driven UI development by describing WHAT views belong together and WHY, not HOW they should be rendered.

**Purpose**: Bridge the gap between atomic views and complete user experiences.

---

### PresentationComposition

Defines a semantic grouping of views into a workspace.

```yaml
presentationComposition:
  name: <string>
  version: <vN>
  description: <string>
  primary: <view_ref>
  navigation:
    label: <string>
    purpose: <purpose_ref>
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
      basis: <relationship_ref> | [<relationship_ref>, ...]
      alias: <string>
      trigger: <action>
      policy: <policy_ref>
      policy_set: <policy_set_ref>
      data_scope: { <field>: <expression>, ... }
      navigation:
        label: <string>
        scope: within_composition | always_visible | on_action
```

**Primary View**: The main view that defines the composition's identity

**Navigation Properties**:
- `label`: Display label (supports template expressions like `{{entity.name}}`)
- `purpose`: Semantic category for navigation grouping
- `parent`: Parent composition for hierarchy inference
- `param`: Parameter required to navigate to this composition
- `policy_set`: Authorization for accessing this composition
- `indicator`: Dynamic badge/indicator for navigation item

**Related View Properties**:
- `relationship`: Semantic relationship type
- `cardinality`: One or many related items
- `required`: Whether the related view must be present
- `basis`: Link to entity relationship definition
- `alias`: Disambiguation when same view appears multiple times
- `trigger`: Action that triggers this related view (for action relationships)
- `data_scope`: Filter expression for related view data

---

### View Relationship Types

| Type | Description | Example |
|------|-------------|---------|
| `derived` | Computed from primary | Balance from Account |
| `detail` | Child items of primary | Transactions of Account |
| `contextual` | Related entity | Account owner (Customer) |
| `action` | Triggered by user action | Transfer form |
| `supplementary` | Additional information | Settings, preferences |
| `historical` | Audit/history data | Activity log |

---

### Navigation Scope Types

| Scope | Description |
|-------|-------------|
| `within_composition` | Visible only when parent composition is active |
| `always_visible` | Visible in primary navigation regardless of context |
| `on_action` | Visible only when triggered by specific action |

---

## PART XIV: EXPERIENCE GRAMMAR (NEW in v1.4)

### Overview

Experience defines an application-level grouping of compositions for a specific user type or journey. It represents a complete navigable context, not a deployment unit.

**Purpose**: Define user journeys that group related compositions with consistent authorization.

---

### Experience

Defines an application-level user experience.

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

**Entry Point**: The default composition when entering this experience

**Includes**: All compositions that belong to this experience

**Authorization Behavior**:
- `on_unauthorized`: What happens when user lacks permission
  - `conceal`: Hide the navigation item entirely
  - `indicate`: Show item but indicate it's inaccessible
  - `redirect`: Redirect to another composition
  - `deny`: Show access denied message

**Unauthenticated Handling**:
- `redirect_to`: Where to send unauthenticated users (typically login)

---

### Experience vs Composition

| Aspect | Composition | Experience |
|--------|-------------|------------|
| Scope | Single workspace | Complete application |
| Contains | Views | Compositions |
| Navigation | Defines position in hierarchy | Defines available hierarchy |
| Policy | Per-composition access | Default policy for all contained |
| Example | AccountDetailsWorkspace | CustomerBanking |

---

## PART XV: NAVIGATION GRAMMAR (NEW in v1.4)

### Overview

Navigation grammar defines semantic categories and behaviors for navigation without prescribing visual implementation.

**Purpose**: Express navigation intent (what destinations exist, when accessible) not layout.

---

### Navigation Purpose Categories

Semantic categories for grouping navigation items:

```yaml
navigation:
  purposes:
    - name: home
      description: "Primary entry point / dashboard"
    - name: accounts
      description: "Account-related destinations"
    - name: transactions
      description: "Transaction-related destinations"
    - name: customers
      description: "Customer management destinations"
    - name: alerts
      description: "Alert and notification destinations"
    - name: compliance
      description: "Compliance and KYC destinations"
    - name: audit
      description: "Audit and logging destinations"
    - name: reports
      description: "Reporting destinations"
    - name: settings
      description: "User settings and preferences"
    - name: help
      description: "Help and support destinations"
    - name: authentication
      description: "Login and authentication"
    - name: action
      description: "Action-triggered destinations"
```

**Purpose Usage**: Runtime may group items by purpose for rendering (e.g., primary nav, utility nav, settings menu).

---

### Indicator Semantics

Indicators express dynamic state on navigation items:

```yaml
indicator:
  when: <expression>
  severity: info | warning | error | critical
```

**Severity Levels**:
- `info`: Informational badge (e.g., unread count)
- `warning`: Requires attention (e.g., pending approvals)
- `error`: Problem exists (e.g., failed transactions)
- `critical`: Urgent action needed (e.g., security alert)

**Example**:
```yaml
navigation:
  indicator:
    when: open_alerts > 0
    severity: warning
```

---

### Navigation Hierarchy

Hierarchy is inferred from `parent` relationships in compositions:

```yaml
# Implicit hierarchy from parent declarations
CustomerDashboard (root - no parent)
  └─ CustomerAccountListWorkspace (parent: CustomerDashboard)
       └─ CustomerAccountWorkspace (parent: CustomerAccountListWorkspace)
            └─ CustomerTransactionWorkspace (parent: CustomerAccountWorkspace)
```

**Rules**:
- Root compositions have no `parent`
- Hierarchy MUST be acyclic
- Navigation labels support dynamic templates

---

## PART XVI: V1.5 GRAMMAR EXTENSIONS

### Overview

Version 1.5 extends the grammar with intent-driven UI interactions, advanced materialization semantics, asset handling, search capabilities, scheduled triggers, and enhanced user experience features.

**Philosophy**: All v1.5 additions maintain the intent-first, transport-agnostic principles of the specification.

---

### Form Intent Binding (Addendum X)

Extends `presentationView` with form submission semantics.

```yaml
presentationView:
  name: <string>
  version: <vN>
  consumes:
    dataState: <DataState>
  triggers:                           # NEW in v1.5
    intent: <InputIntent>
    binding: form | action | link
    field_mapping:
      - view_field: <field_name>
        supplies: <intent_field>
        source: context | input | computed
        required: true | false
        default: <value>
        validation:
          constraint: <expression>
          message: <string>
    confirmation:
      required: true | false
      message: <string_template>
    on_success:
      action: navigate_back | navigate_to | stay | refresh | close
      target: <composition_ref>
      params: { <key>: <expression> }
      notification:
        type: toast | inline | none
        message: <string_template>
    on_error:
      action: display_inline | display_modal | retry
      show_field_errors: true | false
      notification:
        type: toast | inline | modal
        message: <string_template>
```

**Purpose**: Enables declarative form submission without hardcoding transport or UI framework details.

**Binding Types**:
- `form`: Multi-field input form
- `action`: Single-click action button
- `link`: Navigation that triggers intent

**Field Sources**:
- `context`: From composition/session context
- `input`: User-provided value
- `computed`: Derived from other fields

---

### View Materialization Contract (Addendum Y)

Extends `materialization` with retrieval and temporal streaming semantics.

```yaml
materialization:
  name: <string>
  version: <vN>
  source: <DataState | list>
  targetState: <DataState>
  
  retrieval:                          # NEW in v1.5
    mode: by_entity | by_query | singleton
    entity_key: <field_name>
    query_fields: [<field_name>, ...]
    list:
      max_items: <integer>
      default_order: <field_name>
      order_direction: asc | desc
    freshness:
      max_staleness: <duration>
    on_unavailable:
      strategy: use_cached | fail | degrade | wait
      cache_ttl: <duration>
      degrade_to: <view_ref>
      wait_timeout: <duration>
      
  temporal:                           # NEW in v1.5: Streaming semantics
    window:
      type: tumbling | sliding | session | hopping
      size: <duration>
      slide: <duration>
      gap: <duration>
    aggregation:
      - field: <field_name>
        function: sum | avg | min | max | count | first | last | stddev
        as: <result_field>
    timestamp_field: <field_name>
    watermark: <duration>
    overflow:
      strategy: drop_oldest | pause_producer | sample | buffer
      buffer_size: <integer>
      sample_rate: <decimal>
    emit:
      trigger: on_window_close | on_each_event | periodic
      periodic_interval: <duration>
      include_partial: true | false
```

**Retrieval Modes**:
- `by_entity`: One view instance per entity
- `by_query`: Views filtered by fields
- `singleton`: Single global view

**Temporal Windows**: Enable stream processing patterns for real-time data.

---

### Intent Delivery Contract (Addendum Z)

Extends `inputIntent` with delivery guarantees and response schemas.

```yaml
inputIntent:
  name: <string>
  version: <vN>
  proposes:
    dataState: <DataState>
  supplies:
    required: [<field>, ...]
    optional: [<field>, ...]
  
  delivery:                           # NEW in v1.5
    guarantee: at_least_once | at_most_once | exactly_once
    acknowledgment: required | optional | fire_and_forget
    timeout: <duration>
    retry:
      allowed: true | false
      max_attempts: <integer>
      backoff: linear | exponential | fixed
  
  response:                           # NEW in v1.5
    on_success:
      includes: [<field_name>, ...]
      guarantees: [<guarantee>, ...]
    on_error:
      includes: [error_code, message, ...]
      field_errors:
        enabled: true | false
        format: map | list
      categories: [validation, authorization, conflict, unavailable, internal]
```

**Delivery Guarantees**:
- `at_least_once`: May be delivered multiple times (with idempotency)
- `at_most_once`: Delivered zero or one time
- `exactly_once`: Delivered exactly once

**Acknowledgment Modes**:
- `required`: Caller waits for response
- `optional`: Caller may or may not wait
- `fire_and_forget`: No response expected

---

### Session Contract (Addendum AA)

Extends `experience` with session management semantics.

```yaml
experience:
  name: <string>
  version: <vN>
  entry_point: <composition_ref>
  includes: [<composition_ref>, ...]
  
  session:                            # NEW in v1.5
    required: true | false
    subject_binding:
      mode: authenticated | anonymous | either
    contains:
      required: [subject_id, roles, ...]
      optional: [realm, attributes, preferences, ...]
    lifetime:
      mode: bounded | sliding | permanent
      idle_timeout: <duration>
      absolute_timeout: <duration>
      refresh: auto | manual
    on_expired:
      action: redirect_to_auth | degrade | notify
      redirect_to: <composition_ref>
      notification:
        message: <string>
    multi_device:                     # NEW: Multi-device support
      enabled: true | false
      sync_strategy: real_time | periodic | manual
      sync_interval: <duration>
      conflict_resolution: last_write_wins | device_priority | manual
    offline:                          # NEW: Offline support
      enabled: true | false
      cache_duration: <duration>
      sync_on_reconnect: auto | prompt
```

**Lifetime Modes**:
- `bounded`: Fixed duration from creation
- `sliding`: Extends on activity
- `permanent`: No expiration

**Multi-Device Sync**: Enables consistent state across devices.

**Offline Support**: Declarative offline operation capabilities.

---

### Indicator Source Binding (Addendum AB)

Extends `navigation.indicator` with data source and update semantics.

```yaml
navigation:
  indicator:
    name: <string>
    type: count | boolean | value     # NEW in v1.5
    source:                           # NEW in v1.5
      view: <view_ref>
      field: <field_name>
      scope:
        expression: <expression>
        type: user | entity | global
      filter: <expression>
      aggregation: count | sum | min | max | avg
    when: <expression>
    severity: info | warning | error | critical
    update:                           # NEW in v1.5
      trigger: view_change | interval | manual | event
      interval: <duration>
      throttle: <duration>
      debounce: <duration>
    display:                          # NEW in v1.5
      format: number | badge | dot
      max_value: <integer>
      hide_when_zero: true | false
```

**Indicator Types**:
- `count`: Numeric count (e.g., "5 alerts")
- `boolean`: Binary state (e.g., red dot)
- `value`: Specific field value (e.g., "$1,234")

**Update Triggers**:
- `view_change`: Real-time on view update
- `interval`: Periodic refresh
- `manual`: User-triggered
- `event`: External event-driven

---

### Composition Fetch Semantics (Addendum AC)

Extends `presentationComposition` with data fetching strategies.

```yaml
presentationComposition:
  name: <string>
  version: <vN>
  primary: <view_ref>
  related: [<view_ref>, ...]
  
  fetch:                              # NEW in v1.5
    order: primary_first | all_parallel | sequential
    related:
      strategy: parallel | sequential | on_demand
      max_parallel: <integer>
    on_failure:
      primary: fail | degrade | retry
      related: skip | degrade | retry
      degrade_to: <view_ref>
    cache:
      strategy: stale_while_revalidate | cache_first | network_first
      ttl: <duration>
    prefetch:
      enabled: true | false
      trigger: on_navigation | on_hover | manual
```

**Fetch Orders**:
- `primary_first`: Load primary before related
- `all_parallel`: Load everything in parallel
- `sequential`: Load in declared order

**Cache Strategies**:
- `stale_while_revalidate`: Serve stale, update in background
- `cache_first`: Serve from cache if available
- `network_first`: Fetch fresh, fall back to cache

---

### Mask Expression Grammar (Addendum AD)

Extends `fieldPermissions.mask` with formal transform grammar.

```yaml
fieldPermissions:
  - field: <field_name>
    visible: true | false | { policy: <policy_ref> }
    mask:
      unless: <expression>
      transform: <transform_expr>      # NEW in v1.5
      placeholder: <string>            # NEW in v1.5
      preserve_format: true | false    # NEW in v1.5
      context:                         # NEW in v1.5
        default: <transform_expr>
        display: <transform_expr>
        print: <transform_expr>
        export: <transform_expr>
        log: <transform_expr>
```

**Standard Transforms**:
- Reveal: `reveal_last(n)`, `reveal_first(n)`, `reveal_middle(start, end)`
- Hide: `hide_last(n)`, `hide_first(n)`, `hide_middle(start, end)`
- Replace: `redact`, `hash`, `hash_partial(n)`, `truncate(n)`
- Format-aware: `mask_email`, `mask_phone`, `mask_credit_card`, `mask_ssn`, `mask_account`

**Transform Chaining**: Multiple transforms can be applied: `reveal_last(4) | uppercase`

**Context Types**:
- `default`: General fallback
- `display`: Screen/UI display
- `print`: Printed documents
- `export`: Data export (CSV, Excel)
- `log`: Audit/debug logs

---

### Asset Semantics (Addendum AK)

Adds `asset` as a field type for binary/file content.

```yaml
dataType:
  name: <string>
  version: <vN>
  fields:
    <field_name>:
      type: asset                      # NEW in v1.5
      asset:
        category: image | document | media | archive | generic
        constraints:
          max_size: <size_expression>
          mime_types: [<mime_type>, ...]
          extensions: [<extension>, ...]
        lifecycle:
          type: permanent | temporary | expiring
          expires_after: <duration>
        reference:
          style: by_reference | embedded
        access:
          read: public | signed | authenticated | policy
          policy: <policy_ref>
          signed_url_ttl: <duration>
        variants:
          - name: <string>
            transform: resize | thumbnail | compress | convert | extract_page
            params: { ... }
```

**Asset Categories**:
- `image`: Raster/vector images
- `document`: PDFs, Word docs, spreadsheets
- `media`: Audio/video files
- `archive`: Compressed archives
- `generic`: Any binary content

**Lifecycle Types**:
- `permanent`: Stored indefinitely
- `temporary`: Short-lived staging
- `expiring`: Time-bounded with auto-deletion

**Access Modes**:
- `public`: Publicly accessible
- `signed`: Time-limited signed URL
- `authenticated`: Requires authentication
- `policy`: Policy-evaluated access

---

### Search Semantics (Addendum AL)

Adds search configuration to fields and introduces `searchIntent`.

```yaml
dataType:
  name: <string>
  version: <vN>
  fields:
    <field_name>:
      type: <type>
      search:                          # NEW in v1.5
        indexed: true | false
        strategy: full_text | exact | prefix | fuzzy | semantic
        full_text:
          analyzer: standard | language | custom
          language: <language_code>
          stemming: enabled | disabled
          stop_words: enabled | disabled
          synonyms: <synonym_set_ref>
        fuzzy:
          max_edits: 1 | 2
          prefix_length: <integer>
        weight: <decimal>
        boost_recent: true | false
        suggest:
          enabled: true | false
          type: completion | phrase | term
        facet:
          enabled: true | false
          type: terms | range | date_histogram | nested
          ranges: [<range_definition>, ...]
        highlight:
          enabled: true | false
          fragment_size: <integer>
          pre_tag: <string>
          post_tag: <string>
```

**Search Intent**:

```yaml
searchIntent:
  name: <string>
  version: <vN>
  searches: <DataState>
  query:
    fields: [<field>, ...]
    default_operator: and | or
    phrase_slop: <integer>
  filters:
    required: [<field>, ...]
    optional: [<field>, ...]
  facets:
    include: [<field>, ...]
  pagination:
    default_size: <integer>
    max_size: <integer>
    cursor_based: true | false
  sort:
    default: <field> | _score
    allowed: [<field>, ...]
  result:
    includes: [<field>, ...]
    highlights: [<field>, ...]
  authorization:
    policy_set: <policy_set_ref>
    filter_by: <field>
```

**Search Strategies**:
- `full_text`: Natural language search with analysis
- `exact`: Exact match only
- `prefix`: Prefix/autocomplete matching
- `fuzzy`: Typo-tolerant matching
- `semantic`: Vector/embedding similarity

---

### Scheduled Trigger Semantics (Addendum AM)

Extends `inputIntent` with time-based triggers.

```yaml
inputIntent:
  name: <string>
  version: <vN>
  proposes:
    dataState: <DataState>
  
  scheduled:                          # NEW in v1.5
    enabled: true | false
    trigger:
      type: cron | interval | once
      cron: <cron_expression>
      interval: <duration>
      at: <timestamp>
    timezone:
      mode: fixed | user_context | entity_context
      fixed: <timezone>
      context_field: <field_name>
    window:
      start: <time_expression>
      end: <time_expression>
      skip_outside: true | false
    idempotency:
      key: <expression>
      scope: global | per_subject | per_entity
    on_missed:
      action: skip | execute_immediately | queue
```

**Trigger Types**:
- `cron`: Cron expression (e.g., `0 9 * * MON-FRI`)
- `interval`: Recurring interval (e.g., `1h`, `30m`)
- `once`: Single execution at timestamp

**Timezone Modes**:
- `fixed`: Single timezone for all executions
- `user_context`: Use subject's timezone
- `entity_context`: Use entity's timezone field

---

### Notification Channel Semantics (Addendum AN)

Defines notification delivery channels and preferences.

```yaml
notificationChannel:
  name: <string>
  version: <vN>
  
  channels:                           # NEW in v1.5
    - type: email | sms | push | in_app | webhook
      enabled: true | false
      config:
        email:
          from: <email>
          template: <template_ref>
        sms:
          sender: <sender_id>
          template: <template_ref>
        push:
          platform: fcm | apns | web_push
          priority: high | normal | low
        webhook:
          url: <url>
          method: POST | PUT
          auth: <auth_config>
  
  preferences:                        # User preferences
    user_controllable: true | false
    defaults:
      email: enabled | disabled
      sms: enabled | disabled
      push: enabled | disabled
    quiet_hours:
      enabled: true | false
      start: <time>
      end: <time>
      timezone: <timezone>
  
  delivery:
    retry:
      enabled: true | false
      max_attempts: <integer>
      backoff: linear | exponential
    fallback:
      enabled: true | false
      order: [<channel>, ...]
    batching:
      enabled: true | false
      window: <duration>
      max_batch_size: <integer>
```

**Channel Types**:
- `email`: Email notifications
- `sms`: SMS text messages
- `push`: Mobile/web push notifications
- `in_app`: In-application notifications
- `webhook`: HTTP callback to external systems

---

### Collaborative Session Semantics (Addendum AO)

Enables real-time collaboration features.

```yaml
experience:
  name: <string>
  version: <vN>
  
  collaboration:                      # NEW in v1.5
    enabled: true | false
    session:
      type: document | workspace | entity
      scope: <expression>
      max_participants: <integer>
    
    awareness:                        # Presence indicators
      show_active_users: true | false
      show_cursors: true | false
      show_selections: true | false
      timeout: <duration>
    
    permissions:
      mode: owner_controlled | role_based | policy_based
      policy_set: <policy_set_ref>
    
    conflict_resolution:
      strategy: operational_transform | crdt | last_write_wins
      merge_policy: <policy_ref>
```

**Session Types**:
- `document`: Single document editing
- `workspace`: Entire composition collaboration
- `entity`: Entity-scoped collaboration

**Awareness Features**:
- Active user indicators
- Cursor positions
- Selection highlighting

---

### Structured Content Type (Addendum AP)

Adds rich text/structured content field type.

```yaml
dataType:
  name: <string>
  version: <vN>
  fields:
    <field_name>:
      type: structured_content         # NEW in v1.5
      content:
        format: markdown | rich_text | html_subset | json_schema
        schema: <schema_ref>
        validation:
          max_length: <integer>
          allowed_blocks: [paragraph, heading, list, code, quote, table, ...]
          allowed_marks: [bold, italic, underline, link, code, ...]
        sanitization:
          strip_scripts: true | false
          allowed_tags: [<tag>, ...]
          allowed_attributes: [<attr>, ...]
```

**Content Formats**:
- `markdown`: CommonMark or GFM
- `rich_text`: Structured JSON format (ProseMirror, Slate, etc.)
- `html_subset`: Restricted HTML
- `json_schema`: Custom JSON structure

---

### Spatial Type Semantics (Addendum AR)

Adds geographic/spatial field types.

```yaml
dataType:
  name: <string>
  version: <vN>
  fields:
    <field_name>:
      type: spatial                    # NEW in v1.5
      spatial:
        geometry: point | line | polygon | multi_point | multi_line | multi_polygon
        coordinate_system: wgs84 | web_mercator | custom
        srid: <integer>
        precision: <decimal>
        bounds:
          min_lat: <decimal>
          max_lat: <decimal>
          min_lon: <decimal>
          max_lon: <decimal>
        search:
          indexed: true | false
          strategy: bounding_box | radius | polygon_intersection
```

**Geometry Types**:
- `point`: Single coordinate
- `line`: Line string
- `polygon`: Closed polygon
- `multi_*`: Collections

**Search Strategies**:
- `bounding_box`: Rectangular area search
- `radius`: Distance from point
- `polygon_intersection`: Shape intersection

---

### Graph Query Semantics (Addendum AS)

Adds graph traversal queries for relationships.

```yaml
graphQuery:
  name: <string>
  version: <vN>
  
  start_from: <DataState>             # NEW in v1.5
  traversal:
    relationships: [<relationship_ref>, ...]
    direction: outgoing | incoming | both
    min_depth: <integer>
    max_depth: <integer>
  
  filter:
    node: <expression>
    edge: <expression>
  
  return:
    nodes: [<field>, ...]
    edges: [<field>, ...]
    aggregations:
      - function: count | sum | avg
        field: <field_name>
        as: <result_field>
```

**Purpose**: Enable declarative graph traversal without embedding query language.

---

### Durable Workflow Semantics (Addendum AT)

Extends SMS with long-running workflow patterns.

```yaml
sms:
  flows:
    - name: <string>
      durable:                        # NEW in v1.5
        enabled: true | false
        checkpoint:
          frequency: per_step | per_group | manual
          storage: <storage_ref>
        compensation:
          enabled: true | false
          on_failure: rollback | compensate | manual
        timeout:
          total: <duration>
          per_step: <duration>
        retry:
          enabled: true | false
          max_attempts: <integer>
          backoff: exponential | linear
```

**Purpose**: Long-running processes with fault tolerance and compensation.

---

### Delegation and Consent Semantics (Addendum AU)

Adds delegation and consent management.

```yaml
delegation:
  name: <string>
  version: <vN>
  
  delegate:                           # NEW in v1.5
    from: <subject_ref>
    to: <subject_ref>
    scope:
      actions: [<action>, ...]
      data: [<dataState>, ...]
      compositions: [<composition_ref>, ...]
  
  consent:
    required: true | false
    type: explicit | implicit
    expires_after: <duration>
    revocable: true | false
    audit:
      log_access: true | false
      log_revocation: true | false
```

**Purpose**: Declarative delegation without embedding authorization logic.

---

### Conversational Experience Semantics (Addendum AV)

Enables conversational UI patterns.

```yaml
conversationalExperience:
  name: <string>
  version: <vN>
  
  interaction:                        # NEW in v1.5
    mode: voice | text | multimodal
    
  dialogue:
    intents: [<intent_ref>, ...]
    context:
      max_turns: <integer>
      timeout: <duration>
      
  understanding:
    nlu_model: <model_ref>
    confidence_threshold: <decimal>
    fallback_intent: <intent_ref>
    
  generation:
    template_set: <template_ref>
    personalization: enabled | disabled
```

**Purpose**: Voice assistants and chatbots as first-class experiences.

---

### External Integration Semantics (Addendum AW)

Defines external system integration contracts.

```yaml
externalIntegration:
  name: <string>
  version: <vN>
  
  endpoint:                           # NEW in v1.5
    base_url: <url>
    auth:
      type: bearer | oauth2 | api_key | mtls
      credentials: <credentials_ref>
  
  operations:
    - name: <string>
      method: GET | POST | PUT | DELETE
      path: <path_template>
      request:
        schema: <schema_ref>
        mapping: <field_mapping>
      response:
        schema: <schema_ref>
        mapping: <field_mapping>
      
  reliability:
    timeout: <duration>
    retry:
      enabled: true | false
      max_attempts: <integer>
      backoff: exponential
    circuit_breaker:
      enabled: true | false
      failure_threshold: <integer>
      reset_timeout: <duration>
```

**Purpose**: Declarative third-party API integration.

---

### Data Governance Semantics (Addendum AX)

Adds data classification and retention policies.

```yaml
dataGovernance:
  name: <string>
  version: <vN>
  
  classification:                     # NEW in v1.5
    level: public | internal | confidential | restricted
    categories: [pii, pci, phi, gdpr, ...]
  
  retention:
    policy: retain | archive | delete
    duration: <duration>
    triggers:
      - type: entity_deleted | time_elapsed | external_event
        action: archive | delete
  
  compliance:
    regulations: [gdpr, ccpa, hipaa, ...]
    audit_required: true | false
    encryption:
      at_rest: required | optional | none
      in_transit: required | optional | none
```

**Purpose**: Declarative data governance without embedding compliance logic.

---

### Edge Device Semantics (Addendum AY)

Enables edge computing patterns.

```yaml
edgeDevice:
  name: <string>
  version: <vN>
  
  capabilities:                       # NEW in v1.5
    compute: <compute_spec>
    storage: <storage_spec>
    connectivity: online | offline | intermittent
  
  sync:
    strategy: push | pull | bidirectional
    frequency: real_time | periodic | on_demand
    conflict_resolution: device_wins | server_wins | manual
  
  local_processing:
    enabled: true | false
    fallback_to_cloud: true | false
    models: [<model_ref>, ...]
```

**Purpose**: IoT and edge device integration patterns.

---

## CONFORMANCE

An implementation is conformant with this specification if it:

1. Implements all execution semantics
2. Honors atomic group guarantees
3. Separates scheduling from execution
4. Preserves failure invariants
5. Enforces policy lifecycle correctly
6. Maintains version compatibility rules
7. Implements idempotency correctly
8. Propagates execution context
9. Enforces authority singularity (NEW in v1.3)
10. Respects entity-scoped availability semantics (NEW in v1.3)
11. Validates CAS tokens correctly across authority migrations (NEW in v1.3)
12. Maintains view continuity during authority transitions (NEW in v1.3)
13. Resolves compositions to correct views (NEW in v1.4)
14. Enforces field permissions and masking (NEW in v1.4)
15. Routes experiences to correct entry points (NEW in v1.4)
16. Builds navigation hierarchy from parent relationships (NEW in v1.4)
17. Evaluates indicators dynamically (NEW in v1.4)
18. Processes form intent bindings correctly (NEW in v1.5)
19. Implements materialization retrieval contracts (NEW in v1.5)
20. Enforces intent delivery guarantees (NEW in v1.5)
21. Manages session lifecycles correctly (NEW in v1.5)
22. Applies mask transforms as declared (NEW in v1.5)
23. Handles asset storage and retrieval (NEW in v1.5)
24. Implements search strategies correctly (NEW in v1.5)
25. Executes scheduled triggers (NEW in v1.5)
26. Delivers notifications through declared channels (NEW in v1.5)

---

## LEXICAL GRAMMAR

```ebnf
letter        ::= "a"…"z" | "A"…"Z"
digit         ::= "0"…"9"

identifier    ::= letter { letter | digit | "_" }

number        ::= digit { digit }
float         ::= number "." number
boolean       ::= "true" | "false"

string        ::= "\"" { character } "\""

duration      ::= number ( "ms" | "s" | "m" | "h" )

comment       ::= "//" { character } "\n"
```

---

## V1.5 EBNF GRAMMAR EXTENSIONS

### Form Intent Binding EBNF

```ebnf
triggers        ::= "triggers:" NEWLINE INDENT
                    "intent:" intent_ref NEWLINE
                    "binding:" binding_type NEWLINE
                    field_mapping?
                    confirmation?
                    on_success
                    on_error
                    DEDENT

binding_type    ::= "form" | "action" | "link"

field_mapping   ::= "field_mapping:" NEWLINE INDENT
                    ( field_map NEWLINE )+
                    DEDENT

field_map       ::= "- view_field:" identifier NEWLINE
                    "supplies:" identifier NEWLINE
                    "source:" source_type NEWLINE
                    "required:" boolean NEWLINE
                    ( "default:" value NEWLINE )?
                    validation?

source_type     ::= "context" | "input" | "computed"

validation      ::= "validation:" NEWLINE INDENT
                    "constraint:" expression NEWLINE
                    "message:" string NEWLINE
                    DEDENT

confirmation    ::= "confirmation:" NEWLINE INDENT
                    "required:" boolean NEWLINE
                    "message:" string_template NEWLINE
                    DEDENT

on_success      ::= "on_success:" NEWLINE INDENT
                    "action:" success_action NEWLINE
                    ( "target:" composition_ref NEWLINE )?
                    ( "params:" params_map NEWLINE )?
                    notification?
                    DEDENT

on_error        ::= "on_error:" NEWLINE INDENT
                    "action:" error_action NEWLINE
                    ( "show_field_errors:" boolean NEWLINE )?
                    notification?
                    DEDENT

success_action  ::= "navigate_back" | "navigate_to" | "stay" | "refresh" | "close"

error_action    ::= "display_inline" | "display_modal" | "retry"

notification    ::= "notification:" NEWLINE INDENT
                    "type:" notification_type NEWLINE
                    "message:" string_template NEWLINE
                    DEDENT

notification_type ::= "toast" | "inline" | "modal" | "none"
```

---

### Materialization Contract EBNF

```ebnf
retrieval       ::= "retrieval:" NEWLINE INDENT
                    "mode:" retrieval_mode NEWLINE
                    ( entity_key | query_fields )?
                    list_config?
                    freshness?
                    on_unavailable?
                    DEDENT

retrieval_mode  ::= "by_entity" | "by_query" | "singleton"

entity_key      ::= "entity_key:" identifier NEWLINE

query_fields    ::= "query_fields:" "[" identifier_list "]" NEWLINE

list_config     ::= "list:" NEWLINE INDENT
                    "max_items:" integer NEWLINE
                    ( "default_order:" identifier NEWLINE )?
                    ( "order_direction:" order_dir NEWLINE )?
                    DEDENT

order_dir       ::= "asc" | "desc"

on_unavailable  ::= "on_unavailable:" NEWLINE INDENT
                    "strategy:" unavail_strategy NEWLINE
                    ( "cache_ttl:" duration NEWLINE )?
                    ( "degrade_to:" view_ref NEWLINE )?
                    ( "wait_timeout:" duration NEWLINE )?
                    DEDENT

unavail_strategy ::= "use_cached" | "fail" | "degrade" | "wait"

temporal        ::= "temporal:" NEWLINE INDENT
                    temporal_window
                    temporal_aggregation?
                    ( "timestamp_field:" identifier NEWLINE )?
                    ( "watermark:" duration NEWLINE )?
                    temporal_overflow?
                    temporal_emit?
                    DEDENT

temporal_window ::= "window:" NEWLINE INDENT
                    "type:" window_type NEWLINE
                    "size:" duration NEWLINE
                    ( "slide:" duration NEWLINE )?
                    ( "gap:" duration NEWLINE )?
                    DEDENT

window_type     ::= "tumbling" | "sliding" | "session" | "hopping"

temporal_aggregation ::= "aggregation:" NEWLINE INDENT
                         aggregation_def+
                         DEDENT

aggregation_def ::= "-" "field:" identifier NEWLINE INDENT
                    "function:" agg_function NEWLINE
                    "as:" identifier NEWLINE
                    DEDENT

agg_function    ::= "sum" | "avg" | "min" | "max" | "count" | "first" | "last" | "stddev"

temporal_overflow ::= "overflow:" NEWLINE INDENT
                      "strategy:" overflow_strategy NEWLINE
                      ( "buffer_size:" integer NEWLINE )?
                      ( "sample_rate:" decimal NEWLINE )?
                      DEDENT

overflow_strategy ::= "drop_oldest" | "pause_producer" | "sample" | "buffer"

temporal_emit   ::= "emit:" NEWLINE INDENT
                    "trigger:" emit_trigger NEWLINE
                    ( "periodic_interval:" duration NEWLINE )?
                    ( "include_partial:" boolean NEWLINE )?
                    DEDENT

emit_trigger    ::= "on_window_close" | "on_each_event" | "periodic"
```

---

### Intent Delivery Contract EBNF

```ebnf
delivery        ::= "delivery:" NEWLINE INDENT
                    "guarantee:" delivery_guarantee NEWLINE
                    "acknowledgment:" ack_mode NEWLINE
                    "timeout:" duration NEWLINE
                    retry_config?
                    DEDENT

delivery_guarantee ::= "at_least_once" | "at_most_once" | "exactly_once"

ack_mode        ::= "required" | "optional" | "fire_and_forget"

retry_config    ::= "retry:" NEWLINE INDENT
                    "allowed:" boolean NEWLINE
                    ( "max_attempts:" integer NEWLINE )?
                    ( "backoff:" backoff_strategy NEWLINE )?
                    DEDENT

backoff_strategy ::= "linear" | "exponential" | "fixed"

response        ::= "response:" NEWLINE INDENT
                    on_success_response
                    on_error_response
                    DEDENT

on_success_response ::= "on_success:" NEWLINE INDENT
                        "includes:" "[" identifier_list "]" NEWLINE
                        ( "guarantees:" "[" guarantee_list "]" NEWLINE )?
                        DEDENT

on_error_response ::= "on_error:" NEWLINE INDENT
                      "includes:" "[" identifier_list "]" NEWLINE
                      field_errors_config?
                      ( "categories:" "[" error_category_list "]" NEWLINE )?
                      DEDENT

field_errors_config ::= "field_errors:" NEWLINE INDENT
                        "enabled:" boolean NEWLINE
                        ( "format:" field_error_format NEWLINE )?
                        DEDENT

field_error_format ::= "map" | "list"

error_category_list ::= error_category ( "," error_category )*

error_category  ::= "validation" | "authorization" | "conflict" | "unavailable" | "internal"
```

---

### Mask Transform Grammar EBNF

```ebnf
mask            ::= "mask:" NEWLINE INDENT
                    ( "unless:" expression NEWLINE )?
                    "transform:" transform_expr NEWLINE
                    ( "placeholder:" string NEWLINE )?
                    ( "preserve_format:" boolean NEWLINE )?
                    mask_context?
                    DEDENT

transform_expr  ::= simple_transform
                  | chained_transform
                  | conditional_transform

simple_transform ::= transform_name ( "(" transform_args ")" )?

chained_transform ::= simple_transform ( "|" simple_transform )*

conditional_transform ::= NEWLINE INDENT
                          "when:" expression NEWLINE
                          "then:" transform_expr NEWLINE
                          "else:" transform_expr NEWLINE
                          DEDENT

transform_name  ::= "reveal_last" | "reveal_first" | "reveal_middle"
                  | "reveal_pattern" | "hide_last" | "hide_first"
                  | "hide_middle" | "hide_pattern" | "redact" | "hash"
                  | "hash_partial" | "truncate" | "mask_email"
                  | "mask_phone" | "mask_credit_card" | "mask_ssn"
                  | "mask_account" | "uppercase" | "lowercase"

transform_args  ::= integer
                  | integer "," integer
                  | string

mask_context    ::= "context:" NEWLINE INDENT
                    ( "default:" transform_expr NEWLINE )?
                    ( "display:" transform_expr NEWLINE )?
                    ( "print:" transform_expr NEWLINE )?
                    ( "export:" transform_expr NEWLINE )?
                    ( "log:" transform_expr NEWLINE )?
                    DEDENT
```

---

### Asset Type EBNF

```ebnf
asset_field     ::= "type:" "asset" NEWLINE
                    "asset:" NEWLINE INDENT
                    "category:" asset_category NEWLINE
                    asset_constraints?
                    asset_lifecycle?
                    asset_reference?
                    asset_access?
                    asset_variants?
                    DEDENT

asset_category  ::= "image" | "document" | "media" | "archive" | "generic"

asset_constraints ::= "constraints:" NEWLINE INDENT
                      ( "max_size:" size_expression NEWLINE )?
                      ( "mime_types:" "[" mime_type_list "]" NEWLINE )?
                      ( "extensions:" "[" extension_list "]" NEWLINE )?
                      DEDENT

size_expression ::= number size_unit
size_unit       ::= "B" | "KB" | "MB" | "GB"

asset_lifecycle ::= "lifecycle:" NEWLINE INDENT
                    "type:" lifecycle_type NEWLINE
                    ( "expires_after:" duration NEWLINE )?
                    DEDENT

lifecycle_type  ::= "permanent" | "temporary" | "expiring"

asset_reference ::= "reference:" NEWLINE INDENT
                    "style:" reference_style NEWLINE
                    DEDENT

reference_style ::= "by_reference" | "embedded"

asset_access    ::= "access:" NEWLINE INDENT
                    "read:" access_mode NEWLINE
                    ( "policy:" policy_ref NEWLINE )?
                    ( "signed_url_ttl:" duration NEWLINE )?
                    DEDENT

access_mode     ::= "public" | "signed" | "authenticated" | "policy"

asset_variants  ::= "variants:" NEWLINE INDENT
                    variant_definition+
                    DEDENT

variant_definition ::= "-" "name:" identifier NEWLINE INDENT
                       "transform:" transform_type NEWLINE
                       ( "params:" params_map NEWLINE )?
                       DEDENT

transform_type  ::= "resize" | "thumbnail" | "compress" | "convert" | "extract_page"
```

---

### Search Semantics EBNF

```ebnf
search_config   ::= "search:" NEWLINE INDENT
                    "indexed:" boolean NEWLINE
                    ( "strategy:" search_strategy NEWLINE )?
                    full_text_config?
                    fuzzy_config?
                    ( "weight:" decimal NEWLINE )?
                    ( "boost_recent:" boolean NEWLINE )?
                    suggest_config?
                    facet_config?
                    highlight_config?
                    DEDENT

search_strategy ::= "full_text" | "exact" | "prefix" | "fuzzy" | "semantic"

full_text_config ::= "full_text:" NEWLINE INDENT
                     ( "analyzer:" analyzer_type NEWLINE )?
                     ( "language:" string NEWLINE )?
                     ( "stemming:" enabled_disabled NEWLINE )?
                     ( "stop_words:" enabled_disabled NEWLINE )?
                     ( "synonyms:" identifier NEWLINE )?
                     DEDENT

analyzer_type   ::= "standard" | "language" | "custom"
enabled_disabled ::= "enabled" | "disabled"

fuzzy_config    ::= "fuzzy:" NEWLINE INDENT
                    ( "max_edits:" integer NEWLINE )?
                    ( "prefix_length:" integer NEWLINE )?
                    DEDENT

suggest_config  ::= "suggest:" NEWLINE INDENT
                    "enabled:" boolean NEWLINE
                    ( "type:" suggest_type NEWLINE )?
                    DEDENT

suggest_type    ::= "completion" | "phrase" | "term"

facet_config    ::= "facet:" NEWLINE INDENT
                    "enabled:" boolean NEWLINE
                    ( "type:" facet_type NEWLINE )?
                    ( "ranges:" "[" range_list "]" NEWLINE )?
                    DEDENT

facet_type      ::= "terms" | "range" | "date_histogram" | "nested"

highlight_config ::= "highlight:" NEWLINE INDENT
                     "enabled:" boolean NEWLINE
                     ( "fragment_size:" integer NEWLINE )?
                     ( "pre_tag:" string NEWLINE )?
                     ( "post_tag:" string NEWLINE )?
                     DEDENT

search_intent   ::= "searchIntent:" NEWLINE INDENT
                    "name:" identifier NEWLINE
                    "version:" version NEWLINE
                    "searches:" data_state_ref NEWLINE
                    query_config?
                    filter_config?
                    facet_include?
                    pagination_config?
                    sort_config?
                    result_config?
                    search_authorization?
                    DEDENT

search_index    ::= "search_index:" NEWLINE INDENT
                    "enabled:" boolean NEWLINE
                    document_config?
                    freshness_config?
                    rebuild_config?
                    DEDENT
```

---

### Indicator Source Binding EBNF

```ebnf
indicator       ::= "indicator:" NEWLINE INDENT
                    "name:" identifier NEWLINE
                    "type:" indicator_type NEWLINE
                    indicator_source
                    "when:" expression NEWLINE
                    "severity:" severity_level NEWLINE
                    indicator_update
                    indicator_display?
                    DEDENT

indicator_type  ::= "count" | "boolean" | "value"

indicator_source ::= "source:" NEWLINE INDENT
                     "view:" view_ref NEWLINE
                     ( "field:" identifier NEWLINE )?
                     indicator_scope
                     ( "filter:" expression NEWLINE )?
                     ( "aggregation:" aggregation_type NEWLINE )?
                     DEDENT

indicator_scope ::= "scope:" NEWLINE INDENT
                    ( "expression:" expression NEWLINE )?
                    "type:" scope_type NEWLINE
                    DEDENT

scope_type      ::= "user" | "entity" | "global"

aggregation_type ::= "count" | "sum" | "min" | "max" | "avg"

indicator_update ::= "update:" NEWLINE INDENT
                     "trigger:" update_trigger NEWLINE
                     ( "interval:" duration NEWLINE )?
                     ( "throttle:" duration NEWLINE )?
                     ( "debounce:" duration NEWLINE )?
                     DEDENT

update_trigger  ::= "view_change" | "interval" | "manual" | "event"

indicator_display ::= "display:" NEWLINE INDENT
                      "format:" display_format NEWLINE
                      ( "max_value:" integer NEWLINE )?
                      ( "hide_when_zero:" boolean NEWLINE )?
                      DEDENT

display_format  ::= "number" | "badge" | "dot"
```

---

## SPECIFICATION METADATA

```yaml
specMetadata:
  version: v1.5
  ingestible: true
  exampleSemantics: illustrative
  machineReadable: true
  changelog:
    v1.5:
      - "Added Form Intent Binding (triggers block in presentationView)"
      - "Added View Materialization Contract (retrieval and temporal blocks)"
      - "Added Intent Delivery Contract (delivery and response blocks)"
      - "Added Session Contract (session management in experience)"
      - "Added Indicator Source Binding (data source and update semantics)"
      - "Added Composition Fetch Semantics (fetch strategies and caching)"
      - "Added Mask Expression Grammar (formal transform and context-aware masking)"
      - "Added Navigation Semantics (identity, breadcrumbs, active state)"
      - "Added Asset Semantics (binary/file field type with lifecycle)"
      - "Added Search Semantics (searchable fields, search intents, faceting)"
      - "Added Scheduled Trigger Semantics (cron/interval triggers for intents)"
      - "Added Notification Channel Semantics (multi-channel delivery)"
      - "Added Collaborative Session Semantics (real-time collaboration)"
      - "Added Structured Content Type (rich text/markdown fields)"
      - "Added Spatial Type Semantics (geographic/spatial data)"
      - "Added Graph Query Semantics (relationship traversal)"
      - "Added Durable Workflow Semantics (long-running processes)"
      - "Added Delegation and Consent Semantics (delegation management)"
      - "Added Conversational Experience Semantics (voice/chat interfaces)"
      - "Added External Integration Semantics (third-party API integration)"
      - "Added Data Governance Semantics (classification and retention)"
      - "Added Edge Device Semantics (IoT and edge computing)"
      - "Added Surface and Container Semantics (modal, drawer, split view)"
      - "Added Accessibility Preferences (user accessibility settings)"
      - "Added View Availability Policy (conditional view availability)"
      - "Added Presentation Hints (display type recommendations)"
      - "Added Inference-Derived Fields (AI/ML derived fields)"
      - "Enables complete end-to-end application development"
      - "All additions maintain intent-first, transport-agnostic philosophy"
    v1.4:
      - "Added PresentationComposition grammar (Part XIII)"
      - "Added Experience grammar (Part XIV)"
      - "Added Navigation grammar with purposes and scopes (Part XV)"
      - "Added fieldPermissions to PresentationView"
      - "Added field masking (full, partial, hash)"
      - "Added navigation indicators with severity"
      - "Added view relationship types (derived, detail, contextual, action, supplementary, historical)"
      - "Added navigation scopes (within_composition, always_visible, on_action)"
      - "Added on_unauthorized behavior (conceal, indicate, redirect, deny)"
      - "Enables intent-driven UI composition"
    v1.3:
      - "Added Entity-scoped Authority (Mutability, Authority, Authority State)"
      - "Added Authority Transitions with pause-and-cutover protocol"
      - "Added Storage Roles (control vs data)"
      - "Added Relationship grammar with cardinality and semantics"
      - "Added Invariant Policy for view-level invariant enforcement"
      - "Added CAS Token structure for write safety"
      - "Added Authority & Availability Semantics (Part XII)"
      - "Added View authority_agnostic and epoch_tolerant properties"
      - "Added 8 Formal Invariants for authority correctness"
      - "Enables multi-region survivability patterns"
    v1.2:
      - "Added WorkUnitContract grammar (Part IV)"
      - "Enables typed work unit specifications"
      - "Provides contract coverage validation"
    v1.1:
      - "Added signals and scheduling"
      - "Added policy lifecycle"
```

**Declaration**: This specification is structural and machine-ingestable. Examples are illustrative unless marked normative.

