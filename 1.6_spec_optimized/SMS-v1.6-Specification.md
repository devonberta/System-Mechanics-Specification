# System Mechanics Specification (SMS) v1.6

## Complete LLM-Optimized Specification

This document provides the complete, normative specification for the System Mechanics Specification (SMS) framework, incorporating all features from v1.0 through v1.6. It is organized hierarchically for efficient reference and designed for both human and LLM consumption.

**Version**: 1.6  
**Status**: Production Ready  
**Last Updated**: 2026-02-04

**Related Documents:**
- **SMS-v1.6-EBNF-Grammar.md**: Formal grammar definitions for parser development
- **SMS-v1.6-Implementation-Guide.md**: Runtime implementation patterns and code examples
- **SMS-v1.6-Reference-Examples.md**: Complete worked examples and modeling patterns
- **SMS-v1.6-Quick-Reference.md**: Rapid-lookup cheat sheets and decision trees

---

## READING THIS SPECIFICATION (For LLM Agents)

### Document Purpose

This is the **complete, normative** specification for the System Mechanics Specification (SMS) v1.6. It defines a universal, intent-first framework for describing, executing, and evolving distributed systems with comprehensive storage, caching, and infrastructure capabilities.

### How to Use This Document

**For LLMs Implementing Runtime**:
- This document is self-contained - no external references needed
- All grammar is integrated inline (no separate addendum documents)
- Grammar is hierarchical: start with Part I for principles, Part II for invariants
- Validation rules are explicit - all MUST/SHALL/MAY keywords are normative per RFC 2119
- Examples are illustrative unless marked normative

**For LLMs Writing Specifications**:
- Use Parts I-XXIV as authoritative reference for all grammar constructs
- Refer to Part XXV (Validation and Conformance) to ensure correctness
- Use Part XXVI (Design Rationale) to understand intent behind major decisions
- All concepts are versioned - always specify versions explicitly
- v1.6 adds storage binding, caching, intent routing, resource requirements, and client cache

**Key Structural Notes**:
- This specification uses RFC 2119 keywords: MUST, SHALL, MAY, SHOULD
- All code blocks without line numbers are normative grammar unless marked "example"
- Grammar shown in YAML format is canonical; EBNF in separate document is equivalent
- v1.6 is backward compatible with v1.5 - all v1.5 features remain available
- v1.6 introduces infrastructure layer: storage, caching, routing are now declarative

**Related Documents:**
- **SMS-v1.6-EBNF-Grammar.md**: Complete formal grammar for parser/tooling development
- **SMS-v1.6-Implementation-Guide.md**: Detailed runtime implementation patterns with storage, caching, routing
- **SMS-v1.6-Reference-Examples.md**: Modeling guidance and complete worked examples
- **SMS-v1.6-Quick-Reference.md**: Rapid-lookup cheat sheets and decision trees

---

## TABLE OF CONTENTS

