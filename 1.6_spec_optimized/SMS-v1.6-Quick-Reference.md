# SMS v1.6 Quick Reference

## For LLM Agents and Human Users

This document provides rapid-lookup cheat sheets, decision trees, and troubleshooting guides for SMS v1.6. It is optimized for:

**LLM Agents**: Use this document to quickly validate specification patterns, make design decisions, and troubleshoot common issues without parsing the full specification. Each section is self-contained with complete information.

**Human Users**: Use this as a companion while writing specifications or implementing runtimes. Start with decision trees for design choices, then use cheat sheets for syntax, and troubleshooting guides when things go wrong.

**Version**: 1.6  
**Status**: Production Ready  
**Last Updated**: 2026-02-04

### Document Organization

- **Part I**: Grammar cheat sheets and templates for specification authoring
- **Part II**: Decision trees for design choices (authority, invariants, UI, v1.5/v1.6 features)
- **Part III**: Common patterns and working examples
- **Part IV**: Troubleshooting guides (authority, UI, v1.5/v1.6 features)
- **Part V**: Performance targets and operational references
- **Part VI**: v1.5 feature quick references (forms, assets, search, workflows, etc.)
- **Part VII**: v1.6 feature quick references (storage binding, caching, routing, resources)
- **Part VIII**: Authority, storage, and cheat sheets

### Quick Navigation by Task

- **Writing a spec**: → Grammar Cheat Sheets → Templates → Examples
- **Design decisions**: → Decision Trees → Common Patterns
- **Implementation**: → Error Reference → Performance Targets → NATS Subjects
- **Debugging**: → Troubleshooting Guides → Error Quick Reference
- **v1.5 features**: → Part VI (Forms, Assets, Search, etc.)
- **v1.6 features**: → Part VII (Storage Binding, Caching, Routing, Resources)

### Related Documents

- **SMS-v1.6-Specification.md**: Complete normative specification
- **SMS-v1.6-EBNF-Grammar.md**: Integrated formal grammar for parser/tooling
- **SMS-v1.6-Implementation-Guide.md**: Detailed implementation patterns
- **SMS-v1.6-Reference-Examples.md**: Working examples and modeling guidance

---

## PART I: SPECIFICATION AUTHORING

> **Note**: For formal EBNF grammar and production rules, see **SMS-v1.6-EBNF-Grammar.md**. This section provides practical templates and usage examples.

### Minimal Valid Specification

```yaml
system:
  name: MySystem
  version: v1

dataTypes:
  - name: MyData
    version: v1
    fields:
      id: uuid
      value: string

inputIntents:
  - name: CreateMyData
    version: v1
    proposes:
      dataState: MyDataRecord
      version: v1
    supplies:
      required: [value]
    transitions:
      to: MyDataRecord

workers:
  - name: MyWorker
    acceptsInputs: [CreateMyData v1]
    produces: [MyDataRecord v1]

smsFlows:
  - name: CreateFlow
    triggeredBy:
      inputIntent: CreateMyData
    steps:
      - persist: MyDataRecord
```

**Lines of code**: ~30  
**Functionality**: Basic CRUD operation

### What's New in v1.6

| Feature | Purpose | Key Declaration |
|---------|---------|-----------------|
| Storage Binding | Declarative persistence config | `storageBindings:` |
| Cache Layer | Multi-tier caching | `cacheLayers:` |
| Resource Requirements | Abstract infrastructure | `resourceRequirements:` |
| Intent Routing | Transport configuration | `inputIntents[].routing:` |
| Client Cache | Browser/mobile caching | `experiences[].clientCache:` |
| Cache Signals | Cache observability | `signals[].type: cache` |

### Common Field Types

#### Primitive Types

| Type | Go | TypeScript | Example |
|------|-----|------------|---------|
| `string` | `string` | `string` | `"hello"` |
| `integer` | `int64` | `number` | `42` |
| `decimal` | `decimal.Decimal` | `Decimal` | `99.99` |
| `boolean` | `bool` | `boolean` | `true` |
| `uuid` | `uuid.UUID` | `string` | `"550e8400-..."` |
| `timestamp` | `time.Time` | `Date` | `"2024-01-15T14:23:45Z"` |
| `duration` | `time.Duration` | `number` (ms) | `"5m"` |

#### Complex Types

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

### Template Library

#### Template: Basic CRUD Service

```yaml
system:
  name: {ENTITY}Service

dataTypes:
  - name: {ENTITY}
    version: v1
    fields:
      id: uuid
      created_at: timestamp
      updated_at: timestamp
      {custom_fields}: {types}

dataStates:
  - name: {ENTITY}Record
    type: {ENTITY}
    lifecycle: persistent
    constraintPolicy:
      enforcement: strict

inputIntents:
  - name: Create{ENTITY}
    version: v1
    proposes: {dataState: {ENTITY}Record, version: v1}
    supplies:
      required: [{custom_fields}]
    transitions:
      to: {ENTITY}Record
  - name: Update{ENTITY}
    version: v1
    proposes: {dataState: {ENTITY}Record, version: v1}
    supplies:
      required: [id]
      optional: [{custom_fields}]
    transitions:
      to: {ENTITY}Record

workers:
  - name: {ENTITY}Service
    acceptsInputs:
      - Create{ENTITY} v1
      - Update{ENTITY} v1
    produces: [{ENTITY}Record v1]

smsFlows:
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

#### Template: Realm (Boundary)

```yaml
realms:
  - name: {SERVICE_NAME}
    version: v1
    description: "{Description of service boundary}"
    owns:
      - {OwnedDataType1}
      - {OwnedDataType2}
    workUnits:
      - {WorkUnit1}
      - {WorkUnit2}
    policy_set: {SERVICE_NAME}Policies
```

**Usage**: Replace `{SERVICE_NAME}` with your service name, list owned types and work units

#### Template: Policy for Resource

```yaml
policies:
  - name: {ACTION}{RESOURCE}
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

#### Template: Scaling Policy

```yaml
scalingPolicies:
  - name: {worker}_autoscale
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

### v1.6 Template Library

#### Template: Storage Binding

```yaml
storageBindings:
  - name: {EntityName}Storage
    version: v1
    type: kv
    kv:
      bucket: "SMS_{ENTITY}_STATE"
      key_pattern: "{entity}.{{id}}"
      serialization: json
      history: 5
      ttl: 0s  # No expiration
      replicas: 3
    access:
      read_policy: local
      write_consistency: strong
```

#### Template: Cache Layer (Hybrid)

```yaml
cacheLayers:
  - name: {ViewName}Cache
    version: v1
    topology:
      tier: hybrid
      local:
        max_entries: 1000
        ttl: 5m
        eviction: lru
      distributed:
        resource: {ViewCacheResource}
        ttl: 30m
      hybrid:
        promotion: on_miss
    invalidation:
      strategy: hybrid
      ttl: 30m
      watch_keys:
        - "entity.{{entity_id}}"
      propagate: true
    observability:
      emit_signals: true
      metrics: [hit_rate, miss_rate, latency, size, evictions]
      interval: 10s
```

#### Template: Resource Requirements

```yaml
resourceRequirements:
  name: {DomainName}Resources
  version: v1
  
  eventLogs:
    - name: {Domain}IntentLog
      purpose: intent_ingestion
      ordering: causal
      delivery: at_least_once
      durability: persistent
      retention:
        strategy: bounded
        max_age: 90d
      replication:
        min_replicas: 3
        consistency: strong
  
  entityStores:
    - name: {Entity}State
      purpose: authoritative_state
      access_pattern: key_value
      consistency: strong_per_key
      durability: persistent
      history:
        enabled: true
        depth: 10
      replication:
        min_replicas: 3
  
  provisioning:
    mode: auto
    migrations:
      enabled: true
      strategy: rolling
```

#### Template: Intent Routing

```yaml
inputIntents:
  - name: {IntentName}
    version: v1
    proposes:
      dataState: {TargetState}
    supplies:
      required: [field1, field2]
    routing:
      transport: nats
      nats:
        subject_pattern: "SMS.INTENT.{domain}.{IntentName}.{{realm}}"
        delivery: jetstream
        stream: SMS_INTENTS
        queue: {intent}-workers
      response:
        mode: request_reply
        timeout: 30s
```

#### Template: Client Cache

```yaml
experiences:
  - name: {ExperienceName}
    version: v1
    entry_point: Dashboard
    includes: [Dashboard, ListView, DetailView]
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
        views: [DashboardView, ListItemView]
        trigger: on_connect
      offline:
        enabled: true
        required_views: [DashboardView]
        sync_on_reconnect: true
        conflict_resolution: server_wins
```

### Common Expressions

#### Constraint Expressions

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

#### Policy Expressions

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

#### Signal Conditions

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

## PART II: DECISION TREES

### v1.6 Feature Decisions

#### Which Storage Type?

```
What kind of data are you storing?
├─ Entity state (key-value lookups) → Use 'kv'
│   └─ Need history? → Set history > 0
├─ Event stream (ordered messages) → Use 'stream'
│   └─ Choose retention: limits, interest, or workqueue
└─ Large binary objects → Use 'object_store'
```

#### Which Cache Tier?

```
Where is the cached data accessed from?
├─ Single process only → Use 'local'
│   └─ Fast: < 10μs, lost on restart
├─ Multiple processes/instances → Use 'distributed'
│   └─ Medium: < 5ms, survives restarts
└─ Both patterns → Use 'hybrid'
    └─ Best of both: L1 local, L2 distributed
        └─ Choose promotion: on_miss or on_access
