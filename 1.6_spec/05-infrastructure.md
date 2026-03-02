# System Mechanics Specification - Resource Requirements Semantics v1.6

## Overview

Resource Requirements provides declarative specification of infrastructure capabilities required by SMS applications. This specification defines abstract resource requirements that the runtime provisions using appropriate technology, maintaining the intent-first principle where specifications declare WHAT is needed, not HOW it is implemented.

**Design Principle**: Resource requirements specify capabilities, constraints, and guarantees. The runtime provisioner maps these to concrete implementations (NATS, Kafka, Redis, Postgres, etc.) based on deployment context.

---

## Invariant Compliance

### Intent-First Infrastructure

This specification extends the core principle:

> **"Intent-First, Runtime-Decided"**
> - The specification declares intent, guarantees, and prohibitions
> - Execution strategy, transport, locality, and optimization are runtime responsibilities

**Application to Infrastructure**:

| Specification Declares | Runtime Decides |
|----------------------|-----------------|
| Message ordering guarantees | NATS JetStream vs Kafka vs Pulsar |
| Durability requirements | Replication factor, storage engine |
| Consistency semantics | Implementation-specific protocols |
| Retention policies | Backend-specific configuration |
| Capacity constraints | Sizing, partitioning, sharding |

**Rationale**: By specifying abstract requirements, the same SMS specification can be deployed across different infrastructure backends without modification.

---

## Resource Requirements Declaration

### ResourceRequirements

Defines abstract infrastructure resources required by the SMS application.

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
        max_size: <size_expression>
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
        max_entry_size: <size_expression>
        max_entries: <integer>
  
  objectStores:
    - name: <string>
      purpose: documents | media | assets | archives
      durability: persistent | ephemeral
      replication:
        min_replicas: <integer>
      capacity:
        max_object_size: <size_expression>
  
  provisioning:
    mode: auto | validate | strict
    migrations:
      enabled: true | false
      strategy: rolling | blue_green
```

---

## Event Log Requirements

### Purpose Categories

Event logs capture ordered sequences of messages for different use cases.

| Purpose | Description | Typical Requirements |
|---------|-------------|---------------------|
| `intent_ingestion` | InputIntent submission stream | at_least_once, bounded retention |
| `event_sourcing` | Domain event history | at_least_once, long retention |
| `audit` | Compliance and audit trail | exactly_once, immutable, long retention |
| `dead_letter` | Failed message capture | at_least_once, bounded retention |

### Ordering Guarantees

| Ordering | Description | Use Case |
|----------|-------------|----------|
| `strict` | Total ordering across all messages | Single-writer scenarios |
| `causal` | Ordering preserved per entity/key | Entity-scoped operations |
| `none` | No ordering guarantees | Independent events |

### Delivery Semantics

| Delivery | Description | Trade-off |
|----------|-------------|-----------|
| `at_least_once` | Message delivered one or more times | Requires idempotent consumers |
| `at_most_once` | Message delivered zero or one time | May lose messages |
| `exactly_once` | Message delivered exactly once | Higher overhead, limited backend support |

### Example: Intent Ingestion Log

```yaml
eventLogs:
  - name: IntentLog
    purpose: intent_ingestion
    ordering: causal
    delivery: at_least_once
    durability: persistent
    retention:
      strategy: bounded
      max_age: 90d
      max_size: 50GB
    replication:
      min_replicas: 3
      consistency: strong
```

### Example: Event Sourcing Log

```yaml
eventLogs:
  - name: DomainEventLog
    purpose: event_sourcing
    ordering: causal
    delivery: at_least_once
    durability: persistent
    retention:
      strategy: bounded
      max_age: 2555d  # 7 years for compliance
      max_size: 500GB
    replication:
      min_replicas: 3
      consistency: strong
```

---

## Entity Store Requirements

### Purpose Categories

Entity stores provide key-value access to entity state.

| Purpose | Description | Typical Requirements |
|---------|-------------|---------------------|
| `authoritative_state` | Source of truth for entities | Strong consistency, history |
| `derived_state` | Computed/materialized state | Eventual consistency, rebuildable |
| `cache` | Performance optimization layer | TTL, eventual consistency |
| `session` | User session data | TTL, ephemeral acceptable |

### Access Patterns

| Pattern | Description | Backend Implications |
|---------|-------------|---------------------|
| `key_value` | Single-key lookups | Optimized for point queries |
| `range_scan` | Key range queries | Supports ordered iteration |
| `full_scan` | Full store iteration | May require special indexing |

### Consistency Levels

| Level | Description | Use Case |
|-------|-------------|----------|
| `strong_per_key` | Linearizable per key | Entity state, most common |
| `strong_global` | Linearizable globally | Cross-entity transactions |
| `eventual` | Eventually consistent | Caches, derived data |

### Example: Authoritative Entity Store

```yaml
entityStores:
  - name: AccountState
    purpose: authoritative_state
    access_pattern: key_value
    consistency: strong_per_key
    durability: persistent
    history:
      enabled: true
      depth: 10
    expiration:
      enabled: false
    replication:
      min_replicas: 3
    capacity:
      max_entry_size: 1MB