1. [Overview and Versioning](#part-i-overview-and-versioning)
2. [Core Invariants](#part-ii-core-invariants)
3. [Lexical Structure](#part-iii-lexical-structure)
4. [Core Data Types](#part-iv-core-data-types)
5. [Boundaries and Work Units](#part-v-boundaries-and-work-units)
6. [Flow Definitions](#part-vi-flow-definitions)
7. [UI and Presentation](#part-vii-ui-and-presentation)
8. [Policies and Authorization](#part-viii-policies-and-authorization)
9. [Signals and Observability](#part-ix-signals-and-observability)
10. [Multi-Tenancy and Realms](#part-x-multi-tenancy-and-realms)
11. [Entity Relationships (v1.3)](#part-xi-entity-relationships)
12. [Experiences (v1.4)](#part-xii-experiences)
13. [Session & Asset Management (v1.5)](#part-xiii-session-and-asset-management)
14. [IoT & Device Telemetry (v1.5)](#part-xiv-iot-and-device-telemetry)
15. [Privacy & Compliance (v1.5)](#part-xv-privacy-and-compliance)
16. [External Integration (v1.5)](#part-xvi-external-integration)
17. [Graph Queries (v1.5)](#part-xvii-graph-queries)
18. [Conversational UI (v1.5)](#part-xviii-conversational-ui)
19. [ML/Inference (v1.5)](#part-xix-mlinference)
20. [Storage Binding (v1.6)](#part-xx-storage-binding)
21. [Unified Caching (v1.6)](#part-xxi-unified-caching)
22. [Intent Routing (v1.6)](#part-xxii-intent-routing)
23. [Resource Requirements (v1.6)](#part-xxiii-resource-requirements)
24. [Client Cache Contract (v1.6)](#part-xxiv-client-cache-contract)
25. [Validation and Conformance](#part-xxv-validation-and-conformance)
26. [Design Rationale](#part-xxvi-design-rationale)

---

## PART I: OVERVIEW AND VERSIONING

### 1.1 Purpose

SMS is a declarative specification language for describing distributed systems in terms of:
- **Data structures** and their lifecycle
- **Business logic** organized as work units within boundaries
- **User interfaces** as derived views of data
- **Policies** that govern access and behavior
- **Signals** for observability and coordination
- **Infrastructure** abstracted from implementation

### 1.2 Core Principles (Normative)

These principles MUST be followed by all conformant implementations:

**1. Intent-First, Runtime-Decided**
- The specification declares **what** must be true, not **how** it's achieved
- Transport, locality, and optimization are runtime responsibilities
- Same specification works on cloud, edge, browser, embedded

**2. Versioned Truth**
- All data types, policies, UI, inputs, and workers MUST be versioned
- Semantic change requires new version, even if structure unchanged
- Multiple versions coexist during migration

**3. Linear Evolution for Persisted Data**
- Data versions evolve sequentially (v1→v2→v3)
- Versions MAY NOT be skipped for persistent data types
- Evolution path is explicit and auditable

**4. UI and Input Bind to Data, Not Services**
- UI binds to materialized data versions
- Input binds to target data states
- No service coupling in presentation layer

**5. Zero Downtime by Construction**
- Shadow, promotion, retirement are first-class
- Runtime MAY run multiple versions concurrently
- Gradual rollout prevents incidents

**6. Declarative Authorization**
- Authorization expressed declaratively in grammar
- Enforcement mechanics are runtime-defined
- All authorization evaluated locally with distributed policies

**7. Entity-Scoped Authority** (v1.3)
- Write authority resolved per entity, not per model
- Granular failure isolation
- Per-entity mobility for optimization

**8. Invariants Live in Views** (v1.3)
- Cross-entity invariants enforced by derived views
- Entities remain independently writable
- No distributed transactions required

**9. Compose Views Semantically** (v1.4)
- Group views by relationship, not layout
- Semantic relationship types (derived, detail, contextual, etc.)
- Runtime decides visual presentation

**10. Complete Applications Need Comprehensive Features** (v1.5)
- Real applications require assets, search, notifications, workflows
- All features first-class in grammar
- Compliance and accessibility built-in

### 1.3 Version History

| Version | Features |
|---------|----------|
| v1.0 | Core types, boundaries, work units, SMS flows, WTS, policies, signals |
| v1.1 | Realms, work unit contracts |
| v1.2 | Relationships, authority |
| v1.3 | Presentations, experiences, search, webhooks, external dependencies |
| v1.4 | Graph queries, consent, erasure, legal holds |
| v1.5 | Device capabilities, telemetry, actuator commands, sessions, assets |
| **v1.6** | **Storage binding, unified caching, intent routing, resource requirements, client cache** |

### 1.4 What's New in v1.6

v1.6 introduces five major features for infrastructure abstraction and performance optimization:

1. **Storage Binding**: Declarative configuration for data persistence across KV stores, streams, and object storage
2. **Unified Caching**: Multi-tier caching with invalidation strategies and observability signals
3. **Intent Routing**: Declarative transport configuration for InputIntents with multiple protocols
4. **Resource Requirements**: Abstract infrastructure capability specifications for runtime provisioning
5. **Client Cache Contract**: Client-side caching within Experiences for offline support and performance

### 1.5 v1.5 Capabilities (Included)

This specification includes all v1.5 capabilities for complete application development:

**User Experience & Forms:**
- Form-Intent Binding: Declarative form validation and submission
- Field Permissions: Role-based field visibility and masking

**Data & Search:**
- Asset Semantics: Native file/image/media handling with variants
- Structured Content: Rich text and document types
- Spatial Types: Geographic coordinates and queries
- Search Semantics: Full-text search with faceting and ranking
- Data Governance: Retention, classification, and compliance

**Workflows & Automation:**
- Durable Workflows: Long-running processes with checkpoints
- Scheduled Triggers: Cron-style temporal events
- Notification Channels: Multi-channel push, email, SMS, webhook

**Integration & Authorization:**
- External Integration: Third-party APIs with circuit breakers
- Delegation & Consent: Granular authority transfer with audit
- Graph Queries: Relationship traversal

**Advanced Features:**
- Conversational UI: Voice and chatbot interfaces
- ML/Inference: Inference-derived fields

---

## PART II: CORE INVARIANTS

### 2.1 Fundamental Invariants

These invariants MUST be maintained by all conformant implementations:

| ID | Name | Statement |
|----|------|-----------|
| I1 | Boundary Isolation | Work unit execution is isolated to boundary scope; no cross-boundary state sharing |
| I2 | Policy Before Execution | All operations must evaluate applicable policies before execution |
| I3 | Signal Integrity | Signals are append-only events; emitted signals cannot be modified |
| I4 | CAS Safety | Entity writes with CAS token must fail if entity version has changed |
| I5 | Realm Separation | Data in different realms is logically isolated; cross-realm access requires explicit policy |
| I6 | Idempotency | Work units with idempotency_key must produce same result for repeated invocations |
| I7 | Governance Enforcement | Consent and legal requirements must be validated before data operations |
| I8 | Device Binding | Actuator commands are scoped to registered devices; unregistered devices cannot receive commands |
| **I9** | **Transport Scope Boundary** (v1.6) | Transport bindings SHALL be specified only at system ingress points (InputIntent routing). Transport SHALL NOT be specified within SMS flow definitions |

### 2.2 Invariant Enforcement

**Compile-Time Enforcement:**
- I1: Boundary scope validation - no cross-boundary state sharing in work units
- I5: Realm reference validation - cross-realm access requires explicit policy
- I8: Device registration validation - actuator commands scoped to registered devices
- I9: Transport binding location validation - routing only on InputIntent

**Runtime Enforcement:**
- I2: Policy evaluation hooks - all operations evaluate policies before execution
- I3: Append-only storage - emitted signals cannot be modified
- I4: CAS token comparison - entity writes fail if version changed
- I6: Idempotency tracking - same result for repeated invocations
- I7: Consent/hold checks - governance validated before data operations

### 2.3 Formal Invariant Definitions

**I1. Authority Singularity**: At any point in time, an entity SHALL have at most one write-authoritative region.

**I2. Epoch Monotonicity**: Authority epochs SHALL increase monotonically and SHALL NOT be reused.

**I3. No Overlapping Authority**: No two regions SHALL simultaneously accept writes for the same entity.

**I4. CAS Safety**: A write SHALL be accepted if and only if:
- The authority region is current
- The authority epoch matches
- The entity version matches

**I5. View Continuity**: Derived views SHALL remain readable across authority transitions.

**I6. Rebuildability**: All derived state SHALL be reconstructable from authoritative event history.

**I7. Governance Compliance**: Authority transitions SHALL NOT violate declared governance constraints.

**I8. Failure Determinism**: On failure during authority transition, the system SHALL resolve to a single authoritative region without ambiguity.

**I9. Transport Scope Boundary** (v1.6): Transport bindings SHALL be specified only at system ingress points (InputIntent routing). Transport SHALL NOT be specified within SMS flow definitions.

### 2.5 File Structure and Collections

SMS specifications use **plural collection keys** for all multi-instance declaration types. This prevents YAML duplicate key conflicts when multiple declarations of the same type appear in a single document.

**Standard Document Structure:**

```yaml
version: "1.6"
domain: string

system:
  name: string

# Core Data
dataTypes:
  - name: TypeA
    version: v1
    # ...
  - name: TypeB
    version: v1
    # ...

models:
  - name: ModelA
    version: v1
    # ...

dataStates:
  - name: StateA
    type: TypeA
    # ...

constraints:
  - name: ConstraintA
    appliesTo: TypeA
    # ...

relationships:
  - name: RelA
    # ...

# Presentation
views:
  - name: ViewA
    version: v1
    # ...

materializations:
  - name: MatA
    version: v1
    # ...

presentationViews:
  - name: PresentationViewA
    version: v1
    # ...

experiences:
  - name: ExperienceA
    version: v1
    # ...

# Execution
inputIntents:
  - name: IntentA
    version: v1
    # ...

smsFlows:
  - name: FlowA
    version: v1
    # ...

workers:
  - name: WorkerA
    # ...

realms:
  - name: RealmA
    version: v1
    # ...

policies:
  - name: PolicyA
    version: v1
    # ...

signals:
  - name: SignalA
    version: v1
    # ...

# Infrastructure (v1.6)
storageBindings:
  - name: StoreA
    version: v1
    # ...

cacheLayers:
  - name: CacheA
    version: v1
    # ...

resourceRequirements:
  name: Resources
  version: v1
  # ...

# Additional collections as needed:
# assets, delegations, searchIntents, webhookReceivers,
# externalDependencies, graphQueries, dataGovernances,
# conversationalExperiences, edgeDevices, etc.
```

**Minimal Valid Specification:**

```yaml
version: "1.6"
domain: "orders"

system:
  name: OrderSystem
  version: v1

dataTypes:
  - name: Order
    version: v1
    kind: state
    fields:
      - name: id
        type: uuid
        required: true
      - name: total
        type: money
        required: true

inputIntents:
  - name: CreateOrder
    version: v1
    proposes:
      dataState: OrderRecord
      version: v1
    supplies:
      required: [total]
    transitions:
      to: OrderRecord

smsFlows:
  - name: CreateOrderFlow
    version: v1
    triggeredBy:
      inputIntent: CreateOrder
    steps:
      - persist: OrderRecord

realms:
  - name: OrderService
    version: v1
    owns: [Order]
    workUnits: [CreateOrder]
```

**Singular vs. Plural Keys:**

| Key Type | Key Name | Usage |
|----------|----------|-------|
| Singular | `system` | One per specification |
| Singular | `resourceRequirements` | Infrastructure requirements (at most one) |
| Plural | `dataTypes` | Collection of DataType declarations |
| Plural | `models` | Collection of Model declarations |
| Plural | `dataStates` | Collection of DataState declarations |
| Plural | `constraints` | Collection of Constraint declarations |
| Plural | `dataEvolutions` | Collection of DataEvolution declarations |
| Plural | `materializations` | Collection of Materialization declarations |
| Plural | `transitions` | Collection of Transition declarations |
| Plural | `relationships` | Collection of Relationship declarations |
| Plural | `authorities` | Collection of Authority declarations |
| Plural | `views` | Collection of View declarations (data views) |
| Plural | `presentationViews` | Collection of PresentationView declarations (UI views) |
| Plural | `presentationBindings` | Collection of PresentationBinding declarations |
| Plural | `presentationCompositions` | Collection of PresentationComposition declarations |
| Plural | `experiences` | Collection of Experience declarations |
| Plural | `inputIntents` | Collection of InputIntent declarations |
| Plural | `smsFlows` | Collection of SMS Flow declarations |
| Plural | `workers` | Collection of Worker declarations |
| Plural | `workUnitContracts` | Collection of WorkUnitContract declarations |
| Plural | `realms` | Collection of Realm declarations |
| Plural | `policies` | Collection of Policy declarations |
| Plural | `signals` | Collection of Signal declarations |
| Plural | `searchIntents` | Collection of SearchIntent declarations |
| Plural | `webhookReceivers` | Collection of WebhookReceiver declarations |
| Plural | `externalDependencies` | Collection of ExternalDependency declarations |
| Plural | `graphQueries` | Collection of GraphQuery declarations |
| Plural | `dataGovernances` | Collection of DataGovernance declarations (v1.5) |
| Plural | `consentRecords` | Collection of ConsentRecord declarations (v1.5) |
| Plural | `erasureRequests` | Collection of ErasureRequest declarations (v1.5) |
| Plural | `legalHolds` | Collection of LegalHold declarations (v1.5) |
| Plural | `notificationChannels` | Collection of NotificationChannel declarations (v1.5) |
| Plural | `scheduledTriggers` | Collection of ScheduledTrigger declarations (v1.5) |
| Plural | `durableWorkflows` | Collection of DurableWorkflow declarations (v1.5) |
| Plural | `assets` | Collection of Asset declarations (v1.5) |
| Plural | `delegations` | Collection of Delegation declarations (v1.5) |
| Plural | `conversationalExperiences` | Collection of ConversationalExperience declarations (v1.5) |
| Plural | `deviceCapabilities` | Collection of DeviceCapability declarations (IoT) |
| Plural | `deviceTelemetries` | Collection of DeviceTelemetry declarations (IoT) |
| Plural | `actuatorCommands` | Collection of ActuatorCommand declarations (IoT) |
| Plural | `edgeDevices` | Collection of EdgeDevice declarations (IoT) |
| Plural | `storageBindings` | Collection of StorageBinding declarations (v1.6) |
| Plural | `cacheLayers` | Collection of CacheLayer declarations (v1.6) |

---

## PART III: LEXICAL STRUCTURE

### 3.1 Document Format

SMS specifications are written in YAML format with the following structure:

```yaml
# Root document
version: "1.6"
domain: string
imports: [string]

# Core data
dataTypes: [...]
models: [...]
dataStates: [...]
constraints: [...]
dataEvolutions: [...]
transitions: [...]
relationships: [...]
authorities: [...]

# Presentation
views: [...]
presentationViews: [...]
presentationBindings: [...]
presentationCompositions: [...]
materializations: [...]
experiences: [...]
conversationalExperiences: [...]

# Execution
inputIntents: [...]
smsFlows: [...]
workers: [...]
workUnitContracts: [...]
realms: [...]
policies: [...]
signals: [...]

# Integration
searchIntents: [...]
webhookReceivers: [...]
externalDependencies: [...]
graphQueries: [...]

# Content (v1.5)
assets: [...]
delegations: [...]

# Governance (v1.5)
dataGovernances: [...]
consentRecords: [...]
erasureRequests: [...]
legalHolds: [...]

# Operations (v1.5)
notificationChannels: [...]
scheduledTriggers: [...]
durableWorkflows: [...]

# IoT
deviceCapabilities: [...]
deviceTelemetries: [...]
actuatorCommands: [...]
edgeDevices: [...]

# Infrastructure (v1.6)
storageBindings: [...]
cacheLayers: [...]
resourceRequirements: {...}
```

### 3.2 Identifiers

```
identifier      = letter { letter | digit | "_" } ;
qualified_name  = identifier { "." identifier } ;
reference       = "@" qualified_name ;
```

**Naming Conventions:**
- Types: PascalCase (e.g., `OrderCreated`, `AccountBalance`)
- Fields: camelCase (e.g., `orderId`, `createdAt`)
- Work Units: PascalCase (e.g., `CreateOrder`, `ProcessPayment`)
- Policies: snake_case (e.g., `can_view_order`, `admin_access`)

### 3.3 Versions

```
version_string  = "v" number [ "." number [ "." number ] ] ;
```

All named declarations require a version:
- `v1` - Major version only
- `v1.2` - Major and minor
- `v1.2.3` - Full semantic version

---

## PART IV: CORE DATA TYPES

### 4.1 DataType Declaration

```yaml
dataTypes:
  - name: string          # Required: PascalCase name
    version: version      # Required: Version string
    kind: enum            # Required: event | state | input | view
    description: string   # Optional: Human-readable description
    fields: [Field]       # Required: Field definitions
    validation: [Rule]    # Optional: Cross-field validation
```

### 4.2 Field Definition

```yaml
field:
  name: string          # Required: Field name
  type: type_expr       # Required: Type expression
  required: boolean     # Optional: Default false
  default: any          # Optional: Default value
  description: string   # Optional: Documentation
  validation: [Rule]    # Optional: Field-level rules
  sensitive: boolean    # Optional: Privacy marker
```

**Basic Validation Rules:**

All DataType declarations MUST follow these validation rules:

1. **Name Validation (V1)**: Names MUST start with uppercase letter and contain only alphanumeric and underscore characters
2. **Version Format (V2)**: Versions MUST follow pattern `v<major>[.<minor>][.<patch>]` (e.g., `v1`, `v1.2`, `v1.2.3`)
3. **Reference Validation (V3)**: All type references MUST refer to defined artifacts and include version if versioned
4. **Required Fields**: `name`, `version`, `kind`, and `fields` are required for all DataType declarations
5. **Type Validity**: Field `type` values MUST be valid scalar types, composite types, or defined custom types

For comprehensive validation rules including evolution, circular references, storage binding, caching, and semantic constraints, see **[Appendix H: Extended Validation Rules](#appendix-h-extended-validation-rules)**.

### 4.3 Type Expressions

**Scalar Types:**
- `string`, `int`, `float`, `bool`, `timestamp`, `uuid`, `bytes`, `duration`, `money`, `url`, `email`, `phone`
- `asset` - File attachments with variants (see Section 4.8)
- `structured_content` - Rich text and documents (see Section 4.9)
- `spatial` - Geographic coordinates and shapes (see Section 4.10)

**Composite Types:**
- `list[T]` - Ordered collection
- `set[T]` - Unordered unique collection
- `map[K, V]` - Key-value mapping
- `optional[T]` - Nullable type
- `enum[value1, value2, ...]` - Enumeration of string literals
- `Reference<TypeName>` - Entity reference

**Enum Syntax:**

Enumerations define a closed set of string values:

```yaml
# Inline enum declaration
status: enum[draft, pending, approved, rejected]

# Used in field definitions
fields:
  - name: priority
    type: enum[low, medium, high, critical]
    required: true
```

Enums are:
- **Compile-time validated**: Only declared values are accepted
- **String-based**: All enum values are strings
- **Extensible via evolution**: New values can be added in new versions (see Section 4.6)
- **Case-sensitive**: `draft` and `Draft` are different values

**Example:**
```yaml
dataTypes:
  - name: Order
    version: v1
    kind: state
    fields:
      - name: id
        type: uuid
        required: true
      - name: customerId
        type: Reference<Customer>
        required: true
      - name: items
        type: list[OrderItem]
        required: true
      - name: total
        type: money
        required: true
      - name: status
        type: OrderStatus
        required: true
      - name: createdAt
        type: timestamp
        required: true
```

### 4.4 Data Kinds

| Kind | Purpose | Mutability | Persistence |
|------|---------|------------|-------------|
| `event` | Record state transitions | Immutable | Append-only |
| `state` | Current entity state | Mutable (CAS) | Read-write |
| `input` | External input data | Immutable | Transient |
| `view` | Derived/projected data | Computed | Cached |

### 4.5 Constraint Declaration

Defines semantic truth - what must be true about data. Constraints are version-aware semantic invariants that validate data correctness.

**Grammar:**
```yaml
constraints:
  - name: string              # Required: Constraint name
    appliesTo: DataType       # Required: DataType to validate
    rules:
      version: expression     # Required: Boolean expression per version
```

**Purpose**: Constraints are separate from DataType to enable:
- Constraint evolution without structural change
- Different constraints for different versions
- Enforcement flexibility (strict/best-effort/deferred)

**Expression Language**: Boolean expressions referencing fields:
- Comparisons: `==`, `!=`, `<`, `>`, `<=`, `>=`
- Logical: `AND`, `OR`, `NOT`
- Arithmetic: `+`, `-`, `*`, `/`
- Functions: 
  - `len(collection)`: Returns length of a list, set, or string
  - `sum(collection)`: Returns sum of numeric collection
  - `any(collection, condition)`: Returns true if any element matches condition
  - `all(collection, condition)`: Returns true if all elements match condition
  - `exists(reference)`: Returns true if referenced entity exists

**Example:**
```yaml
constraints:
  - name: OrderTotalValid
    appliesTo: Order
    rules:
      v1: total >= 0 AND total == sum(items.price * items.quantity)
      v2: total >= 0 AND total == sum(items.price * items.quantity * (1 - items.discount))
```

**Validation**: Expressions MUST be deterministic and side-effect-free.

---

### 4.6 Data Evolution Declaration

Declares allowed schema changes between versions for persistent data types.

**Grammar:**
```yaml
dataEvolutions:
  - name: string              # Required: Evolution name
    from:
      type: DataType          # Required: Source type
      version: version        # Required: Source version
    to:
      type: DataType          # Required: Target type
      version: version        # Required: Target version
    changes: [ChangeOp]       # Required: Change operations
    compatibility:
      read: enum              # Required: backward | forward | none
      write: enum             # Required: backward | forward | none
```

**Change Operations:**
- `addField`: Add new field with default value
- `removeField`: Remove field (requires backward compatibility)
- `renameField`: Rename field (requires migration)
- `changeType`: Change field type (usually breaking)
- `extendEnum`: Add enum values (backward compatible)
- `refineConstraint`: Tighten constraints (may break old writers)

**Compatibility Modes:**
- `backward`: New code can read old data
- `forward`: Old code can read new data
- `none`: Breaking change

**Example:**
```yaml
dataEvolutions:
  - name: OrderV1toV2
    from:
      type: Order
      version: v1
    to:
      type: Order
      version: v2
    changes:
      - addField:
          name: discount
          type: decimal
          default: 0.0
      - extendEnum:
          field: status
          values: [cancelled, refunded]
    compatibility:
      read: backward    # v2 code can read v1 data
      write: forward    # v1 code can write data v2 code can read
```

---

### 4.7 Transition Declaration

Transitions are typed edges between DataStates with explicit triggers and guarantees.

**Grammar:**
```yaml
transitions:
  - name: string              # Required: Transition name
    from: DataState           # Required: Source state
    to: DataState             # Required: Target state
    triggeredBy:
      inputIntent | smsFlow   # Required: Trigger type
    guarantees: [string]      # Required: Transition guarantees
    idempotency:
      key: expression         # Required: Idempotency key
      scope: enum             # Required: global | per_subject | per_realm
    authorization:
      policySet: PolicySet    # Optional: Policy set
```

**Purpose**: 
- Typed edges between states
- Explicit triggers (never implicit)
- Declared idempotency
- Authorizable independently

**Idempotency (Normative)**:
- ALL transitions MUST be idempotent
- Executing same transition multiple times produces same result
- Deduplication keys MUST be deterministic
- Workers MUST tolerate duplicate delivery

**Example:**
```yaml
transitions:
  - name: ApproveOrder
    from: OrderRecord
    to: OrderRecord
    triggeredBy:
      inputIntent: SubmitApproval
    guarantees:
      - approval_recorded
      - idempotent_application
    idempotency:
      key:
        - approval.id
        - order.id
      scope: global
    authorization:
      policySet: OrderApprovalPolicies
```

---

### 4.7a Invariant Policy (v1.3)

Declares how invariants are enforced - prevents embedding cross-entity invariants inside entities.

**Grammar:**
```yaml
invariants:
  enforced_by: view | policy
  description: string       # Optional: Documentation
```

**Enforcement Options:**
- `view`: Invariants validated by derived views (RECOMMENDED for cross-entity)
- `policy`: Invariants enforced by policy evaluation

**Why Views?**
- Entities remain independently writable
- No distributed transactions required
- Eventual consistency tolerated
- Violations flagged rather than blocked

**Design Rule**: If an invariant spans multiple entities with different authority, it MUST be enforced by a view, not by the entities themselves.

**Example - Balance Invariant:**
```yaml
# DON'T: Embed cross-account invariant in Account entity
# (Would require distributed transaction)

# DO: Enforce in view
materializations:
  - name: AccountBalanceView
    version: v1
    source: [AccountDebitEvent, AccountCreditEvent]
    target: AccountBalance
    invariants:
      enforced_by: view
      description: "Balance = credits - debits, never negative"
```

**Rationale**: 
- Cross-entity invariants create coordination requirements
- Views provide natural enforcement boundary without locking
- Violations are detected and reported asynchronously
- System remains available even when invariants are temporarily violated
- Repair mechanisms can be triggered on violation detection

---

### 4.7b DataState Examples

DataState defines lifecycle, enforcement, and evolution policy for data.

**Example - Persistent Order (Strong Consistency):**
```yaml
dataStates:
  - name: OrderRecord
    type: Order
    lifecycle: persistent
    constraintPolicy:
      enforcement: strict
    evolutionPolicy:
      allowed: [additive, constraint_refinement]
      forbidden: [destructive, semantic_break]
    mutability:
      scope: entity
      exclusive: true
    storage:
      role: data
      rebuildable: false
```

**Example - Materialized Dashboard (Derived View):**
```yaml
dataStates:
  - name: CustomerDashboard
    type: DashboardData
    lifecycle: materialized
    constraintPolicy:
      enforcement: best_effort
    evolutionPolicy:
      allowed: [additive, destructive]  # Views can rebuild
    storage:
      role: data
      rebuildable: true  # Can regenerate from source events

materializations:
  - name: CustomerDashboardView
    version: v1
    source: [CustomerEvent, OrderEvent, PaymentEvent]
    target: CustomerDashboard
    freshness:
      maxStaleness: 5m
    evolution:
      strategy: rebuild
      blocking: false
```

**Lifecycle Types:**
- `persistent`: Long-lived, explicitly managed data
- `intermediate`: Short-lived workflow data
- `materialized`: Derived, regenerable from sources

**Constraint Enforcement:**
- `strict`: Block invalid writes (strong consistency)
- `best_effort`: Log violations, allow writes (eventual consistency)
- `deferred`: Validate asynchronously
- `none`: No validation

**Evolution Policies:**
- `additive`: Add fields, extend enums
- `constraint_refinement`: Tighten constraints
- `destructive`: Remove fields, change types
- `semantic_break`: Change meaning without structure change

---

### 4.8 Asset Semantics (v1.5)

Assets represent binary content (files, images, videos, documents) as first-class data types.

**Asset Field Configuration:**
```yaml
dataTypes:
  - name: Document
    version: v1
    fields:
      attachment:
        type: asset
        asset:
          category: image | document | media | archive | generic
          constraints:
            max_size: string        # e.g., "10MB", "500KB"
            mime_types: [string]    # e.g., ["image/jpeg", "image/png"]
            extensions: [string]    # e.g., [".jpg", ".png"]
          lifecycle:
            type: permanent | temporary | expiring
            expires_after: duration  # For expiring type
          access:
            read: public | signed | authenticated | policy
            policy: policy_ref       # If access=policy
            signed_url_ttl: duration # If access=signed
          variants:
            - name: thumbnail
              transform: resize
              params: { width: 150, height: 150 }
```

**Asset Categories:**
- `image`: Raster/vector images (JPEG, PNG, SVG, etc.)
- `document`: PDFs, Word docs, spreadsheets
- `media`: Audio/video files
- `archive`: Compressed archives (ZIP, TAR, etc.)
- `generic`: Any binary content

**Lifecycle Types:**
- `permanent`: Stored indefinitely, survives entity deletion
- `temporary`: Short-lived staging area (e.g., upload preprocessing)
- `expiring`: Time-bounded with automatic deletion after expiry

**Access Modes:**
- `public`: Publicly accessible URL (no authentication)
- `signed`: Time-limited signed URL (secure temporary access)
- `authenticated`: Requires valid authentication token
- `policy`: Policy-based access control (most secure)

**Transform Types** (for variants):
- `resize`: Scale to dimensions
- `thumbnail`: Generate thumbnail
- `compress`: Reduce file size
- `convert`: Convert format (e.g., HEIC→JPEG)
- `extract_page`: Extract page from PDF

**Runtime Responsibilities:**
- Upload handling (multipart, resumable, direct-to-storage)
- Variant generation and caching
- CDN integration for delivery
- Garbage collection for expiring assets

**Example - Profile Picture:**
```yaml
dataTypes:
  - name: UserProfile
    version: v1
    kind: state
    fields:
      - name: user_id
        type: uuid
        required: true
      - name: avatar
        type: asset
        asset:
          category: image
          constraints:
            max_size: 5MB
            mime_types: ["image/jpeg", "image/png", "image/webp"]
          lifecycle:
            type: permanent
          access:
            read: public
          variants:
            - name: thumbnail
              transform: thumbnail
              params: { width: 64, height: 64 }
            - name: medium
              transform: resize
              params: { width: 256, height: 256 }
```

**Example - Invoice PDF:**
```yaml
dataTypes:
  - name: Invoice
    version: v1
    kind: state
    fields:
      - name: invoice_id
        type: uuid
        required: true
      - name: pdf_document
        type: asset
        asset:
          category: document
          constraints:
            max_size: 10MB
            mime_types: ["application/pdf"]
          lifecycle:
            type: permanent
          access:
            read: policy
            policy: InvoiceAccessPolicy
```

---

### 4.9 Structured Content (v1.5)

Structured content enables rich text and document editing with semantic structure.

```yaml
dataTypes:
  - name: Article
    version: v1
    fields:
      body:
        type: structured_content
        structured_content:
          blocks:
            allowed: [paragraph, heading, list, blockquote, code, image, video, table]
          marks:
            allowed: [bold, italic, underline, link, code, highlight]
          constraints:
            max_length: 50000  # Character limit
          sanitization:
            strip_scripts: true
            allowed_tags: [p, h1, h2, h3, ul, ol, li, blockquote, code, pre]
```

**Block Types:**
- `paragraph`, `heading`, `list`, `blockquote`, `code`, `table`, `image`, `video`, `embed`

**Mark Types (inline formatting):**
- `bold`, `italic`, `underline`, `strikethrough`, `code`, `link`, `mention`, `hashtag`, `highlight`

**Sanitization:**
- `strip_scripts`: Remove JavaScript and event handlers (security)
- `allowed_tags`: Whitelist of HTML tags (if rendering to HTML)
- `allowed_attributes`: Whitelist of HTML attributes

**Runtime Responsibilities:**
- Parsing and validation of structured content
- Sanitization enforcement
- Rendering to HTML/markdown
- Collaborative editing (Operational Transform or CRDT)
- Asset embedding resolution

**Example - Blog Post:**
```yaml
dataTypes:
  - name: BlogPost
    version: v1
    kind: state
    fields:
      - name: post_id
        type: uuid
        required: true
      - name: title
        type: string
        required: true
      - name: content
        type: structured_content
        structured_content:
          blocks:
            allowed: [paragraph, heading, list, blockquote, code, image]
          marks:
            allowed: [bold, italic, link, code]
          constraints:
            max_length: 100000
          sanitization:
            strip_scripts: true
            allowed_tags: [p, h1, h2, h3, ul, ol, li, blockquote, code, pre, img]
            allowed_attributes: [href, src, alt, class, id]
```

---

### 4.10 Spatial Types (v1.5)

Spatial types enable geographic and location-based data with indexing and queries.

```yaml
dataTypes:
  - name: Store
    version: v1
    fields:
      location:
        type: spatial
        spatial:
          geometry: point | line | polygon | multi_point | multi_line | multi_polygon
          coordinate_system: wgs84 | web_mercator | custom
          srid: integer             # Spatial Reference System ID
          precision: decimal        # Coordinate precision
          search:
            indexed: true | false
            strategy: bounding_box | radius | polygon_intersection
```

**Geometry Types:**
- `point`: Single coordinate (lat, lon)
- `line`: Line string (sequence of coordinates)
- `polygon`: Closed polygon (boundary coordinates)
- `multi_point`, `multi_line`, `multi_polygon`: Collections

**Coordinate Systems:**
- `wgs84`: World Geodetic System 1984 (GPS standard)
- `web_mercator`: Web Mercator projection (web maps)

**Search Strategies:**
- `bounding_box`: Rectangular area search (fast, approximation)
- `radius`: Distance from point (circular area)
- `polygon_intersection`: Precise polygon intersection (slower)

**Runtime Responsibilities:**
- Spatial indexing (R-tree, Geohash, S2, etc.)
- Distance calculations
- Containment queries
- Intersection queries
- Integration with mapping services

**Example - Delivery Zone:**
```yaml
dataTypes:
  - name: DeliveryZone
    version: v1
    kind: state
    fields:
      - name: zone_id
        type: uuid
        required: true
      - name: name
        type: string
        required: true
      - name: boundary
        type: spatial
        spatial:
          geometry: polygon
          coordinate_system: wgs84
          search:
            indexed: true
            strategy: polygon_intersection
```

**Example - GPS Tracker:**
```yaml
dataTypes:
  - name: DeviceLocation
    version: v1
    kind: event
    fields:
      - name: device_id
        type: uuid
        required: true
      - name: position
        type: spatial
        spatial:
          geometry: point
          coordinate_system: wgs84
          precision: 0.000001  # ~10cm accuracy
          search:
            indexed: true
            strategy: radius
      - name: timestamp
        type: timestamp
        required: true
```

---

### 4.11 Search Semantics (v1.5)

Search configuration at the field level for full-text, faceted, and ranked search.

**Field-Level Search:**
```yaml
dataTypes:
  - name: Product
    version: v1
    fields:
      name:
        type: string
        search:
          indexed: true
          strategy: full_text | exact | prefix | fuzzy | semantic
          full_text:
            analyzer: standard
            stemming: enabled
            stop_words: enabled
          weight: 2.0              # Boost relevance
          suggest:
            enabled: true
            type: completion
          highlight:
            enabled: true
            fragment_size: 150
```

**Search Strategies:**
- `full_text`: Natural language search with analysis
- `exact`: Exact match only
- `prefix`: Prefix/autocomplete matching
- `fuzzy`: Typo-tolerant matching
- `semantic`: Vector/embedding similarity (requires ML integration)

**Query Parameters:**
- `phrase_slop`: Number of positions words can be apart in phrase queries. Allows finding phrases where words appear in different orders or with intervening words. For example, with `phrase_slop: 2`, searching for "quick brown fox" would match "quick lazy brown fox" (1 intervening word) or "brown quick fox" (words swapped, distance of 2).

**Search Intent Declaration:**
```yaml
searchIntents:
  - name: ProductSearch
    version: v1
    searches: ProductCatalog
    query:
      fields: [name, description, tags]
      default_operator: and | or
      phrase_slop: int              # Allow words in different order (optional)
    filters:
      required: [category]
      optional: [brand, price_range]
    facets:
      include: [category, brand, price_range]
    pagination:
      default_size: 20
      max_size: 100
    sort:
      default: _score
      allowed: [price, created_at, popularity]
    authorization:
      policy_set: SearchPolicies
```

### 4.12 Data Governance (v1.5)

Compliance, retention, and data classification for regulatory requirements.

```yaml
dataStates:
  - name: CustomerRecord
    type: Customer
    lifecycle: persistent
    governance:
      classification: pii | phi | pci | financial | public | internal | confidential
      retention:
        policy: time_based | event_based | indefinite | legal_hold
        duration: duration
        after_event: event_name
        on_expiry: delete | anonymize | archive
      residency:
        allowed_regions: [us-east, us-west, eu-west]
        forbidden_regions: [cn-north]
        transfer:
          allowed: boolean
          requires_consent: boolean
      audit:
        access_logging: boolean
        modification_logging: boolean
        retention_for_logs: duration
      encryption:
        at_rest: required | optional
        in_transit: required
        key_rotation: duration
```

**Classification Levels:**
- `pii`: Personally Identifiable Information (GDPR)
- `phi`: Protected Health Information (HIPAA)
- `pci`: Payment Card Industry data
- `financial`: Financial records (regulatory retention)
- `public`, `internal`, `confidential`: General classifications

**Retention Policies:**
- `time_based`: Delete/archive after fixed duration
- `event_based`: Delete/archive after specific event (e.g., account_closed)
- `indefinite`: Keep forever (regulatory requirement)
- `legal_hold`: Preserve due to legal requirement (immutable)

---

## PART V: BOUNDARIES AND WORK UNITS

### 5.1 Boundary Declaration

Boundaries are the unit of deployment and isolation:

```yaml
realms:
  - name: string          # Required: Boundary name
    version: version      # Required: Version
    description: string   # Optional: Documentation
    owns: [TypeName]      # Required: Owned data types
    workUnits: [WorkUnit] # Required: Contained work units
    policy_set: string    # Optional: Default policies
```

**Example:**
```yaml
realms:
  - name: OrderService
    version: v1
    description: "Handles order lifecycle"
    owns:
      - Order
      - OrderCreated
      - OrderShipped
    workUnits:
      - CreateOrder
      - ShipOrder
      - CancelOrder
    policy_set: OrderPolicies
```

### 5.2 Work Unit Declaration

Work units are the atomic unit of business logic:

```yaml
workUnits:
  - name: string               # Required: Work unit name
    version: version           # Required: Version
    description: string        # Optional: Documentation
    input: TypeName            # Required: Input data type
    outputs:                   # Required: Possible outputs
      success: TypeName
      failure: TypeName
    emits: [TypeName]          # Optional: Events emitted
    reads: [TypeName]          # Optional: State read
    writes: [TypeName]         # Optional: State written
    policy_set: string         # Optional: Required policies
    idempotency_key: expr      # Optional: Idempotency expression
    timeout: duration          # Optional: Execution timeout
```

### 5.3 Work Unit Contract (v1.1+)

Contracts formalize work unit behavior:

```yaml
workUnitContracts:
  - name: string
    version: version
    realm: RealmRef
    work_unit: WorkUnitRef
    
    preconditions:            # Must be true before execution
      - condition: expr
        error_code: string
        
    postconditions:           # Must be true after execution
      - condition: expr
        
    invariants:               # Must be true throughout
      - condition: expr
```

---

## PART VI: FLOW DEFINITIONS

### 6.1 SMS Flow Declaration

SMS flows define state machine orchestrations:

```yaml
smsFlows:
  - name: string              # Required: Flow name
    version: version          # Required: Version
    description: string       # Optional: Documentation
    initial_state: string     # Required: Starting state
    final_states: [string]    # Required: Terminal states
    states: [State]           # Required: State definitions
    transitions: [Transition] # Required: Transition rules
    timeout: duration         # Optional: Overall timeout
    policy_set: string        # Optional: Required policies
```

### 6.2 State Definition

```yaml
state:
  name: string              # Required: State name
  type: enum                # Required: initial | intermediate | final | error
  on_enter: [Action]        # Optional: Entry actions
  on_exit: [Action]         # Optional: Exit actions
```

### 6.3 Transition Definition

```yaml
transitions:
  - from: string | [string]   # Required: Source state(s)
    to: string                # Required: Target state
    on: string                # Required: Trigger event
    guard: expr               # Optional: Condition
    actions: [Action]         # Optional: Transition actions
```

### 6.4 Workers Declaration

Workers define the progression of work within boundaries:

```yaml
workers:
  - name: string              # Required: Worker name
    version: version          # Required: Version
    realm: RealmRef           # Required: Owning realm
    entity_type: TypeName     # Required: Entity being tracked
    states: [WTSState]        # Required: Work states
    transitions: [WTSTransition]
```

### 6.5 Durable Workflows (v1.5)

Long-running processes with fault tolerance, compensation, and human-in-the-loop steps.

```yaml
smsFlows:
  - name: LoanApprovalFlow
    version: v1
    # ... standard flow fields
    
    durable:
      enabled: true
      checkpoint:
        frequency: per_step | per_group | manual
        storage: storage_ref
      compensation:
        enabled: boolean
        on_failure: rollback | compensate | manual
      timeout:
        total: duration         # Overall workflow timeout
        per_step: duration      # Per-step timeout
      retry:
        enabled: boolean
        max_attempts: int
        backoff: exponential | linear
      human_tasks:
        - step: step_name
          timeout: duration
          escalation:
            after: duration
            to: role_ref
```

**Checkpoint Strategies:**
- `per_step`: Checkpoint after each step (most durable)
- `per_group`: Checkpoint after atomic groups
- `manual`: Explicit checkpoint markers only

**Compensation Modes:**
- `rollback`: Attempt to reverse completed steps
- `compensate`: Execute specific compensation logic
- `manual`: Flag for human intervention

**Example - Loan Approval with Human Tasks:**
```yaml
smsFlows:
  - name: LoanApprovalWorkflow
    version: v1
    initial_state: ApplicationReceived
    durable:
      enabled: true
      checkpoint:
        frequency: per_step
      compensation:
        enabled: true
        on_failure: compensate
      timeout:
        total: 14d
    states:
      - name: ApplicationReceived
        type: initial
      - name: UnderwriterReview
        type: intermediate
      - name: CustomerSignature
        type: intermediate
      - name: Approved
        type: final
    transitions:
      - from: ApplicationReceived
        to: UnderwriterReview
        on: credit_check_passed
      - from: UnderwriterReview
        to: CustomerSignature
        on: underwriter_approved
        human_task:
          timeout: 48h
          escalation:
            after: 24h
            to: senior_underwriter
      - from: CustomerSignature
        to: Approved
        on: customer_signed
        human_task:
          timeout: 7d
```

### 6.6 Scheduled Triggers (v1.5)

Cron-style temporal events for recurring automation.

```yaml
inputIntents:
  - name: GenerateDailyReport
    version: v1
    # ... standard intent fields
    
    scheduled:
      enabled: true
      trigger:
        type: cron | interval | once
        cron: cron_expression      # e.g., "0 9 * * MON-FRI"
        interval: duration         # e.g., "1h", "30m"
        at: timestamp              # For "once" type
      timezone:
        mode: fixed | user_context | entity_context
        fixed: timezone            # e.g., "America/New_York"
        context_field: field_name  # For context modes
      window:
        start: time_expression
        end: time_expression
        skip_outside: boolean
      idempotency:
        key: expression
        scope: global | per_subject | per_entity
      on_missed:
        action: skip | execute_immediately | queue
```

**Trigger Types:**
- `cron`: Cron expression (Unix cron syntax)
- `interval`: Recurring interval
- `once`: Single execution at timestamp

**Missed Execution Strategies:**
- `skip`: Don't run missed executions
- `execute_immediately`: Run as soon as possible
- `queue`: Queue for later execution

**Example - Monthly Statement Generation:**
```yaml
inputIntents:
  - name: GenerateMonthlyStatement
    version: v1
    proposes:
      dataState: StatementRecord
    scheduled:
      enabled: true
      trigger:
        type: cron
        cron: "0 0 1 * *"   # First day of month at midnight
      timezone:
        mode: fixed
        fixed: "UTC"
      idempotency:
        key: "{{year}}-{{month}}"
        scope: global
      on_missed:
        action: execute_immediately
```

---

## PART VII: UI AND PRESENTATION

### 7.1 View Declaration

Views define UI presentation of data:

```yaml
views:
  - name: string              # Required: View name
    version: version          # Required: Version
    description: string       # Optional: Documentation
    data_source: TypeName     # Required: Source data type (legacy)
    consumes:                 # Optional: Explicit version specification (v1.5+)
      dataState: DataState    # Required: Data state reference
      version: version        # Required: Explicit version
    guarantees: [string]      # Optional: Behavioral guarantees
    fields: [ViewField]       # Required: Displayed fields
    actions: [Action]         # Optional: Available actions
    layout: Layout            # Optional: Layout hints
    policy_set: string        # Optional: Access policies
    interactionModel: InteractionModel  # Optional: UI interaction patterns
```

**Data Source Specification:**
- `data_source`: Simplified syntax for unversioned reference (legacy, still supported)
- `consumes`: Explicit version specification with data state and version (RECOMMENDED)

**Guarantees**: What the UI promises to display or enable:
- `shows_all_items`: Display all items from source
- `real_time_updates`: Updates reflect changes immediately
- `sortable`: Users can sort data
- `filterable`: Users can filter data
- `paginated`: Data is paginated for large sets
- `searchable`: Data supports text search

**Interaction Model**:
```yaml
interactionModel:
  badges:
    - if: item.urgent == true
      label: "URGENT"
      severity: info | warning | error | critical
    - if: item.status == "pending"
      label: "{{item.status}}"
      severity: info
  actions:
    - if: item.status == "pending"
      action: approve
      label: "Approve"
      intent: ApproveIntent
  filters:
    - field: status
      type: enum
      options: [pending, approved, rejected]
    - field: created_at
      type: date_range
    - field: amount
      type: numeric_range
  sorts:
    - field: created_at
      default: desc
    - field: amount
      default: asc
```

**Badge Severities:**
- `info`: Informational badge (blue)
- `warning`: Warning badge (yellow)
- `error`: Error badge (red)
- `critical`: Critical badge (dark red)

**Filter Types:**
- `enum`: Enumeration selection (dropdown/checkbox)
- `date_range`: Date/time range picker
- `numeric_range`: Numeric range (min/max)
- `text`: Text search/filter
- `boolean`: Boolean toggle

**Sort Configuration:**
Each sort defines a field with default direction (asc/desc). Users can override defaults in UI.

### 7.2 Materialization Declaration

Materializations derive views from state:

```yaml
materializations:
  - name: string              # Required: Name
    version: version          # Required: Version
    source: TypeName          # Required: Source data
    target: TypeName          # Required: Target view type
    projection: [Projection]  # Required: Field mappings
    triggers: [TypeName]      # Optional: Events that trigger refresh
    cache: CacheConfig        # Optional: v1.6 cache configuration
    retrieval: RetrievalContract  # Optional: Retrieval semantics
    temporal: TemporalContract    # Optional: Streaming semantics
```

**Retrieval Contract**:
```yaml
retrieval:
  mode: by_entity | by_query | singleton
  entity_key: field_name        # For by_entity mode
  query_fields: [fields]        # For by_query mode
  list:
    max_items: integer
    default_order: field_name
    order_direction: asc | desc
  freshness:
    max_staleness: duration
  on_unavailable:
    strategy: use_cached | fail | degrade | wait
    cache_ttl: duration
    degrade_to: view_ref
    wait_timeout: duration
```

**Retrieval Modes**:
- `by_entity`: One view instance per entity (keyed by entity_key)
- `by_query`: Views filtered by query fields (e.g., user_id, date_range)
- `singleton`: Single global view instance

**Unavailability Strategies**:
- `use_cached`: Serve stale data from cache (eventual consistency)
- `fail`: Return error immediately (strong consistency)
- `degrade`: Fall back to simpler/older view
- `wait`: Wait up to timeout for view to become available

**Temporal Semantics** (Streaming Views):
```yaml
temporal:
  window:
    type: tumbling | sliding | session | hopping
    size: duration
    slide: duration           # For sliding/hopping
    gap: duration             # For session windows
  aggregation:
    - field: field_name
      function: sum | avg | min | max | count | first | last | stddev
      as: result_field
  timestamp_field: field_name
  watermark: duration
  overflow:
    strategy: drop_oldest | pause_producer | sample | buffer
    buffer_size: integer
    sample_rate: decimal
  emit:
    trigger: on_window_close | on_each_event | periodic
    periodic_interval: duration
    include_partial: boolean
```

**Window Types**:
- `tumbling`: Non-overlapping fixed windows
- `sliding`: Overlapping windows that slide
- `session`: Dynamic windows based on activity gaps
- `hopping`: Fixed windows with fixed slide

**Example - Entity-Keyed View**:
```yaml
materializations:
  - name: CustomerOrderSummary
    source: OrderRecord
    target: OrderSummaryView
    projection:
      - source: customer_id
        target: customerId
      - source: order_count
        target: orderCount
    retrieval:
      mode: by_entity
      entity_key: customer_id
      on_unavailable:
        strategy: use_cached
        cache_ttl: 1h
```

**Example - Streaming Aggregation**:
```yaml
materializations:
  - name: RealtimeOrderMetrics
    source: OrderEvent
    target: OrderMetrics
    projection:
      - source: amount
        target: totalRevenue
    temporal:
      window:
        type: tumbling
        size: 1m
      aggregation:
        - field: amount
          function: sum
          as: total_revenue
        - field: order_id
          function: count
          as: order_count
    timestamp_field: created_at
    emit:
      trigger: on_window_close
```

### 7.3 Form-Intent Binding (v1.5)

Declarative form validation and submission patterns.

```yaml
views:
  - name: TransferFormView
    version: v1
    data_source: TransferFormData
    
    triggers:
      intent: InitiateTransfer
      binding: form | action | link
      field_mapping:
        - view_field: from_account
          supplies: transfer.from_account
          source: context          # From session/navigation context
          required: true
        - view_field: to_account
          supplies: transfer.to_account
          source: input            # User input
          required: true
        - view_field: amount
          supplies: transfer.amount
          source: input
          required: true
          validation:
            constraint: amount > 0
            message: "Amount must be positive"
        - view_field: total_fee
          supplies: transfer.fee
          source: computed         # Calculated from other fields
          default: calculate_fee(amount)
      confirmation:
        required: true
        message: "Transfer {{amount}} to {{to_account}}?"
      on_success:
        action: navigate_to | navigate_back | stay | refresh | close
        target: composition_ref
        params: { order_id: "{{result.id}}" }
        notification:
          type: toast
          message: "Transfer initiated successfully!"
      on_error:
        action: display_inline | display_modal | retry
        show_field_errors: true
        notification:
          type: inline
          message: "Please correct the errors below"
```

**Binding Types:**
- `form`: Multi-field input form
- `action`: Single-click action button
- `link`: Navigation that triggers intent

**Field Sources:**
- `context`: From composition/session context (navigation params, auth state)
- `input`: User-provided value
- `computed`: Derived from other fields

**Success Actions:**
- `navigate_back`: Return to previous screen
- `navigate_to`: Go to specific composition
- `stay`: Remain on current screen
- `refresh`: Reload current screen
- `close`: Close modal/drawer

### 7.4 Field Permissions (v1.4+)

Declarative field visibility and masking.

```yaml
views:
  - name: CustomerDetailView
    # ...
    fieldPermissions:
      - field: ssn
        visible: true
        mask:
          unless: subject.role == "admin"
          transform: reveal_last(4)
          placeholder: "***-**-####"
          preserve_format: true         # Preserve original format (e.g., dashes, spaces)
          context:
            display: reveal_last(4)
            print: redact
            export: hash
            log: redact
      - field: account_balance
        visible:
          policy: ViewBalancePolicy
      - field: internal_notes
        visible: false
      - field: employee_id
        visible: true
        mask:
          transform: reveal_last(4) | uppercase  # Transform chaining
```

**Mask Transforms:**
- Reveal: `reveal_last(n)`, `reveal_first(n)`, `reveal_middle(start, end)`
- Hide: `hide_last(n)`, `hide_first(n)`, `hide_middle(start, end)`
- Replace: `redact`, `hash`, `hash_partial(n)`, `truncate(n)`
- Format-aware: `mask_email`, `mask_phone`, `mask_credit_card`, `mask_ssn`

**Transform Chaining:**
Multiple transforms can be chained using the pipe operator (`|`):
- `reveal_last(4) | uppercase`: Show last 4 characters in uppercase
- `reveal_first(2) | lowercase`: Show first 2 characters in lowercase
- `mask_email | uppercase`: Mask email and convert to uppercase

**Format Preservation:**
- `preserve_format: true`: Maintains original formatting characters (dashes, spaces, parentheses) when masking. For example, `123-45-6789` becomes `***-**-6789` instead of `*****6789`.

**Context Types:**
- `display`: Screen/UI display
- `print`: Printed documents
- `export`: Data export (CSV, Excel)
- `log`: Audit/debug logs

### 7.5 Presentation Composition (v1.3+)

Semantic grouping of views into workspaces.

```yaml
presentationCompositions:
  - name: AccountWorkspace
    version: v1
    description: "Complete account view with transactions"
    primary: AccountSummaryView
    navigation:
      label: "Account {{account.name}}"
      purpose: accounts
      param: account_id
      policy_set: AccountAccessPolicies
      indicator:
        when: pending_alerts > 0
        severity: info | warning | error | critical
    related:
      - view: AccountBalanceView
        relationship: derived | detail | contextual | action | supplementary | historical
        cardinality: one | many
        required: boolean
        data_scope: { account_id: "{{account.id}}" }
        navigation:
          label: "Transactions"
          scope: within_composition | always_visible | on_action
    fetch:
      order: primary_first | all_parallel | sequential
      related:
        strategy: parallel | sequential | on_demand
        max_parallel: int
      on_failure:
        primary: fail | degrade | retry
        related: skip | degrade | retry
        degrade_to: view_ref
      cache:
        strategy: stale_while_revalidate | cache_first | network_first
        ttl: duration
      prefetch:
        enabled: boolean
        trigger: on_navigation | on_hover | manual
```

**Fetch Orders:**
- `primary_first`: Load primary view before related views
- `all_parallel`: Load all views in parallel
- `sequential`: Load views in declared order

**Failure Handling:**
- `on_failure.primary`:
  - `fail`: Return error if primary view fails
  - `degrade`: Fall back to `degrade_to` view reference
  - `retry`: Retry fetching primary view
- `on_failure.related`:
  - `skip`: Continue without related view
  - `degrade`: Fall back to simpler related view
  - `retry`: Retry fetching related view
- `on_failure.degrade_to`: Reference to fallback view when degrading

**Cache Strategies:**
- `stale_while_revalidate`: Serve stale data while updating in background
- `cache_first`: Serve from cache if available, fetch on miss
- `network_first`: Fetch fresh data, fall back to cache on failure

**Relationship Types:**
- `derived`: Computed from primary (e.g., Balance from Account)
- `detail`: Child items of primary (e.g., Transactions of Account)
- `contextual`: Related entity (e.g., Account owner Customer)
- `action`: Triggered by user action (e.g., Transfer form)
- `supplementary`: Additional information (e.g., Settings)
- `historical`: Audit/history data (e.g., Activity log)

---

## PART VIII: POLICIES AND AUTHORIZATION

### 8.1 Policy Declaration

```yaml
policies:
  - name: string              # Required: Policy name
    version: version          # Required: Version
    description: string       # Optional: Documentation
    targets: [TargetSpec]     # Required: What this applies to
    rules: [Rule]             # Required: Evaluation rules
    effect: enum              # Required: allow | deny
    lifecycle: enum           # Optional: shadow | enforce | deprecated
```

### 8.2 Rule Definition

```yaml
rule:
  name: string              # Required: Rule name
  condition: expr           # Required: Boolean expression
  priority: int             # Optional: Evaluation order
  
# Condition expressions can reference:
# - subject.* (actor attributes)
# - resource.* (target attributes)
# - action (operation name)
# - context.* (request context)
# - realm (current realm)
```

### 8.3 Policy Evaluation

```
Evaluation Order:
1. Collect applicable policies by target
2. Evaluate rules by priority (highest first)
3. Apply effect combination (deny overrides allow)
4. Return decision with reason
```

### 8.4 Delegation & Consent (v1.5)

Declarative delegation for temporary authority transfer with audit trails.

```yaml
delegations:
  - name: TemporaryAccountAccess
    version: v1
    delegate:
      from: subject_ref          # Original authority holder
      to: subject_ref            # Delegated authority recipient
      scope:
        actions: [view_balance, view_transactions, initiate_transfer]
        data: [AccountRecord, TransactionRecord]
        compositions: [AccountWorkspace]
    consent:
      required: boolean
      type: explicit | implicit
      expires_after: duration
      revocable: boolean
      audit:
        log_access: boolean
        log_revocation: boolean
```

**Purpose:** Supports temporary authority transfer with:
- Time-limited delegation
- Scope restrictions (specific actions, data, compositions)
- Audit trails for compliance

**Example - Account Representative Access:**
```yaml
delegations:
  - name: AccountRepresentativeAccess
    version: v1
    delegate:
      from: account_owner
      to: authorized_representative
      scope:
        actions: [view_balance, view_transactions]
        data: [AccountRecord, TransactionRecord]
        compositions: [AccountWorkspace]
    consent:
      required: true
      type: explicit
      expires_after: 30d
      revocable: true
      audit:
        log_access: true
        log_revocation: true
```

### 8.5 Consent Records (v1.5)

Granular consent tracking for compliance requirements.

```yaml
consentRecords:
  - name: MarketingConsent
    version: v1
    purpose: "Marketing communications"
    data_types: [CustomerProfile, EmailAddress, Preferences]
    legal_basis: consent | contract | legal_obligation | vital_interest | public_task | legitimate_interest
    retention: duration
    withdrawal:
      allowed: boolean
      method: self_service | contact_support | automated
      effect: delete | anonymize | restrict
```

---

## PART IX: SIGNALS AND OBSERVABILITY

### 9.1 Signal Declaration

```yaml
signals:
  - name: string              # Required: Signal name
    version: version          # Required: Version
    type: enum                # Required: metric | trace | log | alert | cache
    source: SignalSource      # Required: Emission point
    subject: SignalSubject    # Required: What is observed
    severity: enum            # Required: info | warn | error | critical
    schema: [Field]           # Required: Signal payload
```

### 9.2 Signal Types

| Type | Purpose | Example |
|------|---------|---------|
| `metric` | Quantitative measurement | Request latency, error count |
| `trace` | Distributed tracing | Request flow across boundaries |
| `log` | Event record | Work unit completion |
| `alert` | Actionable notification | Threshold breach |
| `cache` | Cache observability (v1.6) | Hit rate, pressure |

### 9.3 Cache Signal Type (v1.6)

```yaml
signals:
  - name: ViewCacheMetrics
    version: v1
    type: cache
    source:
      component: cache
      name: ViewHybridCache
      tier: hybrid
      region: "{{region}}"
    subject:
      cache: ViewHybridCache
    severity: info
    metrics:
      - name: hit_rate
        type: float
        description: "Cache hit ratio 0.0-1.0"
      - name: miss_rate
        type: float
      - name: latency_p50
        type: duration
      - name: latency_p99
        type: duration
      - name: size
        type: int
      - name: evictions
        type: int
      - name: pressure
        type: enum[low, medium, high, critical]
```

### 9.4 Notification Channels (v1.5)

Multi-channel notification delivery with user preferences and fallbacks.

```yaml
notificationChannels:
  - name: TransactionAlert
    version: v1
    channels:
      - type: email | sms | push | in_app | webhook
        enabled: boolean
        config:
          email:
            from: email_address
            template: template_ref
          sms:
            sender: sender_id
            template: template_ref
          push:
            platform: fcm | apns | web_push
            priority: high | normal | low
          webhook:
            url: url
            method: POST | PUT
            auth: auth_config
    preferences:
      user_controllable: boolean
      defaults:
        email: enabled | disabled
        sms: enabled | disabled
        push: enabled | disabled
      quiet_hours:
        enabled: boolean
        start: time
        end: time
        timezone: timezone
    delivery:
      retry:
        enabled: boolean
        max_attempts: int
        backoff: linear | exponential
      fallback:
        enabled: boolean
        order: [channel_list]
      batching:
        enabled: boolean
        window: duration
        max_batch_size: int
```

**Channel Types:**
- `email`: Email notifications
- `sms`: SMS text messages
- `push`: Mobile/web push notifications
- `in_app`: In-application notifications
- `webhook`: HTTP callback to external systems

**Example - Payment Confirmation:**
```yaml
notificationChannels:
  - name: PaymentConfirmation
    version: v1
    channels:
      - type: push
        enabled: true
        config:
          push:
            platform: fcm
            priority: high
      - type: email
        enabled: true
        config:
          email:
            from: "notifications@bank.com"
            template: payment_confirmation
    preferences:
      user_controllable: true
      defaults:
        push: enabled
        email: disabled
    delivery:
      retry:
        enabled: true
        max_attempts: 3
        backoff: exponential
      fallback:
        enabled: true
        order: [push, email]
```

---

## PART X: MULTI-TENANCY AND REALMS

### 10.1 Realm Declaration (v1.1+)

```yaml
realms:
  - name: string              # Required: Realm identifier
    version: version          # Required: Version
    display_name: string      # Optional: Human name
    parent: RealmRef          # Optional: Parent realm
    config: RealmConfig       # Optional: Realm settings
    policies: [PolicyRef]     # Optional: Realm-specific policies
    data_residency: string    # Optional: Geographic constraint
```

### 10.2 Realm Inheritance

- Child realms inherit parent policies unless overridden
- Data residency constraints propagate to children
- Cross-realm access requires explicit policy grant

---

## PART XI: ENTITY RELATIONSHIPS (v1.3+)

### 11.1 Relationship Declaration

```yaml
relationships:
  - name: string              # Required: Relationship name
    version: version          # Required: Version
    from: EntitySpec          # Required: Source entity
    to: EntitySpec            # Required: Target entity
    cardinality: enum         # Required: one_to_one | one_to_many | many_to_many
    ownership: enum           # Optional: owns | references
    cascade: CascadeConfig    # Optional: Delete behavior
```

### 11.2 Authority Declaration

Authority determines which region can accept writes for an entity at any given time. This is distinct from policy-based authorization and focuses on distributed write coordination.

**Grammar:**
```yaml
authority:
  name: string              # Required: Authority name
  version: version          # Required: Version
  entity_type: TypeName     # Required: Entity type
  scope: enum               # Required: entity | model
  resolution: expression    # Required: Expression determining authoritative region
  migration:
    allowed: boolean        # Optional: Allow authority migration
    strategy: enum          # Required if allowed: pause-and-cutover
    triggers:               # Optional: Migration triggers
      - type: enum          # load | latency | manual | time
        condition: expr     # Condition expression
    constraints:            # Optional: Migration constraints
      - type: enum          # region | governance
        allowed: [regions]  # Allowed regions
        rule: expression    # Constraint rule
    governed_by: PolicyRef  # Optional: Governance policy
```

**Authority Scope:**
- `entity`: Authority resolved per entity instance (RECOMMENDED)
- `model`: Authority resolved for entire model (legacy pattern)

**Resolution**: Expression determining authoritative region for an entity:
```yaml
# Example: US customers in us-east, EU customers in eu-west
resolution: |
  if customer.country in ["US", "CA"] then "us-east"
  else if customer.country in EU_COUNTRIES then "eu-west"
  else "default-region"
```

**Migration Triggers:**
- `load`: Migrate when load exceeds threshold
- `latency`: Migrate when latency exceeds threshold
- `manual`: Operator-initiated migration
- `time`: Scheduled migration (follow-the-sun)

**Migration Strategy: pause-and-cutover:**
1. Current authority region pauses writes for entity
2. CAS epoch incremented
3. New region becomes authority
4. Writes resume in new region

**Pause Duration**: Typically <100ms, bounded by network + coordination latency

**Authority State:**
```yaml
authorityState:
  entity_id: uuid
  authority_region: string
  authority_epoch: integer     # Monotonically increasing
  authority_lease: uuid         # Lease token
  authority_status: ACTIVE | TRANSITIONING
  entity_version: integer
```

**CAS Token Structure:**
Write requests include CAS token for linearizability:
```yaml
cas_token:
  entity_id: uuid
  entity_version: integer
  authority_epoch: integer
  authority_region: string
  lease_id: uuid
```

**CAS Validation:**
- Request MUST be sent to authority_region
- authority_epoch MUST match current epoch
- entity_version MUST match current version
- lease_id MUST be valid

**If CAS fails**: Client gets ErrEpochMismatch or ErrVersionMismatch, retries with fresh CAS token.

**Example - Entity-Scoped Authority:**
```yaml
authority:
  name: AccountAuthority
  version: v1
  entity_type: Account
  scope: entity
  resolution: |
    # Place in region closest to account owner
    nearest_region(account.owner.country)
  migration:
    allowed: true
    strategy: pause-and-cutover
    triggers:
      - type: latency
        condition: p99_latency > 200ms
      - type: manual
    constraints:
      - type: governance
        rule: account.kyc_status == "approved"
```

**Authority Invariants (Normative - MUST hold):**

**I1. Authority Singularity**: At any point in time, an entity SHALL have at most one write-authoritative region.

**I2. Epoch Monotonicity**: Authority epochs SHALL increase monotonically and SHALL NOT be reused.

**I3. No Overlapping Authority**: No two regions SHALL simultaneously accept writes for the same entity.

**I4. CAS Safety**: A write SHALL be accepted if and only if:
- The authority region is current
- The authority epoch matches
- The entity version matches

**I5. View Continuity**: Derived views SHALL remain readable across authority transitions.

**I6. Rebuildability**: All derived state SHALL be reconstructable from authoritative event history.

**I7. Governance Compliance**: Authority transitions SHALL NOT violate declared governance constraints.

**I8. Failure Determinism**: On failure during authority transition, the system SHALL resolve to a single authoritative region without ambiguity.

### 11.3 Storage Role Declaration

Storage roles distinguish control state (coordination) from data state (domain events).

**Grammar:**
```yaml
storage:
  role: enum                # Required: data | control
  rebuildable: boolean      # Required: Can be rebuilt from source
```

**Roles:**
- `data`: Append-only or rebuildable event/state data (domain layer)
- `control`: Coordination, authority, or governance state (system layer)

**Rules:**
- Control storage MUST be strongly consistent
- Control storage SHALL be modified only by designated control components
- Data storage MAY be partitioned, replayed, or materialized
- Implementations SHALL NOT treat control storage as domain data source

**Control Storage Examples:**
- Authority KV (entity authority state)
- Policy definitions
- Worker registrations
- Scheduler state

**Data Storage Examples:**
- Event streams (domain events)
- Materialized views
- Audit logs
- User data

**Example:**
```yaml
dataStates:
  # Authority state (control)
  - name: EntityAuthorityState
    type: AuthorityState
    lifecycle: persistent
    storage:
      role: control
      rebuildable: false
  # Domain events (data)
  - name: OrderEvents
    type: OrderEvent
    lifecycle: persistent
    storage:
      role: data
      rebuildable: true  # Can rebuild views from events
```

---

## PART XII: EXPERIENCES (v1.4+)

### 12.1 Experience Declaration

```yaml
experiences:
  - name: string              # Required: Experience name
    version: version          # Required: Version
    description: string       # Optional: Documentation
    entry_point: ViewRef      # Required: Initial view
    includes: [ViewRef]       # Required: Contained views
    policy_set: string        # Optional: Access policies
    unauthenticated:          # Optional: Unauthenticated user handling
      redirect_to: composition_ref   # Redirect target for unauthenticated users
    on_unauthorized: enum     # Optional: conceal | indicate | redirect | deny
    navigation: NavConfig     # Optional: Navigation structure
    session: SessionConfig    # Optional: Session management
    collaboration: CollaborationConfig  # Optional: Collaboration semantics
    clientCache: ClientCacheConfig  # Optional: v1.6 client cache
```

**Authorization Behavior:**
- `unauthenticated.redirect_to`: Composition to redirect unauthenticated users (e.g., login page)
- `on_unauthorized`: How to handle unauthorized access attempts:
  - `conceal`: Hide navigation item entirely
  - `indicate`: Show item but indicate it's inaccessible
  - `redirect`: Redirect to another composition
  - `deny`: Show access denied message

**Session Management**:
```yaml
session:
  required: boolean
  subject_binding:
    mode: authenticated | anonymous | either
  contains:
    required: [subject_id, roles]
    optional: [realm, attributes, preferences]
  lifetime:
    mode: bounded | sliding | permanent
    idle_timeout: duration
    absolute_timeout: duration
    refresh: auto | manual
  on_expired:
    action: redirect_to_auth | degrade | notify
    redirect_to: composition_ref
  multi_device:
    enabled: boolean
    sync_strategy: real_time | periodic | manual
    sync_interval: duration
    conflict_resolution: last_write_wins | device_priority | manual
  offline:
    enabled: boolean
    cache_duration: duration
    sync_on_reconnect: auto | prompt
```

**Session Lifetime Modes**:
- `bounded`: Fixed duration from creation
- `sliding`: Extends on activity
- `permanent`: No expiration

**Multi-Device Sync**: Consistent state across devices with conflict resolution strategies.

**Offline Support**: Declarative offline operation with automatic or prompted sync on reconnect.

**Example - Experience with Session**:
```yaml
experiences:
  - name: CustomerPortal
    version: v1
    description: "Customer-facing banking experience"
    entry_point: CustomerDashboard
    includes:
      - CustomerDashboard
      - AccountWorkspace
      - TransactionHistory
    policy_set: CustomerPolicies
    unauthenticated:
      redirect_to: LoginComposition
    on_unauthorized: conceal
    session:
      required: true
      subject_binding:
        mode: authenticated
      contains:
        required: [subject_id, customer_id, roles]
      lifetime:
        mode: sliding
        idle_timeout: 30m
        absolute_timeout: 8h
        refresh: auto
      multi_device:
        enabled: true
        sync_strategy: real_time
        conflict_resolution: last_write_wins
      offline:
        enabled: true
        cache_duration: 24h
        sync_on_reconnect: auto
```

**Collaboration in Experiences**:

Experiences can include collaborative features for real-time co-editing and presence. See [Section 12.3](#123-collaboration-semantics-v15) for full collaboration configuration.

```yaml
experiences:
  - name: CollaborativeWorkspace
    version: v1
    # ... other fields
    collaboration:
      enabled: true
      session:
        type: workspace
        scope: workspace.id == "{{workspace_id}}"
        max_participants: 50
      awareness:
        show_active_users: true
        show_cursors: false
        timeout: 1m
      permissions:
        mode: role_based
      conflict_resolution:
        strategy: crdt
```

### 12.2 Search Intent (v1.4+)

```yaml
searchIntents:
  - name: string
    version: version
    index: string
    searchable_fields: [Field]
    facets: [FacetConfig]
    filters: [FilterConfig]
    sort_options: [SortConfig]
```

### 12.3 Collaboration Semantics (v1.5)

Collaborative sessions enable real-time co-editing and presence within experiences.

**Grammar:**
```yaml
collaboration:
  name: string              # Required: Collaboration name
  version: version          # Required: Version
  enabled: boolean          # Required: Enable collaboration
  session:
    type: enum              # Required: document | workspace | entity
    scope: expression       # Required: Collaboration scope
    max_participants: int   # Optional: Max concurrent participants
  awareness:
    show_active_users: boolean    # Optional: Show active users
    show_cursors: boolean         # Optional: Show cursor positions
    show_selections: boolean      # Optional: Show selections
    timeout: duration             # Optional: Presence timeout
  permissions:
    mode: enum              # Required: owner_controlled | role_based | policy_based
    policy_set: PolicySet   # Optional: Policy set (if policy_based)
  conflict_resolution:
    strategy: enum          # Required: operational_transform | crdt | last_write_wins
    merge_policy: PolicyRef # Optional: Merge policy
```

**Collaboration Types:**
- `document`: Single document editing (e.g., collaborative text editor)
- `workspace`: Entire composition collaboration (e.g., shared dashboard)
- `entity`: Entity-scoped collaboration (e.g., shared order form)

**Permission Modes:**
- `owner_controlled`: Owner grants/revokes access
- `role_based`: Based on user roles
- `policy_based`: Evaluated by policy engine

**Conflict Resolution Strategies:**
- `operational_transform`: Transform operations for consistency
- `crdt`: Conflict-free replicated data types
- `last_write_wins`: Simple timestamp-based resolution

**Example - Collaborative Document Editing:**
```yaml
collaboration:
  name: DocumentCollaboration
  version: v1
  enabled: true
  session:
    type: document
    scope: document.id == "{{document_id}}"
    max_participants: 10
  awareness:
    show_active_users: true
    show_cursors: true
    show_selections: true
    timeout: 30s
  permissions:
    mode: owner_controlled
  conflict_resolution:
    strategy: operational_transform
```

**Example - Workspace Collaboration:**
```yaml
collaboration:
  name: WorkspaceCollaboration
  version: v1
  enabled: true
  session:
    type: workspace
    scope: workspace.id == "{{workspace_id}}"
    max_participants: 50
  awareness:
    show_active_users: true
    show_cursors: false
    show_selections: false
    timeout: 1m
  permissions:
    mode: role_based
  conflict_resolution:
    strategy: crdt
```

**Integration with Experience:**
Collaboration can be configured within an Experience declaration:
```yaml
experiences:
  - name: CollaborativeExperience
    version: v1
    collaboration:
      enabled: true
      session:
        type: workspace
        scope: workspace.id == "{{workspace_id}}"
      # ... rest of collaboration config
```

---

## PART XIII: SESSION AND ASSET MANAGEMENT (v1.5+)

### 13.1 Session Extension

Sessions can be configured on experiences:

```yaml
experiences:
  - name: CustomerPortal
    session:
      timeout: 30m
      idle_timeout: 5m
      refresh_strategy: sliding
      bindings:
        - key: currentAccount
          type: Reference<Account>
```

### 13.2 Asset Declaration

```yaml
assets:
  - name: string
    version: version
    category: enum            # document | image | media | archive | generic
    max_size: string          # e.g., "10MB"
    allowed_types: [string]   # MIME types
    lifecycle: enum           # permanent | temporary | expiring
    variants: [Variant]       # Size/format variants
    scan: ScanConfig          # Security scanning
```

---

## PART XIV: IoT AND DEVICE TELEMETRY (v1.5+)

### 14.1 Device Capability

```yaml
deviceCapabilities:
  - name: string
    version: version
    device_class: string
    capabilities:
      sensors: [SensorSpec]
      actuators: [ActuatorSpec]
      connectivity: ConnectivitySpec
    telemetry:
      frequency: duration
      batch_size: int
      priority: enum
```

### 14.1a Edge Device Declaration

Edge devices define IoT and edge deployment capabilities with offline operation and sync strategies.

**Grammar:**
```yaml
edgeDevices:
  - name: string              # Required: Device name
    version: version          # Required: Version
    capabilities:
      compute: ComputeSpec    # Required: Compute capabilities
      storage: StorageSpec    # Required: Storage capabilities
      connectivity: enum      # Required: online | offline | intermittent
    sync:
      strategy: enum          # Required: push | pull | bidirectional
      frequency: enum         # Required: real_time | periodic | on_demand
      interval: duration      # Optional: For periodic frequency
      conflict_resolution: enum # Required: device_wins | server_wins | manual
    local_processing:
      enabled: boolean        # Required: Enable local processing
      fallback_to_cloud: boolean # Required: Fallback on failure
      models: [ModelRef]      # Optional: Local ML models
```

**Connectivity Types:**
- `online`: Continuously connected
- `offline`: Operates independently
- `intermittent`: Periodic connectivity

**Sync Strategies:**
- `push`: Device pushes to cloud
- `pull`: Device pulls from cloud
- `bidirectional`: Two-way sync

**Sync Frequency:**
- `real_time`: Immediate sync when connected
- `periodic`: Fixed interval sync
- `on_demand`: Manual trigger

**Conflict Resolution:**
- `device_wins`: Device changes take precedence
- `server_wins`: Server changes take precedence
- `manual`: Require manual conflict resolution

**Example - ATM (Edge Device):**
```yaml
edgeDevices:
  - name: ATMDevice
    version: v1
    capabilities:
      compute: arm64_2ghz
      storage: 32GB
      connectivity: intermittent
    sync:
      strategy: bidirectional
      frequency: real_time
      conflict_resolution: server_wins
    local_processing:
      enabled: true
      fallback_to_cloud: false
      models: [card_validation, transaction_authorization]
```

**Example - Smart Sensor:**
```yaml
edgeDevices:
  - name: TemperatureSensor
    version: v1
    capabilities:
      compute: arm_cortex_m4
      storage: 4MB
      connectivity: intermittent
    sync:
      strategy: push
      frequency: periodic
      interval: 5m
      conflict_resolution: server_wins
    local_processing:
      enabled: true
      fallback_to_cloud: true
      models: [anomaly_detection]
```

### 14.2 Actuator Command

```yaml
actuatorCommands:
  - name: string
    version: version
    device_class: string
    target_actuator: string
    parameters: [Field]
    validation: [Rule]
    acknowledgment: enum      # none | sent | delivered | executed
```

---

## PART XV: PRIVACY AND COMPLIANCE (v1.5+)

### 15.1 Consent Record

```yaml
consentRecords:
  - name: string
    version: version
    purpose: string
    data_types: [TypeName]
    retention: duration
    withdrawal: WithdrawalConfig
```

### 15.2 Erasure Request

```yaml
erasureRequests:
  - name: string
    version: version
    scope: enum               # full | partial | anonymize
    target_types: [TypeName]
    cascade: boolean
    verification: VerificationConfig
```

### 15.3 Legal Hold

```yaml
legalHolds:
  - name: string
    version: version
    scope: HoldScope
    entities: [EntitySpec]
    preservation: PreservationConfig
```

---

## PART XVI: EXTERNAL INTEGRATION (v1.5)

### 16.1 External Integration Declaration

Declarative third-party API integration with resilience patterns.

```yaml
externalDependencies:
  - name: StripePaymentGateway
    version: v1
    endpoint:
      base_url: url
      auth:
        type: bearer | oauth2 | api_key | mtls
        credentials: credentials_ref
    operations:
      - name: charge
        method: GET | POST | PUT | DELETE
        path: path_template
        request:
          schema: schema_ref
          mapping: field_mapping
        response:
          schema: schema_ref
          mapping: field_mapping
    reliability:
      timeout: duration
      retry:
        enabled: boolean
        max_attempts: int
        backoff: exponential | linear
      circuit_breaker:
        enabled: boolean
        failure_threshold: int
        reset_timeout: duration
```

**Example - Payment Gateway Integration:**
```yaml
externalDependencies:
  - name: StripePaymentGateway
    version: v1
    endpoint:
      base_url: "https://api.stripe.com/v1"
      auth:
        type: api_key
        credentials: stripe_secret_key
    operations:
      - name: charge
        method: POST
        path: "/charges"
        request:
          mapping:
            amount: "{{order.total * 100}}"
            currency: "usd"
            source: "{{payment.token}}"
        response:
          mapping:
            charge_id: "{{response.id}}"
            status: "{{response.status}}"
    reliability:
      timeout: 10s
      retry:
        enabled: true
        max_attempts: 3
        backoff: exponential
      circuit_breaker:
        enabled: true
        failure_threshold: 5
        reset_timeout: 60s
```

### 16.2 Webhook Receiver

Inbound webhook handling for external events.

```yaml
webhookReceivers:
  - name: StripeWebhook
    version: v1
    endpoint:
      path: /webhooks/stripe
      method: POST
    security:
      signature:
        header: Stripe-Signature
        algorithm: hmac_sha256
        secret: webhook_secret
    events:
      - external_type: "payment_intent.succeeded"
        maps_to: PaymentCompleted
        mapping:
          payment_id: "{{data.object.id}}"
          amount: "{{data.object.amount / 100}}"
      - external_type: "payment_intent.failed"
        maps_to: PaymentFailed
```

---

## PART XVII: GRAPH QUERIES (v1.5)

### 17.1 Graph Query Declaration

Relationship traversal for graph patterns.

```yaml
graphQueries:
  - name: FindAllSubordinates
    version: v1
    start_from: DataState
    traversal:
      relationships: [relationship_refs]
      direction: outgoing | incoming | both
      min_depth: int
      max_depth: int
    filter:
      node: expression
      edge: expression
    return:
      nodes: [fields]
      edges: [fields]
      aggregations:
        - function: count | sum | avg
          field: field_name
          as: result_field
```

**Example - Organization Hierarchy:**
```yaml
graphQueries:
  - name: FindAllSubordinates
    version: v1
    start_from: EmployeeRecord
    traversal:
      relationships: [ReportsTo]
      direction: incoming
      min_depth: 1
      max_depth: 5
    filter:
      node: employee.status == "active"
    return:
      nodes: [id, name, title, level]
      aggregations:
        - function: count
          field: id
          as: total_subordinates
```

---

## PART XVIII: CONVERSATIONAL UI (v1.5)

### 18.1 Conversational Experience Declaration

Voice assistants and chatbots as first-class experiences.

```yaml
conversationalExperiences:
  - name: BankingVoiceAssistant
    version: v1
    interaction:
      mode: voice | text | multimodal
    dialogue:
      intents: [intent_refs]
      context:
        max_turns: int
        timeout: duration
    understanding:
      nlu_model: model_ref
      confidence_threshold: decimal
      fallback_intent: intent_ref
    generation:
      template_set: template_ref
      personalization: enabled | disabled
```

**Example - Banking Voice Assistant:**
```yaml
conversationalExperiences:
  - name: BankingVoiceAssistant
    version: v1
    interaction:
      mode: voice
    dialogue:
      intents: [CheckBalance, TransferFunds, PayBill, ListTransactions]
      context:
        max_turns: 10
        timeout: 5m
    understanding:
      nlu_model: banking_nlu_model
      confidence_threshold: 0.7
      fallback_intent: AskForClarification
    generation:
      template_set: banking_response_templates
      personalization: enabled
```

---

## PART XIX: ML/INFERENCE (v1.5)

### 19.1 Inference-Derived Fields

ML model outputs as declarative fields in data types.

```yaml
dataTypes:
  - name: Product
    version: v1
    fields:
      name: string
      description: string
      category: string
      recommended_price:
        type: decimal
        inference:
          enabled: boolean
          source:
            fields: [category, historical_sales, competitor_prices]
          model:
            name: price_recommendation_model
            version: v2
          output:
            type: decimal
            range: { min: 0, max: 10000 }
          freshness:
            strategy: on_write | on_read | scheduled | manual
          fallback:
            on_unavailable: use_default | use_cached | fail
            default_value: 0.0
```

**Inference Strategies:**
- `on_write`: Compute when entity is written
- `on_read`: Compute when entity is read (lazy)
- `scheduled`: Periodic recomputation
- `manual`: Explicit trigger

**Example - Fraud Score:**
```yaml
dataTypes:
  - name: Transaction
    version: v1
    fields:
      amount: decimal
      merchant: string
      fraud_score:
        type: decimal
        inference:
          enabled: true
          source:
            fields: [amount, merchant, customer.history, device.fingerprint]
          model:
            name: fraud_detection_model
            version: v3
          output:
            type: decimal
            range: { min: 0.0, max: 1.0 }
          freshness:
            strategy: on_write
          fallback:
            on_unavailable: use_default
            default_value: 0.5
```

---

## PART XX: STORAGE BINDING (v1.6)

### 20.1 Overview

Storage binding provides declarative configuration for data persistence, abstracting the underlying storage technology while providing precise control over data placement and access patterns.

### 20.2 Storage Binding Declaration

```yaml
storageBindings:
  - name: string              # Required: Binding name
    version: version          # Required: Version
    description: string       # Optional: Documentation
    type: enum                # Required: kv | stream | object_store
    
    # Type-specific configuration
    kv:                       # For type: kv
      bucket: string          # Bucket name
      key_pattern: string     # Optional: Key template with {{vars}}
      serialization: enum     # Optional: json | protobuf | msgpack
      history: int            # History depth (0 = none)
      ttl: duration           # Entry TTL
      replicas: int           # Replication factor
      max_value_size: string  # Max value size
      
    stream:                   # For type: stream
      name: string            # Stream name
      subjects: [string]      # Subject patterns
      retention: enum         # limits | interest | work_queue
      max_age: duration       # Message retention
      max_bytes: string       # Max stream size
      max_msgs: int           # Max message count
      
    object_store:             # For type: object_store
      bucket: string          # Bucket name
      max_size: string        # Max object size
      chunk_size: string      # Chunk size for large objects
      
    access:                   # Access configuration
      read: enum              # strong | eventual
      write:
        consistency: enum     # strong | eventual
        ack: enum             # none | one | quorum | all
        
    replication:              # Replication settings
      type: enum              # none | sync | async
      targets: [string]       # Target regions/clusters
```

### 20.3 Key Pattern Syntax

Key patterns support variable interpolation:

```
key_pattern = segment { "." segment } ;
segment     = literal | variable ;
variable    = "{{" field_path "}}" ;
field_path  = identifier { "." identifier } ;
```

**Examples:**
```yaml
# Simple key
key_pattern: "account.{{id}}"

# Hierarchical key
key_pattern: "{{realm}}.customer.{{customer_id}}.order.{{order_id}}"

# Nested field access
key_pattern: "sensor.{{device_id}}.{{reading.type}}"
```

### 20.4 Storage Binding Examples

**KV Store for Entity State:**
```yaml
storageBindings:
  - name: AccountStore
    version: v1
    type: kv
    key_pattern: "account.{{account_id}}"
    kv:
      bucket: account-state
      history: 10
      ttl: 0                  # No expiry
      replicas: 3
    access:
      read: strong
      write:
        consistency: strong
        ack: quorum
```

**Stream for Event Log:**
```yaml
storageBindings:
  - name: OrderEvents
    version: v1
    type: stream
    key_pattern: "order.{{order_id}}"
    stream:
      name: ORDER-EVENTS
      subjects:
        - "order.>"
      retention: limits
      max_age: 30d
      max_bytes: 10GB
```

**Object Store for Documents:**
```yaml
storageBindings:
  - name: DocumentStore
    version: v1
    type: object_store
    key_pattern: "doc.{{category}}.{{doc_id}}"
    object_store:
      bucket: documents
      max_size: 100MB
      chunk_size: 256KB
```

---

## PART XXI: UNIFIED CACHING (v1.6)

### 21.1 Overview

Unified caching provides multi-tier caching for materialized views with configurable invalidation strategies and observability integration.

### 21.2 Cache Layer Declaration

```yaml
cacheLayers:
  - name: string              # Required: Cache name
    version: version          # Required: Version
    description: string       # Optional: Documentation
    
    topology:
      tier: enum              # Required: local | distributed | hybrid
      
      local:                  # For local or hybrid tiers
        max_entries: int      # Max cached items
        max_size: string      # Max memory (e.g., "256MB")
        ttl: duration         # Entry TTL
        eviction: enum        # lru | lfu | fifo
        
      distributed:            # For distributed or hybrid tiers
        resource: string      # Resource reference
        ttl: duration         # Entry TTL
        serialization: enum   # json | protobuf | msgpack
        
      hybrid:                 # For hybrid tier
        promotion: enum       # on_miss | on_access | never
        write_policy: enum    # write_through | write_back
        
    invalidation:
      strategy: enum          # explicit | ttl | watch | hybrid
      ttl: duration           # For ttl strategy
      watch_keys: [string]    # Key patterns to watch
      propagate: boolean      # Cross-instance propagation
      
    observability:
      emit_signals: boolean   # Enable signal emission
      signal_name: string     # Custom signal name
      metrics: [string]       # Metrics to track
      interval: duration      # Emission interval
```

### 21.3 Cache Tier Types

| Tier | L1 | L2 | Use Case |
|------|----|----|----------|
| `local` | Memory | - | Single instance, sub-ms access |
| `distributed` | - | Remote | Shared across instances |
| `hybrid` | Memory | Remote | Best of both, most common |

### 21.4 Eviction Policies

| Policy | Description | Best For |
|--------|-------------|----------|
| `lru` | Least Recently Used | General purpose |
| `lfu` | Least Frequently Used | Skewed access patterns |
| `fifo` | First In First Out | Time-based access |

### 21.5 Invalidation Strategies

| Strategy | Description |
|----------|-------------|
| `explicit` | Manual invalidation only |
| `ttl` | Time-based expiry |
| `watch` | Invalidate on key changes |
| `hybrid` | Combine TTL and watch |

### 21.6 Cache Layer Examples

**Local-Only Cache:**
```yaml
cacheLayers:
  - name: SessionCache
    version: v1
    topology:
      tier: local
      local:
        max_entries: 10000
        ttl: 15m
        eviction: lru
    invalidation:
      strategy: ttl
      ttl: 15m
```

**Hybrid Cache with Watch Invalidation:**
```yaml
cacheLayers:
  - name: ViewHybridCache
    version: v1
    topology:
      tier: hybrid
      local:
        max_entries: 1000
        ttl: 5m
        eviction: lru
      distributed:
        resource: ViewCacheResource
        ttl: 30m
        serialization: protobuf
      hybrid:
        promotion: on_miss
        write_policy: write_through
    invalidation:
      strategy: hybrid
      ttl: 30m
      watch_keys:
        - "account.{{account_id}}"
        - "customer.{{customer_id}}"
      propagate: true
    observability:
      emit_signals: true
      metrics: [hit_rate, miss_rate, latency, size, evictions]
      interval: 10s
```

### 21.7 Materialization with Cache

```yaml
materializations:
  - name: AccountSummaryMaterialization
    version: v1
    source: Account
    target: AccountSummaryView
    projection:
      - source: id
        target: accountId
      - source: balance
        target: currentBalance
      - source: status
        target: accountStatus
    triggers:
      - AccountUpdated
      - TransactionCompleted
    cache:
      layer: AccountSummaryCache
      key: "view.account.{{account_id}}"
      warm_on_trigger: true
```

---

## PART XXII: INTENT ROUTING (v1.6)

### 22.1 Overview

Intent routing provides declarative transport configuration for InputIntents, enabling protocol abstraction while maintaining the Transport Scope Boundary invariant (I9).

### 22.2 InputIntent Routing Extension

```yaml
inputIntents:
  - name: string              # Required: Intent name
    version: version          # Required: Version
    realm: RealmRef           # Required: Target realm
    work_unit: WorkUnitRef    # Required: Target work unit
    input_type: TypeName      # Required: Input data type
    policy_set: string        # Optional: Access policies
    response: ResponseSchema  # Optional: Response contract
    
    routing:                  # v1.6: Routing configuration
      transport: enum         # Required: nats | http | grpc
      
      nats:                   # For transport: nats
        subject_pattern: string  # Subject pattern (supports {{vars}})
        delivery: enum        # Optional: jetstream | core
        stream: string        # Optional: JetStream stream name
        queue: string         # Optional: Queue group for load balancing
        dead_letter: string   # Optional: Dead letter queue subject
        ack_wait: duration    # Acknowledgment timeout
        max_retries: int      # Retry count
        
      http:                   # For transport: http
        path: string          # URL path (supports {{vars}})
        method: enum          # POST | PUT | PATCH
        timeout: duration     # Request timeout
        retry:
          attempts: int
          backoff: enum       # constant | linear | exponential
          
      grpc:                   # For transport: grpc
        service: string       # Service name
        method: string        # Method name
        timeout: duration     # Request timeout
        
      response:
        mode: enum            # request_reply | async | fire_and_forget
        timeout: duration     # For request_reply
        callback:             # For async
          subject: string     # Callback subject/path
          
      dead_letter:
        enabled: boolean
        subject: string       # DLQ subject
        max_age: duration     # Retention
```

### 22.2a Intent Delivery Contract (v1.5)

The Intent Delivery Contract defines explicit delivery guarantees, acknowledgment semantics, retry behavior, and response contracts for InputIntents. This ensures predictable behavior across different transport implementations.

**Delivery Contract Grammar:**
```yaml
inputIntents:
  - name: string
    version: version
    # ... standard fields
    
    delivery:
      guarantee: at_least_once | at_most_once | exactly_once
      acknowledgment: required | optional | fire_and_forget
      timeout: duration
      retry:
        allowed: boolean
        max_attempts: integer
        backoff: linear | exponential | fixed
        initial_delay: duration
        max_delay: duration
    
    response:
      on_success:
        includes: [field_names]
        guarantees: [guarantee_names]
      on_error:
        includes: [field_names]
        field_errors:
          enabled: boolean
          format: map | list
        categories: [error_categories]
```

**Delivery Guarantees:**

- **`at_least_once`**: Intent MAY be delivered multiple times
  - Runtime MUST ensure delivery even if failures occur
  - Handler MUST be idempotent
  - Suitable for: Most business operations with idempotency keys
  
- **`at_most_once`**: Intent delivered zero or one time
  - Runtime makes best-effort delivery attempt
  - No retries on failure
  - Suitable for: Analytics, logging, non-critical operations
  
- **`exactly_once`**: Intent delivered exactly once
  - Runtime ensures single execution with deduplication
  - Highest reliability guarantee
  - Suitable for: Financial transactions, critical state changes

**Acknowledgment Modes:**

- **`required`**: Handler MUST acknowledge receipt
  - Runtime waits for acknowledgment before considering delivery complete
  - Unacknowledged intents are retried according to retry policy
  
- **`optional`**: Handler MAY acknowledge receipt
  - Runtime proceeds without waiting for acknowledgment
  - Used for best-effort delivery patterns
  
- **`fire_and_forget`**: No acknowledgment expected
  - Runtime publishes intent and immediately returns
  - Suitable for: Event logging, analytics

**Retry Configuration:**

```yaml
retry:
  allowed: true              # Enable retry logic
  max_attempts: 3            # Maximum retry attempts
  backoff: exponential       # Backoff strategy
  initial_delay: 1s          # Initial retry delay
  max_delay: 60s             # Maximum delay between retries
```

**Backoff Strategies:**
- `linear`: Delay increases linearly (initial_delay * attempt)
- `exponential`: Delay doubles each attempt (initial_delay * 2^attempt)
- `fixed`: Constant delay between retries

**Response Schema:**

Response schemas define the contract for success and error responses:

```yaml
response:
  on_success:
    includes: [id, created_at, status]     # Required response fields
    guarantees: [persisted, indexed]       # System guarantees
  on_error:
    includes: [error_code, message]        # Required error fields
    field_errors:
      enabled: true
      format: map                          # field_name → error_message
    categories: [validation, authorization, conflict, unavailable, internal]
```

**Runtime Responsibilities:**

The runtime MUST:
1. Honor delivery guarantees across transport failures
2. Implement retry logic with configured backoff
3. Track idempotency keys to prevent duplicate execution
4. Enforce timeout limits
5. Route failed intents to dead letter queues (if configured)
6. Provide observability for delivery attempts and failures
7. Generate responses conforming to declared schema

**Handler Responsibilities:**

Intent handlers MUST:
1. Be idempotent for `at_least_once` guarantees
2. Return responses matching declared schema
3. Acknowledge receipt within timeout window
4. Handle duplicate delivery gracefully
5. Classify errors using declared categories

**Example - Complete Delivery Contract:**
```yaml
inputIntents:
  - name: CreateOrder
    version: v1
    proposes:
      dataState: OrderRecord
      version: v1
    supplies:
      required: [customer_id, items]
      optional: [notes, promo_code]
    
    delivery:
      guarantee: at_least_once
      acknowledgment: required
      timeout: 30s
      retry:
        allowed: true
        max_attempts: 3
        backoff: exponential
        initial_delay: 1s
        max_delay: 30s
    
    response:
      on_success:
        includes: [order_id, total, status, created_at]
        guarantees: [persisted, indexed, replicated]
      on_error:
        includes: [error_code, message, field_errors]
        field_errors:
          enabled: true
          format: map
        categories: [validation, authorization, conflict, unavailable, internal]
```

**Example - Fire and Forget Analytics:**
```yaml
inputIntents:
  - name: TrackPageView
    version: v1
    proposes:
      dataState: PageViewEvent
      version: v1
    
    delivery:
      guarantee: at_most_once
      acknowledgment: fire_and_forget
      timeout: 5s
      retry:
        allowed: false
    
    response:
      on_success:
        includes: [recorded_at]
        guarantees: []
      on_error:
        includes: [error_code]
        categories: [unavailable, internal]
```

### 22.3 Response Schema

Response schema defines the contract for what is returned on success vs. error:

```yaml
response:
  on_success:
    includes: [id, created_at, status]
    guarantees: [persisted, indexed]
  on_error:
    includes: [error_code, message, field_errors]
    field_errors:
      enabled: boolean
      format: map | list
    categories: [validation, authorization, conflict, unavailable, internal]
```

**Success Response**:
- `includes`: Fields guaranteed to be present in successful response
- `guarantees`: System guarantees about the operation (persisted, indexed, replicated, etc.)

**Error Response**:
- `includes`: Fields guaranteed to be present in error response
- `field_errors`: Optional field-level error details
  - `format: map`: Field name → error message mapping
  - `format: list`: Array of field error objects
- `categories`: Error categories for client handling

**Error Categories**:
- `validation`: Input validation failure
- `authorization`: Permission denied
- `conflict`: CAS conflict or constraint violation
- `unavailable`: Service unavailable or timeout
- `internal`: Internal server error

**Example - Transfer Intent with Response Schema**:
```yaml
inputIntents:
  - name: InitiateTransfer
    version: v1
    realm: BankingService
    work_unit: ProcessTransfer
    input_type: TransferRequest
    response:
      on_success:
        includes: [transfer_id, from_account, to_account, amount, status, created_at]
        guarantees: [persisted, indexed, audited]
      on_error:
        includes: [error_code, message, field_errors]
        field_errors:
          enabled: true
          format: map
        categories: [validation, authorization, conflict, unavailable, internal]
    routing:
      transport: nats
      nats:
        subject_pattern: "SMS.INTENT.banking.InitiateTransfer.{{realm}}"
        delivery: jetstream
        stream: SMS_BANKING_INTENTS
      response:
        mode: request_reply
        timeout: 10s
```

### 22.4 Subject Pattern Variables

| Variable | Description |
|----------|-------------|
| `{{domain}}` | Intent domain |
| `{{intent_name}}` | Intent name |
| `{{realm}}` | Current tenant realm |
| `{{version}}` | Intent version |
| `{{entity_id}}` | Entity ID from payload |
| `{{correlation_id}}` | Request correlation ID |

### 22.5 Response Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| `request_reply` | Synchronous request/response | Interactive operations |
| `async` | Publish with callback | Long-running operations |
| `fire_and_forget` | Publish without response | Analytics, logging |

### 22.6 Intent Routing Examples

**NATS with JetStream:**
```yaml
inputIntents:
  - name: InitiateTransfer
    version: v1
    realm: BankingService
    work_unit: ProcessTransfer
    input_type: TransferRequest
    
    routing:
      transport: nats
      nats:
        subject_pattern: "SMS.INTENT.banking.InitiateTransfer.{{realm}}"
        delivery: jetstream
        stream: BANKING-INTENTS
        queue: transfer-processors
        ack_wait: 30s
        max_retries: 3
      response:
        mode: request_reply
        timeout: 10s
      dead_letter:
        enabled: true
        subject: "SMS.DLQ.banking.InitiateTransfer"
        max_age: 7d
```

**HTTP REST:**
```yaml
inputIntents:
  - name: GetAccountBalance
    version: v1
    realm: AccountService
    work_unit: RetrieveBalance
    input_type: BalanceRequest
    
    routing:
      transport: http
      http:
        path: "/api/v1/accounts/{{account_id}}/balance"
        method: GET
        timeout: 5s
        retry:
          attempts: 3
          backoff: exponential
      response:
        mode: request_reply
        timeout: 5s
```

### 22.7 I9 Invariant

**Transport Scope Boundary (I9)**: Transport bindings SHALL be specified only at system ingress points (InputIntent routing). Transport SHALL NOT be specified within SMS flow definitions.

**Valid:**
```yaml
inputIntents:
  - name: ProcessOrder
    routing:
      transport: nats
      # ...
```

**Invalid:**
```yaml
smsFlows:
  - name: OrderFlow
    states:
      - name: Processing
        transport: nats    # VIOLATION: Transport in flow
```

---

## PART XXIII: RESOURCE REQUIREMENTS (v1.6)

### 23.1 Overview

Resource requirements provide abstract specification of infrastructure capabilities, allowing runtime provisioning to select appropriate backends based on system capabilities.

### 23.2 Resource Requirements Declaration

```yaml
resourceRequirements:
  name: string              # Required: Resource set name
  version: version          # Required: Version
  description: string       # Optional: Documentation
  
  eventLogs:                # Event log requirements
    - name: string
      purpose: enum         # intent_ingestion | event_sourcing | audit | telemetry
      ordering: enum        # strict | causal | none
      delivery: enum        # at_most_once | at_least_once | exactly_once
      durability: enum      # persistent | ephemeral
      retention: duration
      partitions: int
      
  entityStores:             # Entity store requirements
    - name: string
      consistency: enum     # strong | eventual | causal
      history: boolean
      history_depth: int
      indexes: [IndexSpec]
      
  objectStores:             # Object store requirements
    - name: string
      max_size: string
      versioning: boolean
      encryption: boolean
      
  provisioning:
    mode: enum              # auto | explicit
    on_missing: enum        # fail | create | warn
```

### 23.3 Resource Types

| Type | Purpose | Backends |
|------|---------|----------|
| `eventLog` | Append-only event streams | JetStream, Kafka, Kinesis |
| `entityStore` | Entity state storage | NATS KV, Redis, DynamoDB |
| `objectStore` | Large object storage | NATS Object Store, S3, GCS |

### 23.4 Provisioning Modes

| Mode | Behavior |
|------|----------|
| `auto` | Runtime provisions resources automatically |
| `explicit` | Resources must exist; fail if missing |

### 23.5 Resource Requirements Example

```yaml
resourceRequirements:
  name: BankingResources
  version: v1
  description: "Infrastructure requirements for banking system"
  
  eventLogs:
    - name: TransactionEvents
      purpose: event_sourcing
      ordering: causal
      delivery: exactly_once
      durability: persistent
      retention: 365d
      partitions: 12
      
    - name: AuditLog
      purpose: audit
      ordering: strict
      delivery: at_least_once
      durability: persistent
      retention: 2555d  # 7 years
      
  entityStores:
    - name: AccountState
      consistency: strong
      history: true
      history_depth: 50
      indexes:
        - field: customerId
          type: hash
        - field: status
          type: btree
          
  objectStores:
    - name: StatementArchive
      max_size: 50MB
      versioning: true
      encryption: true
      
  provisioning:
    mode: auto
    on_missing: create
```

---

## PART XXIV: CLIENT CACHE CONTRACT (v1.6)

### 24.1 Overview

Client cache contract enables declarative client-side caching within Experiences, supporting offline operation, prefetching, and version-aware cache management.

### 24.2 Client Cache Declaration

```yaml
experiences:
  - name: string
    # ... other experience fields
    
    clientCache:
      enabled: boolean        # Enable client caching
      
      storage:
        type: enum            # memory | persistent | hybrid
        max_size: string      # Max cache size
        encryption: boolean   # Encrypt at rest
        
      strategies:
        primary:
          strategy: enum      # cache_first | network_first | stale_while_revalidate
          ttl: duration       # Entry TTL
        related:
          strategy: enum
          ttl: duration
          lazy_cache: boolean # Cache on access vs eagerly
          
      prefetch:
        enabled: boolean
        views: [string]       # Views to prefetch
        trigger: enum         # on_mount | on_connect | manual
        
      offline:
        enabled: boolean
        required_views: [string]  # Must be cached for offline
        sync_on_reconnect: boolean
        conflict_resolution: enum # client_wins | server_wins | manual
        intent_sync:
          on_cas_failure: enum    # reject_and_notify | queue_for_retry
          retry_with_refresh: boolean
        governance:
          validate_on_sync: boolean
          on_violation: enum      # reject | warn | allow
          
      version_policy:
        on_upgrade: enum      # invalidate_all | invalidate_incompatible | keep
        grace_period: duration # How long to allow old cached data
        migration: enum       # eager | lazy
```

### 24.3 Storage Types

| Type | L0 (Memory) | L1 (Persistent) | Use Case |
|------|-------------|-----------------|----------|
| `memory` | Yes | No | Session-only data |
| `persistent` | No | Yes | Offline support |
| `hybrid` | Yes | Yes | Best performance |

### 24.4 Caching Strategies

| Strategy | Behavior |
|----------|----------|
| `cache_first` | Return cached value; fetch in background |
| `network_first` | Try network; fall back to cache |
| `stale_while_revalidate` | Return stale immediately; revalidate |

### 24.5 Offline Intent Sync

When operating offline, intents are queued and synced on reconnect. CAS failures are handled per configuration:

| on_cas_failure | Behavior |
|----------------|----------|
| `reject_and_notify` | Reject intent, notify user of conflict |
| `queue_for_retry` | Re-queue with option to refresh context |

**Governance Integration (I7)**: When `governance.validate_on_sync` is true, synced intents are validated against current consent and legal hold status.

### 24.6 Client Cache Example

```yaml
experiences:
  - name: MobileBankingExperience
    version: v1
    entry_point: DashboardView
    includes:
      - DashboardView
      - AccountsView
      - TransactionsView
      - TransferView
    policy_set: MobileAccess
    
    clientCache:
      enabled: true
      
      storage:
        type: hybrid
        max_size: 100MB
        encryption: true
        
      strategies:
        primary:
          strategy: stale_while_revalidate
          ttl: 5m
        related:
          strategy: cache_first
          ttl: 15m
          lazy_cache: false
          
      prefetch:
        enabled: true
        views:
          - DashboardView
          - AccountsView
        trigger: on_connect
        
      offline:
        enabled: true
        required_views:
          - DashboardView
          - AccountsView
          - TransactionsView
        sync_on_reconnect: true
        conflict_resolution: server_wins
        intent_sync:
          on_cas_failure: reject_and_notify
          retry_with_refresh: true
        governance:
          validate_on_sync: true
          on_violation: reject
          
      version_policy:
        on_upgrade: invalidate_incompatible
        grace_period: 24h
        migration: lazy
```

---

## PART XXV: VALIDATION AND CONFORMANCE

### 25.1 Validation Levels

| Level | Checks |
|-------|--------|
| **Lexical** | YAML syntax, identifier format |
| **Structural** | Required fields, type correctness |
| **Semantic** | Reference resolution, invariant compliance |
| **Cross-File** | Import resolution, dependency cycles |

### 25.2 Invariant Validation

All invariants (I1-I9) must be validated:

```yaml
validation:
  invariants:
    I1:  # Boundary Isolation
      - check: no_cross_boundary_writes
      - check: work_unit_scope_valid
    I4:  # CAS Safety
      - check: cas_token_required_on_write
      - check: client_cache_cas_handling
    I7:  # Governance Enforcement
      - check: consent_check_present
      - check: legal_hold_respected
      - check: offline_governance_configured
    I9:  # Transport Scope Boundary
      - check: no_transport_in_flows
      - check: routing_only_on_intents
```

### 25.3 Conformance Levels

| Level | Requirements |
|-------|--------------|
| **Minimal** | v1.0 core types, boundaries, work units |
| **Standard** | v1.0-1.2 with policies, signals, realms |
| **Extended** | v1.3-1.4 with experiences, search, privacy |
| **Enterprise** | v1.5 with IoT, sessions, assets, workflows, governance, notifications |
| **Complete** | v1.6 with storage, caching, routing, resources, client cache |

### 25.4 v1.5 Validation Errors

| Code | Message | Resolution |
|------|---------|------------|
| `E1501` | Asset field without category | Asset fields MUST specify category |
| `E1502` | Search indexed without strategy | Search indexed fields MUST specify strategy |
| `E1503` | Scheduled trigger without type | Scheduled triggers MUST specify type (cron/interval/once) |
| `E1504` | Notification channel without type | Notification channels MUST specify type |
| `E1505` | External integration without endpoint | External integrations MUST declare endpoint |
| `E1506` | Durable workflow without checkpoint | Durable workflows MUST specify checkpoint strategy |
| `E1507` | Delegation without scope | Delegations MUST specify scope (actions/data/compositions) |
| `E1508` | Graph query without start | Graph queries MUST declare start_from DataState |
| `E1509` | Form binding without field_mapping | Form triggers MUST specify field_mapping |
| `E1510` | Inference field without model | Inference fields MUST reference a model |

### 25.5 v1.6 Validation Errors

| Code | Message | Resolution |
|------|---------|------------|
| `E1601` | Storage binding references unknown type | Verify type exists and is imported |
| `E1602` | Cache layer references unknown resource | Define resource in resourceRequirements |
| `E1603` | Intent routing in flow definition | Move routing to InputIntent (I9) |
| `E1604` | Key pattern contains invalid variable | Use valid field path in {{var}} |
| `E1605` | Resource requirement not satisfiable | Check available backends |
| `E1606` | Client cache on non-experience type | clientCache only valid in experience |
| `E1607` | Offline required view not in includes | Add view to experience includes |
| `E1608` | Cache invalidation watch key invalid | Verify key pattern format |
| `E1609` | Prefetch view not in experience | Add view to includes list |
| `E1610` | Missing governance in offline config | Add governance config for I7 |

### 25.6 Version Compatibility

| Feature | Min Version | Deprecated | Removed |
|---------|-------------|------------|---------|
| Core types | v1.0 | - | - |
| Policies | v1.0 | - | - |
| Signals | v1.0 | - | - |
| Realms | v1.1 | - | - |
| Contracts | v1.1 | - | - |
| Relationships | v1.2 | - | - |
| Authority | v1.2 | - | - |
| Experiences | v1.3 | - | - |
| Presentations | v1.3 | - | - |
| Consent | v1.4 | - | - |
| Erasure | v1.4 | - | - |
| Legal Hold | v1.4 | - | - |
| Field Permissions | v1.4 | - | - |
| Device Capability | v1.5 | - | - |
| Telemetry | v1.5 | - | - |
| Sessions | v1.5 | - | - |
| Assets | v1.5 | - | - |
| Form-Intent Binding | v1.5 | - | - |
| Search Semantics | v1.5 | - | - |
| Structured Content | v1.5 | - | - |
| Spatial Types | v1.5 | - | - |
| Data Governance | v1.5 | - | - |
| Durable Workflows | v1.5 | - | - |
| Scheduled Triggers | v1.5 | - | - |
| Notification Channels | v1.5 | - | - |
| External Integration | v1.5 | - | - |
| Graph Queries | v1.5 | - | - |
| Conversational UI | v1.5 | - | - |
| ML/Inference | v1.5 | - | - |
| Delegation & Consent | v1.5 | - | - |
| **Storage Binding** | **v1.6** | - | - |
| **Cache Layer** | **v1.6** | - | - |
| **Intent Routing** | **v1.6** | - | - |
| **Resource Requirements** | **v1.6** | - | - |
| **Client Cache** | **v1.6** | - | - |
| **Cache Signal Type** | **v1.6** | - | - |

### 25.7 Conformance Requirements

An implementation is conformant with SMS v1.6 if it:

1. **Implements execution semantics**: All grammar constructs honored
2. **Honors atomic group guarantees**: No partial success within atomic groups
3. **Separates scheduling from execution**: External scheduler model
4. **Preserves failure invariants**: Formal invariants I1-I9 enforced
5. **Enforces policy lifecycle**: Shadow/enforce/deprecated/revoked states
6. **Maintains version compatibility**: Multiple versions coexist during migration
7. **Implements idempotency**: All transitions idempotent with idempotency keys
8. **Propagates execution context**: Trace/causation/correlation IDs across boundaries
9. **Enforces authority singularity** (v1.3): At most one authority per entity instance
10. **Respects entity-scoped availability** (v1.3): Write pause permitted during authority migration
11. **Validates CAS tokens** (v1.3): Epoch/version/lease validation on all writes
12. **Maintains view continuity** (v1.3): Views readable across authority transitions
13. **Resolves compositions correctly** (v1.4): Primary + related views with fieldPermissions
14. **Enforces field permissions** (v1.4): Visibility and masking rules
15. **Routes experiences** (v1.4): Entry points and authorization
16. **Builds navigation hierarchy** (v1.4): From parent relationships and purposes
17. **Evaluates indicators dynamically** (v1.4): Real-time badge updates
18. **Processes form intent bindings** (v1.5): Field mapping, validation, on_success/on_error
19. **Implements materialization contracts** (v1.5): Retrieval modes and temporal windows
20. **Enforces delivery guarantees** (v1.5): At-least-once/exactly-once for intents
21. **Manages session lifecycles** (v1.5): Timeout, refresh, multi-device sync
22. **Applies mask transforms** (v1.5): Context-aware masking with field permissions
23. **Handles asset storage** (v1.5): Upload, variants, access control, lifecycle
24. **Implements search strategies** (v1.5): Full-text, fuzzy, semantic with faceting
25. **Executes scheduled triggers** (v1.5): Cron, interval, timezone handling
26. **Delivers notifications** (v1.5): Multi-channel with user preferences
27. **Supports collaborative sessions** (v1.5): Presence, conflict resolution
28. **Orchestrates durable workflows** (v1.5): Checkpoints, compensation, human tasks
29. **Manages delegations** (v1.5): Temporary authority transfer with audit trail
30. **Integrates external services** (v1.5): Circuit breakers, retries, webhooks
31. **Implements storage binding** (v1.6): KV/stream/object store abstraction with provisioning
32. **Supports cache topology** (v1.6): Local/distributed/hybrid tiers with promotion
33. **Routes intents declaratively** (v1.6): NATS/HTTP/gRPC transport abstraction
34. **Enforces resource requirements** (v1.6): Consistency, durability, observability
35. **Implements client cache contract** (v1.6): Prefetch, offline, version policy, CAS handling

---

## PART XXVI: DESIGN RATIONALE

This section explains the "why" behind major design decisions in SMS v1.6.

### Decision 1: Intent-First, Runtime-Decided

**What**: Grammar describes WHAT must be true, not HOW it's achieved.

**Why**:
- **Future-proofing**: Transport changes don't require spec changes
- **Optimization freedom**: Runtime adapts strategies without rewriting
- **Portability**: Same spec works on cloud, edge, browser, embedded
- **Testing**: Deterministic behavior independent of deployment

**Tradeoff**: Requires sophisticated runtime implementation

**Validation**: Can replace NATS with HTTP/gRPC without changing grammar (v1.6 routing)

---

### Decision 2: Entity-Scoped Authority (v1.3)

**What**: Write authority resolved per entity, not per model.

**Why**:
- **Partial failure tolerance**: One entity's unavailability doesn't affect others
- **Global scalability**: Entities distributed by individual authority
- **Per-entity mobility**: Authority can move for individual entities
- **Isolation**: Entity failures contained

**Tradeoff**: More complex authority management

**Alternative Rejected**: Model-scoped authority
- Would collapse availability when models interdependent
- Would make regional isolation difficult
- Would prevent per-entity optimization

**Validation**: Regional failure affects only entities homed to that region

---

### Decision 3: Invariants Live in Views (v1.3)

**What**: Cross-entity invariants enforced by derived views, not entities.

**Why**:
- **Entity isolation**: Entities remain independently writable
- **No global transactions**: Invariants validated asynchronously
- **Eventual consistency**: Violations flagged rather than blocked
- **Decomposition**: Complex systems broken into manageable pieces

**Tradeoff**: Invariant violations detected, not prevented

**Alternative Rejected**: Entity-embedded invariants
- Would require cross-entity locking
- Would create hidden coupling
- Would break entity independence

**Validation**: Banking balance invariant enforced by view without distributed transactions

---

### Decision 4: Storage as Declarative Intent (v1.6)

**What**: Storage characteristics declared as intent, not as specific backends.

**Why**:
- **Backend independence**: Can swap KV stores without spec changes
- **Automatic provisioning**: Runtime creates necessary infrastructure
- **Uniform abstraction**: Same grammar for JetStream KV, Redis, DynamoDB
- **Intent clarity**: Declares requirements (durability, consistency) not implementations

**Tradeoff**: Runtime must understand multiple backend capabilities

**Alternative Rejected**: Direct backend naming (e.g., "use Redis")
- Would lock specifications to specific technologies
- Would prevent transparent migration
- Would break portability across environments

**Validation**: Same storageBinding works on NATS JetStream, Redis, or cloud KV stores

---

### Decision 5: Unified Cache Layer (v1.6)

**What**: Single cache grammar across local, distributed, and hybrid tiers.

**Why**:
- **Consistency**: Same invalidation semantics regardless of tier
- **Promotion**: Seamless migration from local to distributed
- **Observability**: Unified metrics and tracing
- **Simplicity**: Single mental model for all caching

**Tradeoff**: More complex cache implementation

**Alternative Rejected**: Separate grammars per tier
- Would duplicate invalidation logic
- Would complicate promotion
- Would fragment monitoring

**Validation**: Cache can be promoted from local to distributed without application changes

---

### Decision 6: Intent Routing Abstraction (v1.6)

**What**: Routing configuration separate from intent definitions.

**Why**:
- **Clean separation**: Intents describe business logic, routing describes transport
- **Environment-specific**: Dev uses HTTP, prod uses NATS, no spec changes
- **Scalability**: Can change routing strategy independently
- **Invariant I9**: Transport concerns isolated from business logic

**Tradeoff**: Two-phase configuration (intent + routing)

**Alternative Rejected**: Transport in intent definitions
- Would violate intent-first principle
- Would couple business logic to infrastructure
- Would break environment portability

**Validation**: Same InputIntent works with NATS in prod, HTTP in dev, gRPC in embedded

---

### Decision 7: Client Cache Contract (v1.6)

**What**: Explicit contract for offline-first and client-side caching.

**Why**:
- **Offline capability**: Declarative specification of offline requirements
- **Consistency**: CAS token handling prevents stale writes
- **Performance**: Prefetch and cache-first strategies reduce latency
- **Mobile-friendly**: Acknowledges reality of intermittent connectivity

**Tradeoff**: Increased client complexity

**Alternative Rejected**: Always-online assumption
- Would exclude mobile and PWA use cases
- Would ignore network reality
- Would sacrifice user experience

**Validation**: Experience works offline with eventual sync when reconnected

---

### Decision 8: Resource Requirements (v1.6)

**What**: Declarative requirements for consistency, durability, observability.

**Why**:
- **Clear contracts**: What guarantees does each component need?
- **Validation**: Runtime can verify if requirements are satisfiable
- **Documentation**: Self-documenting operational characteristics
- **Compliance**: Auditable requirements for regulated systems

**Tradeoff**: More verbose declarations

**Alternative Rejected**: Implicit defaults
- Would hide operational assumptions
- Would complicate debugging
- Would prevent validation

**Validation**: Specification declares "replicated durability" requirement explicitly

---

### Decision 9: Complete Applications Need Comprehensive Features (v1.5)

**What**: Real applications require assets, search, notifications, workflows, governance.

**Why**:
- **Completeness**: Can't build production apps with only CRUD
- **Consistency**: All features follow same intent-first principles
- **Integration**: Features compose cleanly (e.g., search + policy + governance)
- **Accessibility**: Voice and conversational UI broaden access

**Tradeoff**: Larger specification surface area

**Alternative Rejected**: Minimal core with extensions
- Would fragment ecosystem
- Would lose consistency
- Would make composition difficult

**Validation**: 15+ production-ready examples demonstrate completeness

---

### Decision 10: Fail-Closed by Default

**What**: Ambiguous authorization defaults to deny.

**Why**:
- **Security**: Safer default posture
- **Debuggability**: Forces explicit policy definition
- **Compliance**: Audit-friendly
- **Predictability**: No undefined behavior

**Tradeoff**: Requires comprehensive policy coverage

**Validation**: Policy gaps surface as denied requests immediately

---

### Decision 11: Local Authorization Enforcement

**What**: Authorization decisions made locally with cached policies.

**Why**:
- **Performance**: No network hop for every request
- **Availability**: Works during network partitions
- **Predictability**: Bounded latency
- **Scale**: No central bottleneck

**Tradeoff**: Policy updates eventually consistent

**Validation**: Sub-second propagation, microsecond evaluation

---

### Decision 12: Shadow Policies for Safe Rollout

**What**: New policies evaluated alongside enforced policies without affecting outcomes.

**Why**:
- **Risk reduction**: See impact before committing
- **Data-driven decisions**: Measure divergence quantitatively
- **Rollback safety**: Easy revert if problematic
- **Gradual deployment**: Canary patterns supported

**Tradeoff**: Dual evaluation adds complexity

**Validation**: Policy changes show <1% divergence before promotion

---

### v1.5 Feature Design Rationale (Detailed)

This section provides in-depth rationale for major v1.5 features, including detailed examples and alternatives considered.

#### Why Form Intent Binding

**Context**: Forms are the primary user input mechanism.

**Decision**: Explicit grammar for form-to-intent binding.

**Reasoning**:
1. **Completeness**: Bridges gap between UI and backend
2. **Validation clarity**: Separates UI validation (fast) from business validation (slow)
3. **User experience**: Declarative success/error handling
4. **Framework agnostic**: Works with React, Vue, mobile, voice

**Field Source Types**:
- `context`: From session/navigation (e.g., current user, selected account)
- `input`: User-provided (e.g., form fields)
- `computed`: Calculated from other fields (e.g., total = sum(items))

**Validation Timing**:
```yaml
field_mapping:
  - view_field: email
    supplies: user.email
    source: input
    validation:
      constraint: is_valid_email(email)
      message: "Invalid email format"
    # Validated in browser BEFORE submission (fast UX)

  - view_field: username
    supplies: user.username
    source: input
    # Business validation: username uniqueness
    # Validated on server AFTER submission
```

**Alternative Rejected**: Implicit form handling
- Every framework does it differently
- Hard to validate correctness
- Can't generate forms from spec

---

#### Why Asset Semantics

**Context**: Real applications handle files, not just structured data.

**Decision**: Binary content as first-class type with lifecycle.

**Reasoning**:
1. **Completeness**: Applications upload invoices, images, documents
2. **Lifecycle control**: Retention policies, expiry, garbage collection
3. **Transformation**: Thumbnails, compression, format conversion
4. **Access control**: Inherits policy from parent entity
5. **CDN integration**: Efficient delivery

**Asset Lifecycle**:
```yaml
# Permanent: Survives entity deletion
avatar:
  type: asset
  lifecycle: permanent
  # Use case: Profile pictures, legal documents

# Temporary: Short-lived staging
upload_staging:
  type: asset
  lifecycle: temporary
  # Use case: Upload processing, virus scanning

# Expiring: Time-bounded auto-deletion
session_file:
  type: asset
  lifecycle: expiring
  expires_after: 24h
  # Use case: Temporary file sharing
```

**Alternative Rejected**: URLs as strings
- Loses lifecycle semantics
- Can't enforce retention
- No transformation hints
- Access control implicit

---

#### Why Search as Grammar

**Context**: Users expect to search data.

**Decision**: Declarative search configuration at field level.

**Reasoning**:
1. **User expectation**: Modern apps require search
2. **Engine agnostic**: Works with Elasticsearch, Algolia, Typesense
3. **Policy integration**: Search respects authorization
4. **Consistency**: Same semantic meaning across engines

**Search Strategy Selection**:
```yaml
# Full-text: Natural language queries
name:
  search:
    strategy: full_text
  # Use case: Product names, descriptions, articles

# Exact: Precise matching
sku:
  search:
    strategy: exact
  # Use case: SKUs, order IDs, codes

# Fuzzy: Typo-tolerant
email:
  search:
    strategy: fuzzy
    fuzzy:
      max_edits: 2
  # Use case: Email search with typos

# Semantic: Embedding similarity
description:
  search:
    strategy: semantic
  # Use case: "Find similar products"
```

**Alternative Rejected**: Application-level search
- Inconsistent across features
- Hard to maintain
- Doesn't respect policies

---

#### Why Scheduled Triggers

**Context**: Recurring tasks are common.

**Decision**: Declarative scheduling in inputIntent.

**Reasoning**:
1. **Common pattern**: Reports, cleanup, reminders
2. **Timezone handling**: DST, user timezones
3. **Idempotency**: Missed execution handling
4. **Authorization**: Scheduled tasks respect policies

**Missed Execution Strategies**:
```yaml
scheduled:
  on_missed:
    action: skip              # Daily report: Skip yesterday, continue today
    action: execute_immediately  # Critical cleanup: Run ASAP
    action: queue             # Batch job: Queue for later
```

**Alternative Rejected**: External cron + API calls
- Bypasses authorization
- No idempotency guarantees
- Duplicates scheduling logic

---

#### Why Durable Workflows

**Context**: Some processes span days/weeks with human steps.

**Decision**: Long-running workflows with checkpoints and compensation.

**Reasoning**:
1. **Real requirement**: Loan approvals, contract signing, multi-step approvals
2. **Human involvement**: Workflows pause for human decisions
3. **Fault tolerance**: Survives restarts, region failures
4. **Compensation**: Explicit undo logic

**Example - Loan Approval (2 weeks)**:
```yaml
smsFlows:
  - name: LoanApproval
    durable:
      enabled: true
      timeout:
        total: 14d        # 2 weeks max
    steps:
      - work_unit: credit_check
      - human_tasks:
          - step: underwriter_review
            timeout: 48h    # 2 days for review
            escalation:
              after: 24h    # Escalate after 1 day
              to: senior_underwriter
      - work_unit: generate_contract
      - human_tasks:
          - step: customer_signature
            timeout: 7d     # 7 days to sign
      - work_unit: disburse_funds
```

**Alternative Rejected**: Short-lived SMS flows only
- Can't handle human tasks
- Can't survive restarts
- No multi-day timeouts

---

#### Why Collaborative Sessions

**Context**: Users expect real-time collaboration (Google Docs style).

**Decision**: Collaborative editing as first-class feature.

**Reasoning**:
1. **User expectation**: Real-time co-editing is table stakes
2. **Conflict resolution**: OT or CRDT strategies
3. **Presence awareness**: See who's editing what
4. **Permission integration**: Respects access control

**Conflict Resolution Strategies**:
```yaml
collaboration:
  conflict_resolution:
    strategy: operational_transform  # Google Docs style
    strategy: crdt                   # Eventual consistency
    strategy: last_write_wins        # Simplest
```

**Alternative Rejected**: Application-level WebSocket
- Inconsistent implementations
- Hard to secure
- Doesn't integrate with policies

---

#### Why Data Governance

**Context**: Regulations require retention and deletion.

**Decision**: Declarative governance in dataState.

**Reasoning**:
1. **Compliance**: GDPR, CCPA, HIPAA, SOX all require data governance
2. **Automation**: Retention enforced by runtime, not application code
3. **Audit trail**: All governance actions logged
4. **Legal hold**: Support for litigation holds

**Retention Example**:
```yaml
dataStates:
  # GDPR: Delete after 30 days of account closure
  - name: CustomerData
    governance:
      retention:
        policy: event_based
        after_event: account_deleted
        grace_period: 30d
        on_expiry: anonymize    # Remove PII, keep aggregates

  # SOX: Financial records for 7 years
  - name: FinancialTransaction
    governance:
      retention:
        policy: time_based
        duration: 7y
        on_expiry: archive      # Move to cold storage
```

**Alternative Rejected**: Application-level deletion
- Easy to miss records
- Hard to audit
- Doesn't scale

---

#### Why Edge Device Support

**Context**: IoT and offline scenarios require edge processing.

**Decision**: Edge device semantics with sync strategies.

**Reasoning**:
1. **Real use cases**: ATMs, POS terminals, manufacturing equipment
2. **Offline operation**: Can't rely on connectivity
3. **Sync strategies**: Eventual consistency with conflict resolution
4. **Resource constraints**: Limited CPU/memory

**Example - ATM**:
```yaml
edgeDevices:
  - name: ATMDevice
    capabilities:
      compute: arm64_2ghz
      storage: 32GB
      connectivity: intermittent
    sync:
      strategy: bidirectional
      frequency: real_time
      conflict_resolution: server_wins  # Bank is source of truth
    local_processing:
      enabled: true
      fallback_to_cloud: false  # Must work offline
      models: [card_validation, pin_verification]
```

**Alternative Rejected**: Cloud-only execution
- Doesn't work offline
- High latency for edge use cases
- Misses optimization opportunities

---

## Conformance

An implementation is conformant with SMS v1.6 if it:

1. **Implements execution semantics**: All grammar constructs honored
2. **Honors atomic group guarantees**: No partial success
3. **Separates scheduling from execution**: External scheduler model
4. **Preserves failure invariants**: Formal invariants I1-I9 enforced
5. **Enforces policy lifecycle**: Shadow/enforce/deprecated/revoked
6. **Maintains version compatibility**: Multiple versions coexist
7. **Implements idempotency**: All transitions idempotent
8. **Propagates execution context**: Trace/causation/correlation IDs
9. **Enforces authority singularity** (v1.3): At most one authority per entity
10. **Respects entity-scoped availability** (v1.3): Write pause permitted during migration
11. **Validates CAS tokens** (v1.3): Epoch/version/lease validation
12. **Maintains view continuity** (v1.3): Views readable across authority transitions
13. **Resolves compositions correctly** (v1.4): Primary + related views
14. **Enforces field permissions** (v1.4): Visibility and masking
15. **Routes experiences** (v1.4): Entry points and authorization
16. **Builds navigation hierarchy** (v1.4): From parent relationships
17. **Evaluates indicators dynamically** (v1.4): Real-time badge updates
18. **Processes form intent bindings** (v1.5): Field mapping and validation
19. **Implements materialization contracts** (v1.5): Retrieval modes and temporal windows
20. **Enforces delivery guarantees** (v1.5): At-least-once/exactly-once
21. **Manages session lifecycles** (v1.5): Timeout, refresh, multi-device
22. **Applies mask transforms** (v1.5): Context-aware masking
23. **Handles asset storage** (v1.5): Upload, variants, access control
24. **Implements search strategies** (v1.5): Full-text, fuzzy, semantic
25. **Executes scheduled triggers** (v1.5): Cron, interval, timezone handling
26. **Delivers notifications** (v1.5): Multi-channel with preferences
27. **Supports collaborative sessions** (v1.5): Presence, conflict resolution
28. **Orchestrates durable workflows** (v1.5): Checkpoints, compensation, human tasks
29. **Manages delegations** (v1.5): Temporary authority transfer with audit
30. **Integrates external services** (v1.5): Circuit breakers, retries
31. **Implements storage binding** (v1.6): Declarative storage configuration
32. **Manages unified caching** (v1.6): Multi-tier cache coordination
33. **Routes intents** (v1.6): Transport-agnostic intent delivery
34. **Enforces resource requirements** (v1.6): Infrastructure abstraction
35. **Implements client cache contracts** (v1.6): Offline-first capabilities

---

## Specification Metadata

```yaml
specMetadata:
  version: v1.6
  status: Production Ready
  completeness: 100%
  breaking_changes: None (backward compatible with v1.5)
  ingestible: true
  machine_readable: true
  
  changelog:
    v1.6:
      - "Storage Binding for declarative storage configuration"
      - "Unified Caching with multi-tier coordination"
      - "Intent Routing for transport-agnostic delivery"
      - "Resource Requirements for infrastructure abstraction"
      - "Client Cache Contract for offline-first capabilities"
      - "Enhanced accessibility support (WCAG compliance)"
      - "Improved versioning and evolution strategy"
      - "Extended validation rules for v1.6 features"
      - "All v1.5 features maintained with full backward compatibility"
    
    v1.5:
      - "Added 28 comprehensive addendums (X-AY)"
      - "Form-Intent Binding for declarative forms"
      - "View Materialization Contracts with streaming"
      - "Session Management with multi-device sync"
      - "Asset Semantics for binary content"
      - "Search Semantics with faceting and ranking"
      - "Scheduled Triggers for temporal automation"
      - "Notification Channels for multi-channel delivery"
      - "Collaborative Sessions for real-time co-editing"
      - "Durable Workflows for long-running processes"
      - "Delegation/Consent for authority transfer"
      - "Conversational UI for voice and chat"
      - "External Integration for third-party APIs"
      - "Data Governance for compliance"
      - "Edge Device support for IoT"
      - "Spatial Types for GIS"
      - "Graph Queries for relationship traversal"
      - "Structured Content for rich text"
      - "ML/Inference integration"
      - "Accessibility and user preferences"
    
    v1.4:
      - "UI Composition Grammar (PresentationComposition)"
      - "Experience Grammar for user journeys"
      - "Field Permissions with masking"
      - "Navigation grammar with purposes and scopes"
      - "Indicators for dynamic badges"
      - "View relationship types"
    
    v1.3:
      - "Entity-Scoped Authority"
      - "Authority Transitions (pause-and-cutover)"
      - "Storage Roles (control vs data)"
      - "Relationship Grammar"
      - "Invariant Policy"
      - "CAS Token semantics"
      - "8 Formal Invariants (expanded to 9 in v1.6)"
      - "Multi-region survivability"
```

---

## APPENDIX A: GRAMMAR QUICK REFERENCE

### Top-Level Structure

```ebnf
document        = system_decl , { collection_decl } ;
collection_decl = datatype_collection | model_collection | datastate_collection
                | constraint_collection | evolution_collection | materialization_collection
                | transition_collection | relationship_collection | authority_collection
                | view_collection | presentation_view_collection
                | presentation_binding_collection | presentation_composition_collection
                | experience_collection | conversational_experience_collection
                | input_intent_collection | sms_flow_collection
                | workers_collection | work_unit_contract_collection
                | policy_collection | signal_collection | realm_collection
                | search_intent_collection | webhook_receiver_collection
                | external_dependency_collection | graph_query_collection
                | asset_collection | delegation_collection
                | data_governance_collection | consent_record_collection
                | erasure_request_collection | legal_hold_collection
                | notification_channel_collection | scheduled_trigger_collection
                | durable_workflow_collection
                | device_capability_collection | device_telemetry_collection
                | actuator_command_collection | edge_device_collection
                | storage_binding_collection | cache_layer_collection
                | resource_requirements_decl ;
```

### v1.5 New Keywords

```
asset, structured_content, spatial, search, inference, governance,
triggers, field_mapping, on_success, on_error, confirmation,
fieldPermissions, mask, durable, checkpoint, compensation, human_tasks,
scheduled, notificationChannel, webhookReceiver,
graphQuery, traversal, conversationalExperience, dialogue, understanding,
delegation, consent, retention, residency, audit, encryption
```

> **Collection key mappings**: Some v1.5 keywords use different names as top-level collection keys:
> `externalIntegration` → `externalDependencies:`, `notificationChannel` → `notificationChannels:`,
> `asset` → `assets:`, `delegation` → `delegations:`,
> `conversationalExperience` → `conversationalExperiences:`.

### v1.5 New Enumerations

```
asset_category = "image" | "document" | "media" | "archive" | "generic" ;
asset_lifecycle = "permanent" | "temporary" | "expiring" ;
access_mode    = "public" | "signed" | "authenticated" | "policy" ;
search_strategy = "full_text" | "exact" | "prefix" | "fuzzy" | "semantic" ;
geometry_type  = "point" | "line" | "polygon" | "multi_point" ;
classification = "pii" | "phi" | "pci" | "financial" | "public" | "internal" ;
retention_policy = "time_based" | "event_based" | "indefinite" | "legal_hold" ;
expiry_action  = "delete" | "anonymize" | "archive" ;
trigger_type   = "cron" | "interval" | "once" ;
channel_type   = "email" | "sms" | "push" | "in_app" | "webhook" ;
interaction_mode = "voice" | "text" | "multimodal" ;
inference_strategy = "on_write" | "on_read" | "scheduled" | "manual" ;
```

### v1.6 New Keywords

```
storageBinding, cacheLayer, resourceRequirements, routing,
topology, invalidation, observability, provisioning, clientCache,
tier, local, distributed, hybrid, promotion, watch_keys,
eventLogs, entityStores, objectStores, prefetch, offline,
version_policy, intent_sync
```

### v1.6 New Enumerations

```
storage_type   = "kv" | "stream" | "object_store" ;
cache_tier     = "local" | "distributed" | "hybrid" ;
eviction       = "lru" | "lfu" | "fifo" ;
invalidation   = "explicit" | "ttl" | "watch" | "hybrid" ;
transport      = "nats" | "http" | "grpc" ;
response_mode  = "request_reply" | "async" | "fire_and_forget" ;
delivery       = "at_most_once" | "at_least_once" | "exactly_once" ;
consistency    = "strong" | "eventual" | "causal" ;
durability     = "memory" | "disk" | "replicated" ;
provisioning   = "auto" | "explicit" ;
client_storage = "memory" | "persistent" | "hybrid" ;
cache_strategy = "cache_first" | "network_first" | "stale_while_revalidate" ;
conflict       = "client_wins" | "server_wins" | "manual" ;
cas_handling   = "reject_and_notify" | "queue_for_retry" ;
upgrade_policy = "invalidate_all" | "invalidate_incompatible" | "keep" ;
migration      = "eager" | "lazy" ;
pressure       = "low" | "medium" | "high" | "critical" ;
```

---

## APPENDIX B: SIGNAL SUBJECT PATTERNS

### Standard Patterns

```
SMS.SIGNAL.<domain>.<signal_name>
SMS.SIGNAL.<domain>.<signal_name>.<realm>
```

### v1.6 Cache Signal Patterns

```
SMS.SIGNAL.cache.<cache_name>
SMS.SIGNAL.cache.<cache_name>.<tier>
SMS.SIGNAL.cache.<cache_name>.<tier>.<region>
```

### v1.6 Intent Routing Patterns

```
SMS.INTENT.<domain>.<intent_name>
SMS.INTENT.<domain>.<intent_name>.<realm>
SMS.DLQ.<domain>.<intent_name>
```

---

## APPENDIX C: PERFORMANCE TARGETS

| Component | Target | P99 |
|-----------|--------|-----|
| Local cache read | < 1ms | < 5ms |
| Distributed cache read | < 10ms | < 50ms |
| Storage binding KV read | < 5ms | < 25ms |
| Storage binding KV write | < 10ms | < 50ms |
| Intent routing (NATS) | < 15ms | < 75ms |
| Intent routing (HTTP) | < 50ms | < 250ms |
| Client cache L0 read | < 1ms | < 3ms |
| Client cache L1 read | < 10ms | < 50ms |

---

## APPENDIX D: IMPLEMENTATION PATTERNS

### Pattern: Multi-Region Authority Management

**Scenario**: Global banking application with customers in US and EU.

**Requirements**:
- US customer data stays in US
- EU customer data stays in EU
- Cross-region viewing allowed
- Authority can migrate for follow-the-sun

**Implementation**:
```yaml
dataStates:
  - name: AccountRecord
    type: Account
    lifecycle: persistent
    mutability:
      scope: entity
      exclusive: true
      authority: AccountAuthority
    governance:
      residency:
        primary_region: |
          if customer.country in ["US", "CA", "MX"] then "us-east"
          else if customer.country in EU_COUNTRIES then "eu-west"
          else "default-region"
        transfer:
          allowed: false  # Data cannot leave primary region

authorities:
  - name: AccountAuthority
    scope: entity
    resolution: |
      # Authority follows primary region by default
      residency.primary_region
    migration:
      allowed: true
      strategy: pause-and-cutover
      triggers:
        - type: manual  # Only manual migration for compliance
      constraints:
        - type: governance
          rule: residency.transfer_allowed == true
```

**Runtime Behavior**:
1. US customer account created → authority in us-east
2. Customer moves to EU → residency unchanged (data stays in US)
3. Authority CANNOT migrate to EU (governance constraint)
4. EU staff can VIEW account (cross-region read)
5. Writes go to us-east (authority region)

---

### Pattern: Eventual Consistency with Invariant Enforcement

**Scenario**: E-commerce inventory across warehouses.

**Requirements**:
- Multiple warehouses can sell concurrently
- Total sold cannot exceed total inventory
- No distributed transactions
- Flag violations for investigation

**Implementation**:
```yaml
dataStates:
  # Each warehouse has independent inventory
  - name: WarehouseInventory
    type: Inventory
    lifecycle: persistent
    mutability:
      scope: entity  # Each warehouse independent

materializations:
  # Derived view enforces global invariant
  - name: GlobalInventoryView
    source: [WarehouseSaleEvent, WarehouseRestockEvent]
    targetState: GlobalInventory
    invariants:
      enforced_by: view
      description: "Total sold <= total inventory"
    retrieval:
      mode: singleton  # One global view

# Violation alert flow
smsFlows:
  - name: InventoryViolationAlert
    triggeredBy:
      dataState: GlobalInventoryView
      condition: total_sold > total_inventory
    steps:
      - work_unit: notify_operations
      - work_unit: pause_sales
      - work_unit: trigger_investigation
```

**Runtime Behavior**:
1. Warehouse A sells 100 units (immediately succeeds)
2. Warehouse B sells 50 units (immediately succeeds)
3. View updates: total_sold = 150
4. View detects: 150 > 120 inventory
5. Alert triggered
6. Operations investigates (oversell happened)
7. Compensation flow initiated

**Why This Works**:
- Warehouses don't block each other (independent writes)
- Violations detected within seconds (view update latency)
- Can't prevent 100% of violations (eventual consistency)
- **Trade-off accepted** for high availability and low latency

---

### Pattern: Progressive Form Validation

**Scenario**: Complex multi-step user onboarding.

**Requirements**:
- Save progress without full validation
- Client-side validation for UX
- Server-side validation for security
- Clear error messages

**Implementation**:
```yaml
presentationViews:
  # Step 1: Personal Info (partial validation)
  - name: PersonalInfoForm
    triggers:
      intent: SavePersonalInfo
      binding: form
      field_mapping:
        - view_field: email
          supplies: user.email
          source: input
          validation:
            constraint: is_valid_email(email)
            message: "Invalid email format"
          # Client-side validation (fast UX)
        
        - view_field: phone
          supplies: user.phone
          source: input
          validation:
            constraint: matches_pattern(phone, "^\\+?[0-9]{10,15}$")
            message: "Invalid phone number"
      
      confirmation:
        required: false  # No confirmation for save draft
      
      on_success:
        action: navigate_to
        target: AddressFormView
        notification:
          type: toast
          message: "Progress saved"

inputIntents:
  - name: SavePersonalInfo
    proposes:
      dataState: UserRecord
    supplies:
      required: [email, phone]
    constraints:
      guarantees: [email_format_valid, phone_format_valid]
      defers: [email_unique, phone_verified]
      # Defer expensive validations to final submission

  # Step 2: Address Info
  # ... similar pattern ...

  # Final Step: Submit Application (full validation)
  - name: SubmitApplication
    proposes:
      dataState: UserRecord
    supplies:
      required: [email, phone, address, identity_verification]
    constraints:
      guarantees: [email_unique, phone_verified, address_valid, identity_confirmed]
      # Now enforce ALL constraints
```

**Validation Timing**:
- **Client-side** (instant): Format validation
- **Server-side save** (fast): Basic validation
- **Server-side submit** (slow): Complete validation including external checks

---

### Pattern: Multi-Channel Notification with Fallback

**Scenario**: Critical payment alerts must reach users.

**Requirements**:
- Try push notification first (fastest)
- Fall back to email if push fails
- Respect user preferences
- Respect quiet hours

**Implementation**:
```yaml
notificationChannels:
  - name: PaymentAlert
    version: v1
    channels:
      - type: push
        enabled: true
        config:
          push:
            platform: fcm
            priority: high
      
      - type: email
        enabled: true
        config:
          email:
            from: "alerts@bank.com"
            template: payment_alert
      
      - type: sms
        enabled: true
        config:
          sms:
            sender: "BANK"
            template: payment_alert_sms
    
    preferences:
      user_controllable: true
      defaults:
        push: enabled
        email: enabled
        sms: disabled
      quiet_hours:
        enabled: true
        start: "22:00"
        end: "08:00"
        timezone: user.timezone
    
    delivery:
      retry:
        enabled: true
        max_attempts: 3
        backoff: exponential
      
      fallback:
        enabled: true
        order: [push, email, sms]  # Try in this order
      
      batching:
        enabled: false  # Critical alerts sent immediately
```

**Runtime Behavior**:
1. Payment processed at 14:00 (not quiet hours)
2. Try push → fails (app not installed)
3. Try email → succeeds
4. User receives email within 1 second
5. Audit log records: "push failed, email succeeded"

**Alternative Scenario**:
1. Payment processed at 23:00 (quiet hours)
2. Check user preferences
3. User has quiet_hours enabled
4. Notification queued until 08:00
5. Delivered at 08:01 next morning

---

### Pattern: Durable Workflow with Compensation

**Scenario**: Multi-step payment processing with external gateway.

**Requirements**:
- Reserve funds
- Call payment gateway
- Update order status
- If gateway fails, refund reservation
- If order update fails, reverse payment

**Implementation**:
```yaml
smsFlows:
  - name: ProcessPayment
    durable:
      enabled: true
      checkpoint:
        frequency: per_step
      compensation:
        enabled: true
        on_failure: compensate
      timeout:
        total: 5m
    
    steps:
      # Step 1: Reserve funds (local)
      - work_unit: reserve_funds
        input: { amount: "{{order.total}}", account: "{{order.account_id}}" }
        output: reservation_id
        compensate: release_reservation
      
      # Step 2: Charge payment gateway (external)
      - work_unit: charge_gateway
        input: { amount: "{{order.total}}", token: "{{payment.token}}" }
        output: charge_id
        timeout: 30s
        retry:
          max_attempts: 3
          backoff: exponential
        compensate: refund_gateway
      
      # Step 3: Update order status (local)
      - work_unit: mark_order_paid
        input: { order_id: "{{order.id}}", charge_id: "{{charge_id}}" }
        compensate: mark_order_pending
```

**Failure Scenarios**:

**Scenario 1: Gateway fails**
1. reserve_funds succeeds (checkpoint saved)
2. charge_gateway fails after 3 retries
3. Compensation: release_reservation executes
4. Funds returned to customer
5. Order remains unpaid

**Scenario 2: Order update fails**
1. reserve_funds succeeds
2. charge_gateway succeeds (checkpoint saved)
3. mark_order_paid fails
4. Compensation: refund_gateway executes
5. Then: release_reservation executes
6. Gateway refunded, funds returned, order unpaid

**Scenario 3: System crash during step 2**
1. reserve_funds succeeds (checkpoint saved)
2. charge_gateway in progress
3. **System crashes**
4. **System restarts**
5. Workflow resumes from checkpoint
6. charge_gateway retried (idempotent)
7. Continues to step 3

---

### Pattern: Real-Time Collaborative Editing

**Scenario**: Multiple users editing shared document.

**Requirements**:
- See other users' cursors
- Conflict-free editing
- Undo/redo per user
- Permission-based editing

**Implementation**:
```yaml
experiences:
  - name: DocumentEditor
    collaboration:
      enabled: true
      session:
        type: document
        scope: document.id
        max_participants: 50
      
      awareness:
        show_active_users: true
        show_cursors: true
        show_selections: true
        timeout: 30s  # Inactive after 30s of no activity
      
      permissions:
        mode: policy_based
        policy_set: DocumentEditPolicies
      
      conflict_resolution:
        strategy: operational_transform  # Google Docs style
        merge_policy: DocumentMergePolicy

policies:
  - name: DocumentEditPolicy
    version: v1
    appliesTo:
      type: dataState
      name: DocumentContent
    effect: allow
    when: |
      subject.id == document.owner_id OR
      subject.id in document.collaborators OR
      subject.role == "admin"
```

**Runtime Behavior**:
1. User A opens document
2. User B opens same document
3. Both see each other in "Active Users"
4. User A types "Hello"
5. User B sees "Hello" appear in real-time
6. User B types "World" at same position
7. Operational Transform resolves conflict
8. Both see "Hello World"
9. User A goes idle for 31 seconds
10. User A shown as "Away"

---

## APPENDIX E: COMPLETE EXAMPLES

### Example 1: Banking Account Management

This comprehensive example demonstrates entity-scoped authority, multi-region deployment, field permissions, and UI composition.

#### Data Model

```yaml
dataTypes:
  # Account entity with entity-scoped authority
  - name: Account
    version: v1
    fields:
      id: uuid
      account_number: string
      owner_id: uuid
      balance: decimal
      currency: string
      status: enum[active, frozen, closed]
      created_at: timestamp
      updated_at: timestamp

  # Transaction events (append-only)
  - name: Transaction
    version: v1
    fields:
      id: uuid
      account_id: uuid
      type: enum[debit, credit, transfer]
      amount: decimal
      currency: string
      description: string
      timestamp: timestamp
      balance_after: decimal

dataStates:
  - name: AccountRecord
    type: Account
    lifecycle: persistent
    constraintPolicy:
      enforcement: strict
    mutability:
      scope: entity
      exclusive: true
      authority: AccountAuthority
    governance:
      classification: pii
      retention:
        policy: event_based
        after_event: account_closed
        grace_period: 90d
        on_expiry: anonymize
      audit:
        access_logging: true
        modification_logging: true
        retention_for_logs: 7y

  - name: TransactionEvent
    type: Transaction
    lifecycle: persistent
    storage:
      role: data
      rebuildable: true

authorities:
  - name: AccountAuthority
    scope: entity
    resolution: |
      if owner.country in ["US", "CA", "MX"] then "us-east"
      else if owner.country in EU_COUNTRIES then "eu-west"
      else "ap-south"
    migration:
      allowed: true
      strategy: pause-and-cutover
      triggers:
        - type: latency
          condition: p99_latency > 200ms
        - type: manual

materializations:
  # Balance view (materialized)
  - name: AccountBalanceView
    source: TransactionEvent
    targetState: AccountBalance
    freshness:
      maxStaleness: 1s
    authority_agnostic: true
    epoch_tolerant: true
    retrieval:
      mode: by_entity
      entity_key: account_id
      on_unavailable:
        strategy: use_cached
        cache_ttl: 5m
    invariants:
      enforced_by: view
      description: "Balance equals sum of credits minus sum of debits"
```

#### UI Layer

```yaml
presentationViews:
  # Account summary view
  - name: AccountSummaryView
    version: v1
    consumes:
      dataState: AccountRecord
      version: v1
    fieldPermissions:
      - field: account_number
        visible: true
        mask:
          unless: subject.role in ["owner", "admin"]
          transform: reveal_last(4)
          placeholder: "****-****-####"
      - field: balance
        visible:
          policy: ViewBalancePolicy
    interactionModel:
      badges:
        - if: account.status == "frozen"
          label: "FROZEN"
          severity: error
      actions:
        - if: account.status == "active"
          action: transfer
          label: "Transfer Funds"
          intent: InitiateTransfer

  # Transaction list view
  - name: TransactionListView
    version: v1
    consumes:
      dataState: TransactionEvent
      version: v1
    interactionModel:
      filters:
        - field: type
          type: enum
        - field: timestamp
          type: date_range
      sorts:
        - field: timestamp
          default: desc

presentationCompositions:
  # Account workspace composition
  - name: AccountWorkspace
    version: v1
    description: "Complete account management workspace"
    primary: AccountSummaryView
    navigation:
      label: "Account {{account.account_number}}"
      purpose: accounts
      param: account_id
      policy_set: AccountAccessPolicies
      indicator:
        when: pending_transactions > 0
        severity: info
    related:
      - view: AccountBalanceView
        relationship: derived
        cardinality: one
        required: true
      - view: TransactionListView
        relationship: detail
        cardinality: many
        required: false
        data_scope: { account_id: "{{account.id}}" }
        navigation:
          label: "Transactions"
          scope: within_composition
      - view: TransferFormView
        relationship: action
        cardinality: one
        trigger: initiate_transfer
        navigation:
          label: "Transfer Funds"
          scope: on_action
    fetch:
      order: primary_first
      related:
        strategy: parallel
        max_parallel: 3
      cache:
        strategy: stale_while_revalidate
        ttl: 30s

experiences:
  # Customer banking experience
  - name: CustomerBanking
    version: v1
    description: "Customer-facing banking portal"
    entry_point: CustomerDashboard
    includes:
      - CustomerDashboard
      - AccountWorkspace
      - TransactionHistory
      - TransferCenter
    policy_set: CustomerPolicies
    session:
      required: true
      subject_binding:
        mode: authenticated
      lifetime:
        mode: sliding
        idle_timeout: 15m
        absolute_timeout: 8h
        refresh: auto
      multi_device:
        enabled: true
        sync_strategy: real_time
        conflict_resolution: last_write_wins
```

#### Input & Flows

```yaml
inputIntents:
  # Transfer intent
  - name: InitiateTransfer
    version: v1
    proposes:
      dataState: TransactionEvent
      version: v1
    supplies:
      required: [from_account_id, to_account_id, amount]
      optional: [description]
    constraints:
      guarantees: [amount_positive, accounts_different]
      defers: [sufficient_balance, account_active]
    delivery:
      guarantee: at_least_once
      acknowledgment: required
      timeout: 30s
      retry:
        allowed: true
        max_attempts: 3
        backoff: exponential
    response:
      on_success:
        includes: [transaction_id, new_balance, timestamp]
      on_error:
        includes: [error_code, message]
        field_errors:
          enabled: true
          format: map

presentationViews:
  # Transfer form
  - name: TransferFormView
    version: v1
    consumes:
      dataState: TransferFormData
      version: v1
    triggers:
      intent: InitiateTransfer
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
          validation:
            constraint: to_account != from_account
            message: "Cannot transfer to same account"
        - view_field: amount
          supplies: amount
          source: input
          required: true
          validation:
            constraint: amount > 0
            message: "Amount must be positive"
        - view_field: description
          supplies: description
          source: input
          required: false
      confirmation:
        required: true
        message: "Transfer {{amount}} {{currency}} to account {{to_account}}?"
      on_success:
        action: navigate_back
        notification:
          type: toast
          message: "Transfer submitted successfully"
      on_error:
        action: display_inline
        show_field_errors: true

# Transfer processing flow
smsFlows:
  - name: ProcessTransfer
    triggeredBy:
      inputIntent: InitiateTransfer
    steps:
      - validate:
          constraints: [amount_positive, accounts_exist]
      - atomic_group:
          realm: transfer_realm
          steps:
            - work_unit: debit_source_account
              input: { account_id: "{{from_account_id}}", amount: "{{amount}}" }
            - work_unit: credit_dest_account
              input: { account_id: "{{to_account_id}}", amount: "{{amount}}" }
            - work_unit: record_transfer
              input: { from: "{{from_account_id}}", to: "{{to_account_id}}", amount: "{{amount}}" }
      - persist:
          state: TransactionEvent
      - work_unit: send_notification
        input: { account_id: "{{from_account_id}}", type: "transfer_completed" }
    authorization:
      policySet: TransferPolicies
```

#### Policies

```yaml
policies:
  # View balance policy
  - name: ViewBalancePolicy
    version: v1
    appliesTo:
      type: dataState
      name: AccountRecord
    effect: allow
    when: |
      subject.id == account.owner_id OR
      subject.role == "admin" OR
      (subject.role == "support" AND subject.has_approval("view_balance"))

  # Transfer policy
  - name: TransferAuthPolicy
    version: v1
    appliesTo:
      type: inputIntent
      name: InitiateTransfer
    effect: allow
    when: |
      subject.id == from_account.owner_id AND
      from_account.status == "active" AND
      to_account.status == "active" AND
      amount <= from_account.balance
```

---

### Example 2: E-Commerce Product Catalog with Search

This example demonstrates search semantics, asset management, and materialized views.

#### Data Model

```yaml
dataTypes:
  - name: Product
    version: v1
    fields:
      id: uuid
      sku: string
      name:
        type: string
        search:
          indexed: true
          strategy: full_text
          full_text:
            analyzer: standard
            stemming: enabled
          weight: 2.0
          suggest:
            enabled: true
            type: completion
          highlight:
            enabled: true
            fragment_size: 150
      description:
        type: string
        search:
          indexed: true
          strategy: full_text
          weight: 1.0
      category: string
      brand: string
      price: decimal
      currency: string
      in_stock: boolean
      image:
        type: asset
        asset:
          category: image
          constraints:
            max_size: 5MB
            mime_types: ["image/jpeg", "image/png", "image/webp"]
          lifecycle:
            type: permanent
          access:
            read: public
          variants:
            - name: thumbnail
              transform: thumbnail
              params: { width: 150, height: 150 }
            - name: large
              transform: resize
              params: { width: 1200, height: 1200 }
      tags:
        type: array<string>
        search:
          indexed: true
          strategy: exact
      rating: decimal
      review_count: integer

dataStates:
  - name: ProductCatalog
    type: Product
    lifecycle: persistent
    governance:
      classification: public

searchIntents:
  - name: ProductSearch
    version: v1
    searches: ProductCatalog
    query:
      fields: [name, description, tags, brand]
      default_operator: and
    filters:
      required: []
      optional: [category, brand, price_range, in_stock]
    facets:
      include: [category, brand, price_range, in_stock, rating]
    pagination:
      default_size: 24
      max_size: 100
      cursor_based: false
    sort:
      default: _score
      allowed: [price, rating, name, created_at]
    result:
      includes: [id, sku, name, price, image, rating, in_stock]
      highlights: [name, description]
    authorization:
      policy_set: PublicSearchPolicies
```

#### UI Layer

```yaml
presentationViews:
  - name: ProductSearchView
    version: v1
    consumes:
      dataState: ProductCatalog
      version: v1
    interactionModel:
      filters:
        - field: category
          type: enum
        - field: brand
          type: enum
        - field: price
          type: range
        - field: in_stock
          type: boolean
      sorts:
        - field: relevance
          default: desc
        - field: price
          default: asc
        - field: rating
          default: desc
      actions:
        - action: view_product
          label: "View Details"
          intent: NavigateToProduct

  - name: ProductDetailView
    version: v1
    consumes:
      dataState: ProductCatalog
      version: v1
    interactionModel:
      actions:
        - if: product.in_stock
          action: add_to_cart
          label: "Add to Cart"
          intent: AddToCart
        - action: view_reviews
          label: "Read Reviews"
          intent: NavigateToReviews

presentationCompositions:
  - name: ProductPageWorkspace
    version: v1
    primary: ProductDetailView
    navigation:
      label: "{{product.name}}"
      purpose: products
      param: product_id
    related:
      - view: ProductImagesView
        relationship: supplementary
        cardinality: many
      - view: ProductReviewsView
        relationship: detail
        cardinality: many
        navigation:
          label: "Reviews ({{review_count}})"
          scope: within_composition
      - view: RecommendedProductsView
        relationship: derived
        cardinality: many
        navigation:
          label: "Recommended"
          scope: within_composition
```

---

### Example 3: Scheduled Report Generation

This example demonstrates scheduled triggers and notification channels.

```yaml
dataTypes:
  # Report data type
  - name: SalesReport
    version: v1
    fields:
      id: uuid
      report_date: timestamp
      period: enum[daily, weekly, monthly]
      total_sales: decimal
      total_orders: integer
      top_products: array<ProductSummary>
      pdf:
        type: asset
        asset:
          category: document
          constraints:
            mime_types: ["application/pdf"]
          lifecycle:
            type: expiring
            expires_after: 90d
          access:
            read: policy
            policy: ReportAccessPolicy

dataStates:
  - name: SalesReportRecord
    type: SalesReport
    lifecycle: persistent
    governance:
      classification: confidential
      retention:
        policy: time_based
        duration: 2y
        on_expiry: archive

inputIntents:
  # Scheduled report generation
  - name: GenerateDailySalesReport
    version: v1
    proposes:
      dataState: SalesReportRecord
      version: v1
    supplies:
      required: [report_date, period]
    scheduled:
      enabled: true
      trigger:
        type: cron
        cron: "0 2 * * *"  # Daily at 2 AM
      timezone:
        mode: fixed
        fixed: "America/New_York"
      idempotency:
        key: ["daily_sales", "{{report_date}}"]
        scope: global
      on_missed:
        action: execute_immediately
    delivery:
      guarantee: exactly_once
      acknowledgment: required
      timeout: 5m

# Report generation flow
smsFlows:
  - name: GenerateSalesReport
    triggeredBy:
      inputIntent: GenerateDailySalesReport
    steps:
      - work_unit: aggregate_sales_data
        input: { date: "{{report_date}}" }
        output: sales_summary
      - work_unit: generate_pdf_report
        input: { data: "{{sales_summary}}" }
        output: pdf_url
      - work_unit: save_report
        input: { summary: "{{sales_summary}}", pdf: "{{pdf_url}}" }
      - work_unit: send_report_notification
        input: { report_id: "{{result.id}}" }
    durable:
      enabled: true
      timeout:
        total: 10m

notificationChannels:
  - name: DailySalesReportNotification
    version: v1
    channels:
      - type: email
        enabled: true
        config:
          email:
            from: "reports@company.com"
            template: daily_sales_report
      - type: webhook
        enabled: true
        config:
          webhook:
            url: "https://slack.com/api/webhooks/..."
            method: POST
    preferences:
      user_controllable: false
      defaults:
        email: enabled
        webhook: enabled
    delivery:
      retry:
        enabled: true
        max_attempts: 5
        backoff: exponential
      batching:
        enabled: false
```

---

### Example 4: Multi-Region Document Collaboration

This example demonstrates collaborative sessions, structured content, and multi-region authority.

```yaml
dataTypes:
  # Document data type
  - name: Document
    version: v1
    fields:
      id: uuid
      title: string
      owner_id: uuid
      collaborators: array<uuid>
      content:
        type: structured_content
        structured_content:
          blocks:
            allowed: [paragraph, heading, list, blockquote, code, image, table]
          marks:
            allowed: [bold, italic, underline, link, code, highlight]
          constraints:
            max_length: 100000
          sanitization:
            strip_scripts: true
      status: enum[draft, published, archived]
      created_at: timestamp
      updated_at: timestamp
      version_number: integer

dataStates:
  - name: DocumentRecord
    type: Document
    lifecycle: persistent
    mutability:
      scope: entity
      exclusive: true
      authority: DocumentAuthority
    governance:
      classification: internal
      retention:
        policy: event_based
        after_event: document_archived
        grace_period: 180d
        on_expiry: delete

authorities:
  - name: DocumentAuthority
    scope: entity
    resolution: |
      nearest_region(owner.location)
    migration:
      allowed: true
      strategy: pause-and-cutover
      triggers:
        - type: manual

experiences:
  # Document editor experience
  - name: DocumentEditor
    version: v1
    entry_point: DocumentEditWorkspace
    includes:
      - DocumentEditWorkspace
      - DocumentVersionHistory
    policy_set: DocumentEditPolicies
    session:
      required: true
      subject_binding:
        mode: authenticated
      multi_device:
        enabled: true
        sync_strategy: real_time
        conflict_resolution: last_write_wins
      offline:
        enabled: true
        cache_duration: 24h
        sync_on_reconnect: auto
    collaboration:
      enabled: true
      session:
        type: document
        scope: document.id
        max_participants: 50
      awareness:
        show_active_users: true
        show_cursors: true
        show_selections: true
        timeout: 30s
      permissions:
        mode: policy_based
        policy_set: DocumentCollaborationPolicies
      conflict_resolution:
        strategy: operational_transform

policies:
  # Collaboration policy
  - name: DocumentCollaborationPolicy
    version: v1
    appliesTo:
      type: dataState
      name: DocumentRecord
    effect: allow
    when: |
      subject.id == document.owner_id OR
      subject.id in document.collaborators OR
      subject.role == "admin"
```

---

### Example 5: IoT Edge Device with Offline Sync

This example demonstrates edge device semantics and spatial types.

```yaml
dataTypes:
  # IoT device data
  - name: IoTDevice
    version: v1
    fields:
      id: uuid
      device_id: string
      type: enum[sensor, actuator, gateway]
      location:
        type: spatial
        spatial:
          geometry: point
          coordinate_system: wgs84
          precision: 0.000001
          search:
            indexed: true
            strategy: radius
      status: enum[online, offline, maintenance]
      last_heartbeat: timestamp
      firmware_version: string

  # Sensor reading data
  - name: SensorReading
    version: v1
    fields:
      id: uuid
      device_id: uuid
      timestamp: timestamp
      temperature: decimal
      humidity: decimal
      pressure: decimal
      location:
        type: spatial
        spatial:
          geometry: point
          coordinate_system: wgs84

dataStates:
  - name: SensorReadingEvent
    type: SensorReading
    lifecycle: persistent
    storage:
      role: data
      rebuildable: true

edgeDevices:
  - name: SensorGateway
    version: v1
    capabilities:
      compute: arm64_2ghz
      storage: 64GB
      connectivity: intermittent
    sync:
      strategy: bidirectional
      frequency: periodic
      conflict_resolution: server_wins
    local_processing:
      enabled: true
      fallback_to_cloud: false
      models: [reading_validation, anomaly_detection]

materializations:
  # Materialized view for aggregated readings
  - name: DeviceMetricsView
    source: SensorReadingEvent
    targetState: DeviceMetrics
    temporal:
      window:
        type: tumbling
        size: 5m
      aggregation:
        - field: temperature
          function: avg
          as: avg_temperature
        - field: humidity
          function: avg
          as: avg_humidity
        - field: pressure
          function: avg
          as: avg_pressure
      timestamp_field: timestamp
      emit:
        trigger: on_window_close

graphQueries:
  # Spatial query for nearby devices
  - name: FindNearbyDevices
    version: v1
    start_from: IoTDevice
    traversal:
      relationships: [DeviceProximity]
      direction: both
      max_depth: 2
    filter:
      node: device.status == "online"
    return:
      nodes: [id, device_id, type, location, status]
      aggregations:
        - function: count
          field: id
          as: nearby_count
```

---

## APPENDIX F: TROUBLESHOOTING GUIDE

### Common Issues and Solutions

#### Issue 1: CAS Token Mismatch

**Symptom**: Writes failing with `ErrEpochMismatch` or `ErrVersionMismatch`.

**Cause**: CAS token from client is stale due to authority migration or concurrent writes.

**Diagnosis**:
```yaml
# Check authority state
GET /authority/{entity_id}
Response:
  entity_id: "abc-123"
  authority_region: "us-east"
  authority_epoch: 15        # Epoch increased
  entity_version: 42

# Client CAS token
cas_token:
  entity_id: "abc-123"
  authority_epoch: 14        # Stale epoch!
  entity_version: 41
```

**Solution**:
1. Fetch fresh CAS token from authority region
2. Retry write with new token
3. If repeated failures, check authority migration in progress

**Prevention**:
- Implement automatic token refresh on mismatch
- Add exponential backoff for retries
- Monitor authority migration frequency

---

#### Issue 2: View Staleness

**Symptom**: Materialized views showing outdated data.

**Cause**: View update latency exceeding `maxStaleness` threshold.

**Diagnosis**:
```yaml
materializations:
  - name: AccountBalanceView
    freshness:
      maxStaleness: 5s    # Target: 5 seconds

# Actual latency: 15 seconds
```

**Solution Options**:

**Option 1: Adjust Staleness Tolerance**
```yaml
freshness:
  maxStaleness: 30s  # Increase tolerance
```

**Option 2: Scale View Workers**
```
# Add more view materialization workers
kubectl scale deployment view-materializer --replicas=5
```

**Option 3: Optimize View Query**
```yaml
# Reduce aggregation complexity
materializations:
  - retrieval:
      mode: by_entity  # Instead of global aggregation
```

**Prevention**:
- Monitor view update latency
- Set alerts on staleness violations
- Load test view materialization

---

#### Issue 3: Atomic Group Crosses Boundaries

**Symptom**: Parser rejects flow with error "Atomic group spans multiple boundaries".

**Cause**: Work units in atomic group execute in different realms.

**Invalid Example**:
```yaml
atomic_group:
  realm: order_realm
  steps:
    - work_unit: create_order        # realm: order_realm
    - work_unit: charge_payment      # realm: payment_realm  ❌
```

**Solution**: Split into separate steps with compensation:
```yaml
steps:
  - work_unit: create_order
  - work_unit: charge_payment
    on_failure:
      compensate: cancel_order
```

**Prevention**:
- Design boundaries around transaction requirements
- Use compensation for cross-boundary operations
- Validate flows with linter before deployment

---

#### Issue 4: Policy Divergence in Shadow Mode

**Symptom**: High divergence rate between shadow and enforced policies.

**Diagnosis**:
```yaml
policyArtifact:
  lifecycle:
    state: shadow
  metrics:
    divergence_rate: 0.35  # 35% divergence! ⚠️
```

**Cause Analysis**:
- **If false positives high**: Policy too restrictive
- **If false negatives high**: Policy too permissive
- **If divergence random**: Policy logic error

**Solution**:
1. Analyze divergence patterns in audit logs
2. Identify specific cases causing divergence
3. Refine policy logic
4. Re-test in shadow mode

**Example Fix**:
```yaml
# Original (too restrictive)
when: subject.role == "admin"

# Fixed (correct logic)
when: subject.role in ["admin", "owner"] OR subject.has_permission("manage_account")
```

---

#### Issue 5: Session Sync Conflicts

**Symptom**: Multi-device sync causing data loss or conflicts.

**Cause**: Conflict resolution strategy not appropriate for use case.

**Diagnosis**:
```yaml
session:
  multi_device:
    conflict_resolution: last_write_wins  # May lose data
```

**Solution Options**:

**Option 1: Device Priority**
```yaml
conflict_resolution: device_priority
# Desktop changes win over mobile
```

**Option 2: Manual Resolution**
```yaml
conflict_resolution: manual
# User chooses which version to keep
```

**Option 3: Field-Level Merge**
```yaml
conflict_resolution: crdt
# Automatic conflict-free merge
```

**Prevention**:
- Choose strategy based on data criticality
- Test multi-device scenarios
- Monitor conflict rates

---

#### Issue 6: Search Results Not Updating

**Symptom**: Search returns stale results after data changes.

**Cause**: Search index not updating in real-time.

**Diagnosis**:
```
# Check search index lag
GET /search/status
Response:
  last_indexed_at: "2024-01-15T10:00:00Z"
  current_time: "2024-01-15T10:05:00Z"
  lag: 300s  # 5 minute lag
```

**Solution**:

**Immediate**: Force reindex
```
POST /search/reindex
{ "entity_type": "Product" }
```

**Long-term**: Optimize indexing
```yaml
dataTypes:
  - fields:
      name:
        search:
          indexed: true
          # Add indexing strategy
          update: real_time | batch | periodic
          batch_size: 1000
          batch_window: 30s
```

---

#### Issue 7: Durable Workflow Stuck

**Symptom**: Workflow not progressing, stuck at human task.

**Diagnosis**:
```yaml
workflow_status:
  workflow_id: "wf-123"
  current_step: "underwriter_review"
  status: waiting_for_human
  timeout: 48h
  elapsed: 72h  # Past timeout! ⚠️
```

**Cause**: Human task timeout exceeded without escalation.

**Solution**:

**Immediate**: Manual intervention
```
POST /workflows/wf-123/complete_step
{
  "step": "underwriter_review",
  "result": "approved",
  "reason": "timeout_recovery"
}
```

**Long-term**: Add escalation
```yaml
human_tasks:
  - step: underwriter_review
    timeout: 48h
    escalation:
      after: 24h
      to: senior_underwriter
      action: reassign
```

---

#### Issue 8: Asset Upload Failing

**Symptom**: Asset uploads timing out or failing validation.

**Diagnosis**:
```yaml
assets:
  - constraints:
      max_size: 5MB
  
# Upload attempt
file_size: 12MB  # Exceeds limit ❌
```

**Solutions**:

**Solution 1: Increase limit (if appropriate)**
```yaml
constraints:
  max_size: 20MB
```

**Solution 2: Client-side compression**
```javascript
// Compress before upload
const compressed = await compressImage(file, { quality: 0.8 });
```

**Solution 3: Progressive upload**
```yaml
assets:
  - upload:
      strategy: chunked
      chunk_size: 1MB
      max_chunks: 20
```

---

#### Issue 9: Notification Not Delivered

**Symptom**: Users not receiving notifications.

**Diagnosis**:
```yaml
notification_logs:
  - channel: push
    status: failed
    reason: "Device token invalid"
  - channel: email
    status: delivered
```

**Cause**: Push notification device token expired.

**Solution**: Implement fallback
```yaml
notificationChannels:
  - delivery:
      fallback:
        enabled: true
        order: [push, email, sms]
      retry:
        enabled: true
        max_attempts: 3
```

**Prevention**:
- Refresh device tokens periodically
- Monitor notification success rates
- Test all channels regularly

---

#### Issue 10: Authority Migration Taking Too Long

**Symptom**: Write pause during authority migration exceeds target.

**Diagnosis**:
```
Authority migration for entity abc-123:
  Target pause: 100ms
  Actual pause: 2500ms  ⚠️
  
Bottleneck: Network latency between regions
```

**Solutions**:

**Solution 1: Optimize network path**
```
# Use dedicated private links between regions
AWS PrivateLink / Azure Private Link / GCP Private Service Connect
```

**Solution 2: Reduce migration frequency**
```yaml
authorities:
  - migration:
      triggers:
        - type: latency
          condition: p99_latency > 500ms  # Increase threshold
          cooldown: 1h  # Add cooldown period
```

**Solution 3: Pre-warm target region**
```yaml
migration:
  strategy: pause-and-cutover
  pre_migration:
    warm_cache: true
    sync_hot_data: true
```

---

## APPENDIX G: REFERENCE IMPLEMENTATION GUIDANCE

### Runtime Architecture

A conformant SMS v1.6 runtime SHOULD implement the following architecture:

```
┌─────────────────────────────────────────────────────────────┐
│                      Control Plane                          │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ Policy      │  │ Worker       │  │ Authority    │      │
│  │ Distribution│  │ Registry     │  │ Management   │      │
│  └─────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                            │
                   ┌────────┴────────┐
                   │  NATS / Message │
                   │     Fabric      │
                   └────────┬────────┘
                            │
┌────────────────────────────┴──────────────────────────────┐
│                      Data Plane                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
│  │ Workers  │  │ Flow     │  │ View     │  │ Policy   │ │
│  │          │  │ Engine   │  │ Material │  │ Enforcer │ │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘ │
└────────────────────────────────────────────────────────────┘
                            │
                   ┌────────┴────────┐
                   │  Storage Layer  │
                   │  ┌───┐  ┌───┐  │
                   │  │KV │  │JS │  │
                   │  └───┘  └───┘  │
                   └─────────────────┘
```

#### Component Responsibilities

**Control Plane**:
- Policy distribution to all workers
- Worker discovery and health monitoring
- Authority state management
- Topology configuration

**Workers**:
- Execute work units
- Local policy enforcement
- Emit operational signals
- Maintain routing cache

**Flow Engine**:
- Parse and execute SMS flows
- Atomic group coordination
- Compensation orchestration
- Idempotency checking

**View Materializer**:
- Subscribe to source events
- Compute derived views
- Maintain freshness SLAs
- Handle temporal windows

**Policy Enforcer**:
- Cache policy definitions
- Evaluate policies locally
- Record audit logs
- Support shadow mode

**Storage Layer**:
- Authority KV: Strong consistency, small volume
- JetStream: Event streams, high throughput
- Object Storage: Assets and large documents

---

### Implementation Checklist

#### Phase 1: Core Grammar (Weeks 1-2)
- [ ] Parse YAML specifications
- [ ] Validate structural rules
- [ ] Validate semantic rules
- [ ] Generate types from DataType
- [ ] Support all primitive types
- [ ] Support complex types (enum, array, reference)

#### Phase 2: Execution (Weeks 3-4)
- [ ] Implement flow engine (FSM)
- [ ] Execute work units
- [ ] Enforce atomic groups
- [ ] Validate constraints
- [ ] Implement idempotency checking
- [ ] Handle execution context propagation

#### Phase 3: Policy (Weeks 5-6)
- [ ] Load policies from control plane
- [ ] Evaluate policies locally
- [ ] Support shadow policies
- [ ] Handle lifecycle transitions
- [ ] Implement audit logging
- [ ] Support policy versioning

#### Phase 4: Distribution (Weeks 7-8)
- [ ] Integrate NATS (or alternative)
- [ ] Worker registration/discovery
- [ ] Heartbeat loop
- [ ] Routing cache
- [ ] Signal emission
- [ ] TTL-based expiry

#### Phase 5: Authority & Multi-Region (Weeks 9-12)
- [ ] Entity-scoped authority resolution
- [ ] CAS token validation
- [ ] Authority migration (pause-and-cutover)
- [ ] Cross-region replication
- [ ] View authority independence
- [ ] Regional failover

#### Phase 6: UI Composition (Weeks 13-14)
- [ ] Parse compositions and experiences
- [ ] Build navigation hierarchy
- [ ] Resolve related views
- [ ] Apply field permissions
- [ ] Evaluate indicators
- [ ] Route experiences

#### Phase 7: Core Features (Weeks 15-18)
- [ ] Form-intent binding with validation
- [ ] View materialization contracts (retrieval modes)
- [ ] Streaming views (temporal windows)
- [ ] Intent delivery guarantees
- [ ] Session management with sync
- [ ] Multi-device conflict resolution
- [ ] Offline support
- [ ] Asset upload and transformation
- [ ] Asset variants generation
- [ ] CDN integration for assets

#### Phase 8: Search & Discovery (Weeks 19-20)
- [ ] Search index configuration
- [ ] Full-text search implementation
- [ ] Fuzzy matching
- [ ] Faceted search
- [ ] Result highlighting
- [ ] Search authorization
- [ ] Index freshness management

#### Phase 9: Automation (Weeks 21-22)
- [ ] Scheduled trigger execution
- [ ] Cron expression parsing
- [ ] Timezone handling
- [ ] Missed execution handling
- [ ] Multi-channel notifications
- [ ] Notification templates
- [ ] Delivery tracking
- [ ] User preferences
- [ ] Quiet hours

#### Phase 10: Advanced Features (Weeks 23-26)
- [ ] Collaborative session handling
- [ ] Presence awareness
- [ ] Operational Transform / CRDT
- [ ] Durable workflow orchestration
- [ ] Workflow checkpoints
- [ ] Compensation logic
- [ ] Human task management
- [ ] Delegation and consent management
- [ ] Conversational UI routing
- [ ] External API integration
- [ ] Circuit breakers
- [ ] Rate limiting

#### Phase 11: Governance & Compliance (Weeks 27-28)
- [ ] Data classification enforcement
- [ ] Retention policy automation
- [ ] Legal hold management
- [ ] Right-to-erasure support
- [ ] Data lineage tracking
- [ ] Audit log immutability
- [ ] Encryption at rest/in transit
- [ ] Regional residency enforcement

#### Phase 12: Edge & Advanced (Weeks 29-30)
- [ ] Edge device support
- [ ] Offline operation
- [ ] Sync strategies
- [ ] Spatial query support
- [ ] GIS indexing
- [ ] Graph query execution
- [ ] Relationship traversal
- [ ] Structured content rendering
- [ ] ML inference integration

#### Phase 13: v1.6 Features (Weeks 31-32)
- [ ] Storage Binding implementation
- [ ] Cache Layer (local/distributed/hybrid)
- [ ] Intent Routing with transports
- [ ] Resource Requirements provisioning
- [ ] Client Cache with offline support
- [ ] Cache invalidation strategies
- [ ] Observability integration

#### Phase 14: Production Readiness (Weeks 33-36)
- [ ] Observability hooks
- [ ] Metrics collection
- [ ] Distributed tracing
- [ ] Chaos testing
- [ ] Performance optimization
- [ ] Load testing
- [ ] Security audit
- [ ] Compliance validation
- [ ] Documentation
- [ ] Operational runbooks

---

### Testing Strategies

#### Unit Testing

**Grammar Parsing**:
```go
func TestParseDataType(t *testing.T) {
    spec := `
dataTypes:
  - name: Order
    version: v1
    fields:
      id: uuid
      total: decimal
`
    dt, err := parser.ParseDataType(spec)
    assert.NoError(t, err)
    assert.Equal(t, "Order", dt.Name)
    assert.Equal(t, "v1", dt.Version)
    assert.Len(t, dt.Fields, 2)
}
```

**Policy Evaluation**:
```go
func TestPolicyEvaluation(t *testing.T) {
    policy := Policy{
        Effect: "allow",
        When:   "subject.role == 'admin'",
    }
    
    ctx := ExecutionContext{
        Actor: Actor{
            Subject: "user-123",
            Roles:   []string{"admin"},
        },
    }
    
    decision := evaluatePolicy(policy, ctx)
    assert.Equal(t, Allow, decision)
}
```

**Idempotency**:
```go
func TestIdempotency(t *testing.T) {
    checker := NewIdempotencyChecker()
    
    key := "transfer-123"
    result := Data{"status": "success"}
    
    // First execution
    seen, _ := checker.Check(key)
    assert.False(t, seen)
    checker.Mark(key, result, 24*time.Hour)
    
    // Second execution (duplicate)
    seen, _ = checker.Check(key)
    assert.True(t, seen)
    cached, _ := checker.GetResult(key)
    assert.Equal(t, result, cached)
}
```

#### Integration Testing

**End-to-End Flow**:
```go
func TestOrderProcessingFlow(t *testing.T) {
    // Setup
    runtime := NewSMSRuntime()
    runtime.LoadSpec("order-spec.yaml")
    
    // Execute flow
    intent := InputIntent{
        Name: "CreateOrder",
        Data: map[string]interface{}{
            "customer_id": "cust-123",
            "items": []OrderItem{
                {SKU: "prod-1", Quantity: 2},
            },
        },
    }
    
    result, err := runtime.ExecuteFlow("ProcessOrder", intent)
    
    // Verify
    assert.NoError(t, err)
    assert.Equal(t, "success", result.Status)
    
    // Check side effects
    order := runtime.GetEntity("OrderRecord", result.OrderID)
    assert.NotNil(t, order)
    assert.Equal(t, "confirmed", order.Status)
}
```

**Multi-Region Authority**:
```go
func TestAuthorityMigration(t *testing.T) {
    // Setup two regions
    usEast := NewRegion("us-east")
    euWest := NewRegion("eu-west")
    
    // Create entity in us-east
    entity := CreateEntity("Account", "acc-123", usEast)
    assert.Equal(t, "us-east", entity.AuthorityRegion)
    assert.Equal(t, 1, entity.AuthorityEpoch)
    
    // Migrate to eu-west
    MigrateAuthority(entity.ID, "eu-west")
    
    // Verify migration
    entity = GetEntity("acc-123")
    assert.Equal(t, "eu-west", entity.AuthorityRegion)
    assert.Equal(t, 2, entity.AuthorityEpoch)
    
    // Old CAS token should fail
    oldToken := CASToken{
        EntityID:        "acc-123",
        AuthorityEpoch:  1,
        AuthorityRegion: "us-east",
    }
    err := WriteWithCAS(entity.ID, data, oldToken)
    assert.Error(t, err)
    assert.Equal(t, ErrEpochMismatch, err)
}
```

#### Chaos Testing

**Network Partition**:
```go
func TestNetworkPartition(t *testing.T) {
    // Start system
    system := StartDistributedSystem()
    
    // Generate load
    loadGen := StartLoadGenerator(100) // 100 req/s
    
    // Partition network
    chaos.PartitionRegion("us-east", 30*time.Second)
    
    // Verify:
    // 1. US entities become unavailable
    // 2. EU entities remain operational
    // 3. No data corruption
    // 4. System recovers after partition heals
    
    metrics := system.GetMetrics()
    assert.Greater(t, metrics.Availability, 0.7) // 70%+ available
    assert.Equal(t, 0, metrics.DataCorruption)
}
```

**Random Worker Failures**:
```go
func TestWorkerFailures(t *testing.T) {
    system := StartDistributedSystem()
    loadGen := StartLoadGenerator(50)
    
    // Kill 20% of workers randomly
    chaos.KillRandomWorkers(0.2, 60*time.Second)
    
    // Verify:
    // 1. System continues operating
    // 2. No work is lost
    // 3. Idempotency prevents duplicates
    
    results := loadGen.GetResults()
    assert.Equal(t, 0, results.LostRequests)
    assert.Equal(t, 0, results.DuplicateProcessing)
}
```

---

### Performance Benchmarks

#### Target Metrics

| Component | Operation | Target | Notes |
|-----------|-----------|--------|-------|
| Parser | Parse 1KB spec | <10ms | Grammar validation |
| Policy | Evaluate simple | <100μs | Local evaluation |
| Policy | Evaluate complex | <1ms | With attribute lookup |
| Flow | Execute simple (3 steps) | <50ms | Local execution |
| Flow | Execute distributed (5 steps) | <200ms | Cross-boundary |
| View | Materialize simple | <500ms | Single source |
| View | Materialize complex | <5s | Multiple sources, aggregation |
| Authority | CAS validation | <1ms | Token verification |
| Authority | Migration | <100ms | Write pause duration |
| Storage (Control) | Read | <5ms | KV lookup |
| Storage (Control) | Write | <10ms | KV update |
| Storage (Data) | Append | <10ms | Event stream |
| Storage (Data) | Read | <20ms | Event replay |
| Search | Simple query | <50ms | Single field |
| Search | Complex query | <200ms | Multiple fields, facets |
| Asset | Upload (1MB) | <2s | Including validation |
| Asset | Download (1MB) | <1s | CDN delivery |
| Notification | Send (push) | <500ms | FCM/APNS |
| Notification | Send (email) | <2s | SMTP delivery |
| Cache (local) | Read | <1ms | L1 cache hit |
| Cache (distributed) | Read | <10ms | L2 cache hit |
| Intent Routing | NATS | <15ms | Message routing |
| Intent Routing | HTTP | <50ms | REST API call |

#### Benchmark Tests

```go
func BenchmarkPolicyEvaluation(b *testing.B) {
    policy := LoadPolicy("test-policy")
    ctx := LoadContext("test-context")
    
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        evaluatePolicy(policy, ctx)
    }
}

func BenchmarkViewMaterialization(b *testing.B) {
    events := LoadEvents("test-events", 1000)
    
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        materializeView("TestView", events)
    }
}

func BenchmarkCASValidation(b *testing.B) {
    token := CASToken{...}
    entity := LoadEntity("test-entity")
    
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        validateCASToken(token, entity)
    }
}

func BenchmarkCacheRead(b *testing.B) {
    cache := NewCache("test-cache")
    key := "test-key"
    cache.Set(key, "test-value")
    
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        cache.Get(key)
    }
}
```

---

## APPENDIX H: ADVANCED PATTERNS & ANTI-PATTERNS

### Pattern: Idempotent State Machines

**Problem**: State transitions must be idempotent but still enforce ordering.

**Solution**:
```yaml
transitions:
  - name: ShipOrder
    from: confirmed
    to: shipped
    guarantees:
      idempotency:
        key: ["order_id", "ship"]
        scope: global
        ttl: 7d
    validation:
      pre_condition: |
        order.status == "confirmed" OR order.status == "shipped"
      post_condition: |
        order.status == "shipped"
```

**Why It Works**:
- Pre-condition accepts both "confirmed" (first execution) and "shipped" (retry)
- Post-condition ensures final state is correct
- Idempotency key prevents double-processing
- TTL allows key cleanup after reasonable period

---

### Pattern: Compensating Transactions Without Distributed Locks

**Problem**: Need to undo operations across boundaries without 2PC.

**Solution**:
```yaml
smsFlows:
  - name: TransferWithCompensation
    steps:
      - work_unit: debit_account
        output: debit_id
        compensate: refund_account
        compensation_input: { debit_id: "{{debit_id}}" }
      
      - work_unit: credit_account
        output: credit_id
        compensate: reverse_credit
        compensation_input: { credit_id: "{{credit_id}}" }
        on_failure:
          execute: [reverse_credit, refund_account]
          order: reverse  # Undo in reverse order
```

**Key Principles**:
1. Each step stores compensation handle (ID)
2. Compensation is idempotent
3. Failure triggers reverse-order unwinding
4. Each compensation is best-effort (may fail)

---

### Pattern: View Composition for Complex Queries

**Problem**: Need to query across multiple entities efficiently.

**Solution**:
```yaml
materializations:
  # Base views
  - name: CustomerView
    source: CustomerEvent
    targetState: Customer
    retrieval:
      mode: by_entity
      entity_key: customer_id

  - name: OrderView
    source: OrderEvent
    targetState: Order
    retrieval:
      mode: by_entity
      entity_key: order_id

  # Composed view
  - name: CustomerOrderSummary
    sources:
      - view: CustomerView
        join_key: customer_id
      - view: OrderView
        join_key: customer_id
    targetState: CustomerOrderSummary
    aggregation:
      - field: order.total
        function: sum
        as: total_spent
      - field: order.id
        function: count
        as: order_count
    retrieval:
      mode: by_entity
      entity_key: customer_id
```

**Benefits**:
- Reuses existing views
- Declarative join and aggregation
- Maintains freshness SLAs
- Scales horizontally

---

### Anti-Pattern: Embedding Business Logic in Policies

**Problem**: Policies with complex business rules become unmaintainable.

**Bad**:
```yaml
policies:
  - when: |
      (subject.role == "manager" AND account.balance < 10000) OR
      (subject.role == "director" AND account.balance < 50000) OR
      (subject.role == "vp" AND account.balance < 100000) OR
      subject.role == "cfo"
```

**Good**:
```yaml
policies:
  # Define permission
  - when: subject.has_permission("approve_transfer")

# Business logic in work unit
work_unit:
  name: ApproveTransfer
  validation:
    pre_condition: |
      determine_approver(transfer.amount, subject.role)
```

**Why**:
- Policies focus on who, not what
- Business rules in appropriate layer
- Easier to test and evolve

---

### Anti-Pattern: Over-Normalized Data Model

**Problem**: Too many entities with complex relationships.

**Bad**:
```yaml
dataTypes:
  - name: Address
    # Separate entity for address

  - name: PhoneNumber
    # Separate entity for phone

  - name: Customer
    fields:
      address_id: uuid
      phone_id: uuid
```

**Good**:
```yaml
dataTypes:
  - name: Customer
    fields:
      address:
        street: string
        city: string
        postal_code: string
      phone: string
```

**Why**:
- Embedded values are simpler
- Fewer joins required
- Better query performance
- Only separate when independent lifecycle

---

### Anti-Pattern: Synchronous Cross-Region Calls

**Problem**: Synchronous calls across regions add latency.

**Bad**:
```yaml
smsFlows:
  - steps:
      - work_unit: process_order        # us-east
      - work_unit: notify_warehouse     # eu-west (synchronous)
```

**Good**:
```yaml
smsFlows:
  - steps:
      - work_unit: process_order        # us-east
      - emit:
          signal: OrderProcessed
      # Warehouse subscribes asynchronously

# Separate flow in eu-west
smsFlows:
  - triggeredBy:
      signal: OrderProcessed
    steps:
      - work_unit: notify_warehouse
```

**Why**:
- Asynchronous = lower latency
- Region failures don't block
- Better scalability

---

### Anti-Pattern: Stale View in Critical Path

**Problem**: Using eventual view for strong consistency requirement.

**Bad**:
```yaml
materializations:
  # View with 5-second staleness
  - name: InventoryView
    freshness:
      maxStaleness: 5s

# Flow using view for critical decision
smsFlows:
  - steps:
      - work_unit: check_inventory_view  # May be stale!
      - work_unit: reserve_item
```

**Good**:
```yaml
# Read authoritative source for critical decisions
smsFlows:
  - steps:
      - work_unit: check_inventory_authoritative
      - work_unit: reserve_item
```

**Why**:
- Strong consistency where needed
- Views for queries, not critical decisions
- Accept staleness only where safe

---

### Anti-Pattern: Unbounded Temporal Windows

**Problem**: Windows without bounds consume infinite memory.

**Bad**:
```yaml
temporal:
  window:
    type: session
    # No timeout specified!
```

**Good**:
```yaml
temporal:
  window:
    type: session
    gap: 5m          # Max 5-minute idle
    max_duration: 4h # Max 4-hour session
```

**Why**:
- Prevents memory leaks
- Ensures windows close
- Predictable resource usage

---

## Appendix D: Performance Optimization

### Optimization Patterns

#### Pattern 1: View Caching Strategy

**Goal**: Reduce view computation latency.

**Strategy**:
```yaml
materializations:
  - name: CustomerDashboard
    freshness:
      maxStaleness: 5m
    retrieval:
      mode: by_entity
      entity_key: customer_id
      on_unavailable:
        strategy: stale_while_revalidate
        cache_ttl: 1h  # Serve stale while updating
```

**Results**:
- P50 latency: 5ms (cache hit)
- P99 latency: 250ms (cache miss)
- Cache hit rate: 95%

---

#### Pattern 2: Composition Prefetching

**Goal**: Reduce composition load time.

**Strategy**:
```yaml
presentationCompositions:
  - name: OrderDetailsWorkspace
    fetch:
      order: all_parallel  # Load all views simultaneously
      prefetch:
        enabled: true
        trigger: on_hover  # Prefetch when user hovers over link
      cache:
        strategy: stale_while_revalidate
        ttl: 30s
```

**Results**:
- Without prefetch: 800ms load time
- With prefetch: 150ms load time (85% improvement)

---

#### Pattern 3: Batch Notification Delivery

**Goal**: Reduce notification overhead.

**Strategy**:
```yaml
notificationChannels:
  - name: ActivityDigest
    delivery:
      batching:
        enabled: true
        window: 5m  # Collect notifications for 5 minutes
        max_batch_size: 50
```

**Results**:
- Individual sends: 1000 API calls/hour
- Batched sends: 20 API calls/hour (98% reduction)

---

#### Pattern 4: Search Result Caching

**Goal**: Improve search performance.

**Strategy**:
```yaml
searchIntents:
  - name: ProductSearch
    cache:
      enabled: true
      ttl: 5m
      key_fields: [query, filters, sort]  # Cache key
      invalidate_on: product_update
```

**Results**:
- Cold search: 250ms
- Cached search: 15ms (94% faster)
- Cache hit rate: 70%

---

#### Pattern 5: Incremental View Updates

**Goal**: Reduce view rebuild time.

**Strategy**:
```yaml
materializations:
  - name: SalesMetrics
    evolution:
      strategy: incremental  # Not rebuild
      blocking: false
    temporal:
      window:
        type: sliding
        size: 1h
        slide: 1m  # Update every minute
```

**Results**:
- Full rebuild: 10 minutes
- Incremental update: 5 seconds (120x faster)

---

#### Pattern 6: v1.6 Multi-Tier Caching

**Goal**: Optimize cache performance with hybrid topology.

**Strategy**:
```yaml
cacheLayers:
  - name: EntityCache
    version: v1
    topology:
      tier: hybrid
      local:
        max_entries: 10000
        ttl: 2m
        eviction: lru
      distributed:
        resource: EntityCacheStore
        ttl: 30m
      hybrid:
        promotion: on_miss
    invalidation:
      strategy: hybrid
      ttl: 30m
      watch_keys:
        - "entity.{{entity_id}}"
      propagate: true
```

**Results**:
- L1 hit: <10μs
- L2 hit: <5ms
- Combined hit rate: 98%

---

## Appendix E: Security Best Practices

### Security Patterns

#### Pattern 1: Field-Level Encryption

**Goal**: Protect sensitive data at rest.

**Implementation**:
```yaml
dataStates:
  - name: CustomerPII
    governance:
      encryption:
        at_rest: required
        in_transit: required
        key_rotation: 90d
      classification: pii

presentationViews:
  # Field-level masking
  - fieldPermissions:
      - field: ssn
        mask:
          unless: subject.role == "compliance_officer"
          transform: hash
      - field: credit_card
        mask:
          unless: subject.role == "payment_processor"
          transform: reveal_last(4)
          context:
            log: redact  # Never log full number
```

---

#### Pattern 2: Audit Trail

**Goal**: Complete audit trail for compliance.

**Implementation**:
```yaml
dataStates:
  - governance:
      audit:
        access_logging: true
        modification_logging: true
        retention_for_logs: 7y
        immutable: true

policies:
  - name: AuditAccessPolicy
    appliesTo:
      type: dataState
      name: FinancialRecord
    effect: allow
    when: subject.role in ["auditor", "compliance", "admin"]
    audit:
      log_access: true
      log_denial: true
      log_level: detailed
```

**Audit Log Format**:
```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "trace_id": "trace-abc-123",
  "actor": {
    "subject_id": "user-789",
    "roles": ["customer"],
    "ip": "192.168.1.100"
  },
  "action": "read",
  "resource": {
    "type": "dataState",
    "name": "AccountRecord",
    "entity_id": "account-456"
  },
  "decision": "allow",
  "policy": "ViewBalancePolicy_v1",
  "fields_accessed": ["balance", "account_number"],
  "fields_masked": ["account_number"]
}
```

---

#### Pattern 3: Rate Limiting

**Goal**: Prevent abuse and DoS attacks.

**Implementation**:
```yaml
inputIntents:
  - name: TransferFunds
    rate_limit:
      enabled: true
      strategy: sliding_window
      limit: 10
      window: 1m
      scope: per_subject
      on_exceeded:
        action: reject
        message: "Rate limit exceeded. Try again in {{retry_after}} seconds"

externalDependencies:
  - name: PaymentGateway
    reliability:
      rate_limit:
        requests_per_second: 100
        burst: 150
      circuit_breaker:
        enabled: true
        failure_threshold: 5
        reset_timeout: 60s
```

---

#### Pattern 4: Delegation with Time Limits

**Goal**: Temporary access with automatic expiry.

**Implementation**:
```yaml
delegations:
  - name: TemporaryAccountantAccess
    delegate:
      from: account_owner
      to: accountant
      scope:
        actions: [view_balance, view_transactions, export_statement]
        data: [AccountRecord, TransactionRecord]
    consent:
      required: true
      type: explicit
      expires_after: 7d  # Automatic expiry
      revocable: true
      audit:
        log_access: true
        log_revocation: true

policies:
  - name: AutoRevokeDelegation
    appliesTo:
      type: delegation
      name: TemporaryAccountantAccess
    effect: deny
    when: |
      current_time > delegation.granted_at + delegation.expires_after OR
      delegation.revoked == true
```

---

#### Pattern 5: v1.6 Client-Side Security

**Goal**: Secure client cache with encryption.

**Implementation**:
```yaml
experiences:
  - name: BankingApp
    version: v1
    clientCache:
      enabled: true
      storage:
        type: persistent
        max_size: 50MB
        encryption: true  # Required for sensitive data
      strategies:
        primary:
          strategy: cache_first
          ttl: 5m
      offline:
        enabled: true
        required_views: [AccountSummary]
        sync_on_reconnect: true
        conflict_resolution: server_wins
      version_policy:
        on_upgrade: invalidate_all  # Security: don't trust old cache format
```

---

## Appendix F: Glossary

### Core Terms

**Addendum**: Extension to core grammar filling semantic gaps. All addendums integrated inline in v1.5+.

**Asset**: Binary content (files, images, media) as first-class data type with lifecycle and transformation (v1.5+).

**Atomic Group**: Set of work units that MUST execute together in same boundary. Failure of any fails all.

**Authority**: Right to accept writes for an entity. Resolved per entity in v1.3+.

**Authority Epoch**: Monotonically increasing number tracking authority changes. Part of CAS token.

**Authority Migration**: Transfer of write authority between regions using pause-and-cutover strategy.

**Boundary**: Execution domain where atomic groups execute. Typically process, container, or node.

**Cache Layer**: Multi-tier caching infrastructure with local, distributed, or hybrid topologies (v1.6+).

**CAS Token**: Compare-And-Set token ensuring linearizable writes across authority migrations.

**Client Cache**: Browser/mobile caching contract with offline support and sync strategies (v1.6+).

**Composition**: Semantic grouping of views into workspace (v1.4+).

**Constraint**: Semantic truth about data. Separate from structure for independent evolution.

**Control Storage**: Strong-consistency storage for coordination (authority KV, policies). Distinct from data storage.

**Data Storage**: Append-only or rebuildable storage for domain events. Distinct from control storage.

**DataState**: Lifecycle, enforcement, and evolution policy for data.

**DataType**: Structural definition without behavior or lifecycle.

**Delegation**: Temporary authority transfer with time limits, scope restrictions, and audit (v1.5+).

**Durable Workflow**: Long-running process with checkpoints, compensation, and human tasks (v1.5+).

**Entity-Scoped Authority**: Authority resolved per entity instance, not per model (v1.3+).

**Experience**: Application-level grouping of compositions for user journey (v1.4+).

**Flow**: SMS execution graph describing ordering and dependencies, not transport.

**Form Intent Binding**: Declarative mapping between UI forms and intents (v1.5+).

**Governance**: Classification, retention, residency, encryption policies for compliance (v1.5+).

**Graph Query**: Relationship traversal for social graphs and hierarchies (v1.5+).

**Idempotency**: Property that executing same operation multiple times produces same result. Required for all transitions.

**Indicator**: Dynamic badge on navigation showing state (count, boolean, value) (v1.4+).

**Inference**: ML model output as declarative field (v1.5+).

**InputIntent**: What user/system may propose before full validation. Inverse of data intent.

**Intent Routing**: Transport configuration for intent ingress points (v1.6+).

**Invariant**: Constraint that MUST always hold. Cross-entity invariants enforced by views (v1.3+).

**Materialization**: Derived, regenerable view. Authority-agnostic and epoch-tolerant by default (v1.3+).

**Mutability**: Write authority semantics for data. Scope: entity or append-only (v1.3+).

**Notification Channel**: Multi-channel delivery (email, SMS, push, webhook) with preferences (v1.5+).

**Pause-and-Cutover**: Authority migration strategy. Writes pause briefly during epoch increment and region change.

**Policy**: Declarative authorization with lifecycle (shadow/enforce/deprecated/revoked).

**Presentation View**: UI contract bound to data. Not implementation.

**Relationship**: Semantic linkage between entities with cardinality and causality (v1.3+).

**Resource Requirements**: Abstract infrastructure specification (event logs, entity stores, object stores) (v1.6+).

**Scheduled Trigger**: Cron-style temporal event generation (v1.5+).

**Search**: Full-text, fuzzy, or semantic search with faceting and ranking (v1.5+).

**Session**: Multi-device, offline-capable session state (v1.5+).

**Shadow Mode**: Test policies/writes/UI with real traffic without affecting outcomes.

**Signal**: Operational observation informing scheduling. Not command.

**Spatial Type**: Geographic/GIS data with indexing and queries (v1.5+).

**Storage Binding**: Declarative storage configuration for persistence (v1.6+).

**Storage Role**: Distinction between control (coordination) and data (domain) storage (v1.3+).

**Structured Content**: Rich text/document with semantic structure (v1.5+).

**Temporal Window**: Time-based aggregation for streaming views (tumbling, sliding, session, hopping) (v1.5+).

**Transition**: Typed edge between DataStates with explicit triggers and guarantees.

**View Authority Independence**: Views tolerate events from multiple regions and epochs (v1.3+).

**Work Unit**: Atomic piece of work executed by worker. Has contract defining input/output/side-effects.

**Worker**: Process that accepts inputs, produces outputs, executes work units.

---

### Acronyms

**ABAC**: Attribute-Based Access Control

**API**: Application Programming Interface

**CAS**: Compare-And-Set

**CDN**: Content Delivery Network

**CRDT**: Conflict-Free Replicated Data Type

**DST**: Daylight Saving Time

**EBNF**: Extended Backus-Naur Form

**GDPR**: General Data Protection Regulation (EU)

**GIS**: Geographic Information System

**HIPAA**: Health Insurance Portability and Accountability Act (US)

**HTTP**: Hypertext Transfer Protocol

**IoT**: Internet of Things

**JWT**: JSON Web Token

**KV**: Key-Value (storage)

**LFU**: Least Frequently Used (cache eviction)

**LRU**: Least Recently Used (cache eviction)

**ML**: Machine Learning

**NATS**: Neural Autonomic Transport System (messaging)

**NLU**: Natural Language Understanding

**OT**: Operational Transform

**PCI**: Payment Card Industry

**PDF**: Portable Document Format

**PII**: Personally Identifiable Information

**RBAC**: Role-Based Access Control

**REST**: Representational State Transfer

**RFC**: Request for Comments

**SDK**: Software Development Kit

**SLA**: Service Level Agreement

**SMS**: System Mechanics Specification / State Machine System

**SQL**: Structured Query Language

**SSN**: Social Security Number

**TTL**: Time To Live

**UI**: User Interface

**UUID**: Universally Unique Identifier

**WCAG**: Web Content Accessibility Guidelines

**WTS**: Worker Topology Spec

**YAML**: YAML Ain't Markup Language

---

## Appendix G: FAQ

### General Questions

**Q: Is SMS v1.6 backward compatible with v1.5?**

A: Yes, completely. All v1.5 specifications remain valid in v1.6. v1.6 adds new optional features (storage binding, caching, routing, resource requirements, client cache) without breaking changes.

---

**Q: Do I need to use all v1.6 features?**

A: No. v1.6 features are additive and optional. Use only what your application needs. A valid v1.6 spec might use none of the new features.

---

**Q: Can I migrate from v1.4 directly to v1.6?**

A: Yes. v1.6 is backward compatible with v1.3, v1.4, and v1.5. Review new features to see what applies to your use case.

---

**Q: Is NATS required?**

A: No. NATS is used in reference implementation for control plane and coordination, but grammar is transport-agnostic. You can use HTTP, gRPC, Kafka, or custom transports. v1.6's intent routing makes this even more explicit.

---

**Q: Can I use SMS for monoliths?**

A: Yes. Same grammar works for monoliths, modular monoliths, microservices, serverless, and hybrid. Grammar describes intent, not deployment topology.

---

### Authority & Multi-Region Questions

**Q: Why entity-scoped authority instead of model-scoped?**

A: Entity-scoped authority provides granular failure isolation. One entity's region failure doesn't affect other entities. Model-scoped would collapse availability for entire model.

---

**Q: What happens during authority migration?**

A: Brief write pause (<100ms typically) while:
1. Current region pauses writes
2. Authority epoch increments
3. New region becomes authority
4. Writes resume in new region

Reads unaffected throughout.

---

**Q: Can authority migrate automatically?**

A: Yes, based on declared triggers (load, latency, time). But migration can be constrained by governance policies.

---

**Q: What if migration fails mid-way?**

A: System resolves deterministically to single authority. Either:
- Old region retains authority (migration aborted)
- New region gains authority (migration completed)

No split-brain possible.

---

### v1.6 Storage & Caching Questions

**Q: When should I use KV vs stream vs object_store?**

A: 
- **KV**: Entity state, configuration, small frequently-accessed data
- **Stream**: Event logs, audit trails, ordered messages
- **Object Store**: Large files, assets, media

---

**Q: What's the difference between local and distributed cache?**

A:
- **Local**: In-process, <10μs latency, lost on restart
- **Distributed**: Shared across instances, <5ms latency, survives restarts
- **Hybrid**: Best of both with L1 local + L2 distributed

---

**Q: How does cache invalidation work?**

A:
- **TTL**: Time-based expiry (simple, predictable)
- **Watch**: Real-time via pub/sub (complex, consistent)
- **Hybrid**: TTL backstop + watch for real-time

---

**Q: Should I use strong or eventual consistency?**

A: 
- **Strong**: Critical writes (payments, inventory)
- **Eventual**: Reads, analytics, views
- Default to eventual where acceptable, use strong only when necessary

---

### Data & Views Questions

**Q: Why are invariants in views, not entities?**

A: Cross-entity invariants in entities would require distributed transactions. Views enforce invariants asynchronously without blocking writes.

---

**Q: What if view update is too slow?**

A: Adjust `maxStaleness` tolerance, scale view workers, or optimize view query. Trade-off between consistency and performance.

---

**Q: Can views query across regions?**

A: Yes. Views are authority-agnostic by default. They tolerate and merge events from multiple regions.

---

### Performance Questions

**Q: What's the latency of authority migration?**

A: Typically <100ms write pause. Depends on network latency between regions. Can be optimized with dedicated links.

---

**Q: How fast is policy evaluation?**

A: <100μs for simple policies (local cache), <1ms for complex (attribute lookup). All evaluation is local.

---

**Q: What's the performance impact of v1.6 caching?**

A:
- L1 cache: <10μs (in-memory)
- L2 cache: <5ms (distributed)
- Hybrid cache hit rate: >98% typical

---

### Development Questions

**Q: How do I test specifications before implementing?**

A: Use parser/validator to check grammar. Use shadow mode to test behavior with real traffic before enforcement.

---

**Q: Can I generate code from specifications?**

A: Yes. Many implementations generate types, APIs, and UI scaffolding from specs. Grammar is machine-readable.

---

**Q: How do I debug policy denials?**

A: Enable detailed audit logging. Logs show which policy denied, which expression evaluated false, and full context.

---

## Appendix H: Extended Validation Rules

### Structural Validation Rules

#### Rule V1: Name Validation
```
All names (dataType, inputIntent, policy, etc.) MUST:
- Start with uppercase letter
- Contain only alphanumeric and underscore
- Be unique within scope
- Not collide with reserved keywords
```

Reserved keywords: `version`, `type`, `system`, `runtime`, `internal`

#### Rule V2: Version Format
```
Versions MUST follow pattern: v<major>[.<minor>][.<patch>]
Examples: v1, v1.0, v1.2.3
Ordering: Lexicographic comparison
```

#### Rule V3: Reference Validation
```
All references (dataType refs, intent refs, etc.) MUST:
- Refer to defined artifact
- Include version if versioned
- Respect visibility (no forward refs unless allowed)
```

#### Rule V4: Expression Syntax
```
All expressions (constraints, policies, signals) MUST:
- Parse successfully
- Reference only available fields/context
- Evaluate to appropriate type (boolean for constraints/policies)
```

#### Rule V5: Circular Reference Detection
```
No circular references allowed in:
- DataType composition (nested types)
- View dependencies
- Flow step dependencies
EXCEPTION: Relationships (intentionally circular)
```

#### Rule V6: v1.6 Storage Binding Validation
```
storageBinding MUST:
- Have type matching config block (kv requires kv:, stream requires stream:, etc.)
- Reference valid resource from resourceRequirements (if distributed)
- Have valid key_pattern with only valid field references (for kv)
- Have valid subject_pattern (for stream)
```

#### Rule V7: v1.6 Cache Layer Validation
```
cacheLayer MUST:
- Have tier matching configuration (local requires local:, etc.)
- Reference valid resource for distributed tier
- Have valid eviction policy (lru/lfu/fifo)
- Have positive TTL values
- Reference valid resources in resourceRequirements
```

#### Rule V8: v1.6 Intent Routing Validation
```
inputIntent.routing MUST:
- Have transport matching configuration block
- Have valid subject_pattern for NATS
- Have valid path pattern for HTTP
- Have valid service/method for gRPC
- NOT appear in flow steps (I9 invariant)
```

### Semantic Validation Rules

#### Rule V9: Evolution Linearity
```
DataType evolution chains MUST:
- Be linear (no branches)
- Not skip versions (v1→v2→v3, not v1→v3)
- Declare compatibility explicitly
```

#### Rule V10: Atomic Group Boundary
```
Atomic groups MUST:
- Execute within single boundary
- Not span multiple boundaries
ALTERNATIVE: Use compensation pattern
```

#### Rule V11: Idempotency Keys
```
All transitions MUST:
- Define idempotency specification
- Use deterministic key generation
- Not rely on timestamps alone (time may drift)
```

#### Rule V12: Authority Singularity
```
For each entity at any moment:
- Exactly one region has authority
- Authority KV is source of truth
- No write accepted without authority check
```

#### Rule V13: Transport Scope (I9)
```
Transport bindings:
- MUST be at intent ingress (inputIntent.routing)
- MUST NOT be in flow steps
- MUST NOT be in atomic groups
- MUST NOT be in work units
```

### Completeness Validation Rules

#### Rule V14: Intent Transitions
```
All inputIntents MUST:
- Define at least one transition
- Specify target dataState
- Include required supplies
```

#### Rule V15: DataState Ownership
```
All dataStates MUST:
- Have declared lifecycle
- Have owner specification (which workers produce)
- Have enforcement policy
```

#### Rule V16: Flow Authorization
```
All flows MUST:
- Have policy governing execution
- Validate inputs before execution
- Handle failures gracefully
```

#### Rule V17: Worker Capabilities
```
All workers MUST:
- Declare accepted inputs with versions
- Declare produced outputs with versions
- Register with control plane
```

#### Rule V18: Policy Scopes
```
All policies MUST:
- Define appliesTo scope
- Have explicit effect (allow/deny)
- Have evaluation expression
```

#### Rule V19: Signal Types
```
All signals MUST:
- Have defined type (capacity, health, backpressure, etc.)
- Have emitter identification
- Have severity classification
```

#### Rule V20: Materialization Sources
```
All materializations MUST:
- Declare source dataState or event
- Define update trigger
- Specify freshness tolerance
```

#### Rule V21: v1.6 Resource References
```
All v1.6 features referencing resources MUST:
- Reference defined resource in resourceRequirements
- Match resource purpose to usage
- Respect consistency constraints
```

#### Rule V22: v1.6 Cache Configuration Completeness
```
cacheLayer declarations MUST:
- Have invalidation strategy
- Have eviction policy (for local/hybrid)
- Have max_entries or max_size limits
```

---

## Appendix I: Versioning & Evolution Strategy

### Version Numbering

SMS follows semantic versioning for specifications:

- **Major version** (v1, v2): Breaking changes to grammar
- **Minor version** (v1.5, v1.6): Additive features, backward compatible
- **Patch version** (v1.5.1, v1.6.1): Clarifications, bug fixes

### Backward Compatibility Guarantees

#### v1.x Promise
All v1.x versions are backward compatible:
- v1.6 specs work with v1.0 runtimes (new features ignored)
- v1.0 specs work with v1.6 runtimes (all features supported)
- Parsers MUST accept older versions
- Parsers MAY accept newer versions (forward compatible)

#### Deprecation Policy
Features deprecated in v1.x:
- Marked deprecated in specification
- Continue to work through v1.x
- Removed in v2.0

Current deprecations: None (v1.6 has no deprecations)

### Adding New Features

New features in minor versions MUST:
1. Be additive (new fields, new artifact types)
2. Have sensible defaults
3. Not change existing semantics
4. Be clearly marked as "NEW in vX.Y"

### Runtime Version Negotiation

Clients and runtimes negotiate versions:

```yaml
# Client declares requirement
metadata:
  requires_sms_version: v1.6
  compatible_with: [v1.5, v1.6]

# Runtime declares support
runtime:
  sms_version: v1.6
  supports: [v1.0, v1.1, v1.2, v1.3, v1.4, v1.5, v1.6]
```

If incompatible:
- Runtime rejects specification
- Error message indicates required version
- Client must upgrade or downgrade spec

### v1.6 Changes Summary

**New in v1.6**:
- Storage Binding: Declarative persistence configuration
- Cache Layer: Multi-tier caching infrastructure
- Intent Routing: Transport configuration at ingress
- Resource Requirements: Abstract infrastructure specification
- Client Cache: Browser/mobile caching contract
- I9 Invariant: Transport scope boundary enforcement

**Backward Compatible**: All v1.5 specs valid in v1.6

### Type Naming Migration (v1.5 to v1.6)

v1.6 standardizes type names for consistency with modern programming languages:

| v1.5 Type | v1.6 Type | Notes |
|-----------|-----------|-------|
| `integer` | `int` | 64-bit signed integer |
| `decimal` | `float` | IEEE 754 floating-point |
| `boolean` | `bool` | true/false values |

**Migration Guide**:
- v1.6 runtimes SHOULD accept both old and new type names for backward compatibility
- New specifications SHOULD use v1.6 type names (`int`, `float`, `bool`)
- v1.5 specifications using old names remain valid in v1.6
- Tooling MAY provide automatic conversion between naming conventions

**Rationale**: The v1.6 type names align with industry conventions in Rust, Go, TypeScript, and Python, improving readability and reducing cognitive load for developers.

### Future Roadmap (Informational)

**v1.7 (Planned)**:
- Enhanced observability grammar
- Cost attribution semantics
- Multi-tenancy improvements
- Streaming window enhancements

**v2.0 (Future)**:
- Remove deprecated features (none currently)
- Simplified grammar based on learnings
- Performance-oriented changes
- Potential grammar consolidation

---

## Appendix J: Accessibility (WCAG Compliance)

### Overview

SMS specifications SHOULD incorporate Web Content Accessibility Guidelines (WCAG) 2.1 Level AA compliance where applicable to UI and presentation layers. This appendix provides guidance for creating accessible experiences using SMS primitives.

### Accessibility Principles in SMS

SMS supports the four core WCAG principles through its declarative grammar:

1. **Perceivable**: Information and UI components must be presentable to users in ways they can perceive
2. **Operable**: UI components and navigation must be operable by all users
3. **Understandable**: Information and UI operation must be understandable
4. **Robust**: Content must be robust enough to work with assistive technologies

### Accessibility Annotations

SMS provides accessibility metadata fields that SHOULD be used in presentation layer declarations:

```yaml
presentationViews:
  - name: OrderList
    version: v1
    accessibility:
      role: list                    # ARIA role
      label: "Customer Orders"      # Screen reader label
      description: "List of all customer orders with status and total"
      keyboard_navigation: true     # Keyboard accessible
      focus_management: auto        # Auto-focus handling
```

### Field-Level Accessibility

Field definitions SHOULD include accessibility metadata:

```yaml
fields:
  - name: orderTotal
    type: money
    accessibility:
      label: "Order total amount"
      description: "Total cost including tax and shipping"
      required_label: true         # Announce as required field
      error_announcement: true     # Announce validation errors
```

### Experience Accessibility Requirements

Experience declarations SHOULD specify accessibility requirements:

```yaml
experiences:
  - name: OrderManagement
    version: v1
    accessibility:
      wcag_level: AA                 # Target WCAG level (A, AA, AAA)
      support:
        - screen_readers: true       # Screen reader compatible
        - keyboard_only: true        # Full keyboard navigation
        - high_contrast: true        # High contrast mode support
        - reduced_motion: respect    # Respect prefers-reduced-motion
      color_contrast_ratio: 4.5      # Minimum contrast ratio
      touch_target_size: 44px        # Minimum touch target size
```

### Accessibility in Conversational UI

Conversational interfaces (Part XVIII) have additional accessibility considerations:

```yaml
conversationalUI:
  name: OrderAssistant
  version: v1
  accessibility:
    text_alternative: true         # Text fallback for voice
    speech_rate: adjustable        # User-controllable speed
    pause_resume: true             # Pause/resume capability
    skip_navigation: true          # Skip to specific topics
```

### Implementation Requirements

Runtimes implementing SMS SHOULD:

1. **Semantic HTML**: Generate semantic HTML elements matching declared `role` values
2. **ARIA Attributes**: Translate accessibility metadata to appropriate ARIA attributes
3. **Keyboard Navigation**: Ensure all interactive elements are keyboard accessible
4. **Focus Management**: Implement logical focus order and visible focus indicators
5. **Screen Reader Support**: Provide meaningful labels and descriptions to assistive technologies
6. **Color Independence**: Never rely solely on color to convey information
7. **Responsive Text**: Support text resizing up to 200% without loss of functionality
8. **Error Identification**: Clearly identify and describe validation errors

### Validation Rules for Accessibility

Implementations MAY enforce the following accessibility validation rules:

1. **V-A1**: All interactive PresentationViews MUST have `accessibility.label` defined
2. **V-A2**: All form fields MUST have `accessibility.label` when `required: true`
3. **V-A3**: Color-based indicators MUST have non-color alternatives (text, icon, pattern)
4. **V-A4**: Touch targets MUST be at least 44x44 pixels when `accessibility.touch_target_size` is specified
5. **V-A5**: Keyboard navigation MUST be enabled for all interactive experiences
6. **V-A6**: Time-based sessions MUST provide warnings and extension mechanisms
7. **V-A7**: Animated content MUST respect `reduced_motion` preferences

### Accessibility Testing Guidance

Specifications declaring accessibility requirements SHOULD be tested for:

- **Screen Reader Compatibility**: NVDA, JAWS, VoiceOver, TalkBack
- **Keyboard Navigation**: Tab order, focus indicators, keyboard shortcuts
- **Color Contrast**: Automated contrast checking tools (4.5:1 for AA, 7:1 for AAA)
- **Touch Target Size**: Minimum 44x44 pixels for mobile interfaces
- **Responsive Design**: Text sizing, zoom support, viewport adaptation

### Accessibility Best Practices

**DO:**
- Provide meaningful `accessibility.label` and `accessibility.description` for all UI components
- Use semantic field types (`email`, `phone`, `url`) for proper input assistance
- Declare `keyboard_navigation: true` for all interactive experiences
- Specify `wcag_level` explicitly in experience declarations
- Use conversational UI as an accessibility enhancement

**DON'T:**
- Rely on placeholder text as labels
- Use only color to indicate validation states
- Create keyboard traps in navigation
- Disable zoom or text resizing
- Auto-play audio or video without user control

### Resources

- **WCAG 2.1 Guidelines**: https://www.w3.org/WAI/WCAG21/quickref/
- **ARIA Authoring Practices**: https://www.w3.org/WAI/ARIA/apg/
- **Section 508**: US federal accessibility standards
- **EN 301 549**: European accessibility standard

### Future Enhancements (v1.7+)

Planned accessibility features:
- Automated accessibility validation in parsers
- Accessibility test generation from specifications
- Multi-modal interaction patterns (voice + touch + keyboard)
- Personalization preferences (font size, contrast, reading level)

---

## Appendix K: Migration Guide - v1.5 to v1.6

### Overview

This guide helps migrate existing v1.5 specifications to v1.6. **All v1.5 specifications remain valid in v1.6** - migration is optional and can be done incrementally.

### What's New in v1.6

v1.6 adds infrastructure-level features for storage, caching, and routing:

1. **Storage Binding**: Declarative storage configuration
2. **Unified Caching**: Multi-tier cache coordination  
3. **Intent Routing**: Transport-agnostic intent delivery
4. **Resource Requirements**: Infrastructure abstraction
5. **Client Cache Contract**: Offline-first capabilities

### Migration Strategy

**Option 1: No Migration Required**
- v1.5 specifications work unchanged in v1.6 runtimes
- Default storage and caching behaviors apply
- No code changes needed

**Option 2: Incremental Adoption**
- Add v1.6 features where they provide value
- Migrate storage-critical entities first
- Add caching for high-traffic views
- Configure client cache for offline scenarios

**Option 3: Full v1.6 Adoption**
- Declare all storage bindings explicitly
- Configure unified caching topology
- Define intent routing rules
- Specify resource requirements
- Implement client cache contracts

### Step 1: Add Storage Bindings (Optional)

Storage bindings make storage requirements explicit and portable.

**Before (v1.5)**: Implicit storage
```yaml
dataStates:
  - name: OrderRecord
    type: Order
    lifecycle: persistent
    mutability:
      scope: entity
```

**After (v1.6)**: Explicit storage binding
```yaml
storageBindings:
  - name: OrderStorage
    stores:
      - type: kv
        for: [OrderRecord, OrderAuthority]
        partitioning: entity_key
        consistency: strong
      - type: stream
        for: [OrderCreatedEvent, OrderUpdatedEvent]
        retention: 90d

dataStates:
  - name: OrderRecord
    type: Order
    lifecycle: persistent
    storage: OrderStorage
    mutability:
      scope: entity
```

**Benefits**: Portable, testable, cloud-agnostic

### Step 2: Configure Unified Caching (Optional)

Unified caching provides coordinated multi-tier caching.

**Before (v1.5)**: Runtime-determined caching
```yaml
materializations:
  - name: OrderSummary
    source: OrderEvent
    targetState: OrderSummaryView
    freshness:
      maxStaleness: 1m
```

**After (v1.6)**: Explicit cache topology
```yaml
cacheLayers:
  - name: OrderCache
    topology:
      - tier: L1
        type: local
        for: [OrderSummaryView]
        max_size: 10000
        eviction: lru
      - tier: L2
        type: distributed
        for: [OrderSummaryView]
        ttl: 5m
    invalidation:
      strategy: write_through
      events: [OrderUpdatedEvent]

materializations:
  - name: OrderSummary
    source: OrderEvent
    targetState: OrderSummaryView
    cache: OrderCache
    freshness:
      maxStaleness: 1m
```

**Benefits**: Predictable latency, cost optimization, observability

### Step 3: Define Intent Routing (Optional)

Intent routing decouples intents from transport.

**Before (v1.5)**: Transport implicit
```yaml
inputIntents:
  - name: CreateOrder
    version: v1
    proposes:
      dataState: OrderRecord
```

**After (v1.6)**: Explicit routing
```yaml
inputIntents:
  - name: CreateOrder
    version: v1
    proposes:
      dataState: OrderRecord
    routing:
      transport: nats
      nats:
        subject_pattern: "orders.create.v1"
        delivery: jetstream
      response:
        mode: request_reply
        timeout: 30s
```

**Benefits**: Transport flexibility, explicit contracts, easier testing

### Step 4: Specify Resource Requirements (Optional)

Resource requirements abstract infrastructure needs.

**New in v1.6**:
```yaml
resourceRequirements:
  name: OrderSystemResources
  event_logs:
    - name: order_events
      retention: 90d
      throughput:
        writes_per_sec: 10000
  entity_stores:
    - name: order_store
      entity_types: [Order, OrderAuthority]
      access_pattern: random
      consistency: strong
  provisioning:
    mode: auto
```

**Benefits**: Infrastructure portability, cost estimation, capacity planning

### Step 5: Implement Client Cache (Optional)

Client cache enables offline-first experiences.

**Before (v1.5)**: Session-based offline
```yaml
experiences:
  - name: MobileBanking
    version: v1
    entry_points:
      - view: AccountDashboard
```

**After (v1.6)**: Client cache contract
```yaml
experiences:
  - name: MobileBanking
    version: v1
    entry_points:
      - view: AccountDashboard
    clientCache:
      enabled: true
      strategies:
        - type: prefetch
          views: [AccountDashboard, TransactionHistory]
        - type: optimistic_write
          intents: [TransferMoney]
      offline:
        read: always
        write: queue
      sync:
        on_reconnect: true
        conflict_resolution: last_write_wins
```

**Benefits**: Offline functionality, reduced latency, bandwidth efficiency

### Step 6: Update Type Names (Breaking Change)

v1.6 renames `dataState` references to be more consistent.

**Before (v1.5)**: Mixed naming
```yaml
materializations:
  - targetState: OrderView
```

**After (v1.6)**: Consistent naming (both valid)
```yaml
materializations:
  - targetState: OrderView    # Still valid
    # OR
    target_state: OrderView   # New style
```

**Note**: This is a cosmetic change; both styles work in v1.6.

### Step 7: Test Migration

1. **Validate spec**: Ensure v1.6 parser accepts your specification
2. **Shadow deploy**: Run v1.6 runtime alongside v1.5
3. **Compare behavior**: Verify identical outcomes
4. **Performance test**: Measure caching and routing improvements
5. **Gradual promotion**: Canary deploy → production

### Validation Checklist

- [ ] All v1.5 declarations remain valid
- [ ] Storage bindings reference existing data states
- [ ] Cache layers reference existing views
- [ ] Intent routing maintains delivery guarantees
- [ ] Client cache sync strategies handle conflicts
- [ ] Resource requirements match actual usage
- [ ] All tests pass with v1.6 runtime
- [ ] Performance metrics meet targets

### Common Pitfalls

**Pitfall 1: Over-specifying storage**
- Don't add storage bindings unless portability/testability matters
- Default behaviors work well for most cases

**Pitfall 2: Cache complexity**
- Start with single-tier caching
- Add L2 only when L1 hit rate insufficient

**Pitfall 3: Premature routing configuration**
- Use default NATS routing initially
- Add custom routing only when needed (e.g., HTTP webhooks)

**Pitfall 4: Client cache conflicts**
- Choose conflict resolution strategy carefully
- Test offline scenarios thoroughly
- Consider entity types (append-only vs mutable)

### Rollback Plan

If issues arise:

1. **Revert to v1.5 runtime**: Specifications remain compatible
2. **Remove v1.6 declarations**: Fall back to defaults
3. **Shadow mode testing**: Validate before promotion
4. **Feature flags**: Toggle v1.6 features independently

### Next Steps

After migration:

1. **Monitor metrics**: Cache hit rates, storage latency, offline sync success
2. **Optimize topology**: Adjust cache sizes, TTLs based on data
3. **Expand coverage**: Add v1.6 features to more entities/views
4. **Document decisions**: Record storage and caching choices

### Resources

- **EBNF Grammar**: Full v1.6 grammar definitions
- **Implementation Guide**: Runtime architecture for v1.6 features
- **Reference Examples**: Complete applications with v1.6 features
- **Appendix K**: Quick reference patterns

---

## Appendix L: Quick Reference - Common Patterns

### Pattern: Entity with Authority

```yaml
dataStates:
  - name: AccountRecord
    type: Account
    lifecycle: persistent
    mutability:
      scope: entity
      exclusive: true
      authority: AccountAuthority

authorities:
  - name: AccountAuthority
    scope: entity
    resolution: nearest_region(account.owner.country)
    migration:
      allowed: true
      strategy: pause-and-cutover
```

### Pattern: Materialized View

```yaml
materializations:
  - name: AccountBalance
    source: AccountTransactionEvent
    targetState: BalanceView
    freshness:
      maxStaleness: 1m
    authority_agnostic: true
    epoch_tolerant: true
    retrieval:
      mode: by_entity
      entity_key: account_id
```

### Pattern: Form with Intent Binding

```yaml
presentationViews:
  - name: TransferForm
    version: v1
    consumes:
      dataState: TransferFormData
    triggers:
      intent: InitiateTransfer
      binding: form
      field_mapping:
        - view_field: from_account
          supplies: transfer.from_account
          source: context
        - view_field: to_account
          supplies: transfer.to_account
          source: input
          required: true
        - view_field: amount
          supplies: transfer.amount
          source: input
          required: true
          validation:
            constraint: amount > 0
            message: "Amount must be positive"
```

### Pattern: Policy with Shadow Mode

```yaml
policies:
  - name: ModifyOrder
    version: v2
    lifecycle: shadow
    rulesFor: ModifyOrderIntent
    condition: order.status == "pending" && actor.role == "customer"
    effect: allow
```

### Pattern: Storage Binding (v1.6)

```yaml
storageBindings:
  - name: OrderStorage
    stores:
      - type: kv
        for: [OrderRecord, OrderAuthority]
        partitioning: entity_key
        consistency: strong
      - type: stream
        for: [OrderCreatedEvent, OrderUpdatedEvent]
        retention: 90d
        partitioning: entity_key
```

### Pattern: Unified Caching (v1.6)

```yaml
cacheLayers:
  - name: OrderCache
    topology:
      - tier: L1
        type: local
        for: [OrderView, OrderSummary]
        max_size: 10000
        eviction: lru
      - tier: L2
        type: distributed
        for: [OrderView]
        max_size: 1000000
        eviction: lru
        ttl: 5m
    invalidation:
      strategy: write_through
      events: [OrderUpdatedEvent]
```

---

## Conclusion

This completes the SMS v1.6 Specification document. This specification represents a comprehensive, production-ready system for defining application behavior through declarative, intent-first specifications.

### What You've Learned

This specification covered:

1. **Core Principles**: Intent-first design, versioning, zero-downtime evolution
2. **Complete Grammar**: Data, UI, Input, Flows, Workers, Policies, v1.5 extensions, v1.6 infrastructure
3. **Authority Management**: Entity-scoped authority, migrations, multi-region
4. **UI Composition**: Semantic grouping, experiences, field permissions, navigation
5. **Advanced Features**: Assets, search, workflows, notifications, collaboration, governance
6. **v1.6 Infrastructure**: Storage binding, unified caching, intent routing, resource requirements
7. **Design Rationale**: Why decisions were made, alternatives considered
8. **Implementation Guidance**: Architecture, testing, performance, security
9. **Practical Examples**: Complete applications demonstrating all features
10. **Troubleshooting**: Common issues and solutions
11. **Evolution Strategy**: How to adopt v1.6 from earlier versions

### Next Steps

**For Specification Authors**:
1. Review examples relevant to your domain
2. Start with core grammar (Data, UI, Input)
3. Add v1.5 features as needed
4. Leverage v1.6 infrastructure features (storage, caching, routing)
5. Validate with parser
6. Test in shadow mode

**For Runtime Implementers**:
1. Follow implementation checklist in Appendix G
2. Start with core grammar and execution
3. Add v1.5 features incrementally
4. Implement v1.6 infrastructure layer
5. Test extensively with chaos testing
6. Optimize for your use case

**For Architects**:
1. Understand design rationale (Decision 1-12)
2. Map your requirements to SMS features
3. Design authority and boundary strategy
4. Plan storage and caching topology
5. Plan multi-region deployment
6. Define governance policies

### Resources

**Specification Documents**:
- This document: Complete normative specification
- EBNF Grammar: Formal grammar definition (separate document)
- Implementation Guide: Runtime architecture and patterns
- Reference Examples: Production-ready applications
- Quick Reference: Cheat sheets and decision trees

**Community**:
- Discussions: Design questions and use cases
- Issues: Bug reports and feature requests
- RFCs: Proposals for future versions

---

**This is the complete SMS v1.6 Specification.**

**Version**: 1.6  
**Status**: Production Ready  
**Completeness**: 100%  
**Last Updated**: 2026-02-04  

**All v1.5 features maintained. New v1.6 infrastructure features added. Fully backward compatible.**

---