```

#### Which Eviction Policy?

```
What's the access pattern?
├─ General purpose, access-pattern sensitive → Use 'lru'
├─ Hot-spot workloads, frequency matters → Use 'lfu'
└─ Time-ordered data, queue-like → Use 'fifo'
```

#### Which Invalidation Strategy?

```
How fresh must cached data be?
├─ Application controls exactly → Use 'explicit'
├─ Predictable staleness tolerance → Use 'ttl'
├─ Real-time consistency required → Use 'watch'
│   └─ Requires pub/sub infrastructure
└─ Balance of both → Use 'hybrid'
    └─ TTL provides backstop, watch provides real-time
```

#### Which Transport Type?

```
Who is calling this intent?
├─ Internal microservices → Use 'nats'
│   └─ Need durability? → Use jetstream delivery
├─ External API clients (web, mobile) → Use 'http'
│   └─ Browser-native, universal
└─ High-performance RPC → Use 'grpc'
    └─ Efficient binary protocol
```

#### Which Response Mode?

```
How should the caller receive results?
├─ Need immediate feedback → Use 'request_reply'
│   └─ Client blocks, best for user-initiated actions
├─ Long-running operation → Use 'async'
│   └─ Immediate ack, result via callback
└─ No response needed → Use 'fire_and_forget'
    └─ Events, audit logs, non-critical notifications
```

#### Which Client Cache Storage?

```
What are the client requirements?
├─ Session-scoped, no persistence → Use 'memory'
│   └─ Fastest, lost on page refresh
├─ Offline-capable, survive restarts → Use 'persistent'
│   └─ IndexedDB/SQLite, requires encryption
└─ Both patterns → Use 'hybrid'
    └─ Memory L1, persistent L2
```

#### Which Cache Strategy (Client)?

```
What's the priority?
├─ Instant response, eventual consistency → Use 'cache_first'
│   └─ Return cached, fetch in background
├─ Fresh data, with offline fallback → Use 'network_first'
│   └─ Fetch first, fall back to cache on failure
└─ Performance + eventual update → Use 'stale_while_revalidate'
    └─ Return stale immediately, update in background
```

### General Design Decisions

#### When to Use Atomic Groups?

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

#### Which Lifecycle for DataState?

```
Is data regenerable from other sources?
├─ YES → Use 'materialized'
└─ NO → Is data temporary during processing?
    ├─ YES → Use 'intermediate'
    └─ NO → Use 'persistent'
```

#### Which Enforcement Level for Constraints?

```
Can system tolerate invalid data temporarily?
├─ YES → Is validation expensive?
│   ├─ YES → Use 'deferred'
│   └─ NO → Use 'best_effort'
└─ NO → Is data financially/legally critical?
    ├─ YES → Use 'strict'
    └─ NO → Use 'best_effort'
```

#### When to Shadow a Change?

```
Is change to production system?
├─ NO → Skip shadow (dev/test)
└─ YES → Does change affect authorization/data/schema?
    ├─ NO → Maybe skip shadow
    └─ YES → ALWAYS shadow first
        └─ Monitor for: 24h minimum
```

#### Which Completion Policy?

```
Must all steps complete?
├─ YES → Use 'all'
└─ NO → Is first response good enough?
    ├─ YES → Use 'any'
    └─ NO → Need N of M responses?
        ├─ YES → Use 'quorum'
        └─ NO → Reconsider design
```

### Authority & Invariant Decisions (v1.3)

#### Which Authority Scope?

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

#### Should Authority Migrate?

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

#### Which Storage Role?

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

#### Where Does Invariant Belong?

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

### UI Pattern Decisions (v1.4)

#### Which Composition Type?

```
What is the primary content?
├─ Single entity focus → Use 'entity_detail'
│   └─ With related data? → Add related views
├─ Collection of entities → Use 'entity_list'
│   └─ With filters/search? → Add filters section
├─ Dashboard/overview → Use 'dashboard'
│   └─ With widgets? → Use composition grid
├─ Form input → Use 'form'
│   └─ Multi-step? → Use wizard composition
└─ Report/analytics → Use 'report'
    └─ Interactive? → Add drill-down views
```

#### Which Navigation Placement?

```
How often is this accessed?
├─ Always available → Primary navigation
├─ Related to current context → Secondary/contextual
├─ Admin/settings → Utility navigation
└─ Rarely used → Hidden (link from other views)
```

#### Which Field Permission Type?

```
Should field be hidden or masked?
├─ Field existence is sensitive → Use visible: policy
│   └─ Unauthorized see nothing
└─ Field exists but value sensitive → Use mask:
    └─ What level of masking?
        ├─ Full replacement → type: full
        ├─ Show partial → type: partial + reveal
        └─ Show hash → type: hash
```

### v1.5 Feature Decisions

#### Which Asset Lifecycle?

```
Is the asset permanent or temporary?
├─ Permanent → Use 'permanent'
│   └─ Follows entity lifecycle
└─ Temporary → How long needed?
    ├─ Until upload completes → Use 'temporary'
    └─ Fixed duration → Use 'expiring'
        └─ Set expires_after duration
```

#### Which Search Strategy?

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

#### Which Session Storage?

```
What's your scaling strategy?
├─ Traditional monolith → Use 'stateful'
│   └─ Redis/Memcached session store
├─ Stateless microservices → Use 'stateless_jwt'
│   └─ Everything in JWT claims
└─ Hybrid/scalable web app → Use 'hybrid'
    └─ Session ID + essential claims in JWT
```

#### Which Notification Channel?

```
What's the message urgency?
├─ Critical/immediate → SMS or Push
├─ Important but not urgent → Email
├─ Informational → In-app notification
└─ Depends on user preference?
    └─ Use preference-based routing
        └─ Define fallback chain
```

#### Which Workflow Type?

```
Is the process long-running?
├─ NO → Use regular SMS flow
│   └─ Completes in seconds/minutes
└─ YES → Use durable workflow
    └─ What persistence needed?
        ├─ Just state → checkpoint_strategy: after_each_state
        ├─ Full history → checkpoint_strategy: after_each_event
        └─ Custom → checkpoint_strategy: on_checkpoint
```

#### Which Collaboration Strategy?

```
How do users edit shared content?
├─ Sequential edits (one at a time) → Use 'lock_based'
├─ Simple concurrent edits → Use 'last_write_wins'
├─ Text/document editing → Use 'operational_transform'
└─ Complex structured data → Use 'crdt'
```

---

## PART III: COMMON PATTERNS

### v1.6 Patterns

#### Pattern: Full-Stack Caching

```yaml
# 1. Define resource requirements
resourceRequirements:
  name: CachingResources
  version: v1
  entityStores:
    - name: ViewCache
      purpose: cache
      consistency: eventual
      durability: persistent
      expiration:
        enabled: true
        ttl: 1h
      replication:
        min_replicas: 3

# 2. Define server-side cache layer
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
        resource: ViewCache
        ttl: 30m
      hybrid:
        promotion: on_miss
    invalidation:
      strategy: hybrid
      propagate: true

# 3. Bind cache to materialization
materializations:
  - name: AccountSummaryView
    source: AccountState
    targetState: AccountSummaryState
    cache:
      layer: ViewHybridCache
      key: "account.{{account_id}}"

# 4. Define client-side cache
experiences:
  - name: BankingApp
    version: v1
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
      offline:
        enabled: true
        required_views: [AccountSummaryView]
        sync_on_reconnect: true
```

#### Pattern: Intent with Routing

```yaml
inputIntents:
  - name: InitiateTransfer
    version: v1
    proposes:
      dataState: TransferRequest
    supplies:
      required: [source_account, destination_account, amount]
    routing:
      transport: nats
      nats:
        subject_pattern: "SMS.INTENT.banking.InitiateTransfer.{{realm}}"
        delivery: jetstream
        stream: SMS_BANKING_INTENTS
        queue: transfer-workers
      response:
        mode: request_reply
        timeout: 30s
```

#### Pattern: Complete Storage Configuration

```yaml
# 1. Define abstract resources
resourceRequirements:
  name: BankingResources
  version: v1
  eventLogs:
    - name: BankingIntentLog
      purpose: intent_ingestion
      ordering: causal
      delivery: at_least_once
      durability: persistent
      retention:
        strategy: bounded
        max_age: 90d
      replication:
        min_replicas: 3
        consistency: strong
  entityStores:
    - name: AccountState
      purpose: authoritative_state
      access_pattern: key_value
      consistency: strong_per_key
      durability: persistent
      history:
        enabled: true
        depth: 10
      replication:
        min_replicas: 3