```

### Example: View Cache Store

```yaml
entityStores:
  - name: ViewCache
    purpose: cache
    access_pattern: key_value
    consistency: eventual
    durability: persistent
    history:
      enabled: false
    expiration:
      enabled: true
      ttl: 1h
    replication:
      min_replicas: 3
    capacity:
      max_entry_size: 10MB
```

### Example: Session Store

```yaml
entityStores:
  - name: SessionStore
    purpose: session
    access_pattern: key_value
    consistency: strong_per_key
    durability: ephemeral
    history:
      enabled: false
    expiration:
      enabled: true
      ttl: 24h
    replication:
      min_replicas: 3
    capacity:
      max_entry_size: 100KB
```

---

## Object Store Requirements

### Purpose Categories

Object stores handle large binary data.

| Purpose | Description | Typical Requirements |
|---------|-------------|---------------------|
| `documents` | User-uploaded documents | Persistent, replicated |
| `media` | Images, video, audio | Persistent, CDN-friendly |
| `assets` | Application assets | Persistent, versioned |
| `archives` | Long-term storage | Persistent, cold storage ok |

### Example: Document Store

```yaml
objectStores:
  - name: CustomerDocuments
    purpose: documents
    durability: persistent
    replication:
      min_replicas: 3
    capacity:
      max_object_size: 100MB
```

---

## Provisioning Configuration

### Provisioning Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| `auto` | Create resources if missing | Development, staging |
| `validate` | Verify resources exist, warn on drift | Production with IaC |
| `strict` | Fail if resources missing or misconfigured | Production, regulated |

### Migration Strategies

| Strategy | Description | Downtime |
|----------|-------------|----------|
| `rolling` | Gradual migration, old and new coexist | Zero |
| `blue_green` | Create new, switch, retire old | Minimal |

---

## Complete Example

### Banking Resource Requirements

```yaml
resourceRequirements:
  name: BankingResources
  version: v1
  description: "Resource requirements for SMS Banking application"
  
  eventLogs:
    # Intent ingestion
    - name: BankingIntentLog
      purpose: intent_ingestion
      ordering: causal
      delivery: at_least_once
      durability: persistent
      retention:
        strategy: bounded
        max_age: 90d
        max_size: 50GB
      replication:
        min_replicas: 3
        consistency: strong
    
    # Domain events for event sourcing
    - name: BankingEventLog
      purpose: event_sourcing
      ordering: causal
      delivery: at_least_once
      durability: persistent
      retention:
        strategy: bounded
        max_age: 2555d  # 7 years for compliance
        max_size: 500GB
      replication:
        min_replicas: 3
        consistency: strong
    
    # Dead letter queue
    - name: BankingDeadLetterLog
      purpose: dead_letter
      ordering: none
      delivery: at_least_once
      durability: persistent
      retention:
        strategy: bounded
        max_age: 30d
      replication:
        min_replicas: 3
        consistency: eventual
  
  entityStores:
    # Account authoritative state
    - name: AccountState
      purpose: authoritative_state
      access_pattern: key_value
      consistency: strong_per_key
      durability: persistent
      history:
        enabled: true
        depth: 10
      expiration:
        enabled: false
      replication:
        min_replicas: 3
      capacity:
        max_entry_size: 1MB
    
    # Transaction records
    - name: TransactionState
      purpose: authoritative_state
      access_pattern: key_value
      consistency: strong_per_key
      durability: persistent
      history:
        enabled: false
      expiration:
        enabled: false
      replication:
        min_replicas: 3
      capacity:
        max_entry_size: 100KB
    
    # Materialized view cache
    - name: ViewCache
      purpose: cache
      access_pattern: key_value
      consistency: eventual
      durability: persistent
      history:
        enabled: false
      expiration:
        enabled: true
        ttl: 1h
      replication:
        min_replicas: 3
      capacity:
        max_entry_size: 10MB
    
    # User sessions
    - name: SessionStore
      purpose: session
      access_pattern: key_value
      consistency: strong_per_key
      durability: ephemeral
      history:
        enabled: false
      expiration:
        enabled: true
        ttl: 24h
      replication:
        min_replicas: 3
      capacity:
        max_entry_size: 50KB
  
  objectStores:
    # Customer documents
    - name: CustomerDocuments
      purpose: documents
      durability: persistent
      replication:
        min_replicas: 3
      capacity:
        max_object_size: 100MB
  
  provisioning:
    mode: auto
    migrations:
      enabled: true
      strategy: rolling
