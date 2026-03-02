# System Mechanics Specification - Grammar Reference v1.4

## Overview

This document provides the complete formal grammar for the Unified System Specification, covering Data, Evolution, UI, Input, Flow (SMS), Worker Topology (WTS), Policy, Signals, Scheduling, Authority, Multi-Region Survivability, UI Composition, and Experiences.

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

## SPECIFICATION METADATA

```yaml
specMetadata:
  version: v1.4
  ingestible: true
  exampleSemantics: illustrative
  machineReadable: true
  changelog:
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