# 2. Bind DataState to storage
storageBindings:
  - name: AccountStateStorage
    version: v1
    type: kv
    kv:
      bucket: "SMS_ACCOUNT_STATE"
      key_pattern: "account.{{account_id}}"
      serialization: json
      history: 10
      replicas: 3
    access:
      read_policy: local
      write_consistency: strong

# 3. Reference in DataState
dataStates:
  - name: AccountState
    type: Account
    lifecycle: persistent
    storage:
      binding: AccountStateStorage
```

### Core Patterns (v1.5 and Earlier)

#### Pattern: Create Entity

```yaml
# 1. Define data
dataTypes:
  - name: Entity
    version: v1
    fields:
      id: uuid
      name: string

dataStates:
  - name: EntityRecord
    type: Entity
    lifecycle: persistent

# 2. Define input
inputIntents:
  - name: CreateEntity
    version: v1
    proposes: {dataState: EntityRecord, version: v1}
    supplies:
      required: [name]
    transitions:
      to: EntityRecord

# 3. Define flow
smsFlows:
  - name: CreateEntityFlow
    triggeredBy: {inputIntent: CreateEntity}
    steps:
      - validate: constraints
      - persist: EntityRecord

# 4. Define worker
workers:
  - name: EntityService
    acceptsInputs: [CreateEntity v1]
    produces: [EntityRecord v1]
```

#### Pattern: Cross-Boundary Transaction

```yaml
smsFlows:
  - name: OrderCheckout
    steps:
      - work_unit: reserve_inventory
        realm: inventory
      - work_unit: charge_payment
        realm: billing
      - work_unit: confirm_order
    failure:
      policy: compensate
      compensations:
        - on: charge_payment
          do: release_inventory
```

#### Pattern: Data Evolution

```yaml
# Step 1: Define new version
dataTypes:
  - name: Entity
    version: v2
    fields:
      id: uuid
      name: string
      description: string  # NEW

# Step 2: Define evolution
dataEvolutions:
  - name: Entity_v1_to_v2
    from: {type: Entity, version: v1}
    to: {type: Entity, version: v2}
    changes:
      - addField: {name: description, default: ""}
    compatibility:
      read: backward
      write: forward

# Step 3: Update workers to support both
workers:
  - name: EntityService
    compatibleVersions: [v1, v2]  # Both!
      
# Step 4: Shadow write
# (Implementation handles dual writes)

# Step 5: Promote when safe
# (Routing preferences shift to v2)
```

#### Pattern: Policy Rollout

```yaml
# Step 1: Deploy shadow
policyArtifacts:
  - policy: {name: NewAccessPolicy, version: v2}
    lifecycle:
      state: shadow
    scope: {domain: entity, component: worker.EntityService}

# Step 2: Monitor divergence
# (Watch signals for shadow vs enforced differences)

# Step 3: Promote
  - policy: {name: NewAccessPolicy, version: v2}
    lifecycle:
      state: enforce
      supersedes: v1

# Step 4: Deprecate old
  - policy: {name: NewAccessPolicy, version: v1}
    lifecycle:
      state: deprecated
```

#### Pattern: Multi-Region Deployment

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
  
  scalingPolicies:
    - signals:
        queue_depth > 100 => scale_up
        region.health == "degraded" => scale_up
  
  failure_domain:
    name: us_east_domain
    affected_locations: [us_east]
    failover:
      automatic: true
      target_domains: [eu_west_domain]
```

### v1.5 Patterns

#### Pattern: File Upload with Validation

```yaml
# 1. Define asset field
dataTypes:
  - name: Document
    fields:
      file:
        type: asset
        asset:
          category: document
          constraints:
            max_size: "10MB"
            mime_types: ["application/pdf"]

# 2. Define upload intent
inputIntents:
  - name: UploadDocument
    supplies:
      required: [title, file]
    asset_upload:
      field: file
      validation:
        scan_virus: true
        validate_mime: true

# 3. Define form
presentationViews:
  - name: DocumentUploadForm
    triggers:
      intent: UploadDocument
      binding: form
      field_mapping:
        - view_field: file_input
          supplies: file
          source: input
```

#### Pattern: Searchable Product Catalog

```yaml
# 1. Define searchable fields
dataTypes:
  - name: Product
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
materializations:
  - name: ProductSearchIndex
    source: ProductRecord
    search_index:
      enabled: true

# 3. Search results view
presentationViews:
  - name: ProductSearch
    search:
      target_index: ProductSearchIndex
      facets:
        - field: category
        - field: price
```

#### Pattern: Scheduled Report Generation

```yaml
# 1. Define trigger
scheduledTriggers:
  - name: DailyReport
    schedule:
      type: cron
      expression: "0 6 * * *"
      timezone: "America/New_York"
    triggers:
      intent: GenerateReport
    supplies:
      date: "{{ execution_date }}"
      type: "daily"
    idempotency:
      key: ["date", "type"]
      scope: global
    execution:
      concurrency: forbid
      timeout: "30m"
      retry:
        max_attempts: 3
        backoff: exponential

# 2. Define notification
notificationChannels:
  - name: ReportDelivery
    delivery:
      type: email

# 3. Link in flow
smsFlows:
  - name: GenerateReportFlow
    steps:
      - work_unit: generate_report
      - notify:
          channel: ReportDelivery
          template: ReportReady
```

#### Pattern: Collaborative Document Editing

```yaml
# 1. Define session
collaborativeSessions:
  - name: DocEditing
    scope:
      entity_type: Document
    participants:
      max_concurrent: 10
    synchronization:
      strategy: operational_transform

# 2. Define composition
compositions:
  - name: DocumentEditor
    collaborative_session:
      type: DocEditing
      entity_binding: document_id

# 3. Define structured content
dataTypes:
  - name: Document
    fields:
      content:
        type: structured_content
        structured_content:
          format: prosemirror
```

#### Pattern: External Payment Integration

```yaml
# 1. Define integration
externalDependencies:
  - name: PaymentGateway
    operations:
      - name: process_payment
        method: POST
        retry:
          max_attempts: 3
          idempotency_key: "{{ payment_id }}"

# 2. Use in flow
smsFlows:
  - name: CheckoutFlow
    steps:
      - external_call:
          integration: PaymentGateway
          operation: process_payment
      - persist: OrderRecord
```

#### Pattern: Durable Order Fulfillment Workflow

```yaml
durableWorkflows:
  - name: OrderFulfillment
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
```

---

## PART IV: TROUBLESHOOTING

### Authority Troubleshooting (v1.3)

#### CAS Failure Diagnosis

| Error | Cause | Fix |
|-------|-------|-----|
| `ErrEpochMismatch` | Authority migrated | Refresh CAS token |
| `ErrVersionMismatch` | Concurrent write | Reload entity, retry |
| `ErrWrongRegion` | Sent to wrong region | Route to authority_region |
| `ErrLeaseExpired` | Lease not renewed | Wait for new lease |
| `ErrTransitioning` | Authority moving | Wait, retry after transition |

#### Authority Migration Issues

| Symptom | Possible Cause | Investigation |
|---------|---------------|---------------|
| Writes rejected everywhere | Transition stuck | Check `SMS.AUTHORITY` KV |
| Reads stale after migration | View not caught up | Check view worker lag |
| Epoch higher than expected | Rapid migrations | Review migration triggers |
| Region mismatch in token | Stale client cache | Force cache refresh |

#### View Authority Issues

| Symptom | Possible Cause | Fix |
|---------|---------------|-----|
| View stops updating | Epoch boundary not received | Check epoch markers |
| Events from wrong region | Multi-region merge issue | Verify merger config |
| Duplicate events in view | Epoch rollback | Should not happen (investigate) |

### UI Troubleshooting (v1.4)

#### "Composition not rendering"
→ Check primary view exists  
→ Verify policy_set grants access  
→ Check data source returns data  
→ Verify view version matches

#### "Navigation item missing"
→ Check composition in experience.includes  
→ Verify on_unauthorized is not `conceal` for denied user  
→ Check parent composition is accessible  
→ Verify purpose is in rendered nav group

#### "Field still visible when shouldn't be"
→ Check policy evaluation  
→ Verify field name matches exactly  
→ Check mask.unless condition  
→ Clear permission cache

#### "Indicator not updating"
→ Check indicator.when expression  
→ Verify data context available  
→ Check WebSocket/SSE connection  
→ Verify polling interval

#### "Related view not loading"
→ Check relationship basis exists  
→ Verify data_scope expression  
→ Check required vs optional  
→ Verify cardinality matches data

### v1.5 Feature Troubleshooting

#### Form Binding Issues

| Symptom | Possible Cause | Fix |
|---------|---------------|-----|
| Form won't submit | Required field missing | Check field_mapping required |
| Field mapping fails | Source type mismatch | Verify source: context/input/computed |
| on_success not triggering | Intent failed | Check flow execution logs |
| Validation not running | constraint not defined | Add constraint expression |

#### Asset Upload Issues

| Symptom | Possible Cause | Fix |
|---------|---------------|-----|
| Upload rejected | Size/type constraint | Check constraints |
| File corrupt after upload | Serialization issue | Verify mime validation |
| Variant not generated | Transform missing | Check variants config |
| Access denied | Policy not evaluated | Check access.policy |