```

---

## Binding to Other Declarations

### StorageBinding Reference

StorageBindings reference resource requirements:

```yaml
storageBinding:
  name: AccountStateStorage
  version: v1
  type: entity_store
  entityStore:
    resource: AccountState  # References resourceRequirements.entityStores[].name
    key_pattern: "account.{{account_id}}"
```

### IntentRouting Reference

IntentRouting references event logs:

```yaml
inputIntent:
  name: InitiateTransfer
  routing:
    eventLog: BankingIntentLog  # References resourceRequirements.eventLogs[].name
    # Transport (nats, kafka, etc.) is runtime-decided based on deployment
```

### CacheLayer Reference

CacheLayers reference entity stores:

```yaml
cacheLayer:
  name: ViewCacheLayer
  topology:
    tier: distributed
    distributed:
      resource: ViewCache  # References resourceRequirements.entityStores[].name
```

---

## Runtime Provisioner Mapping

The runtime provisioner maps abstract requirements to concrete implementations:

### Example Mappings

| Requirement | NATS Implementation | Kafka Implementation | Redis Implementation |
|-------------|--------------------|--------------------|---------------------|
| EventLog (at_least_once) | JetStream Stream | Topic with acks | Stream with XACK |
| EntityStore (strong_per_key) | KV Bucket | Compacted Topic + Cache | Hash with Lua scripts |
| EntityStore (eventual) | KV Bucket (single replica) | Topic + Local Cache | Standard Hash |
| ObjectStore | Object Store | N/A (use S3) | N/A (use S3) |

### Provisioner Behavior

```
1. Load resource requirements
2. Detect available infrastructure backends
3. Select optimal backend per resource type
4. Map abstract requirements to backend config:
   - ordering: causal → NATS subjects with key, Kafka partitioning
   - delivery: at_least_once → enable acks, replay
   - retention: bounded → set limits, TTL
   - replication: min_replicas → replica count, consistency
5. Provision resources
6. Report mapping and status
```

### Provisioning Report

```yaml
provisioningReport:
  status: success | partial | failed
  backend: nats | kafka | redis | postgres
  mappings:
    - resource: BankingIntentLog
      backend: nats_jetstream
      config:
        stream: SMS_BANKING_INTENTS
        subjects: ["SMS.INTENT.banking.>"]
        replicas: 3
    - resource: AccountState
      backend: nats_kv
      config:
        bucket: SMS_ACCOUNT_STATE
        history: 10
        replicas: 3
  warnings: []
```

---

## Validation Rules

### Structural Violations (MUST reject)

1. EventLog without name
2. EntityStore without name
3. ObjectStore without name
4. Invalid ordering value
5. Invalid delivery value
6. Invalid consistency value
7. min_replicas less than 1

### Semantic Violations (MUST reject)

1. Duplicate resource names across all types
2. `exactly_once` delivery with `ephemeral` durability
3. `history.enabled: true` with `purpose: cache`
4. `strong_global` consistency with `ephemeral` durability
5. `purpose: authoritative_state` with `eventual` consistency (warning)

### Reference Violations (MUST reject)

1. StorageBinding referencing non-existent resource
2. IntentRouting referencing non-existent event log
3. CacheLayer referencing non-existent entity store

---

## EBNF Grammar

```ebnf
resource_requirements ::= "resourceRequirements:" NEWLINE INDENT
                          "name:" identifier NEWLINE
                          ( "version:" version NEWLINE )?
                          ( "description:" string NEWLINE )?
                          event_logs_config?
                          entity_stores_config?
                          object_stores_config?
                          provisioning_config?
                          DEDENT

event_logs_config   ::= "eventLogs:" NEWLINE INDENT
                        event_log_decl+
                        DEDENT

event_log_decl      ::= "- name:" identifier NEWLINE INDENT
                        ( "purpose:" event_log_purpose NEWLINE )?
                        ( "ordering:" ordering_guarantee NEWLINE )?
                        ( "delivery:" delivery_semantic NEWLINE )?
                        ( "durability:" durability_type NEWLINE )?
                        ( retention_config )?
                        ( replication_config )?
                        DEDENT

event_log_purpose   ::= "intent_ingestion" | "event_sourcing" | "audit" | "dead_letter"