#### Search Issues

| Symptom | Possible Cause | Fix |
|---------|---------------|-----|
| No results | Index not built | Trigger reindex |
| Slow search | Too many fields indexed | Reduce indexed fields |
| Facets empty | facet.enabled not set | Add facet config |
| Wrong relevance | Weight misconfigured | Adjust field weights |

#### Session Issues

| Symptom | Possible Cause | Fix |
|---------|---------------|-----|
| Session expires early | idle_timeout too short | Increase timeout |
| Data not persisting | stateless_jwt storage | Switch to hybrid |
| Cross-device sync fails | scope: entity but needs global | Change scope |

#### Workflow Issues

| Symptom | Possible Cause | Fix |
|---------|---------------|-----|
| Workflow stuck | State timeout reached | Check timeout config |
| Compensation not running | Not defined for state | Add compensation |
| Checkpoint not saving | checkpoint_strategy wrong | Use after_each_state |

### v1.6 Troubleshooting

#### Storage Binding Issues

| Symptom | Possible Cause | Fix |
|---------|---------------|-----|
| Key pattern not resolving | Missing field in entity | Check key_pattern references valid fields |
| Storage not created | Wrong type specified | Verify type matches config block (kv/stream/object_store) |
| Write failures | Strong consistency with local-only reads | Ensure read_policy allows distributed reads |
| Data not persisting | TTL too short | Check ttl configuration |
| History not available | History set to 0 | Set history > 0 for KV stores |

#### Cache Layer Issues

| Symptom | Possible Cause | Fix |
|---------|---------------|-----|
| Cache not populated | Resource reference missing | Verify distributed.resource exists in resourceRequirements |
| High miss rate | TTL too short | Increase TTL values |
| Stale data | Invalidation not propagating | Set propagate: true, check watch_keys |
| Memory pressure | max_entries too high | Reduce max_entries or max_size |
| Eviction too aggressive | Wrong eviction policy | Try different eviction policy (lru/lfu/fifo) |
| Cache cold on restart | No warmup configured | Enable warmup with eager or scheduled strategy |

#### Intent Routing Issues

| Symptom | Possible Cause | Fix |
|---------|---------------|-----|
| Intent not delivered | Subject pattern wrong | Check subject_pattern syntax |
| No workers receiving | Queue group mismatch | Verify queue name matches worker subscription |
| Timeout errors | Response timeout too short | Increase response.timeout |
| Async callback missing | Callback URL invalid | Check callback pattern/URL |
| Dead letters accumulating | Processing failures | Check worker logs, verify work unit exists |

#### Client Cache Issues

| Symptom | Possible Cause | Fix |
|---------|---------------|-----|
| Offline not working | memory storage type | Change to persistent or hybrid |
| Encryption errors | No encryption key | Ensure session provides encryption key |
| Views missing offline | Not in required_views | Add view to required_views list |
| Sync conflicts | Wrong conflict_resolution | Choose appropriate strategy |
| Cache too large | max_size too high | Reduce max_size, add eviction |
| Prefetch not working | Wrong trigger | Check prefetch.trigger matches user behavior |

### General Troubleshooting

#### Symptom: High Latency

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

#### Symptom: Duplicate Processing

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

#### Symptom: Policy Not Enforcing

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

#### Symptom: Worker Not Receiving Work

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

#### Symptom: Zero-Downtime Deployment Failed

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

### Multi-Region Troubleshooting

#### "Writes rejected in all regions"

**Check**:
1. Authority KV state (check `SMS.AUTHORITY`)
2. Current epoch and transition state
3. Network connectivity between regions
4. Lease status

**Resolution**:
- If transitioning: wait for transition to complete
- If stuck: manually resolve authority state
- If lease expired: wait for new lease election

#### "Data inconsistent across regions"

**Check**:
1. Replication lag between regions
2. Event ordering and epoch markers
3. View materialization worker health
4. JetStream mirror status

**Resolution**:
- Check replication backlog
- Verify epoch boundaries being processed
- Restart stuck view workers
- Force re-sync if necessary

---

## PART V: OPERATIONAL REFERENCES

### Performance Targets

#### Latency Budgets

| Operation | Target | Acceptable | Investigate |
|-----------|--------|------------|-------------|
| In-process invocation | <50μs | <100μs | >100μs |
| Local socket | <500μs | <1ms | >2ms |
| NATS invocation | <5ms | <10ms | >20ms |
| Policy evaluation | <100μs | <500μs | >1ms |
| L1 cache lookup | <10μs | <50μs | >100μs |
| L2 cache lookup | <5ms | <10ms | >20ms |
| Client memory cache | <1ms | <5ms | >10ms |
| Client persistent cache | <10ms | <50ms | >100ms |
| Constraint validation | <1ms | <5ms | >10ms |
| Flow step transition | <50ms | <100ms | >200ms |

#### v1.6 Performance Targets

| Component | Metric | Target |
|-----------|--------|--------|
| L1 (Local) Cache | Lookup latency | < 10μs |
| L1 (Local) Cache | Insert latency | < 10μs |
| L1 (Local) Cache | Eviction latency | < 100μs |
| L2 (Distributed) Cache | Lookup latency | < 5ms |
| L2 (Distributed) Cache | Insert latency | < 10ms |
| L2 (Distributed) Cache | Invalidation propagation | < 100ms |
| Hybrid Cache | L1 hit latency | < 10μs |
| Hybrid Cache | L1 miss + L2 hit | < 5ms |
| Hybrid Cache | Promotion latency | < 1ms |
| Client Cache | Memory lookup | < 1ms |
| Client Cache | Persistent lookup | < 10ms |
| Client Cache | Encryption overhead | < 5ms |
| Sync Operations | Initial sync | < 5s |
| Sync Operations | Reconnection sync | < 2s |

#### Throughput Targets

| Component | Target | Scaling |
|-----------|--------|---------|
| Worker (per instance) | 100-1000 req/sec | Horizontal |
| Flow engine | 1000+ flows/sec | Horizontal |
| Policy evaluator | 10,000+ eval/sec | Vertical (CPU) |
| Signal aggregator | 100,000+ sig/sec | Horizontal |
| Routing cache | 1M+ lookups/sec | Vertical (RAM) |
| View materialization | 10,000+ events/sec | Horizontal |
| Search indexing | 1,000+ docs/sec | Horizontal |

#### Resource Guidelines

| Component | CPU | Memory | Notes |
|-----------|-----|--------|-------|
| Worker | 0.5-2 cores | 512MB-2GB | Depends on work units |
| Flow engine | 1-4 cores | 1-4GB | State machine overhead |
| Control plane | 1-2 cores | 2-8GB | Artifact storage |
| Scheduler | 0.5-1 core | 512MB-1GB | Signal aggregation |
| Cache layer (L1) | 0.5 cores | 256MB-4GB | Size depends on max_entries |
| Cache layer (L2) | 1-2 cores | 1-8GB | Depends on dataset |
| Search service | 2-8 cores | 4-32GB | Index-dependent |
| Asset processor | 1-4 cores | 1-4GB | Transformation-dependent |

#### Capacity Planning

| Metric | Formula | Example |
|--------|---------|---------|
| Workers needed | `peak_rps / worker_capacity` | 5000 / 500 = 10 workers |
| Cache memory | `avg_entry_size × max_entries` | 1KB × 100K = 100MB |
| L2 cache size | `unique_entities × avg_size × 1.5` | 1M × 2KB × 1.5 = 3GB |
| JetStream storage | `events/day × avg_size × retention_days` | 1M × 500B × 90 = 45GB |
| Search index | `documents × avg_doc_size × 1.2` | 1M × 5KB × 1.2 = 6GB |

### NATS Subject Quick Reference

#### v1.6 Subject Patterns

| Purpose | Pattern | Example |
|---------|---------|---------|
| Intent routing | `SMS.INTENT.{{domain}}.{{intent_name}}.{{realm}}` | `SMS.INTENT.banking.InitiateTransfer.acme` |
| Cache signal | `SMS.SIGNAL.cache.{{cache_name}}.{{tier}}.{{region}}` | `SMS.SIGNAL.cache.ViewCache.hybrid.us-east-1` |
| Dead letter | `SMS.DLQ.{{domain}}.{{intent_name}}` | `SMS.DLQ.banking.InitiateTransfer` |
| Async callback | `SMS.CALLBACK.{{correlation_id}}` | `SMS.CALLBACK.abc123` |

#### Core Subject Patterns

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

#### Subject Subscription Patterns

| Component | Subscribes To | Why |
|-----------|---------------|-----|
| Worker | `spec.{domain}.>`, `policy.{domain}.worker.{name}.>`, `intent.{domain}.>` | Get specs, policies, work |
| Flow Engine | `spec.{domain}.>`, `policy.{domain}.sms.>` | Get flow definitions, policies |
| Scheduler | `signal.worker.>`, `topology.>` | Get signals, topology changes |
| UI | `spec.{domain}.>`, `policy.{domain}.ui.>` | Get view definitions, policies |
| Control Plane | (none - publishes only) | Distributes artifacts |

### Signal Severity Guidelines

#### When to Emit Each Severity

| Severity | Condition | Example | Action |
|----------|-----------|---------|--------|
| `info` | Normal operation | `queue_depth: 15` | Log only |
| `warn` | Approaching limits | `cpu_usage: 0.75` | Alert on-call |
| `error` | Threshold exceeded | `error_rate: 0.08` | Page on-call |
| `critical` | Service degraded | `all_workers_unhealthy` | Wake everyone |

#### Signal Types by Component

| Component | Emits | Frequency | Consumers |
|-----------|-------|-----------|-----------|
| Worker | capacity, health, latency | 5s | Scheduler, monitoring |
| SMS | backpressure, flow_completion | on event | Scheduler, audit |
| Policy Engine | shadow_divergence | on divergence | Ops, control plane |
| Infrastructure | region_health, node_status | 30s | Schedulers |
| Cache (v1.6) | pressure, hit_rate, evictions | 10s | Monitoring, auto-scaling |

#### Cache Pressure Levels

| Level | Capacity Threshold | Hit Rate Threshold | Action |
|-------|-------------------|-------------------|--------|
| `low` | < 50% | > 80% | Normal operation |
| `medium` | 50-75% | 60-80% | Monitor closely |
| `high` | 75-90% | 40-60% | Consider scaling or tuning |
| `critical` | > 90% | < 40% | Immediate attention |

### Error Quick Reference

#### v1.6 Validation Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `missing_storage_type` | storageBinding without type | Add type: kv/stream/object_store |
| `missing_bucket` | KV/object_store without bucket | Add bucket name |
| `missing_resource` | Distributed cache without resource | Add resource reference |
| `invalid_tier` | Unknown cache tier | Use local/distributed/hybrid |
| `missing_transport` | Routing without transport | Add transport: nats/http/grpc |
| `missing_subject` | NATS routing without subject_pattern | Add subject_pattern |
| `invalid_eviction` | Unknown eviction policy | Use lru/lfu/fifo |
| `resource_not_found` | Reference to non-existent resource | Define resource first |
| `transport_in_flow` | Transport specified in flow step | Remove transport from flow (I9 violation) |

---

## PART VI: v1.5 FEATURE QUICK REFERENCES

### Form Intent Binding

#### Form Binding Structure

```yaml
presentationViews:
  - name: <FormName>
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

#### Binding Types Quick Reference

| Binding | When to Use | Example |
|---------|-------------|---------|
| `form` | Multi-field user input | Transfer form, account creation |
| `action` | Single-click operation | Confirm payment, mark read |
| `link` | Navigation with data | View details, open document |

#### Field Source Types

| Source | Description | Example |
|--------|-------------|---------|
| `context` | From session/composition | `session.account_id` |
| `input` | User provides | Text field, dropdown, file upload |
| `computed` | Derived from other fields | `source.balance - amount` |

#### Success/Error Actions

| Action | Behavior |
|--------|----------|
| `navigate_back` | Return to previous screen |
| `navigate_to` | Go to specific composition |
| `stay` | Remain on current view |
| `refresh` | Reload current data |
| `close` | Close modal/drawer |
| `display_inline` | Show errors in form |
| `display_modal` | Show errors in popup |

### Asset Management

#### Asset Field Declaration

```yaml
dataTypes:
  - name: Document
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

#### Asset Categories

| Category | Use Case | Typical Mime Types |
|----------|----------|-------------------|
| `image` | Photos, diagrams | image/png, image/jpeg, image/webp |
| `document` | PDFs, Office docs | application/pdf, application/msword |
| `media` | Audio, video | video/mp4, audio/mpeg |
| `archive` | ZIP files, backups | application/zip, application/tar |
| `generic` | Any binary file | application/octet-stream |

#### Access Modes

| Mode | Description | When to Use |
|------|-------------|-------------|
| `public` | Anyone can access | Public images, marketing materials |
| `signed` | Time-limited URL | Temporary access, downloads |
| `authenticated` | Requires login | User documents, private files |
| `policy` | Policy-evaluated | Sensitive data, permission-based |

#### Asset Upload Pattern

```yaml
inputIntents:
  - name: UploadDocument
    version: v1
    supplies:
      required: [title, file]
    asset_upload:
      field: file
      stage: pre_submit | inline | post_submit
      validation:
        scan_virus: true
        validate_mime: true

presentationViews:
  - name: DocumentUploadForm
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

### Search & Discovery

#### Searchable Field Declaration

```yaml
dataTypes:
  - name: Product
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

#### Search Strategies

| Strategy | Description | Use Case |
|----------|-------------|----------|
| `full_text` | Natural language search | Product descriptions, articles |
| `exact` | Exact match only | SKU, email, order ID |
| `prefix` | Prefix matching | Autocomplete, type-ahead |
| `fuzzy` | Typo-tolerant | User-entered search terms |
| `semantic` | Vector similarity | AI-powered recommendations |

#### Faceted Search Pattern

```yaml
presentationViews:
  - name: ProductSearchResults
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
          - name: newest
            label: "Newest First"
            field: created_at
            order: desc
```

### Session Management

#### Session Declaration

```yaml
sessions:
  - name: UserSession
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

#### Session Scope Types

| Scope | Lifetime | Use Case |
|-------|----------|----------|
| `global` | Across entire application | User authentication |
| `realm` | Within single tenant/realm | Multi-tenant isolation |
| `entity` | Bound to specific entity | Document editing session |

#### Session Storage Backends

| Backend | Characteristics | Use Case |
|---------|----------------|----------|
| `stateful` | Server-side storage | Traditional web apps |
| `stateless_jwt` | Client-side JWT | Stateless APIs, microservices |
| `hybrid` | Session ID + JWT claims | Scalable web apps |

### Scheduled Triggers

```yaml
scheduledTriggers:
  - name: DailyReportGeneration
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

#### Schedule Types

| Type | Expression Format | Example |
|------|------------------|---------|
| `cron` | Standard cron syntax | `0 2 * * *` (2 AM daily) |
| `interval` | Duration between runs | `every 15m` |
| `calendar` | Business calendar | `weekdays at 9am` |

#### Concurrency Control

| Mode | Behavior |
|------|----------|
| `forbid` | Skip if previous still running |
| `allow` | Run concurrently |
| `replace` | Cancel previous, start new |

### Durable Workflows (v1.5)

#### Durable Workflow Declaration

```yaml
durableWorkflows:
  - name: OrderFulfillment
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

#### Checkpoint Strategies

| Strategy | Description | Use Case |
|----------|-------------|----------|
| `after_each_state` | Checkpoint on state transition | Standard workflows |
| `after_each_event` | Checkpoint on every event | Audit-critical |
| `on_checkpoint` | Manual checkpoint points | Long-running with batches |
| `periodic` | Time-based checkpoints | High-throughput |

### Notification Channels (v1.5)

#### Notification Channel Declaration

```yaml
notificationChannels:
  - name: UserAlerts
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

#### Delivery Types Quick Reference

| Type | Delivery Speed | Reliability | Use Case |
|------|---------------|-------------|----------|
| `push` | Immediate | Best-effort | Real-time alerts |
| `sms` | Seconds | High | Critical alerts |
| `email` | Minutes | High | Reports, receipts |
| `in_app` | Immediate | Guaranteed | Activity updates |
| `webhook` | Immediate | Configurable | Integrations |

### Collaborative Sessions (v1.5)

#### Collaborative Session Pattern

```yaml
collaborativeSessions:
  - name: DocumentEditing
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

#### Synchronization Strategies

| Strategy | Characteristics | Use Case |
|----------|-----------------|----------|
| `operational_transform` | Low latency, complex | Text editing, docs |
| `crdt` | Eventually consistent, simple | Counters, sets, maps |
| `lock_based` | Pessimistic, simple | Sequential edits |

### Spatial Data (v1.5)

#### Spatial Type Declaration

```yaml
dataTypes:
  - name: Location
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

#### Spatial Geometry Types

| Geometry | Description | Use Case |
|----------|-------------|----------|
| `point` | Single coordinate | Store location, POI |
| `linestring` | Sequence of points | Route, path, boundary |
| `polygon` | Closed area | Delivery zone, region |
| `multipoint` | Multiple points | Store locations, stops |

#### Spatial Query Examples

```yaml
# Find within radius
spatial_queries:
  - type: within_distance
    center: { lat: 40.7128, lon: -74.0060 }
    distance: "5km"

# Find within polygon
  - type: within_polygon
    boundary: DeliveryZone

# Find nearest
  - type: nearest
    point: UserLocation
    limit: 10
```

### External Integration (v1.5)

#### External Integration Declaration

```yaml
externalDependencies:
  - name: StripePayments
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
          response:
            - source_field: id
              target_field: transaction_id
        retry:
          max_attempts: 3
          backoff: exponential
          idempotency_key: "{{ payment_id }}"
        circuit_breaker:
          enabled: true
          failure_threshold: 5
          timeout: "30s"
```

#### Circuit Breaker States

| State | Behavior | Transition |
|-------|----------|------------|
| `closed` | Normal operation | failures > threshold → open |
| `open` | All calls fail fast | after timeout → half-open |
| `half-open` | Test single call | success → closed, fail → open |