ordering_guarantee  ::= "strict" | "causal" | "none"

delivery_semantic   ::= "at_least_once" | "at_most_once" | "exactly_once"

durability_type     ::= "persistent" | "ephemeral"

retention_config    ::= "retention:" NEWLINE INDENT
                        ( "strategy:" retention_strategy NEWLINE )?
                        ( "max_age:" duration NEWLINE )?
                        ( "max_size:" size_expression NEWLINE )?
                        ( "max_count:" integer NEWLINE )?
                        DEDENT

retention_strategy  ::= "bounded" | "unbounded"

replication_config  ::= "replication:" NEWLINE INDENT
                        ( "min_replicas:" integer NEWLINE )?
                        ( "consistency:" replication_consistency NEWLINE )?
                        DEDENT

replication_consistency ::= "strong" | "eventual"

entity_stores_config ::= "entityStores:" NEWLINE INDENT
                         entity_store_decl+
                         DEDENT

entity_store_decl   ::= "- name:" identifier NEWLINE INDENT
                        ( "purpose:" entity_store_purpose NEWLINE )?
                        ( "access_pattern:" access_pattern NEWLINE )?
                        ( "consistency:" store_consistency NEWLINE )?
                        ( "durability:" durability_type NEWLINE )?
                        ( history_config )?
                        ( expiration_config )?
                        ( replication_config )?
                        ( capacity_config )?
                        DEDENT

entity_store_purpose ::= "authoritative_state" | "derived_state" | "cache" | "session"

access_pattern      ::= "key_value" | "range_scan" | "full_scan"

store_consistency   ::= "strong_per_key" | "strong_global" | "eventual"

history_config      ::= "history:" NEWLINE INDENT
                        ( "enabled:" boolean NEWLINE )?
                        ( "depth:" integer NEWLINE )?
                        DEDENT

expiration_config   ::= "expiration:" NEWLINE INDENT
                        ( "enabled:" boolean NEWLINE )?
                        ( "ttl:" duration NEWLINE )?
                        DEDENT

capacity_config     ::= "capacity:" NEWLINE INDENT
                        ( "max_entry_size:" size_expression NEWLINE )?
                        ( "max_entries:" integer NEWLINE )?
                        DEDENT

object_stores_config ::= "objectStores:" NEWLINE INDENT
                         object_store_decl+
                         DEDENT

object_store_decl   ::= "- name:" identifier NEWLINE INDENT
                        ( "purpose:" object_store_purpose NEWLINE )?
                        ( "durability:" durability_type NEWLINE )?
                        ( replication_config )?
                        ( object_capacity_config )?
                        DEDENT

object_store_purpose ::= "documents" | "media" | "assets" | "archives"

object_capacity_config ::= "capacity:" NEWLINE INDENT
                           ( "max_object_size:" size_expression NEWLINE )?
                           DEDENT

provisioning_config ::= "provisioning:" NEWLINE INDENT
                        ( "mode:" provisioning_mode NEWLINE )?
                        ( migrations_config )?
                        DEDENT

provisioning_mode   ::= "auto" | "validate" | "strict"

migrations_config   ::= "migrations:" NEWLINE INDENT
                        ( "enabled:" boolean NEWLINE )?
                        ( "strategy:" migration_strategy NEWLINE )?
                        DEDENT

migration_strategy  ::= "rolling" | "blue_green"
```

---

## Conformance

An implementation is conformant with Resource Requirements semantics if it:

1. Parses and validates all resource requirement declarations
2. Maps abstract requirements to appropriate backend implementations
3. Provisions resources with equivalent or stronger guarantees than specified
4. Honors ordering guarantees per requirement
5. Implements delivery semantics correctly
6. Respects durability requirements
7. Enforces retention policies
8. Maintains replication minimums
9. Respects provisioning mode on startup
10. Validates references from other declarations
11. Reports provisioning mappings accurately
12. Supports rolling updates without downtime

---

## Specification Metadata

```yaml
specMetadata:
  feature: resource-requirements
  version: v1.6
  status: normative
  invariant_compliance:
    principle: "Intent-First, Runtime-Decided"
    application: "Specification declares capabilities; runtime selects implementation"
  dependencies:
    - storageBinding
    - cacheLayer
    - intentRouting
  changelog:
    v1.6:
      - "Renamed from infrastructure to resource-requirements"
      - "Abstract resource types (eventLog, entityStore, objectStore)"
      - "Capability-based specification (ordering, delivery, consistency)"
      - "Purpose categorization for semantic clarity"
      - "Runtime provisioner mapping to backends"
      - "Technology-agnostic, intent-first design"
      - "Explicit invariant compliance documentation"
```