#### Authentication Types

| Type | Configuration | Use Case |
|------|---------------|----------|
| `api_key` | Header or query param | Simple APIs |
| `oauth2` | Token refresh, scopes | User-context APIs |
| `jwt` | Self-signed tokens | Service-to-service |
| `mutual_tls` | Client certificate | High-security APIs |

### Data Governance (v1.5)

#### Data Governance Declaration

```yaml
dataGovernances:
  - name: CustomerDataGovernance
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

#### Field-Level Classification

```yaml
dataTypes:
  - name: Customer
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

#### Compliance Quick Reference

| Regulation | Key Requirements | SMS Mapping |
|------------|------------------|-------------|
| GDPR | Right to erasure, consent | governance.erasure, consent records |
| HIPAA | PHI protection, audit | classification: restricted, audit_all_access |
| SOX | Financial audit trail | audit logging, retention |
| PCI-DSS | Cardholder data protection | encryption, masking, access_controls |

### Graph Queries (v1.5)

#### Graph Query Declaration

```yaml
graphQueries:
  - name: CustomerOrderHistory
    version: v1
    traversal:
      start: Customer
      path:
        - relationship: places
          to: Order
        - relationship: contains
          to: OrderItem
        - relationship: for
          to: Product
    filters:
      - entity: Order
        condition: status == "completed"
      - entity: Order
        condition: created_at > now() - duration("90d")
    projection:
      - entity: Order
        fields: [id, total, created_at]
      - entity: Product
        fields: [name, category]
    pagination:
      strategy: cursor
      default_limit: 50
      max_limit: 200
```

#### Traversal Patterns

| Pattern | Description | Example |
|---------|-------------|---------|
| One-hop | Direct relationship | Customer → Orders |
| Multi-hop | Relationship chain | Customer → Orders → Products |
| Bidirectional | Both directions | Customer ↔ Accounts |
| Filtered | With conditions | Active orders only |

### Delegation & Consent (v1.5)

#### Delegation Chain

```yaml
delegations:
  - name: AccountAccessDelegation
    version: v1
    grantor: Account.owner
    grantee: authorized_user
    scope:
      permissions: [view, transact]
      entity_scope: Account
      constraints:
        max_amount: 1000
        valid_until: "2025-12-31"
    chain:
      max_depth: 2
      allow_further_delegation: false
    audit:
      log_all_actions: true
      notify_grantor: true
```

#### Consent Record

```yaml
consentRecords:
  - name: MarketingConsent
    version: v1
    purpose: marketing_communications
    collection:
      method: explicit
      timestamp_field: consent_given_at
      evidence_field: consent_signature
    scope:
      channels: [email, sms]
      data_categories: [contact_info, preferences]
    withdrawal:
      allowed: true
      method: self_service
      propagate_to: [analytics, crm]
```

---

## PART VII: v1.6 FEATURE QUICK REFERENCES

### Storage Binding Cheat Sheet

#### Declaration Structure

```yaml
storageBindings:
  - name: <string>
    version: <vN>
    type: kv | stream | object_store
    kv:                              # For type: kv
      bucket: <string>
      key_pattern: <pattern>
      serialization: json | protobuf | msgpack
      history: <integer>
      ttl: <duration>
      replicas: <integer>
    stream:                          # For type: stream
      name: <string>
      subjects: [<pattern>, ...]
      retention: limits | interest | workqueue
      max_msgs: <integer>
      max_bytes: <size>
      max_age: <duration>
      storage: file | memory
      replicas: <integer>
    object_store:                    # For type: object_store
      bucket: <string>
      description: <string>
      max_object_size: <size>
      replicas: <integer>
    access:
      read_policy: local | replicated | eventual
      write_consistency: strong | eventual
```

#### Key Pattern Examples

```yaml
# Simple entity key
key_pattern: "account.{{id}}"

# Composite key with realm
key_pattern: "{{realm}}.customer.{{customer_id}}"

# Nested field reference
key_pattern: "order.{{id}}.{{status.code}}"
```

### Cache Layer Cheat Sheet

#### Declaration Structure

```yaml
cacheLayers:
  - name: <string>
    version: <vN>
    topology:
      tier: local | distributed | hybrid
      local:
        max_entries: <integer>
        ttl: <duration>
        eviction: lru | lfu | fifo
        max_size: <size>
      distributed:
        resource: <resource_ref>
        ttl: <duration>
      hybrid:
        promotion: on_miss | on_access
    invalidation:
      strategy: explicit | ttl | watch | hybrid
      ttl: <duration>
      watch_keys: [<pattern>, ...]
      propagate: true | false
    warmup:
      enabled: true | false
      strategy: lazy | eager | scheduled
      on_startup: true | false
      queries: [<query>, ...]
    observability:
      emit_signals: true | false
      metrics: [hit_rate, miss_rate, latency, size, evictions]
      interval: <duration>
```

#### Topology Quick Reference

| Tier | Access Latency | Persistence | Sharing |
|------|---------------|-------------|---------|
| local | < 10μs | Process-local | None |
| distributed | < 5ms | Across restarts | All instances |
| hybrid | Best of both | Both | Promotion-based |

### Intent Routing Cheat Sheet

#### Declaration Structure

```yaml
inputIntents:
  - name: <string>
    version: <vN>
    proposes:
      dataState: <DataState>
    supplies:
      required: [<field>, ...]
      optional: [<field>, ...]
    routing:
      transport: nats | http | grpc
      nats:
        subject_pattern: <pattern>
        delivery: jetstream | core
        stream: <stream_name>
        queue: <queue_group>
        dead_letter: <subject>
      http:
        path: <path_pattern>
        method: POST | PUT | PATCH
        timeout: <duration>
      grpc:
        service: <service_name>
        method: <method_name>
        streaming: true | false
      response:
        mode: request_reply | async | fire_and_forget
        timeout: <duration>
        callback: <pattern | url>
```

#### Transport Selection Guide

| Scenario | Transport | Reason |
|----------|-----------|--------|
| Internal microservices | nats (jetstream) | Native, durable, low latency |
| External API clients | http | Universal compatibility |
| High-performance RPC | grpc | Efficient binary protocol |
| Browser clients | http | Browser-native |
| Event-driven | nats (core) | Lightweight, ephemeral |

### Resource Requirements Cheat Sheet

#### Declaration Structure

```yaml
resourceRequirements:
  name: <string>
  version: <vN>
  description: <string>
  
  eventLogs:
    - name: <string>
      purpose: intent_ingestion | event_sourcing | audit | dead_letter
      ordering: strict | causal | none
      delivery: at_least_once | at_most_once | exactly_once
      durability: persistent | ephemeral
      retention:
        strategy: bounded | unbounded
        max_age: <duration>
        max_size: <size>
        max_count: <integer>
      replication:
        min_replicas: <integer>
        consistency: strong | eventual
  
  entityStores:
    - name: <string>
      purpose: authoritative_state | derived_state | cache | session
      access_pattern: key_value | range_scan | full_scan
      consistency: strong_per_key | strong_global | eventual
      durability: persistent | ephemeral
      history:
        enabled: true | false
        depth: <integer>
      expiration:
        enabled: true | false
        ttl: <duration>
      replication:
        min_replicas: <integer>
      capacity:
        max_entry_size: <size>
        max_entries: <integer>
  
  objectStores:
    - name: <string>
      purpose: documents | media | assets | archives
      durability: persistent | ephemeral
      replication:
        min_replicas: <integer>
      capacity:
        max_object_size: <size>
  
  provisioning:
    mode: auto | validate | strict
    migrations:
      enabled: true | false
      strategy: rolling | blue_green
```

### Client Cache Cheat Sheet

#### Declaration Structure

```yaml
experiences:
  - name: <string>
    version: <vN>
    clientCache:
      enabled: true | false
      storage:
        type: memory | persistent | hybrid
        max_size: <size>
        encryption: true | false
      strategies:
        primary:
          strategy: cache_first | network_first | stale_while_revalidate
          ttl: <duration>
        related:
          strategy: cache_first | network_first | stale_while_revalidate
          ttl: <duration>
          lazy_cache: true | false
      prefetch:
        enabled: true | false
        views: [<view_ref>, ...]
        trigger: on_navigate | on_idle | on_connect
      offline:
        enabled: true | false
        required_views: [<view_ref>, ...]
        sync_on_reconnect: true | false
        conflict_resolution: last_write_wins | server_wins | client_wins | manual
        intent_sync:
          on_cas_failure: reject_and_notify | queue_for_retry
          retry_with_refresh: true | false
        governance:
          validate_on_sync: true | false
          on_violation: reject | queue_for_review | notify_and_proceed
      version_policy:
        on_upgrade: invalidate_all | invalidate_incompatible | keep_compatible
        grace_period: <duration>
        migration: eager | lazy
```

#### Strategy Quick Reference

| Strategy | Behavior | Use Case |
|----------|----------|----------|
| cache_first | Return cached, refresh in background | Read-heavy, tolerate staleness |
| network_first | Fetch first, fallback to cache | Fresh data required |
| stale_while_revalidate | Return stale, update in background | Performance + eventual update |

---

## PART VIII: CHEAT SHEETS & RULES

### Authority Declaration Cheat Sheet (v1.3)

#### Minimal Authority Declaration

```yaml
authorities:
  - scope: entity
    resolution: entity.home_region
    migration:
      allowed: false
```

#### Full Authority with Migration

```yaml
authorities:
  - scope: entity
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

#### CAS Token Structure

```
entity_id        : UUID of entity
entity_version   : Current version number
authority_epoch  : Monotonic epoch counter
authority_region : Current authoritative region
lease_id         : Current lease ID
```

#### Authority State Values

| Status | Meaning | Writes Allowed |
|--------|---------|----------------|
| `ACTIVE` | Region is authoritative | Yes |
| `TRANSITIONING` | Authority transferring | No |

### Invariant Placement Cheat Sheet (v1.3)

| Invariant Type | Placement | Example |
|----------------|-----------|---------|
| Single-field constraint | Entity | `age >= 0` |
| Cross-field constraint | Entity | `end_date > start_date` |
| Cross-entity sum | View | `account.balance >= 0` |
| Business rule | Policy | `user.verified == true` |
| Audit requirement | View + Policy | Transaction history |

#### View Invariant Template

```yaml
views:
  - name: BalanceView
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

#### Invariant Enforcement Options

| Action | Behavior | Use Case |
|--------|----------|----------|
| `reject` | Block the write | Critical constraints |
| `flag` | Allow write, mark for review | Soft constraints |
| `log` | Allow write, log violation | Monitoring only |
| `compensate` | Trigger compensation flow | Eventually consistent |

### UI Composition Cheat Sheet (v1.4)

#### Composition Structure

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

#### View Relationship Types

| Type | When to Use | Example |
|------|-------------|---------|
| `derived` | Computed from primary | Balance from Account |
| `detail` | Child collection | Order items |
| `contextual` | Related entity | Customer of Account |
| `action` | User-triggered form | Transfer form |
| `supplementary` | Additional info | Settings panel |
| `historical` | Audit/history | Activity log |

#### Navigation Purpose Categories

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

### Experience Cheat Sheet (v1.4)

#### Experience Structure

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

#### On_unauthorized Behaviors

| Behavior | Navigation | Content | Use Case |
|----------|------------|---------|----------|
| `conceal` | Hidden | N/A | Customer-facing |
| `indicate` | Visible, disabled | Hint | Admin tools |
| `redirect` | Visible, redirects | Target | SSO flows |
| `deny` | Visible | Error | Debug mode |

### Field Permissions Cheat Sheet (v1.4)

#### Field Permission Structure

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

#### Mask Types

| Type | Example Input | Example Output |
|------|---------------|----------------|
| `full` | `123-45-6789` | `***-**-****` |
| `partial` + `last4` | `123-45-6789` | `***-**-6789` |
| `partial` + `first3` | `123-45-6789` | `123-**-****` |
| `hash` | `123-45-6789` | `a1b2c3d4` |

### Storage Role Cheat Sheet (v1.3)

| Role | Characteristics | Examples |
|------|-----------------|----------|
| `control` | Strongly consistent, small payloads, coordination | Authority KV, leases, locks |
| `data` | Append-only allowed, rebuildable, partitionable | Events, state, views |

#### Control vs Data Role Decision

```yaml
# CONTROL role examples
- Authority tracking: SMS.AUTHORITY KV
- Worker registration: SMS.WORKERS KV
- Lease management: SMS.LEASES KV
- Policy distribution: policy.* subjects

# DATA role examples
- Intent streams: SMS.INTENT.* streams
- Entity state: SMS.*_STATE KV buckets
- View materializations: view worker outputs
- Audit logs: SMS.AUDIT.* streams
```

### Error Code Quick Reference

#### Common Validation Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `unknown_data_type` | Referenced DataType doesn't exist | Define DataType first |
| `version_skip` | Evolution skips version | Add intermediate version |
| `cross_boundary_atomic` | Atomic group spans boundaries | Use compensation instead |
| `missing_idempotency` | Transition without idempotency | Add idempotency spec |
| `duplicate_authority` | Multiple schedulers with authority=true | Set one to authority=false |
| `undefined_work_unit` | Flow references unknown work unit | Define work unit in worker |
| `invalid_compensation` | Compensation references undefined work unit | Define compensation work unit |

#### Common Runtime Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `ErrBoundaryViolation` | Work unit executed in wrong boundary | Check routing logic |
| `ErrNoCompatibleVersion` | Version negotiation failed | Add version compatibility |
| `ErrPolicyDenied` | Authorization failed | Check policy conditions |
| `ErrIdempotencyViolation` | Duplicate processing detected | Idempotency key collision |
| `ErrNoWorkerAvailable` | No worker registered for work unit | Deploy worker |
| `ErrAtomicGroupFailed` | Step in atomic group failed | Check failure logs |
| `ErrCircuitOpen` | Circuit breaker tripped | Investigate downstream health |

### Version Compatibility Matrix

#### Read Compatibility

| From (stored) | To (code) | Backward | Forward |
|---------------|-----------|----------|---------|
| v1 | v2 | New code reads old data | ✅ |
| v2 | v1 | Old code reads new data | ✅ |

**Backward Read**: New code + old data  
**Forward Read**: Old code + new data

#### Write Compatibility

| From (code) | To (stored) | Backward | Forward |
|-------------|-------------|----------|---------|
| v1 | v2 | Old code writes, new code reads | ✅ |
| v2 | v1 | New code writes, old code reads | ✅ |

**Backward Write**: Old code writes → new code reads  
**Forward Write**: New code writes → old code reads

#### Deployment Strategies by Compatibility

| Read | Write | Strategy |
|------|-------|----------|
| Backward | Forward | Deploy code first, migrate data later |
| Forward | Backward | Migrate data first, deploy code later |
| Both | Both | Deploy anytime (ideal) |
| None | None | Requires downtime (avoid) |

### v1.6 Invariants

#### I9. Transport Scope Boundary

> Transport bindings SHALL be specified only at system ingress points (InputIntent routing).
> Transport SHALL NOT be specified within SMS flow definitions.
> Flow steps, atomic groups, and work unit execution SHALL remain transport-agnostic.

**Correct:**
```yaml
# Transport at intent ingress (external contract)
inputIntents:
  - name: InitiateTransfer
    routing:
      transport: nats
      nats:
        subject_pattern: "SMS.INTENT.banking.InitiateTransfer"

# Flow remains transport-agnostic (internal execution)
smsFlows:
  - name: TransferFlow
    triggeredBy:
      inputIntent: InitiateTransfer
    steps:
      - work_unit: ValidateTransfer  # No transport here
      - work_unit: ExecuteTransfer
```

**Incorrect:**
```yaml
# VIOLATION: Transport in flow step
smsFlows:
  - name: TransferFlow
    steps:
      - work_unit: ValidateTransfer
        transport: nats              # VIOLATION
        subject: "SMS.WORK.validate" # VIOLATION
```

### Core Invariants (v1.3-v1.5)

| Invariant | Rule |
|-----------|------|
| I1. Authority Singularity | Single authority per entity |
| I3. No Overlapping Authority | Entity authorities don't overlap |
| I4. CAS Safety | Compare-and-swap for writes |
| I7. Governance Enforcement | Policies enforced at boundaries |
| I9. Transport Scope | Transport only at ingress |

---

## APPENDIX A: SPECIFICATION VALIDATION

### Structural Validation Checklist

- [ ] All DataTypes have unique names
- [ ] All versions use format `v{integer}`
- [ ] All references point to existing elements
- [ ] All expressions use valid syntax
- [ ] All required fields present
- [ ] No circular references in types
- [ ] No duplicate field names within types

### Semantic Validation Checklist

- [ ] Evolution chains are linear (no skips)
- [ ] Atomic groups single boundary
- [ ] Compensations reference existing work units
- [ ] Policies have explicit effects
- [ ] Idempotency keys deterministic
- [ ] One authority per scope
- [ ] Min ≤ max for all placements
- [ ] Quorum ≤ step count
- [ ] Storage bindings match resource requirements
- [ ] Cache layers reference existing resources

### Completeness Validation Checklist

- [ ] All InputIntents have transitions
- [ ] All DataStates have owners
- [ ] All Flows have authorization
- [ ] All Workers declare versions
- [ ] All Policies have scopes
- [ ] All Signals have types
- [ ] All Materializations have sources
- [ ] All storage bindings have access config
- [ ] All cache layers have invalidation strategy

### v1.6-Specific Validation

- [ ] All storageBinding.type matches config block (kv/stream/object_store)
- [ ] All cacheLayer.distributed.resource exists in resourceRequirements
- [ ] All inputIntent.routing.transport has required config
- [ ] No transport specified in flow steps (I9 violation)
- [ ] All client cache strategies are valid
- [ ] All resource requirements have provisioning mode

---

## APPENDIX B: IMPLEMENTATION GUIDE

### Implementation Phases

#### Week 1: Bootstrap
- [ ] Set up project structure
- [ ] Choose parser library (or hand-write)
- [ ] Define AST types
- [ ] Implement lexer
- [ ] Implement parser
- [ ] Add validation rules
- [ ] Write unit tests

#### Week 2: Core Runtime
- [ ] Define runtime interfaces
- [ ] Implement WorkUnit registry
- [ ] Implement FlowEngine (FSM)
- [ ] Implement constraint validator
- [ ] Add execution context
- [ ] Add basic observability
- [ ] Create single-binary demo

#### Week 3: Control Plane
- [ ] Set up NATS (embedded or external)
- [ ] Create JetStream streams
- [ ] Implement artifact publishing
- [ ] Implement artifact loading
- [ ] Add versioning support
- [ ] Test artifact distribution

#### Week 4: Workers
- [ ] Implement Worker runtime
- [ ] Add KV registration
- [ ] Implement heartbeat loop
- [ ] Add subscription management
- [ ] Implement work unit execution
- [ ] Add graceful shutdown

#### Week 5: Policy
- [ ] Implement PolicyRegistry
- [ ] Add policy evaluation engine
- [ ] Implement shadow evaluation
- [ ] Add lifecycle handling
- [ ] Integrate with Workers/SMS/UI
- [ ] Add audit logging

#### Week 6: Optimization
- [ ] Implement routing cache
- [ ] Add locality-first logic
- [ ] Optimize hot paths
- [ ] Add connection pooling
- [ ] Implement batching
- [ ] Performance testing

#### Week 7: Signals
- [ ] Define signal types
- [ ] Implement signal emission
- [ ] Add signal aggregation
- [ ] Create scheduler integration
- [ ] Test autoscaling
- [ ] Add dashboards

#### Week 8: Evolution
- [ ] Implement version negotiation
- [ ] Add shadow write support
- [ ] Implement gradual rollout
- [ ] Add rollback support
- [ ] Test zero-downtime deploy
- [ ] Document procedures

#### Week 9: Multi-Region
- [ ] Regional deployment
- [ ] Cross-region policy sync
- [ ] Regional schedulers
- [ ] Failover testing
- [ ] Partition handling
- [ ] Chaos testing

#### Week 10: Production Hardening
- [ ] Security audit
- [ ] Performance tuning
- [ ] Observability polish
- [ ] Runbook creation
- [ ] Incident procedures
- [ ] Documentation

### Quick Wins

#### Easy Optimizations

1. **Enable Routing Cache**: 100x speedup for lookups
2. **Use Unix Sockets**: 10x faster than HTTP for local calls
3. **Batch Cache Updates**: Reduce lock contention
4. **Compile Policy Expressions**: 50x faster evaluation
5. **Connection Pooling**: Reuse connections
6. **Use L1 Cache**: < 10μs vs < 5ms for L2

#### Common Mistakes to Avoid

1. ❌ Synchronous NATS requests in hot path
2. ❌ No routing cache
3. ❌ Storing state in workers
4. ❌ Skipping shadow validation
5. ❌ Manual cleanup processes
6. ❌ Ignoring graceful shutdown
7. ❌ Inferring missing semantics
8. ❌ Coupling workers directly
9. ❌ Transport in flow definitions (I9 violation)
10. ❌ No cache invalidation strategy

### Getting Unstuck

#### "I don't know where to start"
→ Use Template: Basic CRUD Service  
→ Follow the minimal valid specification  
→ Start with single binary

#### "My specification won't validate"
→ Check Common Validation Errors table  
→ Validate incrementally  
→ Use examples as reference

#### "Performance is poor"
→ Check Performance Targets  
→ Enable routing cache  
→ Use local invocation  
→ Profile hot paths  
→ Check cache hit rates

#### "Deployment failed"
→ Check Troubleshooting Guide  
→ Verify shadow phase completed  
→ Check compatibility declarations  
→ Review rollback procedure

#### "I need a feature not in spec"
→ Check if it's runtime concern (probably is)  
→ Review Design Rationale for "why not"  
→ Consider external integration pattern  
→ Propose extension via RFC process

#### "Cache not working"
→ Check resource reference exists  
→ Verify invalidation strategy  
→ Check warmup configuration  
→ Monitor cache signals

#### "Intent routing fails"
→ Verify subject pattern syntax  
→ Check queue group configuration  
→ Verify worker subscription  
→ Check response timeout

---

## APPENDIX C: ONE-PAGERS

### For Developers

**What**: System Mechanics Spec defines what your system does, not how it does it

**Benefits**:
- Zero-downtime upgrades
- Automatic scaling
- Built-in authorization
- End-to-end tracing
- Multi-region by default
- Declarative caching (v1.6)
- Abstract infrastructure (v1.6)

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
- Caching configurations (v1.6)
- Infrastructure definitions (v1.6)

**Enables**:
- Code generation
- Automated validation
- Safe evolution
- Policy enforcement
- Dynamic optimization
- Multi-tier caching (v1.6)
- Transport abstraction (v1.6)

**Decision**: Use for new systems or modernization efforts

### For Operators

**What**: Declarative specification of system behavior

**Operations Benefits**:
- No manual scaling
- Automatic failover
- Graceful deploys
- Self-healing
- Observable by default
- Cache monitoring (v1.6)
- Resource abstraction (v1.6)

**Requirements**:
- NATS cluster
- JetStream storage
- Scheduler integration (K8s/Nomad)
- Monitoring setup

**Maintenance**: GitOps-friendly, version controlled

---

## APPENDIX D: v1.6 DESIGN CHECKLIST

### Before Starting

- [ ] Identify authoritative data (needs strong consistency)
- [ ] Identify cacheable data (can tolerate staleness)
- [ ] Map entity relationships
- [ ] List external integrations
- [ ] Define invariants and constraints

### Storage Design

- [ ] Choose storage type for each DataState (kv/stream/object_store)
- [ ] Define key patterns for KV stores
- [ ] Set appropriate history depth
- [ ] Configure replication
- [ ] Define retention policies

### Cache Design

- [ ] Identify hot paths for caching
- [ ] Choose cache tier (local/distributed/hybrid)
- [ ] Define invalidation strategy
- [ ] Set appropriate TTLs
- [ ] Configure warmup if needed

### Routing Design

- [ ] Define transport for each ingress intent
- [ ] Configure response modes
- [ ] Set appropriate timeouts
- [ ] Define dead letter handling
- [ ] Keep flows transport-agnostic (I9)

### Client Design

- [ ] Define offline requirements
- [ ] Choose cache strategies per view
- [ ] Configure prefetch patterns
- [ ] Define conflict resolution
- [ ] Set version upgrade policy

---

## GLOSSARY

### v1.6 Terms

- **Cache Layer**: Multi-tier caching configuration
- **Storage Binding**: Declarative storage configuration for persistence
- **Resource Requirements**: Abstract infrastructure specification
- **Intent Routing**: Transport configuration for InputIntents
- **Client Cache**: Browser/mobile caching contract
- **Cache Signal**: Observability for cache layers
- **Pressure Level**: Cache health indicator (low/medium/high/critical)
- **Promotion**: Moving cache entry from L2 to L1 in hybrid cache
- **Invalidation**: Strategy for expiring cached data
- **I9**: Transport Scope Boundary invariant (transport only at ingress)

### Core Terms

- **SMS**: State Machine System (flow orchestration)
- **WTS**: Worker Topology Specification (placement/scaling)
- **CAS**: Compare-And-Swap
- **TTL**: Time To Live
- **EBNF**: Extended Backus-Naur Form
- **CRDT**: Conflict-free Replicated Data Type
- **RBAC**: Role-Based Access Control
- **ABAC**: Attribute-Based Access Control
- **KV**: Key-Value (NATS KV bucket)
- **FSM**: Finite State Machine
- **AST**: Abstract Syntax Tree
- **SLA**: Service Level Agreement
- **P50/P95/P99**: Latency percentiles
- **PII**: Personally Identifiable Information
- **GDPR**: General Data Protection Regulation
- **HIPAA**: Health Insurance Portability and Accountability Act

---

## ADDITIONAL RESOURCES

### Documentation
- **SMS-v1.6-Specification.md**: Complete normative specification
- **SMS-v1.6-EBNF-Grammar.md**: Integrated formal grammar for parser/tooling
- **SMS-v1.6-Implementation-Guide.md**: Detailed implementation patterns
- **SMS-v1.6-Reference-Examples.md**: Working examples and modeling guidance
- **This Document**: Quick reference for rapid lookup

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

**Version**: 1.6  
**Last Updated**: 2026-02-04  
**Status**: Production Ready

This quick reference is maintained alongside the full specification and updated with each release.

## v1.6 Feature Summary

**Storage & Persistence**: Declarative storage binding with KV, stream, and object store support  
**Unified Caching**: Local, distributed, and hybrid cache topologies with invalidation strategies  
**Intent Routing**: NATS, HTTP, and gRPC transport configuration at intent ingress  
**Resource Requirements**: Abstract infrastructure requirements with runtime provisioning  
**Client Caching**: Browser/mobile caching with offline support and sync strategies  
**Cache Signals**: Operational observability for cache layers with pressure monitoring

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
