# SMS v1.6 Reference Examples

## For LLM Learning

This document consolidates modeling guidance, examples, and patterns for learning and applying the System Mechanics Specification v1.6. Use this document to:

- **Understand invariant-oriented design** - Learn the philosophy behind SMS modeling
- **Model new applications** - Follow the playbook for entity, relationship, and view design
- **Find patterns** - Reference common patterns for typical requirements
- **Learn by example** - Study complete worked examples from simple to complex
- **Validate designs** - Use checklists and decision trees to verify your models
- **Master v1.6 features** - Storage binding, caching, routing, and resource requirements

**Document Status**: Normative Guidance and Examples  
**Version**: 1.6  
**Last Updated**: 2026-02-04

**Related Documents:**
- **SMS-v1.6-Specification.md**: Normative specification with grammar definitions
- **SMS-v1.6-EBNF-Grammar.md**: Formal grammar for parser/tooling development
- **SMS-v1.6-Implementation-Guide.md**: Detailed implementation patterns
- **SMS-v1.6-Quick-Reference.md**: Rapid-lookup cheat sheets and decision trees

---

## Table of Contents

- [Part I: Modeling Guidance](#part-i-modeling-guidance)
- [Part II: Complete Banking Example (All Features)](#part-ii-complete-banking-example-all-features)
- [Part III: v1.6 Feature Examples](#part-iii-v16-feature-examples)
- [Part IV: Grammar Quick Examples](#part-iv-grammar-quick-examples)
- [Part V: Common Patterns Library](#part-v-common-patterns-library)
- [Appendix A: v1.5 Feature Examples](#appendix-a-v15-feature-examples)
- [Appendix B: Migration Examples](#appendix-b-migration-examples)
- [Appendix C: v1.6 Design Checklist](#appendix-c-v16-design-checklist)

---

# Part I: Modeling Guidance

## Core Philosophy

### The Problem

Many systems try to enforce complex invariants (e.g., "account balance must be non-negative") by:
- Embedding invariants in single entities
- Using distributed transactions
- Creating global locks

These approaches create:
- Hidden coupling between entities
- Availability dependencies
- Scaling bottlenecks
- Complex failure modes

### The SMS Solution

SMS decomposes invariants across:
1. **Entities**: Independent, authority-scoped units of state
2. **Relationships**: Semantic links with cardinality and ordering
3. **Views**: Derived state that aggregates and validates

This enables:
- Entity independence (each writable without coordination)
- Eventual consistency (invariants validated asynchronously)
- Failure isolation (violations detected, not blocking)

---

## The First Question

Before writing any model, ask:

> **"What must never be wrong?"**

This identifies your invariants. Everything else flows from this.

### Examples

| Domain | Invariant |
|--------|-----------|
| Banking | Account balance must be non-negative |
| Inventory | Stock count must match physical items |
| Scheduling | No overlapping appointments |
| Voting | One vote per eligible voter |
| Billing | Invoice total equals sum of line items |

---

## Invariant Classification

### Step 1: List All Invariants

Write down every correctness condition.

**Example (Banking)**:
1. Account balance >= 0
2. Transaction amount > 0
3. Transfer debits match credits
4. Account must exist before transactions
5. Closed accounts cannot receive funds

### Step 2: Classify Each Invariant

| Invariant | Scope | Enforcement |
|-----------|-------|-------------|
| balance >= 0 | Cross-entity | View |
| amount > 0 | Single-entity | Entity constraint |
| debits = credits | Cross-entity | View |
| Account before Transaction | Cross-entity | Relationship |
| No funds to closed | Business rule | Policy |

---

## Entity Design Rules

### Rule E1: Entities Are Authority-Independent

Each entity should be writable without coordination with other entities.

**Good**:
```yaml
models:
  - name: Account
    mutability:
      scope: entity
      exclusive: true
```

**Bad**:
```yaml
# Anti-pattern: Entity depends on another entity's state for writes
models:
  - name: Transfer
    precondition: source_account.balance >= amount  # Creates dependency
```

### Rule E2: Entities Contain Identity and Constraints, Not Derived State

Entities should store their identity and local constraints, not values that depend on other entities.

**Good**:
```yaml
dataTypes:
  - name: Account
    fields:
      id: uuid
      owner_id: uuid
      created_at: timestamp
      account_type: enum[checking, savings]
```

**Bad**:
```yaml
# Anti-pattern: Balance is derived from transactions
dataTypes:
  - name: Account
    fields:
      id: uuid
      balance: decimal  # Requires coordination to update
```

### Rule E3: Append-Only for Immutable Facts

Facts that never change should be append-only.

```yaml
models:
  - name: Transaction
    mutability:
      scope: append-only
```

---

## Entity Design Checklist

For each entity, verify:

- [ ] **Independence**: Can be written without reading other entities
- [ ] **Identity**: Has clear, immutable identifier
- [ ] **No derived state**: Does not contain values calculated from other entities
- [ ] **Authority scope**: Has explicit authority declaration
- [ ] **Local constraints**: All single-entity invariants declared

### Entity Template

```yaml
models:
  - name: <EntityName>
    
    mutability:
      scope: entity | append-only
      exclusive: true
    
    authority:
      scope: entity
      resolution: <expression>
      migration:
        allowed: true | false

dataTypes:
  - name: <EntityName>
    version: v1
    fields:
      id: uuid
      # Identity fields only
      # NO derived or aggregate fields

constraints:
  - name: <EntityName>_valid
    appliesTo: <EntityName>
    rules:
      v1: <single-entity constraints>
```

---

## Relationship Design Rules

### Rule R1: Relationships Express Semantic Links

Relationships describe how entities relate, not how they coordinate.

```yaml
relationships:
  - name: debit
    from: Transaction
    to: Account
    type: affects
    cardinality: many-to-one
    semantics:
      causal: true
      ordered: true
```

### Rule R2: Relationships Do Not Imply Transactions

A relationship says "Transaction affects Account" - it does NOT say "Transaction and Account must be updated atomically."

**What relationships imply**:
- Semantic linkage
- Causality ordering
- Reference validation

**What relationships do NOT imply**:
- Synchronous coordination
- Shared authority
- Atomic multi-entity mutation

### Rule R3: Cardinality Constrains Valid Instances

Use cardinality to express valid relationship counts:

| Cardinality | Meaning |
|-------------|---------|
| `one-to-one` | Each source maps to exactly one target |
| `one-to-many` | Each source maps to many targets |
| `many-to-one` | Many sources map to one target |
| `many-to-many` | Many sources map to many targets |

---

## Relationship Design Checklist

For each relationship, verify:

- [ ] **Semantic meaning**: Describes how entities relate
- [ ] **Cardinality**: Valid counts specified
- [ ] **Causality**: Whether ordering matters
- [ ] **No coordination**: Does not imply synchronous updates

### Relationship Template

```yaml
relationships:
  - name: <verb>
    from: <SourceEntity>
    to: <TargetEntity>
    type: <relationship_type>
    cardinality: one-to-one | one-to-many | many-to-one | many-to-many
    semantics:
      causal: true | false
      ordered: true | false
```

---

## View Design Rules

### Rule V1: Views Absorb Complexity

Views combine data from multiple entities and relationships into queryable, validated state.

```yaml
views:
  - name: AccountBalance
    inputs:
      - Account
      - Transaction
    aggregation:
      type: sum
      expression: sum(credits) - sum(debits)
    invariant:
      expression: result >= 0
      on_violation: flag
```

### Rule V2: Views Are Authority-Agnostic

Views must tolerate events from multiple regions and epochs.

```yaml
views:
  - authority_agnostic: true
    epoch_tolerant: true
```

### Rule V3: Views Validate, Not Block

Views detect invariant violations after the fact rather than preventing writes.

**Violation handling options**:
- `flag`: Mark violation for review
- `log`: Record violation in audit
- `reject`: Block reads of invalid state

---

## View Design Checklist

For each view, verify:

- [ ] **Derived only**: All data reconstructible from entities
- [ ] **Authority agnostic**: Tolerates multi-region events
- [ ] **Invariant declared**: Cross-entity invariants specified
- [ ] **Violation handling**: Clear action on invariant breach

### View Template

```yaml
views:
  - name: <ViewName>
    
    inputs:
      - <Entity1>
      - <Entity2>
    
    aggregation:
      type: <aggregation_type>
      expression: <expression>
    
    invariant:
      expression: <cross-entity invariant>
      on_violation: flag | log | reject
    
    authority_agnostic: true
    epoch_tolerant: true
```

---

## Invariant Placement Table

| Invariant Type | Placement | Enforcement | Example |
|----------------|-----------|-------------|---------|
| Single-field constraint | Entity | Write-time | `age >= 0` |
| Cross-field constraint | Entity | Write-time | `end > start` |
| Cross-entity aggregate | View | Read-time | `balance >= 0` |
| Business rule | Policy | Evaluation-time | `user.verified` |
| Audit requirement | View + Policy | Continuous | Transaction history |

---

## Authority Migration Guidance

### When to Allow Migration

| Scenario | Recommendation |
|----------|----------------|
| User-local data | ✅ Allow with latency triggers |
| Financial records | ⚠️ Prefer stable authority |
| High write contention | ❌ Disable migration |
| Follow-the-sun workloads | ✅ Allow with time triggers |
| Regulatory-bound data | ❌ Constrain by jurisdiction |

### Migration Configuration

```yaml
authorities:
  - migration:
      allowed: true
      strategy: pause-and-cutover
      
      triggers:
        # Performance-based
        - type: latency
          condition: avg_write_latency > 100ms
        
        # Pattern-based
        - type: load
          condition: primary_access_region != authority_region
        
        # Manual
        - type: manual
          condition: operator_request
        
        # Scheduled
        - type: time
          condition: time_of_day in follow_sun_schedule
      
      constraints:
        - type: region
          allowed: [us-east, eu-west, ap-southeast]
        - type: governance
          rule: entity.classification != "restricted"
```

---

## Authority Interaction Rules

### Rule A1: Invariants Must Not Create Authority Dependencies

An invariant that spans multiple entities must not require coordinated writes.

**Good**: View calculates balance from transactions (no coordination)  
**Bad**: Transfer requires locking both accounts (creates dependency)

### Rule A2: Authority Scope Matches Entity Scope

If entities are independently authoritative, their invariants must be independently enforceable.

### Rule A3: View Continuity During Authority Changes

Views must remain readable during authority transitions.

```yaml
views:
  - authority_agnostic: true  # Tolerates events from multiple regions
```

---

## Common Anti-Patterns

### Anti-Pattern 1: Balance as Entity Field

**Problem**: Storing balance in Account requires coordinated updates.

```yaml
# BAD
dataTypes:
  - name: Account
    fields:
      balance: decimal  # Who updates this?
```

**Solution**: Derive balance in view.

```yaml
# GOOD
views:
  - name: AccountBalance
    inputs: [Account, Transaction]
    aggregation:
      expression: sum(credits) - sum(debits)
```

### Anti-Pattern 2: Cross-Entity Precondition

**Problem**: Write to entity A depends on state of entity B.

```yaml
# BAD
transition:
  precondition: target_account.accepts_deposits == true
```

**Solution**: Validate in view, handle violations gracefully.

### Anti-Pattern 3: Global Lock for Consistency

**Problem**: Acquire lock across multiple entities.

**Solution**: Accept eventual consistency, detect and handle violations.

### Anti-Pattern 4: Embedding Relationship State

**Problem**: Storing relationship count in entity.

```yaml
# BAD
dataTypes:
  - name: Account
    fields:
      transaction_count: integer  # Derived, requires coordination
```

**Solution**: Calculate in view.

---

## Modeling Decision Tree

```
Start with invariant
       │
       ▼
Does it span multiple entities?
       │
    ┌──┴──┐
   NO    YES
    │      │
    ▼      ▼
Entity   Must it hold synchronously?
Constraint      │
            ┌───┴───┐
           NO      YES
            │        │
            ▼        ▼
         View    Reconsider
       Invariant   Model
```

---

## v1.5 Modeling Extensions

### Assets (Addendum AK)

When modeling entities that include binary content (images, documents, videos), use asset semantics:

**Decision Point**: Does your entity reference files?
- **YES** → Use `asset` type with lifecycle, transformations, and retention

```yaml
# Example: Customer profile with photo
dataTypes:
  - name: Customer
    fields:
      id: uuid
      name: string
      profile_photo: asset

assets:
  - name: profile_photo
    types: [image/jpeg, image/png]
    max_size: 5MB
    transformations:
      thumbnail: resize(width: 200, height: 200)
    retention:
      duration: account_lifetime
      delete_on_entity_deletion: true
```

---

### Search (Addendum AL)

When users need to find entities by complex criteria, model search explicitly:

**Decision Point**: Will users search for this entity?
- **YES** → Define search configuration with fields, weights, and ranking

```yaml
# Example: Search customers by name, email, phone
searchIntents:
  - name: CustomerSearch
    target: CustomerView
    fields:
      - name: name
        weight: 2.0
        analyzers: [standard, edge_ngram]
      - name: email
        weight: 1.5
        analyzers: [keyword]
      - name: phone
        weight: 1.0
        analyzers: [keyword]
    ranking:
      function: bm25
```

---

### Spatial Data (Addendum AR)

When modeling location-based entities or constraints:

**Decision Point**: Does location matter for this entity?
- **YES** → Use `geo_point` or `geo_shape` types

```yaml
# Example: Store with service area
dataTypes:
  - name: Store
    fields:
      id: uuid
      name: string
      location: geo_point
      service_area: geo_shape

constraints:
  - name: ServiceAreaValid
    appliesTo: Store
    rules:
      v1: within(location, service_area)
```

---

### Scheduled Actions (Addendum AM)

When entities require time-based automation:

**Decision Point**: Does this entity need recurring or scheduled actions?
- **YES** → Define scheduled triggers

```yaml
# Example: Monthly account statement generation
scheduledTriggers:
  - name: GenerateMonthlyStatement
    schedule: "0 0 1 * *"  # First day of month at midnight
    intent: GenerateStatement
    scope: per_entity
    entity_filter: account.status == active
```

---

### External Integration (Addendum AW)

When entities depend on external services:

**Decision Point**: Does this entity interact with third-party systems?
- **YES** → Model external integration with resilience patterns

```yaml
# Example: Credit score from external bureau
externalDependencies:
  - name: CreditBureauAPI
    endpoint: credit_bureau
    operation: getCreditScore
    circuit_breaker:
      threshold: 5
      timeout: 30s
      reset_after: 60s
    fallback:
      strategy: use_cached
      cache_ttl: 24h
```

---

### Data Governance (Addendum AX)

For every entity containing sensitive data, define governance:

**Decision Point**: Does this entity contain PII or regulated data?
- **YES** → Define classification, retention, and erasure policy

```yaml
# Example: Customer with PII
dataGovernances:
  - model: Customer
    classification:
      level: pii
      categories: [personal_identifiable, financial]
    retention:
      duration: 7_years
      delete_after: true
    right_to_erasure:
      supported: true
      cascade: [AuditLog, Transaction]
```

---

### Structured Content (Addendum AP)

When entities need rich text or formatted content:

**Decision Point**: Does this entity need more than plain text?
- **YES** → Use structured content type

```yaml
# Example: Product with formatted description
dataTypes:
  - name: Product
    fields:
      id: uuid
      name: string
      description: structured_content

structuredContents:
  - format: markdown
    sanitization: strict
    max_length: 50000
```

---

### Durable Workflows (Addendum AT)

For long-running processes spanning multiple entities:

**Decision Point**: Does this process take days/weeks with human steps?
- **YES** → Model as durable workflow, not SMS flow

```yaml
# Example: Comprehensive loan approval process with human steps
durableWorkflows:
  - name: LoanApproval
    trigger:
      intent: SubmitLoanApplication
    steps:
      - name: creditCheck
        action: callExternalIntegration
        integration: CreditBureauAPI
        timeout: 5m
        retry:
          max_attempts: 3
        compensation: logCreditCheckFailure
      
      - name: automaticDecision
        action: evaluatePolicy
        policy: LoanAutoApprovalPolicy
        branches:
          auto_approved:
            condition: credit_score > 750 AND amount < 50000
            next: approved
          auto_rejected:
            condition: credit_score < 600
            next: rejected
          manual_review:
            condition: true
            next: manualReview
      
      - name: manualReview
        action: human_task
        assigned_to: underwriting_team
        timeout: 3_days
        escalation:
          after: 2_days
          to: senior_underwriter
        ui: LoanReviewComposition
        branches:
          approved: { next: approved }
          rejected: { next: rejected }
          request_more_info: { next: waitForCustomer }
      
      - name: waitForCustomer
        action: wait_for_event
        event: CustomerDocumentsUploaded
        timeout: 7_days
        on_timeout: rejected
        on_event: manualReview
      
      - name: approved
        action: submit_intent
        intent: ApproveLoan
        notification: LoanApprovedNotification
        end: true
      
      - name: rejected
        action: submit_intent
        intent: RejectLoan
        notification: LoanRejectedNotification
        end: true
    
    state_persistence:
      duration: 90_days
    observability:
      track_step_duration: true
      alert_on_timeout: true

# v1.6 Enhancement: Storage Binding for workflow state
storageBindings:
  - name: WorkflowStateStore
    storage_type: stream
    stream:
      retention: 90_days
      replicas: 3
    access:
      read_policy: sequential
      durability: persistent
```

**Key Features Demonstrated**:
- Multi-day workflow with human decision points
- Automatic branching based on policy evaluation
- Escalation for time-sensitive tasks
- Wait-for-event pattern for asynchronous customer input
- Compensation logic for rollback scenarios
- State persistence for fault tolerance
- Integration with external APIs (credit bureau)
- Notification on completion

**v1.6 Enhancements**: Workflow state stored in durable stream with 90-day retention and replicated durability.

---

## Testing Invariant Models

### Test 1: Independence Test

Can each entity be written without coordination? If a write to entity A requires reading entity B, the model is coupled.

### Test 2: Failure Isolation Test

If region X fails, do entities in region Y continue operating? If not, authority scope is wrong.

### Test 3: Eventual Consistency Test

Can temporary invariant violations be detected and handled? If blocking is required, reconsider the model.

### Test 4: View Rebuild Test

Can all views be rebuilt from entity events? If not, views contain authoritative data incorrectly.

---

## Modeling Checklist

Before finalizing a model:

### Core (v1.1-v1.3)
- [ ] All invariants identified and classified
- [ ] Cross-entity invariants in views
- [ ] No derived state in entities
- [ ] Relationships explicit with cardinality
- [ ] Authority scope appropriate
- [ ] Migration policy defined
- [ ] Violation handling specified
- [ ] Views are rebuildable
- [ ] Model passes independence test

### v1.5 Extensions
- [ ] Assets defined with lifecycle and retention (if binary content)
- [ ] Search configuration specified (if user search required)
- [ ] Spatial constraints declared (if location-based)
- [ ] Scheduled triggers defined (if time-based automation)
- [ ] External integrations with resilience patterns (if third-party APIs)
- [ ] Data governance classification and retention (if PII/regulated)
- [ ] Structured content sanitization (if rich text)
- [ ] Durable workflows for long-running processes (if multi-day spans)

### v1.6 Extensions
- [ ] Storage bindings defined for authoritative state
- [ ] Cache layers configured for read performance
- [ ] Intent routing specified for all input intents
- [ ] Resource requirements declared for infrastructure
- [ ] Client cache configured for offline-capable experiences

---

## Quick Reference Card

### Entity Rules
1. Independent writes
2. No derived state
3. Local constraints only

### Relationship Rules
1. Semantic, not transactional
2. Cardinality declared
3. Causality explicit

### View Rules
1. Derived and rebuildable
2. Authority agnostic
3. Invariants validated

### Authority Rules
1. Entity-scoped
2. Migration governed
3. CAS for writes

---

## Migration from Traditional Models

### From ACID to SMS

| Traditional | SMS Equivalent |
|-------------|----------------|
| Transaction | Atomic group (single boundary) |
| Foreign key | Relationship |
| Computed column | View |
| Constraint | Entity constraint OR view invariant |
| Trigger | Policy OR view worker |

### Step-by-Step Migration

1. **Identify invariants** in existing system
2. **Classify** as single-entity or cross-entity
3. **Move cross-entity invariants** to views
4. **Define relationships** explicitly
5. **Set authority scope** per entity
6. **Implement view workers** for derived state

---

# Part II: Complete Banking Example (All Features)

## Overview

This section provides a **complete, production-ready banking system** that demonstrates all SMS v1.6 features including storage binding, caching, routing, resource requirements, and all 28 v1.5 addendums in contextually-appropriate banking scenarios.

| Layer | Features Demonstrated |
|-------|----------------------|
| **Data Model** | Entities, constraints, assets, spatial types, structured content, inference fields |
| **Relationships** | Graph queries, delegation, causality, cardinality |
| **Views** | Materialization, search indexes, temporal windows, invariant enforcement |
| **Experiences** | Accessibility, conversational UI, collaborative sessions, client cache |
| **Workflows** | Durable workflows, offline support, compensation, checkpoints |
| **Integrations** | External APIs, notifications, scheduled triggers |
| **Edge Devices** | ATM terminals, POS devices, self-service kiosks |
| **Governance** | Data classification, retention, consent, right to erasure |
| **v1.6 Features** | Storage binding, caching, intent routing, resource requirements |

### Feature Coverage Matrix

All v1.5 addendums and v1.6 features are demonstrated:

| # | Feature | Banking Context |
|---|---------|-----------------|
| X | Form-Intent Binding | Transfer form binding |
| Y | View Materialization | Temporal windows, streaming |
| Z | Intent Delivery | Exactly-once guarantees |
| AA | Session Contract | Offline support, multi-device |
| AB | Indicator Source | Dynamic badges |
| AC | Composition Fetch | Related view loading |
| AD | Mask Expression | SSN masking |
| AE | Navigation Semantics | Hierarchical navigation |
| AF | View Availability | Regional fallback |
| AG | Presentation Hints | Display hints |
| AH | Surface/Container | View grouping |
| AI | Accessibility | Screen reader, keyboard nav |
| AJ | End-to-End Flow | Complete banking flows |
| AK | Asset Semantics | Check images, signatures |
| AL | Search Semantics | Transaction search |
| AM | Scheduled Triggers | Statements, payments |
| AN | Notification Channel | Multi-channel alerts |
| AO | Collaborative Session | Joint account planning |
| AP | Structured Content | Loan documentation |
| AQ | Inference/Derived | Fraud scoring, risk |
| AR | Spatial Types | Branch/ATM locator |
| AS | Graph Query | Customer relationships |
| AT | Durable Workflow | Loan approval |
| AU | Delegation/Consent | Authorized users |
| AV | Conversational | Banking assistant |
| AW | External Integration | Credit bureau API |
| AX | Data Governance | PII, retention |
| AY | Edge Device | ATM, POS terminals |
| v1.6 | Storage Binding | Account state, transaction streams |
| v1.6 | Cache Layer | Hybrid view caching |
| v1.6 | Intent Routing | NATS/HTTP/gRPC routing |
| v1.6 | Resource Requirements | Event logs, entity stores |
| v1.6 | Client Cache | Offline mobile banking |

---

## System Declaration

```yaml
system:
  name: BankingSystem
  version: v1
  description: "Complete banking application with SMS v1.6"
  realm: banking
```

## Resource Requirements

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

## Storage Bindings

```yaml
storageBindings:
  - name: AccountStateStorage
    version: v1
    type: kv
    kv:
      bucket: "SMS_ACCOUNT_STATE"
      key_pattern: "account.{{account_id}}"
      serialization: json
      history: 10
      ttl: 0s  # No expiration
      replicas: 3
    access:
      read_policy: local
      write_consistency: strong

  - name: TransactionEventStream
    version: v1
    type: stream
    stream:
      name: "SMS_TRANSACTIONS"
      subjects:
        - "SMS.TXN.{{realm}}.>"
        - "SMS.TXN.*.created"
      retention: limits
      max_msgs: 10000000
      max_bytes: 10GB
      max_age: 365d
      storage: file
      replicas: 3
    access:
      read_policy: eventual
      write_consistency: strong
```

## Cache Layers

```yaml
cacheLayers:
  - name: ViewHybridCache
    version: v1
    description: "Hybrid cache for materialized views"
    topology:
      tier: hybrid
      local:
        max_entries: 1000
        ttl: 5m
        eviction: lru
        max_size: 100MB
      distributed:
        resource: ViewCache
        ttl: 30m
      hybrid:
        promotion: on_miss
    invalidation:
      strategy: hybrid
      ttl: 30m
      watch_keys:
        - "account.{{account_id}}"
        - "view.{{view_name}}.{{account_id}}"
      propagate: true
    warmup:
      enabled: true
      strategy: eager
      on_startup: true
      queries:
        - "SELECT * FROM active_accounts"
    observability:
      emit_signals: true
      metrics:
        - hit_rate
        - miss_rate
        - latency
        - size
        - evictions
      interval: 10s

  - name: SessionLocalCache
    version: v1
    topology:
      tier: local
      local:
        max_entries: 5000
        ttl: 30m
        eviction: lru
    invalidation:
      strategy: ttl
      ttl: 30m
```

## Data Types

### Entities: Account, Customer, Transaction, Alert, AuditLog

```yaml
dataTypes:
  - name: Account
    version: v1
    fields:
      id: uuid
      owner_id: uuid
      account_number: string
      account_type: enum[checking, savings, credit]
      status: enum[active, frozen, closed, pending_approval]
      currency: string
      jurisdiction: string
      created_at: timestamp
      updated_at: timestamp

  - name: Customer
    version: v1
    fields:
      id: uuid
      email: string
      full_name: string
      phone: string
      ssn: string
      date_of_birth: timestamp
      kyc_status: enum[pending, verified, rejected]
      risk_tier: enum[low, medium, high]
      created_at: timestamp

  - name: Transaction
    version: v1
    fields:
      id: uuid
      timestamp: timestamp
      debit_account_id: uuid
      credit_account_id: uuid
      amount: decimal
      currency: string
      reference: string
      category: enum[transfer, payment, deposit, withdrawal, fee, interest]
      status: enum[pending, completed, failed, reversed]
      metadata: map

  - name: Alert
    version: v1
    fields:
      id: uuid
      type: enum[fraud_suspected, large_transaction, negative_balance, kyc_expired]
      severity: enum[low, medium, high, critical]
      entity_type: string
      entity_id: uuid
      message: string
      status: enum[open, acknowledged, resolved, escalated]
      created_at: timestamp
      resolved_at: timestamp
      resolved_by: uuid

  - name: AuditLog
    version: v1
    fields:
      id: uuid
      actor_id: uuid
      actor_type: enum[customer, admin, system]
      action: string
      entity_type: string
      entity_id: uuid
      changes: map
      ip_address: string
      timestamp: timestamp
```

### Models: Account, Transaction

```yaml
models:
  - name: Account
    mutability:
      scope: entity
      exclusive: true
    authority:
      scope: entity
      resolution: entity.jurisdiction_region
      migration:
        allowed: false  # Financial data stays in region

  - name: Transaction
    mutability:
      scope: append-only  # Transactions never change
```

## Data States

```yaml
dataStates:
  - name: AccountRecord
    type: Account
    lifecycle: persistent
    constraintPolicy:
      enforcement: strict
    storage:
      binding: AccountStateStorage

  - name: TransactionRecord
    type: Transaction
    lifecycle: persistent
    constraintPolicy:
      enforcement: strict
```

## Relationships

```yaml
relationships:
  - name: customer_owns_account
    from: Account
    to: Customer
    cardinality: many-to-one
    semantics:
      causal: true
      
  - name: transaction_debits_account
    from: Transaction
    to: Account
    cardinality: many-to-one
    semantics:
      causal: true
      ordered: true
      
  - name: transaction_credits_account
    from: Transaction
    to: Account
    cardinality: many-to-one
    semantics:
      causal: true
      ordered: true
      
  - name: alert_references_entity
    from: Alert
    to: Account
    cardinality: many-to-one
    
  - name: audit_references_actor
    from: AuditLog
    to: Customer
    cardinality: many-to-one
```

## Views

```yaml
views:
  - name: AccountBalance
    inputs: [Account, Transaction]
    aggregation:
      credits: sum(t.amount WHERE t.credit_account_id == account.id AND t.status == "completed")
      debits: sum(t.amount WHERE t.debit_account_id == account.id AND t.status == "completed")
      pending: sum(t.amount WHERE t.status == "pending")
      balance: credits - debits
      available: balance - pending
    invariant:
      expression: balance >= 0 OR account.account_type == "credit"
      on_violation: flag
    authority_agnostic: true

  - name: CustomerSummary
    inputs: [Customer, Account, AccountBalance]
    aggregation:
      account_count: count(accounts)
      total_balance: sum(balances)
      has_alerts: exists(alerts WHERE status == "open")

  - name: SystemMetrics
    inputs: [Customer, Account, Transaction, Alert]
    aggregation:
      total_customers: count(customers)
      active_accounts: count(accounts WHERE status == "active")
      pending_transactions: count(transactions WHERE status == "pending")
      open_alerts: count(alerts WHERE status == "open")
      critical_alerts: count(alerts WHERE severity == "critical" AND status == "open")
```

## Materialization with Cache

```yaml
materializations:
  - name: AccountSummaryView
    source: [AccountRecord, TransactionRecord]
    targetState: AccountSummaryState
    freshness:
      maxStaleness: 5m
    evolution:
      onSourceChange: recompute
    cache:
      layer: ViewHybridCache
      key: "account.{{account_id}}.summary"
```

## Input Intents with Routing

```yaml
inputIntents:
  - name: InitiateTransfer
    version: v1
    proposes:
      dataState: TransferRequest
    supplies:
      required: [source_account, destination_account, amount, currency]
      optional: [description, scheduled_date]
    constraints:
      amount: "amount > 0"
      currency: "currency in ['USD', 'EUR', 'GBP']"
    transitions:
      to: TransferRequest
    routing:
      transport: nats
      nats:
        subject_pattern: "SMS.INTENT.banking.InitiateTransfer.{{realm}}"
        delivery: jetstream
        stream: SMS_BANKING_INTENTS
        queue: transfer-workers
        dead_letter: "SMS.DLQ.banking.InitiateTransfer"
      response:
        mode: request_reply
        timeout: 30s

  - name: GetAccountSummary
    version: v1
    proposes:
      dataState: AccountSummaryQuery
    supplies:
      required: [account_id]
    routing:
      transport: http
      http:
        path: "/api/v1/accounts/{account_id}/summary"
        method: GET
        timeout: 10s
      response:
        mode: request_reply
        timeout: 10s
```

## SMS Flows

### Transfer Flow, Bill Payment Flow, Loan Origination Flow

```yaml
smsFlows:
  - name: TransferFlow
    version: v1
    description: "Process fund transfer with fraud check and atomic execution"
    
    triggeredBy:
      inputIntent: InitiateTransferIntent
      
    steps:
      - work_unit: ValidateTransfer
        description: "Validate transfer parameters and account status"
        input:
          from_account: "intent.from_account_id"
          to_account: "intent.to_account_id"
          amount: "intent.amount"
        output:
          validation_result: "validation.status"
          
      - work_unit: CheckFraud
        description: "Real-time fraud scoring"
        input:
          transaction: "transfer"
          customer_id: "intent.customer_id"
        output:
          fraud_score: "fraud_result.score"
          fraud_reasons: "fraud_result.reasons"
          
      - if: "fraud_score > 0.8"
        then:
          - work_unit: FlagForReview
            input:
              transfer: "transfer"
              fraud_score: "fraud_score"
              reasons: "fraud_reasons"
          - signal: TransferFlaggedSignal
            payload:
              transfer_id: "transfer.id"
              fraud_score: "fraud_score"
        else:
          - atomicGroup:
              name: ExecuteTransfer
              realm: account_authority
              description: "Atomically debit and credit accounts"
              steps:
                - work_unit: DebitAccount
                  input:
                    account_id: "intent.from_account_id"
                    amount: "intent.amount"
                    reference: "transfer.reference"
                - work_unit: CreditAccount
                  input:
                    account_id: "intent.to_account_id"
                    amount: "intent.amount"
                    reference: "transfer.reference"
              compensation:
                - work_unit: ReverseDebit
                  input:
                    account_id: "intent.from_account_id"
                    amount: "intent.amount"
                    original_reference: "transfer.reference"
                - work_unit: ReverseCredit
                  input:
                    account_id: "intent.to_account_id"
                    amount: "intent.amount"
                    original_reference: "transfer.reference"
              
      - work_unit: RecordTransaction
        description: "Create immutable transaction record"
        input:
          transfer: "transfer"
          status: "completed"
          
      - signal: TransferCompletedSignal
        payload:
          transfer_id: "transfer.id"
          amount: "intent.amount"
          from_account: "intent.from_account_id"
          to_account: "intent.to_account_id"
          
    failure:
      policy: compensate
      on_failure:
        - signal: TransferFailedSignal
        - work_unit: NotifyCustomer
          input:
            type: "transfer_failed"

  - name: BillPaymentFlow
    version: v1
    description: "Process scheduled or immediate bill payments"
    
    triggeredBy:
      inputIntent: PayBillIntent
      
    steps:
      - work_unit: ValidatePayee
        description: "Verify payee is valid and active"
        input:
          payee_id: "intent.payee_id"
        output:
          payee: "payee_result"
          
      - work_unit: ValidateAccount
        description: "Check account status and available balance"
        input:
          account_id: "intent.account_id"
          amount: "intent.amount"
        output:
          available_balance: "account.available_balance"
          
      - if: "available_balance < intent.amount"
        then:
          - signal: InsufficientFundsSignal
            payload:
              account_id: "intent.account_id"
              required: "intent.amount"
              available: "available_balance"
          - work_unit: RejectPayment
            input:
              reason: "insufficient_funds"
        else:
          - atomicGroup:
              name: ExecutePayment
              realm: account_authority
              steps:
                - work_unit: DebitAccount
                  input:
                    account_id: "intent.account_id"
                    amount: "intent.amount"
                - work_unit: CreatePaymentRecord
                  input:
                    payee: "payee"
                    amount: "intent.amount"
              compensation:
                - work_unit: ReverseDebit
                - work_unit: VoidPaymentRecord
                
          - work_unit: SubmitToPaymentNetwork
            description: "Send payment to external network (ACH/wire)"
            external_dependency: PaymentNetwork
            input:
              payment: "payment_record"
            timeout: 30s
            retry:
              max_attempts: 3
              backoff: exponential
              
      - signal: BillPaymentProcessedSignal

  - name: LoanOriginationFlow
    version: v1
    description: "Multi-week loan approval with human review and durable state"
    
    triggeredBy:
      inputIntent: SubmitLoanApplicationIntent
      
    durability:
      enabled: true
      state: LoanApplicationFlowState
      ttl: 60d
      versioning:
        strategy: run_to_completion
        
    steps:
      - work_unit: ValidateApplication
        description: "Validate loan application data"
        input:
          application: "intent"
        output:
          validation_errors: "validation.errors"
          
      - checkpoint: post_validation
        description: "Snapshot after validation for recovery"
      
      - work_unit: PullCreditReport
        description: "Fetch credit report from bureau"
        external_dependency: CreditBureau
        input:
          customer_id: "intent.applicant_id"
        output:
          credit_score: "credit.score"
          credit_history: "credit.history"
        timeout: 60s
        retry:
          max_attempts: 3
          
      - work_unit: CalculateRiskScore
        description: "ML-based risk assessment"
        input:
          application: "intent"
          credit_score: "credit_score"
        output:
          risk_score: "risk.score"
          risk_factors: "risk.factors"
      
      - if: "risk_score < 300"
        then:
          - work_unit: AutoApprove
            input:
              application_id: "intent.application_id"
              reason: "low_risk_auto_approval"
          - goto: generate_documents
        else:
          - if: "risk_score > 700"
            then:
              - work_unit: AutoDeny
                input:
                  application_id: "intent.application_id"
                  reason: "high_risk_auto_denial"
              - goto: send_denial
            else:
              - await:
                  name: await_underwriter_review
                  description: "Manual underwriter review required"
                  trigger:
                    type: inputIntent
                    intent: UnderwriterDecisionIntent
                    assignment:
                      policy: skill_based
                      pool: UnderwriterPool
                  sla:
                    target: 24h
                    warning_at: 20h
                  escalation:
                    - after: 24h
                      action: notify
                      to: UnderwriterManager
                    - after: 48h
                      action: escalate_pool
                      to: SeniorUnderwriterPool
                  timeout: 72h
                  on_timeout: escalate
                  result:
                    captures: [decision, conditions, notes]
                    
      - checkpoint: post_underwriting
        state_snapshot: full
        description: "Full state snapshot after underwriting decision"
      
      - if: "underwriter_decision == 'approved' OR underwriter_decision == 'approved_with_conditions'"
        then:
          - label: generate_documents
          - work_unit: GenerateLoanDocuments
            description: "Generate promissory note and disclosures"
            input:
              application: "intent"
              decision: "underwriter_decision"
              conditions: "conditions"
            output:
              documents: "generated_documents"
          
          - await:
              name: await_document_signing
              description: "Wait for customer to sign loan documents"
              trigger:
                type: inputIntent
                intent: SignLoanDocumentsIntent
                assignment:
                  policy: direct
                  subject: "application.applicant_id"
              sla:
                target: 7d
              timeout: 14d
              on_timeout: cancel
              
          - work_unit: FundLoan
            description: "Disburse loan funds to customer account"
            input:
              application_id: "intent.application_id"
              account_id: "intent.disbursement_account_id"
              amount: "intent.approved_amount"
              
          - signal: LoanFundedSignal
            payload:
              application_id: "intent.application_id"
              amount: "intent.approved_amount"
          
        else:
          - label: send_denial
          - work_unit: SendDenialNotice
            input:
              application_id: "intent.application_id"
              reason: "underwriter_decision"
              adverse_action_reasons: "risk_factors"
              
          - signal: LoanDeniedSignal
            payload:
              application_id: "intent.application_id"
          
    checkpoints:
      - name: post_validation
        after_step: ValidateApplication
        retention: 7d
      - name: post_underwriting
        after_step: await_underwriter_review
        state_snapshot: full
        retention: 90d
        
    compensation:
      - from_checkpoint: post_underwriting
        steps:
          - work_unit: ReverseApproval
          - work_unit: NotifyApplicant
            input:
              type: "application_cancelled"
      - from_checkpoint: post_validation
        steps:
          - work_unit: CancelApplication
          - work_unit: RefundApplicationFee
```

### Task Pools for Human Review

```yaml
taskPool:
  name: UnderwriterPool
  version: v1
  description: "Pool of loan underwriters for manual review"
  
  membership:
    type: role_based
    roles: [underwriter, senior_underwriter]
    
  capacity:
    max_concurrent_per_member: 10
    rebalance_on_overflow: true
    
  skills:
    - name: credit_analysis
      required: true
    - name: high_value_loans
      required: false
      threshold: "loan.amount > 500000"
    - name: commercial_lending
      required: false
      
  assignment:
    algorithm: round_robin_with_skills
    consider_workload: true

taskPool:
  name: SeniorUnderwriterPool
  version: v1
  description: "Senior underwriters for escalated reviews"
  
  membership:
    type: role_based
    roles: [senior_underwriter]
    
  capacity:
    max_concurrent_per_member: 5
```

### Signals

```yaml
signals:
  - name: TransferCompletedSignal
    version: v1
    payload:
      transfer_id: uuid
      amount: decimal
      from_account: uuid
      to_account: uuid
      timestamp: timestamp
    subscribers:
      - NotificationService
      - AuditService
      - AnalyticsService

  - name: TransferFlaggedSignal
    version: v1
    payload:
      transfer_id: uuid
      fraud_score: decimal
      reasons: array[string]
    subscribers:
      - FraudReviewQueue
      - SecurityAlertService

  - name: InsufficientFundsSignal
    version: v1
    payload:
      account_id: uuid
      required: decimal
      available: decimal
    subscribers:
      - NotificationService
      - OverdraftProtectionService

  - name: LoanFundedSignal
    version: v1
    payload:
      application_id: uuid
      amount: decimal
      funded_at: timestamp
    subscribers:
      - NotificationService
      - AccountingService
      - ReportingService

  - name: LoanDeniedSignal
    version: v1
    payload:
      application_id: uuid
      denial_reason: string
    subscribers:
      - NotificationService
      - ComplianceService
```

### Work Unit Contracts

```yaml
workUnitContracts:
  - name: DebitAccount
    version: v1
    description: "Debit funds from an account"

    input:
      account_id: uuid
      amount: decimal
      reference: string

    output:
      new_balance: decimal
      transaction_id: uuid

    preconditions:
      - "account.status == 'active'"
      - "account.available_balance >= amount"

    postconditions:
      - "account.balance == old(account.balance) - amount"

    authority:
      requires: account_write
      scope: entity

    idempotency:
      key: [account_id, reference]
      ttl: 24h

  - name: CreditAccount
    version: v1
    description: "Credit funds to an account"

    input:
      account_id: uuid
      amount: decimal
      reference: string

    output:
      new_balance: decimal
      transaction_id: uuid

    preconditions:
      - "account.status IN ['active', 'frozen']"

    postconditions:
      - "account.balance == old(account.balance) + amount"

    authority:
      requires: account_write
      scope: entity

    idempotency:
      key: [account_id, reference]
      ttl: 24h

  - name: CheckFraud
    version: v1
    description: "Score transaction for fraud risk"

    input:
      transaction: Transaction
      customer_id: uuid

    output:
      score: decimal
      reasons: array[string]
      confidence: decimal

    external_dependency:
      service: FraudDetectionService
      timeout: 5s
      fallback:
        strategy: default_score
        value: 0.5
```

## Policy Layer

### Subjects and Roles

```yaml
policies:
  - subjects:
      - name: customer
        type: user
        attributes:
          id: uuid
          email: string
          kyc_status: string
          
      - name: admin
        type: user
        attributes:
          id: uuid
          email: string
          department: string
          clearance_level: integer
          
    roles:
      - name: customer
        description: "Regular banking customer"
        grants:
          - view_own_accounts
          - view_own_transactions
          - create_transfers
          - update_own_profile
          
      - name: customer_service
        description: "Customer service representative"
        grants:
          - view_customer_accounts
          - view_transactions
          - acknowledge_alerts
          
      - name: compliance_officer
        description: "Compliance and KYC officer"
        grants:
          - view_all_customers
          - view_kyc_details
          - approve_kyc
          - reject_kyc
          - view_audit_logs
        inherits: [customer_service]
        
      - name: fraud_analyst
        description: "Fraud detection analyst"
        grants:
          - view_all_transactions
          - view_alerts
          - resolve_alerts
          - escalate_alerts
          - freeze_accounts
        inherits: [customer_service]
        
      - name: operations_manager
        description: "Operations manager"
        grants:
          - view_system_metrics
          - view_all_accounts
          - unfreeze_accounts
          - reverse_transactions
        inherits: [fraud_analyst, compliance_officer]
```

### Policies

```yaml
policies:
  - name: CustomerViewOwnData
    version: v1
    effect: allow
    when: |
      subject.role == "customer" AND
      subject.id == data.owner_id
      
  - name: AdminViewSensitiveData
    version: v1
    effect: allow
    when: |
      subject.role in ["compliance_officer", "operations_manager", "system_admin"] AND
      subject.clearance_level >= 3
      
  - name: AdminFreezeAccount
    version: v1
    effect: allow
    when: |
      subject.role in ["fraud_analyst", "operations_manager", "system_admin"]

policy_sets:
  - name: CustomerAccess
    resolution: deny_overrides
    policies: [RequireAuthentication, CustomerViewOwnData]
    
  - name: AdminBasicAccess
    resolution: deny_overrides
    policies: [RequireAuthentication, AdminViewCustomers]
    
  - name: AdminSensitiveAccess
    resolution: deny_overrides
    policies: [RequireAuthentication, AdminViewSensitiveData]
```

## UI Layer

### Customer Views

```yaml
ui:
  views:
    - name: CustomerDashboardView
      version: v1
      consumes:
        dataState: CustomerSummary
        version: v1
      guarantees:
        - show_account_overview
        - show_recent_activity
        - show_quick_actions
      interactionModel:
        badges:
          - if: has_alerts
            label: "Action Required"
            severity: warning
        actions:
          - if: true
            action: new_transfer
            label: "New Transfer"
            
    - name: CustomerAccountListView
      version: v1
      consumes:
        dataState: AccountList
        version: v1
      guarantees:
        - show_all_accounts
        - show_balances
      interactionModel:
        filters:
          - field: account_type
            type: enum
          - field: status
            type: enum
        sorts:
          - field: created_at
            default: desc
          - field: balance
            
    - name: CustomerAccountDetailsView
      version: v1
      consumes:
        dataState: AccountRecord
        version: v1
      guarantees:
        - show_account_info
        - show_account_status
      interactionModel:
        badges:
          - if: status == "frozen"
            label: "Account Frozen"
            severity: error
          - if: status == "active"
            label: "Active"
            severity: success
        actions:
          - if: status == "active"
            action: transfer_funds
            label: "Transfer"
          - if: true
            action: download_statement
            label: "Download Statement"
      fieldPermissions:
        - field: account_number
          visible: true
        - field: balance
          visible: true
        - field: available_balance
          visible: true
          
    - name: CustomerBalanceView
      version: v1
      consumes:
        dataState: AccountBalance
        version: v1
      guarantees:
        - show_current_balance
        - show_available_balance
        - show_pending
      interactionModel:
        badges:
          - if: balance < 100
            label: "Low Balance"
            severity: warning
          - if: balance < 0
            label: "Overdrawn"
            severity: error
            
    - name: CustomerTransactionListView
      version: v1
      consumes:
        dataState: TransactionHistory
        version: v1
      guarantees:
        - show_transactions
        - show_running_balance
      interactionModel:
        filters:
          - field: timestamp
            type: date_range
          - field: category
            type: enum
          - field: amount
            type: range
        sorts:
          - field: timestamp
            default: desc
          - field: amount
```

### Admin Views

```yaml
    - name: AdminDashboardView
      version: v1
      consumes:
        dataState: SystemMetrics
        version: v1
      guarantees:
        - show_system_health
        - show_key_metrics
        - show_alert_summary
      interactionModel:
        badges:
          - if: open_alerts > 0
            label: "{{open_alerts}} Open Alerts"
            severity: warning
          - if: critical_alerts > 0
            label: "{{critical_alerts}} Critical"
            severity: error
        actions:
          - if: open_alerts > 0
            action: view_alerts
            label: "View Alerts"
            
    - name: AdminCustomerDetailsView
      version: v1
      consumes:
        dataState: CustomerRecord
        version: v1
      guarantees:
        - show_full_customer_info
        - show_kyc_details
        - show_risk_assessment
      interactionModel:
        badges:
          - if: kyc_status == "pending"
            label: "KYC Pending"
            severity: warning
          - if: kyc_status == "verified"
            label: "Verified"
            severity: success
          - if: kyc_status == "rejected"
            label: "Rejected"
            severity: error
          - if: risk_tier == "high"
            label: "High Risk"
            severity: error
        actions:
          - if: kyc_status == "pending"
            action: approve_kyc
            label: "Approve KYC"
            intent: ApproveKYC
          - if: kyc_status == "pending"
            action: reject_kyc
            label: "Reject KYC"
            intent: RejectKYC
      fieldPermissions:
        - field: full_name
          visible: true
        - field: email
          visible: true
        - field: phone
          visible: true
        - field: ssn
          visible:
            policy: AdminViewSensitiveData
          mask:
            unless: subject.clearance_level >= 4
            type: partial
            reveal: last4
        - field: date_of_birth
          visible:
            policy: AdminViewSensitiveData
        - field: kyc_status
          visible: true
        - field: risk_tier
          visible: true
```

## Experience with Client Cache

```yaml
experiences:
  - name: CustomerBankingExperience
    version: v1
    description: "Mobile-first banking with offline support"
    entry_point: AccountOverview
    includes:
      - AccountOverview
      - AccountList
      - AccountDetails
      - TransferForm
      - TransactionHistory
    policy_set: CustomerAccess
    
    navigation:
      root: AccountOverview
      tree:
        - name: Accounts
          view: AccountListView
        - name: Transfers
          view: TransferFormView
        - name: History
          view: TransactionHistoryView
    
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
          - AccountSummaryView
          - RecentTransactionsView
        trigger: on_connect
      offline:
        enabled: true
        required_views:
          - AccountSummaryView
          - RecentTransactionsView
          - PendingTransfersView
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

## Realms

```yaml
realms:
  - name: TransferService
    version: v1
    description: "Handles fund transfers between accounts"
    owns:
      - TransferRequest
      - TransactionRecord
    workUnits:
      - InitiateTransfer
      - ProcessTransfer
    policy_set: TransferPolicies

  - name: AccountService
    version: v1
    description: "Manages account lifecycle and queries"
    owns:
      - Account
      - AccountSummaryState
    workUnits:
      - GetAccountSummary
    policy_set: AccountPolicies
```

## Workers

```yaml
workers:
  - name: TransferService
    acceptsInputs: 
      - InitiateTransfer v1
    produces: 
      - TransferRequest v1
      - TransactionRecord v1
    compatibleVersions: [v1]
    boundedContext: transfers
  
  - name: AccountQueryService
    acceptsInputs:
      - GetAccountSummary v1
    produces:
      - AccountSummaryState v1
    compatibleVersions: [v1]
    boundedContext: accounts
```

## v1.5 Extensions in Banking Context

### Assets: Check Images (Addendum AK)

```yaml
assets:
  - name: check_image
    types: [image/jpeg, image/png, image/tiff]
    max_size: 10MB
    transformations:
      thumbnail: resize(width: 300, height: 150)
      full: compress(quality: 85)
    retention:
      duration: 7_years  # Banking regulation
      delete_after: true
    access:
      policy: ViewCheckImage
```

### Notifications: Transaction Alerts (Addendum AN)

```yaml
notificationChannels:
  - name: LargeTransactionAlert
    trigger:
      event: Transaction.created
      condition: amount > account.alert_threshold
    channels:
      - type: email
        template: large_transaction_email
        to: customer.email
      - type: sms
        template: large_transaction_sms
        to: customer.phone
        condition: customer.sms_alerts_enabled
      - type: push
        template: large_transaction_push
        condition: customer.push_enabled
    delivery:
      retry: 3
      timeout: 30s
```

### Scheduled Triggers: Monthly Statements (Addendum AM)

```yaml
scheduledTriggers:
  - name: GenerateMonthlyStatement
    schedule: "0 0 1 * *"  # First day of month at midnight
    timezone: account.timezone
    intent: GenerateStatement
    scope: per_entity
    entity_filter: account.status == active AND account.statement_delivery == monthly
    idempotency:
      key: [account.id, date.month]
    retry:
      max_attempts: 3
      backoff: exponential
```

### External Integration: Credit Bureau (Addendum AW)

```yaml
externalDependencies:
  - name: CreditBureauAPI
    endpoint: https://api.creditbureau.example
    authentication:
      type: api_key
      credential_ref: credit_bureau_key
    operations:
      - name: getCreditScore
        method: POST
        path: /v1/credit-score
        timeout: 5s
    resilience:
      circuit_breaker:
        threshold: 5
        timeout: 30s
        reset_after: 60s
      retry:
        max_attempts: 3
        backoff: exponential
      fallback:
        strategy: use_cached
        cache_ttl: 24h
```

### Data Governance: PII Classification (Addendum AX)

```yaml
dataGovernances:
  - model: Customer
    classification:
      level: pii
      categories: [personal_identifiable, financial, contact]
    retention:
      duration: 7_years  # After account closure
      delete_after: true
      archive_after: 1_year
    right_to_erasure:
      supported: true
      cascade: [AuditLog, Transaction]
      exceptions: [Transaction]  # Keep for audit
      pseudonymize: [Transaction.metadata.customer_name]
    data_lineage:
      track: true
    audit:
      log_access: true
      log_modification: true
      retention: 10_years
```

### Search: Transaction Search (Addendum AL)

```yaml
searchIntents:
  - name: TransactionSearch
    target: TransactionView
    scope: account.id == session.account_id  # User can only search their own
    fields:
      - name: reference
        weight: 2.0
        analyzers: [keyword, edge_ngram]
      - name: category
        weight: 1.5
        facet: true
      - name: amount
        weight: 1.0
        range_facet: true
      - name: timestamp
        weight: 0.5
        range_facet: true
    ranking:
      function: bm25
      boost: recency
    pagination:
      default_size: 25
      max_size: 100
```

### Form Binding: Transfer Form (Addendum X)

```yaml
presentationViews:
  - name: TransferForm
    version: v1
    consumes:
      dataState: AccountBalance
    triggers:
      intent: InitiateTransfer
      binding: form
      field_mapping:
        - view_field: from_account_id
          supplies: from_account
          source: context
          required: true
        - view_field: to_account_number
          supplies: to_account
          source: input
          required: true
          validation:
            constraint: matches_account_format(to_account_number)
            message: "Invalid account number format"
        - view_field: amount
          supplies: amount
          source: input
          required: true
          validation:
            constraint: amount > 0 AND amount <= balance
            message: "Amount must be positive and not exceed balance"
      confirmation:
        required: true
        message: "Transfer {{amount}} from {{from_account}} to {{to_account}}?"
      on_success:
        action: navigate_back
        notification:
          type: toast
          message: "Transfer initiated successfully"
      on_error:
        action: display_inline
        show_field_errors: true
```

### Spatial Types: Branch and ATM Locator (Addendum AR)

```yaml
dataTypes:
  - name: Branch
    version: v1
    fields:
      id: uuid
      name: string
      address: string
      location: geo_point
      service_area: geo_shape
      hours_of_operation: map
      services: array[enum[teller, atm, safe_deposit, mortgage, business]]
      status: enum[open, closed, temporarily_closed]

searchIntents:
  - name: NearbyBranchesIntent
    version: v1
    description: "Find bank branches near user location"
    
    searches: BranchState
    
    query:
      spatial:
        field: location
        type: within_distance
        within_distance:
          from: "request.user_location"
          max_distance: 25km
        sort:
          by_distance_from: "request.user_location"
          
    filter:
      expression: "branch.status == 'open'"
      
    result:
      includes: [id, name, address, location, distance, hours_of_operation, services]
      max_results: 20
```

### Graph Query: Customer Relationship Network (Addendum AS)

```yaml
graphQueries:
  - name: HouseholdAccountsQuery
    version: v1
    description: "Find all accounts for household members"

    traverses: [Household_Member_Of, Owns_Account]

    start:
      from: CustomerState
      where: "customer.id == request.customer_id"

    pattern:
      type: path
      path:
        - step: Household_Member_Of
          direction: outbound
        - step: Household_Member_Of
          direction: inbound
        - step: Owns_Account
          direction: outbound
        max_hops: 3

    result:
      includes: [customer_id, full_name, account_id, account_type, balance]
      depth_field: relationship_degree
```

### Delegation: Authorized User (Addendum AU)

```yaml
delegations:
  - name: AuthorizedUser
    delegator: account.owner_id
    delegate: authorized_user.id
    permissions:
      - ViewBalance
      - ViewTransactions
      - InitiateTransfer
    scope:
      accounts: [delegator.accounts]
      max_transfer_amount: 1000
    temporal:
      starts_at: delegation.created_at
      expires_at: delegation.created_at + 1_year
      auto_revoke: true
    consent:
      required: true
      captured_at: delegation.created_at
      method: explicit
      evidence: delegation.consent_signature
    audit:
      log_all_actions: true
      notify_delegator: true
```

### Conversational UI: Banking Assistant (Addendum AV)

```yaml
conversationalExperiences:
  - name: BankingAssistant
    channels: [chat, voice]
    intents:
      - intent: CheckBalance
        utterances:
          - "What's my balance"
          - "How much money do I have"
          - "Show my account balance"
        response:
          type: template
          template: "Your {{account_type}} account balance is {{balance}}"
          data_source: AccountBalance
      - intent: InitiateTransfer
        utterances:
          - "Transfer money"
          - "Send money to [account]"
          - "Pay [amount] to [account]"
        multi_turn: true
        slots:
          - name: to_account
            type: account_number
            prompt: "Which account would you like to transfer to?"
          - name: amount
            type: decimal
            prompt: "How much would you like to transfer?"
        confirmation:
          required: true
          template: "Transfer {{amount}} to {{to_account}}?"
        action: submit_intent
    fallback:
      strategy: human_handoff
      message: "Let me connect you with a representative"
```

### Collaborative Sessions: Joint Account Financial Planning (Addendum AO)

Banks support joint account holders and advisors collaborating in real-time:

```yaml
# Example 1: Financial Planning Session
collaborativeSessions:
  - name: FinancialPlanningSession
    version: v1
    description: "Joint account holders reviewing finances together"

    binds_to:
      dataState: AccountState
      entity_key: account_id

    authorization:
      requires: joint_account_holder OR authorized_delegate
      max_participants: 3

    presence:
      enabled: true
      fields:
        cursor:
          type: position
          scope: per_client
          description: "Current view position in transaction list"
        viewing_section:
          type: enum
          values: [overview, transactions, statements, settings]
          scope: per_client
        user_color:
          type: string
          scope: per_session
        is_typing_note:
          type: boolean
          scope: per_client
      ttl: 30s
      broadcast:
        to: same_entity
        throttle: 100ms

    operations:
      mode: stream
      types:
        - name: add_note
          payload:
            text: string
            transaction_id: uuid
          ordering:
            guarantee: causal
        - name: flag_transaction
          payload:
            transaction_id: uuid
            reason: string
          conflict:
            hint: merge
        - name: category_override
          payload:
            transaction_id: uuid
            new_category: string
          conflict:
            hint: last_write_wins

    shared_state:
      - name: session_notes
        type: structured_content
        sync: realtime
        conflict_resolution: operational_transform
      - name: highlighted_transactions
        type: array[uuid]
        sync: realtime
        conflict_resolution: union

  # Example 2: Loan Advisor Session
  - name: LoanAdvisorSession
    version: v1
    description: "Advisor-client co-review of loan application"

    binds_to:
      dataState: LoanApplicationState
      entity_key: application_id

    authorization:
      requires: (applicant AND advisor) OR underwriter
      max_participants: 3

    presence:
      enabled: true
      fields:
        viewing_document:
          type: string
          scope: per_client
        annotation_mode:
          type: boolean
          scope: per_client
      ttl: 30s
      broadcast:
        to: same_entity

    operations:
      mode: stream
      types:
        - name: advisor_comment
          payload:
            section: string
            comment: structured_content
          ordering:
            guarantee: causal
        - name: request_document
          payload:
            document_type: string
            instructions: string
        - name: answer_question
          payload:
            question_id: uuid
            answer: string

    features:
      screen_share:
        enabled: true
        initiator: advisor
      cobrowse:
        enabled: true
        control: advisor_led

# v1.6 Enhancement: Storage Binding for session state
storageBindings:
  - name: CollaborationSessionStore
    storage_type: kv
    kv:
      ttl: 24h
      replicas: 2
    access:
      write_consistency: strong
      read_policy: leader_preferred

# v1.6 Enhancement: Cache Layer for presence data
cacheLayers:
  - name: PresenceCache
    topology:
      tiers: [local]
    eviction:
      policy: ttl
      ttl: 30s
    observability:
      emit_signals: true
```

**Key Features Demonstrated**:
- Real-time presence awareness (cursor position, viewing section)
- Multi-user conflict resolution strategies (merge, last-write-wins, operational transform)
- Causal ordering for comments and annotations
- Screen sharing and co-browsing capabilities
- Throttled presence broadcasts to reduce network overhead
- Shared state synchronization with conflict resolution

**v1.6 Enhancements**: 
- Session state stored in KV store with strong consistency and 24h TTL
- Presence data cached locally with 30s TTL for low-latency updates
- Storage binding provides automatic replication and durability

---

### Edge Device: ATM Terminal (Addendum AY)

```yaml
edgeDevices:
  - name: ATMTerminal
    version: v1
    description: "Automated Teller Machine with offline capability"
    
    device:
      type: kiosk
      runtime: embedded_linux
      
      capabilities:
        - cash_dispense
        - card_read
        - pin_entry
        - receipt_print
        - deposit_accept
        
    connectivity:
      primary: ethernet
      fallback: cellular_4g
      
      offline:
        enabled: true
        max_duration: 4h
        capabilities:
          - balance_inquiry_cached
          - cash_withdrawal_limited
          - deposit_accept_deferred
          
        limits:
          max_withdrawal_offline: 200
          max_transactions_offline: 20
          
        sync:
          on_reconnect: immediate
          priority: high
          
    security:
      encryption:
        pin_entry: hardware_encrypted
        storage: aes_256
      tamper_detection:
        enabled: true
        on_detect: lockdown
```

---

### Enhanced Accessibility: Banking for All (Addendum AI)

Comprehensive accessibility support for customers with various needs:

```yaml
accessibilityPreferences:
  name: BankingAccessibilityComplete
  version: v1
  scope: session
  
  presets:
    - name: vision_impaired
      settings:
        font_scale: 1.5
        high_contrast: true
        screen_reader_optimized: true
        motion: reduced
        
    - name: low_vision
      settings:
        font_scale: 2.0
        high_contrast: true
        large_touch_targets: true
        
    - name: motor_impaired
      settings:
        large_touch_targets: true
        extended_timeouts: true
        single_tap_actions: true
        
    - name: cognitive
      settings:
        simplified_interface: true
        step_by_step_guidance: true
        confirmation_dialogs: extended
        
  preferences:
    - name: high_contrast
      type: boolean
      default: false
      applies_to: [all_views]
      
    - name: font_scale
      type: enum
      values: [0.85, 1.0, 1.25, 1.5, 2.0]
      default: 1.0
      applies_to: [all_views]
      
    - name: reduce_motion
      type: boolean
      default: false
      applies_to: [animations, transitions, auto_refresh]
      
    - name: screen_reader_hints
      type: boolean
      default: false
      applies_to: [all_views]
      description: "Add extra context for screen readers"
      
    - name: color_blind_mode
      type: enum
      values: [none, protanopia, deuteranopia, tritanopia]
      default: none
      applies_to: [charts, status_indicators, badges]
      
    - name: audio_descriptions
      type: boolean
      default: false
      applies_to: [charts, images, complex_ui]
      
    - name: timeout_extensions
      type: enum
      values: [standard, extended, maximum]
      default: standard
      mapping:
        standard: 30s
        extended: 60s
        maximum: 120s
        
  keyboard_navigation:
    enabled: true
    focus_visible: always
    skip_links: true
    
    shortcuts:
      - key: "Alt+H"
        action: navigate_home
        description: "Go to dashboard"
      - key: "Alt+T"
        action: navigate_transactions
        description: "View transactions"
      - key: "Alt+N"
        action: new_transfer
        description: "Start new transfer"
      - key: "Alt+B"
        action: announce_balance
        description: "Read balance aloud"
      - key: "Alt+S"
        action: open_search
        description: "Search transactions"
      - key: "Escape"
        action: cancel_current
        description: "Cancel current action"
        
  screen_reader:
    balance_announcement:
      template: "Your {{account_type}} account ending in {{last_four}} has a balance of {{balance | currency}}"
      announce_on: [page_load, balance_change]
      
    transaction_announcement:
      template: "{{transaction_type}} of {{amount | currency}} {{direction}} {{counterparty}} on {{date}}"
      
    form_guidance:
      enabled: true
      announce_required_fields: true
      announce_validation_errors: immediately
      
    navigation_landmarks:
      - role: banner
        label: "Main navigation"
      - role: main
        label: "Account content"
      - role: navigation
        label: "Account menu"
      - role: complementary
        label: "Quick actions"
        
  wcag_compliance:
    level: AA
    exceptions: []
```

**Cross-reference**: Appendix J (Accessibility - WCAG Compliance)

---

### Structured Content: Loan Documentation (Addendum AP)

Banks need rich document formats for loan contracts and disclosures:

```yaml
dataTypes:
  - name: LoanDocument
    version: v1
    fields:
      id: uuid
      loan_application_id: uuid
      document_type: enum[promissory_note, disclosure, agreement, amendment]
      content: structured_content
      signatures: array[SignatureBlock]
      version: integer
      status: enum[draft, pending_signature, signed, archived]
      created_at: timestamp
    
structuredContents:
  - name: loan_document_content
    schema: loan_document
    
    blocks:
      allowed:
        - paragraph
        - heading
        - list
        - table
        - signature_block
        - disclosure_box
        - terms_section
        - amortization_table
        
    marks:
      allowed:
        - bold
        - italic
        - underline
        - legal_reference
        - defined_term
        
    custom_blocks:
      - name: disclosure_box
        fields:
          title: string
          severity: enum[info, warning, important]
          content: paragraph[]
        presentation:
          border: true
          background: severity_based
          
      - name: signature_block
        fields:
          signer_role: enum[borrower, co_borrower, guarantor, notary]
          date_field: boolean
          witness_required: boolean
        presentation:
          layout: signature_line
          
      - name: amortization_table
        fields:
          loan_amount: decimal
          interest_rate: decimal
          term_months: integer
          payment_schedule: auto_generated
        presentation:
          display_as: table
          columns: [payment_number, payment_date, principal, interest, balance]
          
    constraints:
      max_length: 500000
      required_blocks: [heading, signature_block]
      
    sanitization: strict
    
    versioning:
      enabled: true
      track_changes: true

# v1.6 Enhancement: Storage Binding for document storage
storageBindings:
  - name: LoanDocumentStore
    storage_type: object_store
    object_store:
      purpose: document_storage
      retention:
        policy: time_based
        duration: 7y
      access:
        encryption: aes_256
        versioning: enabled
    access:
      durability: persistent
      read_policy: nearest
```

**Key Features Demonstrated**:
- Custom block types for legal documents (disclosure_box, signature_block, amortization_table)
- Semantic marks for legal references and defined terms
- Strict sanitization for security
- Version tracking for audit trails
- Rich presentation hints for rendering

**v1.6 Enhancements**: Documents stored in object store with 7-year retention for compliance, AES-256 encryption, and versioning enabled for audit trails.

---

### Inference/Derived Fields: Risk and Fraud Scoring (Addendum AQ)

Banks use ML models for real-time fraud detection and credit risk assessment:

```yaml
dataTypes:
  - name: Transaction
    version: v2
    fields:
      amount: decimal
      counterparty: string
      timestamp: timestamp
      account_id: uuid
      location: geo_point
      device_id: string
      description: string
      mcc_code: string
      
      fraud_score:
        type: decimal
        inference:
          enabled: true
          source:
            fields: [amount, counterparty, timestamp, account_id, location, device_id]
          model:
            name: fraud_detector
            version: v3
            type: ml_model
          output:
            type: decimal
            range: { min: 0, max: 1 }
            confidence_field: fraud_confidence
          freshness:
            strategy: on_write
          fallback:
            on_unavailable: default
            default_value: 0.5
          thresholds:
            high_risk: 0.8
            medium_risk: 0.5
          
      spending_category:
        type: string
        inference:
          enabled: true
          source:
            fields: [description, counterparty, amount, mcc_code]
          model:
            name: transaction_categorizer
            version: v2
          output:
            type: enum
            values: [groceries, dining, transportation, utilities, entertainment, 
                     healthcare, shopping, travel, income, transfer, other]
          freshness:
            strategy: on_write
          fallback:
            on_unavailable: default
            default_value: "other"

  - name: Customer
    version: v2
    fields:
      account_history: array
      payment_behavior: map
      credit_utilization: decimal
      income_estimate: decimal
      login_frequency: integer
      transaction_volume: decimal
      product_usage: map
      support_tickets: integer
      
      credit_risk_score:
        type: decimal
        inference:
          enabled: true
          source:
            fields: [account_history, payment_behavior, credit_utilization, income_estimate]
            external: CreditBureauAPI
          model:
            name: credit_risk_scorer
            version: v4
          output:
            type: decimal
            range: { min: 300, max: 850 }
          freshness:
            strategy: periodic
            interval: 24h
          fallback:
            on_unavailable: use_cached
            cache_ttl: 7d
          
      churn_probability:
        type: decimal
        inference:
          enabled: true
          source:
            fields: [login_frequency, transaction_volume, product_usage, support_tickets]
          model:
            name: churn_predictor
            version: v1
          output:
            type: decimal
            range: { min: 0, max: 1 }
          freshness:
            strategy: periodic
            interval: 7d
          fallback:
            on_unavailable: default
            default_value: 0.0

# v1.6 Enhancement: Cache Layer for ML inference results
cacheLayers:
  - name: InferenceCache
    topology:
      tiers: [local, distributed]
    local:
      max_entries: 10000
      eviction: lru
    distributed:
      replicas: 2
      eviction: ttl
      ttl: 1h
    invalidation:
      strategy: on_model_update
    observability:
      emit_signals: true
```

**Key Features Demonstrated**:
- Real-time fraud scoring using ML models
- Automatic transaction categorization
- Credit risk assessment with external data integration
- Churn prediction for customer retention
- Freshness strategies (on_write vs periodic)
- Fallback strategies for model unavailability
- Confidence thresholds for risk-based decisions

**v1.6 Enhancements**: ML inference results cached in hybrid local+distributed cache with 1h TTL and automatic invalidation on model updates for optimal performance.

---

### Accessibility Preferences (Addendum AI)

Support customer accessibility needs for inclusive banking experiences:

```yaml
accessibilityPreferences:
  name: BankingAccessibility
  version: v1
  scope: session
  
  preferences:
    - name: high_contrast
      type: boolean
      default: false
      applies_to: [all_views]
      description: "High contrast mode for visual clarity"
      
    - name: font_scale
      type: enum
      values: [small, medium, large, xlarge]
      default: medium
      applies_to: [all_views]
      description: "Text size adjustment"
      
    - name: reduce_motion
      type: boolean
      default: false
      applies_to: [animations, transitions]
      description: "Reduce animations for motion sensitivity"
      
    - name: screen_reader_hints
      type: boolean
      default: false
      applies_to: [all_views]
      description: "Enhanced ARIA labels and descriptions"
      
  keyboard_navigation:
    enabled: true
    shortcuts:
      - key: "Alt+H"
        action: navigate_home
        description: "Go to home screen"
      - key: "Alt+T"
        action: navigate_transactions
        description: "View transactions"
      - key: "Alt+N"
        action: new_transfer
        description: "Start new transfer"
      - key: "Alt+A"
        action: navigate_accounts
        description: "View accounts"

# v1.6 Enhancement: Client Cache for preferences
experiences:
  - name: AccessibleBankingExperience
    version: v1
    
    clientCache:
      storage:
        type: persistent
        max_size: 1MB
      
      strategies:
        preferences:
          cache: cache_first
          ttl: 30d
          sync: background
      
      prefetch:
        triggers: [on_login]
        entities: [accessibility_preferences]
```

**Key Features Demonstrated**:
- User-configurable accessibility preferences
- High contrast mode for visual impairment
- Font scaling for readability
- Motion reduction for vestibular disorders
- Screen reader enhancements with ARIA hints
- Keyboard navigation shortcuts for non-mouse users
- Session-scoped preferences for immediate application

**v1.6 Enhancements**: Accessibility preferences cached on client with 30-day TTL and cache-first strategy for instant application, prefetched on login for seamless experience.

---

### Offline Support: Mobile Banking (Addendum AA)

Mobile banking apps need offline capability for areas with poor connectivity. This example shows v1.6 client cache integration:

```yaml
experiences:
  - name: CustomerMobileBanking
    version: v1
    description: "Mobile banking app with offline support using v1.6 client cache"
    
    entry_point: MobileDashboard
    
    session:
      required: true
      lifetime:
        mode: sliding
        idle_timeout: 30m
        max_duration: 24h
    
    # v1.6 Client Cache Contract
    clientCache:
      enabled: true
      
      strategies:
        - type: prefetch
          views:
            - AccountBalanceView
            - TransactionHistoryView
            - PayeeListView
            - ScheduledPaymentsView
          on_login: true
          
        - type: optimistic_write
          intents:
            - InitiateTransfer
            - SchedulePayment
            - UpdatePayee
          
      offline:
        read: always
        write: queue
        
      cache:
        max_size: 50MB
        ttl: 24h
        encryption: required
        
      sync:
        on_reconnect: immediate
        conflict_resolution: server_wins
        field_strategies:
          - field: transaction.status
            strategy: server_wins
          - field: payee.nickname
            strategy: client_wins
          - field: scheduled_payment
            strategy: merge
            
      queue:
        max_size: 100
        max_age: 72h
        persist: true
        priority_fields: [transaction.amount]
        
      degraded_features:
        - feature: balance_check
          behavior: show_cached_with_warning
          warning: "Balance as of {{cache_time}}"
        - feature: transfer
          behavior: queue_for_sync
          confirmation: "Transfer will complete when online"
        - feature: bill_pay
          behavior: queue_for_sync
        - feature: check_deposit
          behavior: disabled
          reason: "Requires network connection"
        - feature: external_transfer
          behavior: disabled
          
      indicators:
        offline_mode:
          type: banner
          message: "You're offline. Some features may be limited."
          severity: warning
        pending_sync:
          type: badge
          message: "{{count}} pending transactions"
          
    multi_device:
      enabled: true
      sync:
        fields: [payees, preferences, alerts]
        strategy: merge
      conflict:
        on_conflict: prompt_user
```

**Cross-reference**: Part XVI (v1.6 Client Cache Contract)

---

### Enhanced Search: Full-Text Transaction Search (Addendum AL)

Comprehensive search with facets for transaction discovery:

```yaml
searchIntents:
  - name: TransactionFullTextSearch
    version: v1
    target: TransactionView

    scope: "account.owner_id == session.customer_id"

    fields:
      - name: description
        weight: 2.0
        analyzers: [standard, edge_ngram]
        highlight: true

      - name: counterparty
        weight: 1.5
        analyzers: [keyword, phonetic]
        highlight: true

      - name: reference
        weight: 1.0
        analyzers: [keyword]

      - name: category
        weight: 0.5
        facet:
          enabled: true
          type: terms

      - name: amount
        range_facet:
          enabled: true
          ranges: [0-25, 25-100, 100-500, 500-1000, 1000+]

      - name: timestamp
        range_facet:
          enabled: true
          type: date_histogram
          interval: month

    ranking:
      function: bm25
      boost:
        recency:
          field: timestamp
          decay: exponential
          scale: 30d
        amount:
          field: amount
          function: log2p

    suggestions:
      enabled: true
      min_prefix: 2
      sources: [description, counterparty, category]

    pagination:
      mode: cursor
      page_size: 50
      max_results: 10000

    export:
      formats: [csv, pdf]
      max_rows: 100000
```

**Statement Search**:
```yaml
searchIntents:
  - name: StatementFullTextSearch
    version: v1
    target: StatementView

    scope: "statement.account_id IN session.customer_accounts"

    fields:
      - name: period
        type: date_range
        facet:
          enabled: true
          type: date_histogram
          interval: year
      - name: balance_range
        type: numeric_range
        facet:
          enabled: true

    filters:
      - name: year
        field: period.year
        type: terms
      - name: account
        field: account_id
        type: terms
```

**Cross-reference**: Addendum AL (Search Semantics)

---

### Enhanced Assets: Document Lifecycle (Addendum AK)

Complete asset management for banking documents:

```yaml
assets:
  - name: check_image
    version: v1
    types: [image/jpeg, image/png, image/tiff]
    max_size: 10MB

    transformations:
      thumbnail:
        operation: resize
        params: { width: 300, height: 150, fit: contain }
      full:
        operation: compress
        params: { quality: 85, format: jpeg }
      archive:
        operation: convert
        params: { format: tiff, compression: lzw }

    lifecycle:
      type: retention_based
      retention: 7_years

      stages:
        - name: active
          duration: 90d
          storage_tier: hot
        - name: archived
          duration: 2555d
          storage_tier: cold
          access: on_demand_restore

    access:
      policy: ViewCheckImage
      audit_access: true

    compliance:
      regulation: check_21
      preserve_original: true

  - name: identity_document
    version: v1
    types: [image/jpeg, image/png, application/pdf]
    max_size: 25MB

    transformations:
      redacted:
        operation: custom
        handler: pii_redaction_service
        params: { mask_ssn: true, mask_account: true }
      thumbnail:
        operation: resize
        params: { width: 200, height: 200, fit: cover }

    verification:
      required: true
      service: identity_verification_service
      checks:
        - authenticity
        - expiration
        - face_match

    lifecycle:
      type: permanent

    governance:
      classification: pii
      encryption: required
      audit_access: true
      retention: until_account_closed

  - name: signature_card
    version: v1
    types: [image/jpeg, image/png]
    max_size: 5MB

    transformations:
      normalized:
        operation: custom
        handler: signature_normalization_service
        params: { deskew: true, enhance_contrast: true }

    verification:
      service: signature_verification_service

    access:
      requires_clearance: compliance_officer
```

**Cross-reference**: Addendum AK (Asset Semantics)

---

### Enhanced Scheduled Triggers: Banking Automation (Addendum AM)

Comprehensive scheduling for banking operations:

```yaml
scheduledTriggers:
  - name: GenerateMonthlyStatement
    version: v1
    schedule: "0 0 1 * *"  # First day of month
    timezone: account.timezone

    intent: GenerateStatement

    scope: per_entity
    entity_filter: "account.status == 'active' AND account.statement_delivery == 'monthly'"

    idempotency:
      key: [account.id, statement.year, statement.month]

    retry:
      max_attempts: 3
      backoff: exponential
      initial_delay: 5m

    notification:
      on_success:
        channel: email
        template: statement_ready
      on_failure:
        channel: internal
        to: operations_team

  - name: ProcessRecurringPayments
    version: v1
    schedule: "0 6 * * *"  # Daily at 6 AM
    timezone: UTC

    intent: ProcessScheduledPayment

    scope: per_entity
    entity_filter: "payment.next_date == today AND payment.status == 'active'"

    batch:
      enabled: true
      size: 100
      parallel: 10

    idempotency:
      key: [payment.id, execution_date]

    sla:
      complete_within: 2h
      alert_on_breach: true

  - name: DormantAccountCheck
    version: v1
    schedule: "0 1 * * 0"  # Weekly, Sunday at 1 AM
    timezone: UTC

    intent: CheckDormantAccounts

    scope: singleton

    notification:
      on_match:
        channel: email
        to: operations_team
        template: dormant_accounts_report

  - name: FraudModelRefresh
    version: v1
    schedule: "0 3 * * *"  # Daily at 3 AM
    timezone: UTC

    intent: RefreshFraudModel

    scope: singleton

    dependencies:
      - TransactionDataExport
      - ModelTrainingComplete

    retry:
      max_attempts: 1

    notification:
      on_failure:
        channel: pager
        to: ml_oncall
        severity: critical
```

**Cross-reference**: Addendum AM (Scheduled Triggers)

---

# Part II.5: Additional Complete Production Examples

This section provides complete, production-ready implementations for common application patterns beyond banking, demonstrating SMS v1.6 features in diverse domains.

---

## Example: E-Commerce Marketplace

### Overview

Complete e-commerce platform with product catalog, cart, checkout, inventory management, and seller operations.

**Domain**: Retail/Commerce  
**Complexity**: High  
**Key Features**: Search, inventory management, distributed transactions, seller multi-tenancy

### Core Entities

```yaml
models:
  # Product Catalog
  - name: Product
    version: v1
    
    mutability:
      scope: entity
      exclusive: true
    
    authority:
      scope: entity
      resolution: seller_id
    
    fields:
      id:
        type: uuid
        immutable: true
      seller_id:
        type: uuid
        immutable: true
      sku:
        type: string
        constraints:
          unique: true
      name:
        type: string
        constraints:
          max_length: 200
        search:
          indexed: true
          strategy: full_text
          weight: 2.0
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
      price:
        type: money
        constraints:
          min: 0
      images:
        type: list[asset]
        constraints:
          max_items: 10
      attributes:
        type: map[string, string]
      status:
        type: enum[draft, active, inactive, archived]
      created_at:
        type: timestamp
      updated_at:
        type: timestamp
    
    storage:
      role: authoritative
      binding:
        provider: postgres
        database: marketplace_products
        table: products
        indexes:
          - fields: [seller_id, status]
          - fields: [category, status]
    
    transitions:
      - from: draft
        to: active
        guard: has_required_fields
      - from: active
        to: inactive
        guard: seller_action
      - from: [active, inactive]
        to: archived
        guard: retention_policy

  # Inventory Tracking
  - name: InventoryItem
    version: v1
    
    mutability:
      scope: append-only
    
    authority:
      scope: entity
      resolution: product_id
    
    fields:
      id:
        type: uuid
      product_id:
        type: uuid
        relationship:
          target: Product
          cardinality: many_to_one
      warehouse_id:
        type: uuid
      quantity_change:
        type: int  # Positive for restock, negative for sale
      reason:
        type: enum[restock, sale, adjustment, return, damage]
      reference_order_id:
        type: optional[uuid]
      timestamp:
        type: timestamp
      recorded_by:
        type: string
    
    storage:
      role: authoritative
      binding:
        provider: event_stream
        stream: inventory_events
        retention: 7y

  # Order Processing
  - name: Order
    version: v1
    
    mutability:
      scope: entity
      exclusive: true
    
    authority:
      scope: entity
      resolution: customer_id
    
    fields:
      id:
        type: uuid
        immutable: true
      customer_id:
        type: uuid
        immutable: true
      items:
        type: list[OrderItem]
      subtotal:
        type: money
      tax:
        type: money
      shipping_cost:
        type: money
      total:
        type: money
      status:
        type: enum[pending, confirmed, processing, shipped, delivered, cancelled, refunded]
      payment_status:
        type: enum[pending, authorized, captured, failed, refunded]
      payment_method_id:
        type: uuid
      shipping_address:
        type: Address
      created_at:
        type: timestamp
      confirmed_at:
        type: optional[timestamp]
      shipped_at:
        type: optional[timestamp]
      delivered_at:
        type: optional[timestamp]
    
    storage:
      role: authoritative
      binding:
        provider: postgres
        database: marketplace_orders
        table: orders
        indexes:
          - fields: [customer_id, created_at]
          - fields: [status, created_at]
    
    transitions:
      - from: pending
        to: confirmed
        guard: payment_authorized
      - from: confirmed
        to: processing
        guard: inventory_reserved
      - from: processing
        to: shipped
        guard: shipment_created
      - from: shipped
        to: delivered
        guard: delivery_confirmed
      - from: [pending, confirmed]
        to: cancelled
        guard: cancellation_window
```

**Shopping Cart (Ephemeral State)**:
```yaml
dataStates:
  - name: ShoppingCart
    version: v1
    kind: transient
    
    sourced_from:
      - type: CartItem
        aggregation: collect
    
    materialization:
      strategy: on_demand
      freshness: immediate
    
    fields:
      customer_id:
        type: uuid
      items:
        type: list[CartItemView]
      subtotal:
        type: money
      tax:
        type: money
      shipping_estimate:
        type: money
      total:
        type: money
      last_updated:
        type: timestamp
    
    storage:
      binding:
        provider: redis
        key_pattern: "cart:{customer_id}"
        ttl: 30d
    
    cache:
      enabled: true
      ttl: 5m
      invalidate_on: [cart_item_added, cart_item_removed, cart_item_updated]
```

### Views and Materialization

```yaml
dataStates:
  # Product Catalog View
  - name: ProductCatalog
    version: v1
    kind: view
    
    sourced_from:
      - type: Product
        filter: status == "active"
      - type: InventoryItem
        aggregation: sum
        group_by: product_id
    
    materialization:
      strategy: continuous
      freshness: near_real_time
      rebuild:
        strategy: incremental
        checkpoint_frequency: 1m
    
    fields:
      product_id:
        type: uuid
      seller_id:
        type: uuid
      sku:
        type: string
      name:
        type: string
      description:
        type: string
      category:
        type: string
      price:
        type: money
      images:
        type: list[asset]
      available_quantity:
        type: int
      rating_average:
        type: optional[float]
      review_count:
        type: int
      in_stock:
        type: bool
        derived: available_quantity > 0
    
    storage:
      role: derived
      binding:
        provider: elasticsearch
        index: product_catalog
        refresh_interval: 1s
    
    cache:
      enabled: true
      ttl: 5m
      strategy: stale_while_revalidate
      key_fields: [product_id]

  # Seller Dashboard View
  - name: SellerDashboard
    version: v1
    kind: view
    
    sourced_from:
      - type: Product
        filter: seller_id == context.seller_id
      - type: Order
        filter: items[*].seller_id == context.seller_id
      - type: Review
        filter: product.seller_id == context.seller_id
    
    materialization:
      strategy: on_demand
      freshness: stale_ok
      ttl: 1h
    
    fields:
      seller_id:
        type: uuid
      total_products:
        type: int
      active_products:
        type: int
      pending_orders:
        type: int
      revenue_today:
        type: money
      revenue_month:
        type: money
      average_rating:
        type: float
      total_reviews:
        type: int
    
    cache:
      enabled: true
      ttl: 15m
      invalidate_on: [product_status_changed, order_confirmed]
```

### Search Intent

```yaml
searchIntents:
  - name: ProductSearch
    version: v1
    searches: ProductCatalog
    
    query:
      fields: [name, description, category, sku]
      default_operator: and
      phrase_slop: 2
    
    filters:
      required: []
      optional: [category, seller_id, price_range, in_stock, rating]
    
    facets:
      include: [category, seller_id, price_range, in_stock, rating]
    
    pagination:
      default_size: 24
      max_size: 100
      cursor_based: false
    
    sort:
      default: _score
      allowed: [price, rating, name, created_at, popularity]
    
    result:
      includes: [product_id, sku, name, price, images, rating_average, in_stock]
      highlights: [name, description]
    
    authorization:
      policy_set: PublicSearchPolicies
      filter_by: visibility
    
    cache:
      enabled: true
      ttl: 5m
      key_fields: [query, filters, sort]
      invalidate_on: product_updated
    
    routing:
      strategy: broadcast
      timeout: 500ms
```

### Checkout Flow

```yaml
smsFlows:
  - name: CheckoutFlow
    version: v1

    triggeredBy:
      inputIntent: InitiateCheckout

    steps:
      - validate:
          name: validate_cart
          constraints: [cart_not_empty, items_available, prices_current]

      - work_unit:
          name: calculate_totals
          computes: [subtotal, tax, shipping, total]

      - atomic_group:
          name: reserve_and_charge
          realm: order_realm
          steps:
            - work_unit: reserve_inventory
              compensate_with: release_inventory

            - work_unit: authorize_payment
              compensate_with: void_authorization

            - work_unit: create_order
              compensate_with: cancel_order

      - work_unit:
          name: capture_payment
          retry:
            enabled: true
            max_attempts: 3
            backoff: exponential

      - persist:
          state: Order
          fields: [id, customer_id, items, total, status]

      - parallel:
          - work_unit: send_confirmation_email
          - work_unit: notify_sellers
          - work_unit: clear_cart

    on_failure:
      compensation_strategy: automatic
      max_compensation_attempts: 5

    durable:
      enabled: true
      checkpoint:
        frequency: per_step
        storage: checkout_checkpoints
      timeout:
        total: 5m
        per_step: 30s

    resourceRequirements:
      compute:
        min_instances: 2
        max_instances: 20
        scale_on: pending_checkouts > 100
```

### Presentation Layer

```yaml
presentationViews:
  # Product Detail View
  - name: ProductDetailView
    version: v1
    
    consumes:
      dataState: ProductCatalog
      version: v1
      selection: product_id == params.product_id
    
    interactionModel:
      actions:
        - if: in_stock == true
          action: add_to_cart
          label: "Add to Cart"
          intent: AddToCart
          prominence: primary
        - action: add_to_wishlist
          label: "Save for Later"
          intent: AddToWishlist
          prominence: secondary
    
    fieldPermissions:
      - field: seller_contact
        visible:
          policy: SellerContactPolicy
      - field: cost_breakdown
        visible: false
    
    navigation:
      breadcrumb:
        - label: "Home"
          target: /
        - label: "{{category}}"
          target: /category/{{category}}
        - label: "{{name}}"
    
    cache:
      client_side:
        enabled: true
        ttl: 10m
        invalidate_on: [product_updated, price_changed]
        offline_mode: read_only

  # Search Results View
  - name: ProductSearchView
    version: v1
    
    consumes:
      searchIntent: ProductSearch
      version: v1
    
    interactionModel:
      filters:
        - field: category
          type: multi_select
          position: sidebar
        - field: price_range
          type: range_slider
          position: sidebar
        - field: rating
          type: rating_filter
          position: sidebar
        - field: in_stock
          type: toggle
          position: toolbar
      
      sort:
        - field: relevance
          label: "Best Match"
          default: true
        - field: price
          label: "Price: Low to High"
        - field: price_desc
          label: "Price: High to Low"
        - field: rating
          label: "Customer Rating"
      
      pagination:
        type: infinite_scroll
        page_size: 24
    
    cache:
      client_side:
        enabled: true
        ttl: 5m
        strategy: cache_first
```

### Experience Routes

```yaml
experiences:
  - name: MarketplaceExperience
    version: v1
    
    routes:
      - path: /
        composition: HomeComposition
        public: true
      
      - path: /search
        composition: SearchComposition
        public: true
      
      - path: /product/{product_id}
        composition: ProductDetailComposition
        public: true
      
      - path: /cart
        composition: CartComposition
        requires_session: true
      
      - path: /checkout
        composition: CheckoutComposition
        requires_session: true
        requires_auth: true
      
      - path: /orders
        composition: OrderHistoryComposition
        requires_auth: true
      
      - path: /seller/dashboard
        composition: SellerDashboardComposition
        requires_auth: true
        authorization:
          policy_set: SellerPolicies
    
    session:
      provider: jwt
      storage:
        provider: redis
        key_pattern: "session:{session_id}"
        ttl: 30d
      idle_timeout: 30m
      absolute_timeout: 7d
```

### v1.6 Infrastructure

**Storage Binding**:
```yaml
storageBindings:
  - name: marketplace_products
    provider: postgres
    config:
      host: ${POSTGRES_HOST}
      database: marketplace
      pool_size: 20
      connection_timeout: 5s
  
  - name: inventory_events
    provider: nats_stream
    config:
      subject: inventory.*
      retention_policy: limits
      max_age: 7y
      storage: file
  
  - name: product_catalog
    provider: elasticsearch
    config:
      hosts: ${ES_HOSTS}
      index_prefix: marketplace
      replicas: 2
      shards: 5
```

**Caching Strategy**:
```yaml
cacheLayers:
  - target: ProductCatalog
    strategy: stale_while_revalidate
    ttl: 5m
    backend: redis
    invalidation:
      signals: [product_updated, inventory_changed]
  
  - target: ShoppingCart
    strategy: write_through
    ttl: 30d
    backend: redis
  
  - target: ProductSearch
    strategy: cache_first
    ttl: 5m
    backend: redis
    cache_key_fields: [query, filters, sort, page]
```

**Intent Routing**:
```yaml
routingConfig:
  - intent: InitiateCheckout
    strategy: consistent_hashing
    hash_key: customer_id
    timeout: 5s
  
  - intent: ProductSearch
    strategy: broadcast
    timeout: 500ms
    aggregate_results: true
  
  - intent: AddToCart
    strategy: locality_preferred
    fallback: any_available
    timeout: 1s
```

**Resource Requirements**:
```yaml
resourceRequirements:
  name: EcommerceResources
  version: v1
  
  components:
    checkout_workers:
      compute:
        min_instances: 2
        max_instances: 50
        scale_on: queue_depth > 100
      memory:
        per_instance: 512MB
    
    search_indexer:
      compute:
        min_instances: 1
        max_instances: 10
        scale_on: index_lag > 60s
      memory:
        per_instance: 2GB
```

---

## Example: Collaborative Document Editor

### Overview

Real-time collaborative document editing with operational transformation, version history, and offline support.

**Domain**: Productivity  
**Complexity**: High  
**Key Features**: Real-time collaboration, OT/CRDT, offline sync, rich content

### Core Entities

```yaml
models:
  # Document Structure
  - name: Document
    version: v1
    
    mutability:
      scope: entity
      exclusive: true
    
    authority:
      scope: entity
      resolution: owner_id
      migration:
        allowed: true
        requires: owner_consent
    
    fields:
      id:
        type: uuid
        immutable: true
      owner_id:
        type: uuid
        immutable: true
      title:
        type: string
        constraints:
          max_length: 500
        search:
          indexed: true
          strategy: full_text
      content:
        type: structured_content
        schema: document_schema
      version_number:
        type: int
      created_at:
        type: timestamp
      updated_at:
        type: timestamp
      last_edited_by:
        type: uuid
    
    storage:
      role: authoritative
      binding:
        provider: postgres
        database: documents
        table: documents
        indexes:
          - fields: [owner_id, updated_at]
    
    versioning:
      enabled: true
      strategy: snapshot_on_major_change
      retention: all_versions

  # Document Operations (Operational Transform)
  - name: DocumentOperation
    version: v1
    
    mutability:
      scope: append-only
    
    authority:
      scope: session
    
    fields:
      id:
        type: uuid
      document_id:
        type: uuid
        relationship:
          target: Document
          cardinality: many_to_one
      session_id:
        type: uuid
      user_id:
        type: uuid
      operation_type:
        type: enum[insert, delete, format, move]
      position:
        type: int
      content:
        type: optional[string]
      attributes:
        type: optional[map[string, any]]
      client_timestamp:
        type: timestamp
      server_timestamp:
        type: timestamp
      sequence_number:
        type: int
      parent_sequence:
        type: int
    
    storage:
      role: authoritative
      binding:
        provider: event_stream
        stream: document_ops_{document_id}
        retention: 90d
        partitioning: by_document
```

**Collaboration Session**:
```yaml
dataStates:
  - name: CollaborationSession
    version: v1
    kind: transient
    
    sourced_from:
      - type: DocumentOperation
        aggregation: apply_ot
    
    materialization:
      strategy: continuous
      freshness: real_time
    
    fields:
      document_id:
        type: uuid
      active_users:
        type: list[ActiveUser]
      pending_operations:
        type: list[DocumentOperation]
      current_version:
        type: int
      lock_holder:
        type: optional[uuid]
    
    storage:
      binding:
        provider: redis
        key_pattern: "collab:{document_id}"
        ttl: 1h
    
    collaboration:
      enabled: true
      session:
        max_participants: 50
        presence_timeout: 30s
        cursor_sharing: true
      conflict_resolution:
        strategy: operational_transform
        priority: server_timestamp
```

### Real-Time Collaboration Flow

```yaml
smsFlows:
  - name: CollaborativeEditFlow
    version: v1

    triggeredBy:
      inputIntent: ApplyDocumentEdit

    steps:
      - validate:
          name: validate_operation
          constraints: [valid_position, valid_content, session_active]

      - work_unit:
          name: transform_operation
          description: "Apply OT to resolve conflicts"
          algorithm: operational_transform

      - persist:
          state: DocumentOperation
          fields: [id, document_id, operation_type, position, content]

      - work_unit:
          name: broadcast_to_collaborators
          description: "Send operation to all active sessions"
          delivery: at_least_once

      - work_unit:
          name: update_document_version
          conditional: is_major_change

    routing:
      strategy: consistent_hashing
      hash_key: document_id
      sticky_session: true

    resourceRequirements:
      compute:
        min_instances: 5
        max_instances: 100
        scale_on: active_sessions > 1000
```

### Presentation Layer

```yaml
presentationViews:
  # Editor View
  - name: DocumentEditorView
    version: v1
    
    consumes:
      dataState: Document
      version: v1
      selection: id == params.document_id
    
    collaboration:
      enabled: true
      session:
        join_on_load: true
        presence_indicator: cursor_and_highlight
        show_active_users: true
    
    interactionModel:
      actions:
        - action: save
          label: "Save"
          intent: SaveDocument
          trigger: auto_save_interval
          prominence: secondary
        - action: share
          label: "Share"
          intent: ShareDocument
          prominence: primary
        - action: version_history
          label: "History"
          intent: ViewVersionHistory
          prominence: secondary
    
    cache:
      client_side:
        enabled: true
        ttl: indefinite
        strategy: write_through
        offline_mode: full_editing
        sync_strategy: operational_transform
        conflict_resolution: server_wins
```

### v1.6 Infrastructure

**Intent Routing for Collaboration**:
```yaml
routingConfig:
  - intent: ApplyDocumentEdit
    strategy: consistent_hashing
    hash_key: document_id
    sticky_session: true
    timeout: 100ms
    retry:
      enabled: true
      max_attempts: 3
```

**Client Cache Contract**:
```yaml
client_cache:
  - entity: Document
    strategy: write_through
    offline_capabilities:
      read: full_document
      write: queued_operations
    sync_protocol: operational_transform
    conflict_resolution: three_way_merge
```

---

## Example: Logistics & Fleet Management

### Overview

IoT-based fleet tracking with geospatial queries, route optimization, and device telemetry.

**Domain**: Logistics/IoT  
**Complexity**: High  
**Key Features**: Geospatial, IoT telemetry, real-time tracking, route optimization

### Core Entities

```yaml
models:
  # Vehicle Fleet
  - name: Vehicle
    version: v1
    
    mutability:
      scope: entity
      exclusive: true
    
    authority:
      scope: entity
      resolution: fleet_id
    
    fields:
      id:
        type: uuid
        immutable: true
      fleet_id:
        type: uuid
        immutable: true
      vin:
        type: string
        constraints:
          pattern: "^[A-HJ-NPR-Z0-9]{17}$"
      license_plate:
        type: string
      vehicle_type:
        type: enum[truck, van, car, motorcycle]
      status:
        type: enum[active, maintenance, inactive, retired]
      current_driver_id:
        type: optional[uuid]
      home_depot_id:
        type: uuid
    
    storage:
      role: authoritative
      binding:
        provider: postgres
        database: fleet
        table: vehicles

  # GPS Telemetry (IoT Stream)
  - name: VehicleTelemetry
    version: v1
    
    mutability:
      scope: append-only
    
    authority:
      scope: device
      resolution: vehicle_id
    
    fields:
      id:
        type: uuid
      vehicle_id:
        type: uuid
        relationship:
          target: Vehicle
          cardinality: many_to_one
      location:
        type: spatial
        spatial_type: point
        srid: 4326  # WGS 84
      speed:
        type: float  # km/h
      heading:
        type: float  # degrees
      odometer:
        type: float  # kilometers
      fuel_level:
        type: float  # percentage
      engine_status:
        type: enum[on, off, idle]
      timestamp:
        type: timestamp
      device_id:
        type: string
    
    storage:
      role: authoritative
      binding:
        provider: timeseries
        database: vehicle_telemetry
        retention: 90d
        downsample:
          after: 7d
          interval: 5m
    
    device:
      protocol: mqtt
      topic: vehicles/{vehicle_id}/telemetry
      qos: 1
      compression: enabled

  # Delivery Route
  - name: DeliveryRoute
    version: v1
    
    mutability:
      scope: entity
      exclusive: true
    
    authority:
      scope: entity
      resolution: dispatcher_id
    
    fields:
      id:
        type: uuid
      vehicle_id:
        type: uuid
        relationship:
          target: Vehicle
          cardinality: many_to_one
      driver_id:
        type: uuid
      stops:
        type: list[RouteStop]
        constraints:
          min_items: 1
          max_items: 50
      route_geometry:
        type: spatial
        spatial_type: linestring
        srid: 4326
      estimated_duration:
        type: duration
      estimated_distance:
        type: float  # kilometers
      status:
        type: enum[planned, in_progress, completed, cancelled]
      started_at:
        type: optional[timestamp]
      completed_at:
        type: optional[timestamp]
    
    storage:
      role: authoritative
      binding:
        provider: postgres
        database: fleet
        table: routes
        indexes:
          - fields: [vehicle_id, started_at]
          - spatial_index: route_geometry
```

### Geospatial Views

```yaml
dataStates:
  # Live Fleet View
  - name: LiveFleetView
    version: v1
    kind: view
    
    sourced_from:
      - type: Vehicle
        filter: status == "active"
      - type: VehicleTelemetry
        aggregation: latest
        group_by: vehicle_id
        window: 5m
    
    materialization:
      strategy: continuous
      freshness: real_time
      rebuild:
        strategy: streaming
    
    fields:
      vehicle_id:
        type: uuid
      vehicle_type:
        type: string
      current_location:
        type: spatial
        spatial_type: point
      current_speed:
        type: float
      heading:
        type: float
      fuel_level:
        type: float
      last_update:
        type: timestamp
      status:
        type: string
    
    spatial_operations:
      - name: nearby_vehicles
        operation: within_distance
        max_distance: 10km
      - name: vehicles_in_zone
        operation: within_polygon
    
    storage:
      role: derived
      binding:
        provider: postgis
        database: fleet_realtime
        table: live_vehicles
        spatial_index: current_location
    
    cache:
      enabled: true
      ttl: 30s
      strategy: stale_while_revalidate
```

### Route Optimization Flow

```yaml
smsFlows:
  - name: RouteOptimizationFlow
    version: v1

    triggeredBy:
      inputIntent: OptimizeRoutes

    steps:
      - work_unit:
          name: fetch_pending_deliveries
          computes: delivery_list

      - work_unit:
          name: fetch_available_vehicles
          computes: vehicle_list

      - work_unit:
          name: calculate_optimal_routes
          algorithm: vehicle_routing_problem
          timeout: 10s

      - parallel:
          - work_unit: assign_routes_to_vehicles
          - work_unit: notify_drivers

      - persist:
          state: DeliveryRoute
          fields: [id, vehicle_id, stops, route_geometry]

    resourceRequirements:
      compute:
        min_instances: 1
        max_instances: 10
        scale_on: pending_optimizations > 50
      memory:
        per_instance: 4GB  # Route optimization is memory-intensive
```

### IoT Device Integration

```yaml
inputIntents:
  # Telemetry Ingestion
  - name: IngestVehicleTelemetry
    version: v1
    
    input_schema:
      device_id: string
      vehicle_id: uuid
      location: spatial
      speed: float
      fuel_level: float
      timestamp: timestamp
    
    validation:
      - location.is_valid_coordinates
      - speed >= 0 && speed <= 200
      - fuel_level >= 0 && fuel_level <= 100
    
    routing:
      strategy: partition_by_key
      partition_key: vehicle_id
      timeout: 500ms
    
    delivery:
      guarantee: at_least_once
      ordering: per_partition
    
    acknowledgment:
      mode: async
    
    rate_limit:
      per_device: 10/second
      per_fleet: 1000/second
```

### v1.6 Infrastructure

**Storage Binding for IoT**:
```yaml
storageBindings:
  - name: vehicle_telemetry
    provider: timescaledb
    config:
      host: ${TIMESCALE_HOST}
      database: telemetry
      chunk_interval: 1d
      compression:
        enabled: true
        after: 7d
      retention: 90d
  
  - name: fleet_realtime
    provider: postgis
    config:
      host: ${POSTGIS_HOST}
      database: fleet
      spatial_extensions: [postgis, postgis_topology]
```

**Edge Computing**:
```yaml
edge_config:
  - location: vehicle_gateway
    capabilities:
      - telemetry_buffering
      - local_alerting
      - data_compression
    storage:
      type: local_cache
      capacity: 100MB
      retention: 24h
    sync:
      strategy: batch_upload
      interval: 5m
      on_connectivity_restore: immediate
```

---

## Example: Scheduled Report Generation

### Overview

Automated report generation with scheduling, parameterization, and delivery.

**Domain**: Analytics/Reporting  
**Complexity**: Medium  
**Key Features**: Scheduled triggers, data aggregation, multi-format export, email delivery

### Report Definition

```yaml
models:
  - name: ReportDefinition
    version: v1
    
    mutability:
      scope: entity
      exclusive: true
    
    authority:
      scope: entity
      resolution: owner_id
    
    fields:
      id:
        type: uuid
      name:
        type: string
      description:
        type: string
      owner_id:
        type: uuid
      data_source:
        type: string  # Reference to DataState
      filters:
        type: map[string, any]
      aggregations:
        type: list[Aggregation]
      output_format:
        type: enum[pdf, excel, csv, json]
      schedule:
        type: schedule
      recipients:
        type: list[email]
      enabled:
        type: bool
```

### Scheduled Report Flow

```yaml
smsFlows:
  - name: GenerateScheduledReport
    version: v1

    triggeredBy:
      scheduledTrigger: ReportSchedule

    steps:
      - work_unit:
          name: fetch_report_definition
          computes: report_config

      - work_unit:
          name: query_data_source
          timeout: 60s

      - work_unit:
          name: aggregate_data
          computes: report_data

      - work_unit:
          name: generate_output
          format: from_report_config
          timeout: 120s

      - work_unit:
          name: store_report
          storage: report_archive

      - parallel:
          - work_unit: send_email
            retry:
              enabled: true
              max_attempts: 3
          - work_unit: notify_completion

    on_failure:
      notify: report_owner
      store_error: true

    durable:
      enabled: true
      checkpoint:
        frequency: per_step
      timeout:
        total: 10m
```

### Schedule Configuration

```yaml
scheduledTriggers:
  - name: ReportSchedule
    version: v1

    schedule:
      cron: "0 8 * * MON"  # Every Monday at 8 AM
      timezone: America/New_York

    triggers:
      flow: GenerateScheduledReport

    parameters:
      report_id: "{{report_definition.id}}"

    retry:
      on_failure: true
      max_attempts: 3
      backoff: exponential

    resourceRequirements:
      compute:
        min_instances: 1
        max_instances: 5
```

---

# Part III: v1.6 Feature Examples

## Storage Binding Examples

### Key-Value Storage

```yaml
storageBindings:
  - name: UserPreferencesStorage
    version: v1
    type: kv
    kv:
      bucket: "SMS_USER_PREFS"
      key_pattern: "user.{{user_id}}.prefs"
      serialization: json
      history: 3
      ttl: 0s
      replicas: 3
    access:
      read_policy: local
      write_consistency: strong
```

### Stream Storage

```yaml
storageBindings:
  - name: AuditEventStream
    version: v1
    type: stream
    stream:
      name: "SMS_AUDIT"
      subjects:
        - "SMS.AUDIT.{{domain}}.>"
      retention: limits
      max_msgs: 50000000
      max_bytes: 100GB
      max_age: 2555d  # 7 years
      storage: file
      replicas: 3
    access:
      read_policy: eventual
      write_consistency: strong
```

### Object Store Storage

```yaml
storageBindings:
  - name: DocumentStorage
    version: v1
    type: object_store
    object_store:
      bucket: "SMS_DOCUMENTS"
      description: "Customer documents and attachments"
      max_object_size: 100MB
      replicas: 3
    access:
      read_policy: replicated
      write_consistency: eventual
```

## Cache Layer Examples

### Local-Only Cache

```yaml
cacheLayers:
  - name: RequestLocalCache
    version: v1
    topology:
      tier: local
      local:
        max_entries: 10000
        ttl: 1m
        eviction: lru
        max_size: 50MB
    invalidation:
      strategy: ttl
```

### Distributed Cache

```yaml
cacheLayers:
  - name: SessionDistributedCache
    version: v1
    topology:
      tier: distributed
      distributed:
        resource: SessionStore
        ttl: 30m
    invalidation:
      strategy: watch
      watch_keys:
        - "session.{{session_id}}"
      propagate: true
    observability:
      emit_signals: true
      metrics:
        - hit_rate
        - miss_rate
        - latency
      interval: 30s
```

### Hybrid Cache with All Options

```yaml
cacheLayers:
  - name: ProductCatalogCache
    version: v1
    description: "Multi-tier cache for product catalog"
    topology:
      tier: hybrid
      local:
        max_entries: 5000
        ttl: 2m
        eviction: lfu
        max_size: 200MB
      distributed:
        resource: CatalogCache
        ttl: 1h
      hybrid:
        promotion: on_access
    invalidation:
      strategy: hybrid
      ttl: 1h
      watch_keys:
        - "product.{{product_id}}"
        - "category.{{category_id}}"
      propagate: true
    warmup:
      enabled: true
      strategy: scheduled
      queries:
        - "REFRESH_VIEW FeaturedProducts"
        - "REFRESH_VIEW TopCategories"
    observability:
      emit_signals: true
      metrics:
        - hit_rate
        - miss_rate
        - latency
        - size
        - evictions
      interval: 10s
```

## Intent Routing Examples

### NATS Routing (JetStream)

```yaml
inputIntents:
  - name: ProcessOrder
    version: v1
    proposes:
      dataState: OrderRequest
    supplies:
      required: [customer_id, items, payment_method]
    routing:
      transport: nats
      nats:
        subject_pattern: "SMS.INTENT.orders.ProcessOrder.{{realm}}"
        delivery: jetstream
        stream: SMS_ORDERS
        queue: order-processors
        dead_letter: "SMS.DLQ.orders.ProcessOrder"
      response:
        mode: request_reply
        timeout: 30s
```

### HTTP Routing

```yaml
inputIntents:
  - name: CreateUser
    version: v1
    proposes:
      dataState: UserRecord
    supplies:
      required: [email, name]
      optional: [phone, preferences]
    routing:
      transport: http
      http:
        path: "/api/v1/users"
        method: POST
        timeout: 10s
      response:
        mode: request_reply
        timeout: 10s
```

### gRPC Routing

```yaml
inputIntents:
  - name: StreamAnalytics
    version: v1
    proposes:
      dataState: AnalyticsRequest
    supplies:
      required: [query, time_range]
    routing:
      transport: grpc
      grpc:
        service: analytics.AnalyticsService
        method: StreamMetrics
        streaming: true
      response:
        mode: async
        timeout: 5m
        callback: "SMS.CALLBACK.analytics.{{correlation_id}}"
```

### Async with Callback

```yaml
inputIntents:
  - name: GenerateReport
    version: v1
    proposes:
      dataState: ReportRequest
    supplies:
      required: [report_type, date_range]
    routing:
      transport: nats
      nats:
        subject_pattern: "SMS.INTENT.reports.GenerateReport"
        delivery: jetstream
        stream: SMS_REPORTS
      response:
        mode: async
        timeout: 1h
        callback: "SMS.CALLBACK.reports.{{correlation_id}}"
```

## Resource Requirements Examples

### E-Commerce Resources

```yaml
resourceRequirements:
  name: EcommerceResources
  version: v1
  description: "Resource requirements for e-commerce platform"
  
  eventLogs:
    - name: OrderEventLog
      purpose: event_sourcing
      ordering: causal
      delivery: at_least_once
      durability: persistent
      retention:
        strategy: bounded
        max_age: 365d
        max_size: 100GB
      replication:
        min_replicas: 3
        consistency: strong
    
    - name: InventoryEventLog
      purpose: event_sourcing
      ordering: strict
      delivery: exactly_once
      durability: persistent
      retention:
        strategy: bounded
        max_age: 90d
      replication:
        min_replicas: 3
        consistency: strong
  
  entityStores:
    - name: ProductCatalog
      purpose: derived_state
      access_pattern: key_value
      consistency: eventual
      durability: persistent
      history:
        enabled: false
      expiration:
        enabled: true
        ttl: 24h
      replication:
        min_replicas: 3
      capacity:
        max_entry_size: 5MB
        max_entries: 1000000
    
    - name: CartStore
      purpose: session
      access_pattern: key_value
      consistency: strong_per_key
      durability: ephemeral
      expiration:
        enabled: true
        ttl: 7d
      replication:
        min_replicas: 3
  
  objectStores:
    - name: ProductImages
      purpose: media
      durability: persistent
      replication:
        min_replicas: 3
      capacity:
        max_object_size: 50MB
  
  provisioning:
    mode: validate
    migrations:
      enabled: true
      strategy: blue_green
```

## Client Cache Examples

### Mobile Banking App

```yaml
experiences:
  - name: MobileBankingApp
    version: v1
    
    clientCache:
      enabled: true
      storage:
        type: persistent
        max_size: 200MB
        encryption: true
      strategies:
        primary:
          strategy: network_first
          ttl: 2m
        related:
          strategy: stale_while_revalidate
          ttl: 10m
          lazy_cache: true
      prefetch:
        enabled: true
        views:
          - AccountBalanceView
          - RecentActivityView
        trigger: on_connect
      offline:
        enabled: true
        required_views:
          - AccountBalanceView
          - RecentActivityView
          - PendingPaymentsView
        sync_on_reconnect: true
        conflict_resolution: server_wins
        intent_sync:
          on_cas_failure: reject_and_notify
          retry_with_refresh: true
        governance:
          validate_on_sync: true
          on_violation: reject
      version_policy:
        on_upgrade: invalidate_all
        migration: eager
```

### Web Dashboard

```yaml
experiences:
  - name: AdminDashboard
    version: v1
    
    clientCache:
      enabled: true
      storage:
        type: memory
        max_size: 50MB
      strategies:
        primary:
          strategy: cache_first
          ttl: 5m
        related:
          strategy: cache_first
          ttl: 15m
          lazy_cache: false
      prefetch:
        enabled: true
        views:
          - DashboardSummaryView
          - ActiveAlertsView
        trigger: on_idle
```

## Cache Signal Examples

### Healthy Cache Signal

```yaml
signals:
  - type: cache
    source:
      component: cache
      name: ViewHybridCache
      tier: hybrid
      region: us-east-1
    subject:
      cache: ViewHybridCache
    metrics:
      hit_rate: 0.92
      miss_rate: 0.08
      hit_count: 45000
      miss_count: 4000
      latency_p50: 0.1ms
      latency_p99: 2ms
      size: 8500
      max_entries: 10000
      evictions: 120
      pressure: low
    severity: info
    timestamp: "2024-01-15T10:30:00Z"
```

### Degraded Cache Signal

```yaml
signals:
  - type: cache
    source:
      component: cache
      name: SessionCache
      tier: distributed
      region: us-west-2
    subject:
      cache: SessionCache
    metrics:
      hit_rate: 0.45
      miss_rate: 0.55
      latency_p50: 15ms
      latency_p99: 150ms
      size: 95000
      max_entries: 100000
      evictions: 5000
      pressure: critical
    severity: critical
    timestamp: "2024-01-15T10:30:00Z"
```

---

# Part IV: Grammar Quick Examples

## Minimal Examples

### Minimal Storage Binding

```yaml
storageBindings:
  - name: SimpleKV
    version: v1
    type: kv
    kv:
      bucket: "SIMPLE_STORE"
```

### Minimal Cache Layer

```yaml
cacheLayers:
  - name: SimpleCache
    version: v1
    topology:
      tier: local
      local:
        eviction: lru
```

### Minimal Intent Routing

```yaml
inputIntents:
  - name: SimpleIntent
    version: v1
    proposes:
      dataState: SimpleState
    supplies:
      required: [data]
    routing:
      transport: nats
      nats:
        subject_pattern: "SMS.INTENT.simple"
```

### Minimal Resource Requirements

```yaml
resourceRequirements:
  name: SimpleResources
  version: v1
  entityStores:
    - name: SimpleStore
      purpose: authoritative_state
```

### Minimal Client Cache

```yaml
experiences:
  - name: SimpleExperience
    version: v1
    entry_point: MainView
    includes: [MainView]
    clientCache:
      enabled: true
      storage:
        type: memory
```

## Basic Entity with Authority

```yaml
dataTypes:
  - name: Order
    version: v1
    fields:
      id: uuid
      customer_id: uuid
      total: decimal
      status: enum[draft, paid, fulfilled]

models:
  - name: Order
    mutability:
      scope: entity
      exclusive: true
    authority:
      scope: entity
      resolution: entity.region
```

## Append-Only Entity (Immutable Facts)

```yaml
dataTypes:
  - name: Transaction
    version: v1
    fields:
      id: uuid
      timestamp: timestamp
      amount: decimal
      type: enum[debit, credit]

models:
  - name: Transaction
    mutability:
      scope: append-only
```

## Relationship with Causality

```yaml
relationships:
  - name: order_contains_item
    from: OrderItem
    to: Order
    type: belongs_to
    cardinality: many-to-one
    semantics:
      causal: true
      ordered: true
```

## View with Invariant

```yaml
views:
  - name: OrderTotal
    inputs: [Order, OrderItem]
    aggregation:
      type: sum
      expression: sum(items.price * items.quantity)
    invariant:
      expression: result == order.total
      on_violation: flag
    authority_agnostic: true
```

## Simple Input Intent

```yaml
inputIntents:
  - name: CreateOrder
    version: v1
    proposes:
      dataState: OrderDraft
      version: v1
    supplies:
      required:
        - customer_id
        - items
      optional:
        - notes
    constraints:
      guarantees:
        - items.length > 0
```

## SMS Flow with Atomic Group

```yaml
smsFlows:
  - name: CheckoutFlow
    triggeredBy:
      inputIntent: InitiateCheckout
    steps:
      - type: atomic
        atomicGroup:
          name: ValidateAndReserve
          realm: inventory
          steps:
            - work_unit: validate_order
            - work_unit: reserve_inventory
      - work_unit: charge_payment
        realm: billing
    failure:
      policy: compensate
      compensations:
        - on: charge_payment
          do: release_inventory
```

## Policy with RBAC

```yaml
policies:
  - name: ViewOrders
    version: v1
    appliesTo:
      type: presentationView
      name: CustomerOrderList
    effect: allow
    when: subject.role in [customer, admin, support]
```

## v1.5: Form Intent Binding

```yaml
presentationViews:
  - name: CreateOrderForm
    version: v1
    triggers:
      intent: CreateOrder
      binding: form
      field_mapping:
        - view_field: customer_id
          supplies: customer_id
          source: context
          required: true
        - view_field: items
          supplies: items
          source: input
          required: true
```

## v1.5: Asset Field

```yaml
dataTypes:
  - name: Product
    version: v1
    fields:
      id: uuid
      name: string
      image: asset

assets:
  - name: image
    types: [image/jpeg, image/png]
    max_size: 5MB
    transformations:
      thumbnail: resize(width: 200, height: 200)
    retention:
      duration: permanent
```

## v1.5: Scheduled Trigger

```yaml
scheduledTriggers:
  - name: DailyReport
    schedule: "0 0 * * *"  # Daily at midnight
    intent: GenerateDailyReport
    scope: singleton
    idempotency:
      key: [date]
```

## v1.5: Notification

```yaml
notificationChannels:
  - name: OrderShipped
    trigger:
      event: Order.status_changed
      condition: new_status == "shipped"
    channels:
      - type: email
        template: order_shipped_email
        to: customer.email
      - type: push
        template: order_shipped_push
        condition: customer.push_enabled
```

## v1.5: Search Configuration

```yaml
searchIntents:
  - name: ProductSearch
    target: ProductView
    fields:
      - name: name
        weight: 2.0
        analyzers: [standard, edge_ngram]
      - name: description
        weight: 1.0
        analyzers: [standard]
    ranking:
      function: bm25
```

## v1.5: Data Governance

```yaml
dataGovernances:
  - model: Customer
    classification:
      level: pii
      categories: [personal_identifiable]
    retention:
      duration: 7_years
      delete_after: true
    right_to_erasure:
      supported: true
```

---

# Part V: Common Patterns Library

## Pattern: Ledger/Balance

**Use Case**: Banking accounts, wallet balances, inventory levels

**Entities**:
- `Account`: Identity, type, ownership
- `Transaction`: Immutable fact (amount, parties, timestamp)

**Relationships**:
- Transaction → Account (debit)
- Transaction → Account (credit)

**Views**:
- `AccountBalance`: sum(credits) - sum(debits)

**Invariant**: `AccountBalance.result >= 0`

**Key Principle**: Balance is derived, not stored. Transactions are append-only.

---

## Pattern: Inventory

**Use Case**: Stock management, warehouse inventory

**Entities**:
- `Product`: Identity, description, SKU
- `StockEvent`: Immutable adjustment (quantity, reason, timestamp)

**Relationships**:
- StockEvent → Product (affects)

**Views**:
- `ProductStock`: sum(adjustments)

**Invariant**: `ProductStock.result >= 0`

**Key Principle**: Stock level is derived from append-only events.

---

## Pattern: Scheduling

**Use Case**: Appointments, reservations, resource booking

**Entities**:
- `Resource`: Identity, type, availability rules
- `Booking`: Immutable reservation (start, end, resource)

**Relationships**:
- Booking → Resource (reserves)

**Views**:
- `ResourceSchedule`: time slots with bookings

**Invariant**: No overlapping bookings for same resource

**Key Principle**: View detects conflicts; compensate or notify.

---

## Pattern: Voting

**Use Case**: Elections, polls, surveys

**Entities**:
- `Voter`: Identity, eligibility
- `Vote`: Immutable ballot (choice, timestamp)

**Relationships**:
- Vote → Voter (cast_by)

**Views**:
- `VoterStatus`: has_voted flag
- `TallyView`: count per choice

**Invariant**: At most one vote per voter

**Key Principle**: View enforces uniqueness; detect duplicates.

---

## Pattern: Multi-Region Authority

**Use Case**: User profiles, regional data

**Authority Configuration**:
```yaml
authorities:
  - scope: entity
    resolution: entity.home_region
    migration:
      allowed: true
      strategy: pause-and-cutover
      triggers:
        - type: latency
          condition: avg_write_latency > 100ms
```

**Key Principle**: Authority follows access patterns; views remain global.

---

## Pattern: Multi-Tier Caching with Invalidation

**Problem**: Need fast reads with eventual consistency and real-time updates.

**Solution**:

```yaml
# 1. Define resources
resourceRequirements:
  name: CacheResources
  entityStores:
    - name: L2Cache
      purpose: cache
      consistency: eventual
      expiration:
        enabled: true
        ttl: 1h

# 2. Define hybrid cache with watch
cacheLayers:
  - name: DataCache
    topology:
      tier: hybrid
      local:
        max_entries: 1000
        ttl: 1m
        eviction: lru
      distributed:
        resource: L2Cache
        ttl: 30m
      hybrid:
        promotion: on_miss
    invalidation:
      strategy: watch
      watch_keys:
        - "entity.{{id}}"
      propagate: true
    observability:
      emit_signals: true
      interval: 10s
```

---

## Pattern: Event Sourcing with Streaming

**Problem**: Need audit trail and event replay capability.

**Solution**:

```yaml
# 1. Define event stream
resourceRequirements:
  name: EventResources
  eventLogs:
    - name: DomainEvents
      purpose: event_sourcing
      ordering: causal
      delivery: at_least_once
      durability: persistent
      retention:
        strategy: bounded
        max_age: 2555d  # 7 years
      replication:
        min_replicas: 3
        consistency: strong

# 2. Bind storage
storageBindings:
  - name: DomainEventStream
    type: stream
    stream:
      name: "DOMAIN_EVENTS"
      subjects:
        - "EVENTS.{{aggregate}}.>"
      retention: limits
      max_age: 2555d
      storage: file
      replicas: 3
```

---

## Pattern: Offline-First Mobile App

**Problem**: App must work without network connectivity.

**Solution**:

```yaml
experiences:
  - name: OfflineFirstApp
    clientCache:
      enabled: true
      storage:
        type: persistent
        max_size: 500MB
        encryption: true
      strategies:
        primary:
          strategy: cache_first
          ttl: 1h
      prefetch:
        enabled: true
        views:
          - AllCriticalViews
        trigger: on_connect
      offline:
        enabled: true
        required_views:
          - CriticalView1
          - CriticalView2
        sync_on_reconnect: true
        conflict_resolution: manual
        intent_sync:
          on_cas_failure: queue_for_retry
          retry_with_refresh: true
        governance:
          validate_on_sync: true
          on_violation: queue_for_review
      version_policy:
        on_upgrade: keep_compatible
        grace_period: 48h
        migration: lazy
```

---

## Pattern: High-Throughput Processing

**Problem**: Need to process millions of events with exactly-once semantics.

**Solution**:

```yaml
# 1. Intent with fire-and-forget
inputIntents:
  - name: ProcessEvent
    routing:
      transport: nats
      nats:
        subject_pattern: "SMS.EVENTS.process.{{partition}}"
        delivery: jetstream
        stream: SMS_EVENTS
        queue: event-processors
      response:
        mode: fire_and_forget

# 2. Resource with exactly-once
resourceRequirements:
  eventLogs:
    - name: ProcessedEvents
      purpose: audit
      ordering: causal
      delivery: exactly_once
      durability: persistent
      retention:
        strategy: bounded
        max_age: 30d
```

---

## Pattern: Real-Time Collaboration

**Use Case**: Document editing, whiteboard, design tools

**Session**:
```yaml
collaborativeSessions:
  - name: DocumentEditSession
    scope: document
    state:
      - name: document_content
        type: structured_content
        sync: realtime
        conflict_resolution: operational_transform
    awareness:
      presence: true
      cursor_tracking: true
```

**Key Principle**: Operational transform for conflict-free merges.

---

## Pattern: Durable Workflows

**Use Case**: Loan approval, compliance review, multi-step processes

**Workflow Structure**:
```yaml
durableWorkflows:
  - name: MultiStepProcess
    steps:
      - name: automated_check
        type: work_unit
      - name: human_review
        type: human_task
        timeout: 3_days
        escalation:
          after: 2_days
      - name: wait_for_event
        type: wait_for_event
        timeout: 7_days
```

**Key Principle**: Persistent state across days/weeks with human interaction.

---

## Pattern: Delegation with Consent

**Use Case**: Power of attorney, authorized users, shared access

**Delegation**:
```yaml
delegations:
  - name: AccountDelegate
    delegator: account.owner_id
    delegate: authorized_user.id
    permissions: [ViewBalance, InitiateTransfer]
    consent:
      required: true
      captured_at: delegation.created_at
    audit:
      log_all_actions: true
```

**Key Principle**: Explicit consent with full audit trail.

---

## Pattern: Search with Facets

**Use Case**: Product search, transaction search, document search

**Search Configuration**:
```yaml
searchIntents:
  - name: ProductSearch
    fields:
      - name: name
        weight: 2.0
        analyzers: [standard, edge_ngram]
      - name: category
        facet: true
      - name: price
        range_facet: true
    ranking:
      function: bm25
```

**Key Principle**: Full-text search with categorical and range facets.

---

## Pattern: External API Integration with Caching

**Problem**: Need to cache responses from external APIs.

**Solution**:

```yaml
# 1. Cache for external responses
cacheLayers:
  - name: ExternalAPICache
    topology:
      tier: distributed
      distributed:
        resource: APIResponseCache
        ttl: 5m
    invalidation:
      strategy: ttl
      ttl: 5m
    observability:
      emit_signals: true
      metrics:
        - hit_rate
        - miss_rate
      interval: 30s

# 2. External integration
externalDependencies:
  - name: PaymentGateway
    provider:
      type: rest_api
      base_url: "https://api.payments.com/v1"
    operations:
      - name: process_payment
        method: POST
        path: "/charges"
        retry:
          max_attempts: 3
          backoff: exponential
          idempotency_key: "{{ payment_id }}"
        circuit_breaker:
          enabled: true
          failure_threshold: 5
          timeout: "30s"
```

---

## Pattern: Data Governance

**Use Case**: GDPR compliance, PII management, retention policies

**Governance**:
```yaml
dataGovernances:
  - model: Customer
    classification:
      level: pii
    retention:
      duration: 7_years
    right_to_erasure:
      supported: true
      cascade: [AuditLog]
```

**Key Principle**: Classification drives retention and subject rights.

---

## Anti-Patterns to Avoid

### ❌ Mutable Aggregate in Entity

Don't store derived values like `balance`, `total_orders`, or `count` in entities.

### ❌ Cross-Entity Preconditions

Don't check another entity's state before writing.

### ❌ Synchronous Invariants

Don't require invariants to hold immediately after write.

### ❌ Authority Coupling

Don't design entities that must update together but have different authority.

### ❌ Transport in Flow Steps (I9 Violation)

Don't specify transport configuration inside SMS flow steps.

---

## Pattern Selection Guide

| Requirement | Pattern |
|-------------|---------|
| Balance/Total tracking | Ledger/Balance |
| Stock management | Inventory |
| Time-based booking | Scheduling |
| One action per entity | Voting |
| Regional data | Multi-Region Authority |
| Fast reads with updates | Multi-Tier Caching |
| Audit trail | Event Sourcing |
| Mobile offline | Offline-First |
| High throughput | High-Throughput Processing |
| Live editing | Real-Time Collaboration |
| Multi-day process | Durable Workflows |
| Shared access | Delegation with Consent |
| User search | Search with Facets |
| Third-party service | External API Integration |
| PII compliance | Data Governance |

---

# Appendix A: v1.5 Feature Examples

This appendix provides additional examples for v1.5 features not covered in the main Banking example.

## Asset Upload Flow Example

```yaml
smsFlows:
  - name: DocumentUploadFlow
    version: v1
    triggeredBy:
      inputIntent: UploadDocumentIntent
    steps:
      - work_unit: ValidateFile
        input:
          file: "intent.file"
          allowed_types: ["application/pdf", "image/jpeg", "image/png"]
          max_size: "25MB"
      - work_unit: ScanForVirus
        input:
          file: "intent.file"
        timeout: 60s
      - work_unit: ExtractMetadata
        input:
          file: "intent.file"
      - work_unit: StoreAsset
        input:
          file: "intent.file"
          metadata: "extracted_metadata"
          retention: "7_years"
      - work_unit: UpdateEntityReference
        input:
          entity_id: "intent.entity_id"
          asset_id: "stored_asset.id"
```

## Scheduled Batch Processing Example

```yaml
scheduledTriggers:
  - name: NightlyReconciliation
    version: v1
    schedule: "0 2 * * *"  # 2 AM daily
    timezone: UTC

    intent: RunReconciliation

    scope: singleton

    batch:
      enabled: true
      size: 1000
      parallel: 20

    idempotency:
      key: [date]

    monitoring:
      sla:
        target_completion: 4h
        alert_at: 80%
      track_duration: true

    retry:
      max_attempts: 3
      backoff: exponential

    notification:
      on_success:
        channel: internal
        template: reconciliation_complete
      on_failure:
        channel: pager
        to: on_call_team
```

## Governance Compliance Example

```yaml
dataGovernances:
  - model: PatientRecord
    classification:
      level: phi  # Protected Health Information
      categories: [health_data, personal_identifiable]
      regulations: [hipaa, gdpr]
      
    retention:
      duration: 10_years
      from: last_treatment_date
      delete_after: true
      archive_after: 3_years
      
    access_control:
      requires: [hipaa_training, role_healthcare]
      audit: always
    
  encryption:
    at_rest: aes_256
    in_transit: tls_1_3
    key_rotation: 90d
    
  right_to_erasure:
    supported: true
    exceptions: [legal_hold, ongoing_treatment]
    pseudonymize_on_erasure: true
    
  data_residency:
    required_regions: [us-east, us-west]
    prohibited_regions: [*]
```

---

# Appendix B: Migration Examples

## v1.4 → v1.5 Migration

### Adding Asset Support

**Before (v1.4)**:
```yaml
dataTypes:
  - name: Document
    version: v1
    fields:
      id: uuid
      file_url: string  # External URL to file
      file_type: string
      file_size: integer
```

**After (v1.5)**:
```yaml
dataTypes:
  - name: Document
    version: v2
    fields:
      id: uuid
      content: asset  # Managed asset with lifecycle

assets:
  - name: document_content
    types: [application/pdf, image/jpeg, image/png]
    max_size: 50MB
    lifecycle:
      retention: 7_years
    transformations:
      thumbnail: resize(width: 200, height: 200)
```

### Adding Search Support

**Before (v1.4)**:
```yaml
# Manual filtering in views
views:
  - name: ProductList
    inputs: [Product]
    filters:
      - field: name
        operator: contains
```

**After (v1.5)**:
```yaml
searchIntents:
  - name: ProductSearch
    target: ProductView
    fields:
      - name: name
        weight: 2.0
        analyzers: [standard, edge_ngram]
      - name: description
        weight: 1.0
    ranking:
      function: bm25
```

## v1.5 → v1.6 Migration

### Adding Storage Binding

**Before (v1.5)**: Implicit storage

**After (v1.6)**:
```yaml
storageBindings:
  - name: ProductStateStorage
    version: v1
    type: kv
    kv:
      bucket: "SMS_PRODUCTS"
      key_pattern: "product.{{product_id}}"
      serialization: json
      replicas: 3
    access:
      read_policy: local
      write_consistency: strong

dataStates:
  - name: ProductRecord
    type: Product
    lifecycle: persistent
    storage:
      binding: ProductStateStorage
```

### Adding Cache Layer

**Before (v1.5)**: No explicit caching

**After (v1.6)**:
```yaml
cacheLayers:
  - name: ProductCache
    version: v1
    topology:
      tier: hybrid
      local:
        max_entries: 1000
        ttl: 5m
        eviction: lru
      distributed:
        resource: ProductCacheStore
        ttl: 30m
      hybrid:
        promotion: on_miss
    invalidation:
      strategy: watch
      watch_keys:
        - "product.{{product_id}}"
```

### Adding Intent Routing

**Before (v1.5)**: Implicit routing

**After (v1.6)**:
```yaml
inputIntents:
  - name: CreateProduct
    version: v1
    proposes:
      dataState: ProductRecord
    supplies:
      required: [name, price, category]
    routing:
      transport: nats
      nats:
        subject_pattern: "SMS.INTENT.products.CreateProduct"
        delivery: jetstream
        stream: SMS_PRODUCTS
        queue: product-workers
      response:
        mode: request_reply
        timeout: 10s
```

---

# Appendix C: v1.6 Design Checklist

For each v1.6 feature, verify:

## Storage Binding Checklist

- [ ] Type matches configuration block (kv/stream/object_store)
- [ ] Key pattern references valid fields
- [ ] Replicas >= 1
- [ ] Consistency levels compatible (no strong consistency with local-only reads)
- [ ] TTL appropriate for use case (0s for permanent data)

## Cache Layer Checklist

- [ ] Tier matches configuration blocks
- [ ] Distributed cache has valid resource reference
- [ ] Hybrid cache has both local and distributed config
- [ ] Watch invalidation has watch_keys specified
- [ ] Propagate enabled for multi-instance deployment
- [ ] Warmup queries valid for warmup strategy

## Intent Routing Checklist

- [ ] Transport matches configuration block
- [ ] NATS has subject_pattern
- [ ] HTTP has path and method
- [ ] gRPC has service and method
- [ ] Async response has callback
- [ ] JetStream delivery has stream reference
- [ ] No transport configuration in flow steps (I9 compliance)

## Resource Requirements Checklist

- [ ] No duplicate names across all resource types
- [ ] Purpose appropriate for consistency level
- [ ] History disabled for cache purpose
- [ ] Exactly-once delivery not with ephemeral durability
- [ ] Replication minimum >= 1

## Client Cache Checklist

- [ ] Storage type supports use case (persistent for offline)
- [ ] Encryption enabled for sensitive data
- [ ] Required views defined for offline mode
- [ ] Conflict resolution appropriate for data type
- [ ] CAS failure handling respects I4 invariant

---

## Conclusion

This document provides:
1. **Modeling philosophy** - Invariant-oriented design principles
2. **Practical guidance** - Checklists, templates, and decision trees
3. **Complete Banking example** - Production-ready reference demonstrating all v1.5 addendums and v1.6 features
4. **v1.6 Feature examples** - Storage binding, caching, routing, and resource requirements
5. **Quick reference** - Grammar examples for common elements
6. **Pattern library** - Reusable solutions for common problems
7. **Migration guidance** - Upgrading from v1.4 to v1.5 to v1.6
8. **Design checklists** - Validation for v1.6 features

Use this document to learn SMS v1.6, model new applications, and find proven patterns for your requirements.

---

# Part VI: Comprehensive Production Examples Catalog

This section catalogs 15 real-world application examples. Each example represents a production-ready architecture demonstrating the full SMS v1.6 specification.

## Example Catalog

| # | Example | Industry | Complexity | Primary Features |
|---|---------|----------|------------|------------------|
| 01 | Enterprise Banking Platform | Financial | High | Multi-region, workflows, delegation, compliance |
| 02 | Telemedicine Platform | Healthcare | High | Consent, scheduling, video, HIPAA compliance |
| 03 | E-Commerce Marketplace | Retail | Medium-High | Search, assets, payments, recommendations |
| 04 | Logistics & Fleet Management | Transportation | High | IoT, spatial, real-time, offline mobile |
| 05 | Customer Service Center | Cross-Industry | Medium | Chatbot, voice, human tasks, knowledge base |
| 06 | Corporate HR Platform | Enterprise | Medium | Graph hierarchy, workflows, GDPR, delegation |
| 07 | Property Management | Real Estate | Medium | Spatial, IoT, assets, temporal access |
| 08 | Learning Management System | Education | Medium | Rich content, assets, collaboration, offline |
| 09 | Social Networking Platform | Consumer | High | Graph, streaming, ML, governance |
| 10 | Smart Factory / Industrial IoT | Manufacturing | High | IoT, streaming, ML, safety, compliance |
| 11 | Legal Case Management | Legal | High | Workflows, legal holds, delegation, search |
| 12 | Voice-First Smart Home | Consumer IoT | Medium | Voice, IoT, device groups, temporal access |
| 13 | Subscription Commerce | SaaS | Medium | External APIs, webhooks, scheduling, metering |
| 14 | Collaborative Document Editor | Productivity | High | Real-time collaboration, rich content, offline |
| 15 | Government Permitting System | Government | High | Durable workflows, spatial, accessibility |

These examples demonstrate production patterns including:

- **Entity-scoped authority** with multi-region failover
- **View-based invariant enforcement** for correctness guarantees
- **Durable workflows** with human tasks and compensation
- **External API integration** with circuit breakers and fallbacks
- **Data governance** for GDPR/HIPAA compliance
- **Real-time collaboration** and streaming updates
- **IoT and edge device** patterns with offline support
- **Accessibility and conversational** interfaces
- **Search, spatial, and graph** queries
- **Asset management** with lifecycle policies
- **v1.6 storage binding** with appropriate backend selection
- **v1.6 caching strategies** for performance optimization
- **v1.6 intent routing** for scalable request handling
- **v1.6 resource requirements** for declarative provisioning
- **v1.6 client cache** for offline-first experiences

For a complete demonstration of core features, see [Part II: Complete Banking Example](#part-ii-complete-banking-example-all-features) which provides a fully-worked example suitable for production reference.

### Implementation Notes

These examples serve as architectural references demonstrating:

1. **Feature Coverage**: Each example showcases specific SMS features appropriate to its domain
2. **Real-World Complexity**: Examples model actual production requirements, not simplified demos
3. **Best Practices**: Patterns demonstrate proven approaches to common problems
4. **Integration Points**: Show how different SMS features work together in complete applications

### Example 01: Enterprise Banking Platform

**Industry**: Financial Services  
**Complexity**: High  
**Key Features**: Multi-region authority, entity-scoped write control, durable workflows, delegation, compliance

**Architecture Highlights**:
- Customer accounts with region-specific authority placement
- Cross-border viewing with local authority for writes
- Compliance-driven data residency enforcement
- Audit trails for regulatory requirements
- Durable payment workflows with external gateway integration
- Delegation for temporary account access (e.g., accountant, executor)
- Real-time fraud detection with ML inference
- Customer service workflows with human approval steps

**Data Governance**:
- PII classification for customer data
- Event-based retention (account closure + grace period)
- Legal hold support for investigations
- Encryption at rest and in transit
- Right-to-erasure implementation

**v1.6 Enhancements**:
- Storage binding: KV for accounts, object store for documents
- Cache layer: Hybrid caching for account balances
- Intent routing: NATS for high-volume transactions
- Resource requirements: Auto-provisioning for seasonal load
- Client cache: Offline balance viewing

### Example 02: Telemedicine Platform

**Industry**: Healthcare  
**Complexity**: High  
**Key Features**: Consent management, scheduled appointments, video integration, HIPAA compliance

**Architecture Highlights**:
- Patient health records with strict access control
- Provider-patient consent for data access
- Time-bounded delegation for care team members
- Scheduled video consultations with temporal triggers
- Prescription workflows with external pharmacy integration
- HIPAA audit logging
- Edge device integration for vital signs monitoring

**Data Governance**:
- PHI classification for all patient data
- Retention policies for medical records (7-10 years)
- Patient consent tracking with revocation support
- Break-glass emergency access with audit
- Anonymization for research datasets

**v1.6 Enhancements**:
- Storage binding: HIPAA-compliant object store for medical images
- Cache layer: Local caching for provider dashboards
- Intent routing: Priority routing for emergency cases
- Client cache: Offline patient history viewing

### Example 03: E-Commerce Marketplace

**Industry**: Retail  
**Complexity**: Medium-High  
**Key Features**: Full-text search, asset management, payment processing, ML recommendations

**Architecture Highlights**:
- Product catalog with full-text and faceted search
- Image assets with CDN delivery and transformations
- Shopping cart with session management
- Durable order workflows with payment gateway
- Inventory materialized views with eventual consistency
- Product recommendations using ML inference
- Review and rating system with moderation
- Multi-seller marketplace with policy-based commission

**v1.6 Enhancements**:
- Storage binding: Object store for product images, KV for cart state
- Cache layer: Distributed caching for product catalog
- Intent routing: Load-balanced order processing
- Resource requirements: Auto-scale search indexers
- Client cache: Offline shopping cart with sync

### Example 04: Logistics & Fleet Management

**Industry**: Transportation  
**Complexity**: High  
**Key Features**: IoT device integration, spatial queries, real-time tracking, offline mobile

**Architecture Highlights**:
- Vehicle fleet with GPS tracking
- Delivery routes with spatial optimization
- Driver mobile apps with offline operation
- Real-time package tracking for customers
- Geofencing for delivery zone enforcement
- Temporal windows for delivery time slots
- IoT sensor data for vehicle diagnostics
- Edge processing for route optimization

**v1.6 Enhancements**:
- Storage binding: Time-series store for GPS data
- Cache layer: Local caching for driver apps
- Intent routing: Regional routing for delivery zones
- Resource requirements: Scale based on active vehicles
- Client cache: Offline route updates with background sync

### Example 05: Customer Service Center

**Industry**: Cross-Industry  
**Complexity**: Medium  
**Key Features**: Conversational UI, voice interface, human tasks, knowledge base search

**Architecture Highlights**:
- Multi-channel support (voice, chat, email)
- NLU-driven intent recognition
- Knowledge base with semantic search
- Human escalation workflows
- Agent assignment with skill matching
- Case management with durable workflows
- Real-time collaboration between agents
- Customer sentiment analysis

**v1.6 Enhancements**:
- Storage binding: Search index for knowledge base
- Cache layer: Distributed caching for frequent queries
- Intent routing: Priority routing for premium customers
- Client cache: Offline case viewing for mobile agents

### Example 06: Corporate HR Platform

**Industry**: Enterprise  
**Complexity**: Medium  
**Key Features**: Organizational hierarchy, approval workflows, GDPR compliance, delegation

**Architecture Highlights**:
- Employee records with privacy controls
- Organizational graph for reporting structure
- Leave request workflows with manager approval
- Performance review cycles with scheduled triggers
- Compensation planning with delegation chains
- GDPR right-to-access and right-to-erasure
- Document management for policies and forms
- Onboarding/offboarding automation

**Data Governance**:
- PII classification for employee data
- Residency enforcement for EU employees (GDPR)
- Retention policies tied to employment status
- Audit logging for compensation changes
- Anonymization for analytics

**v1.6 Enhancements**:
- Storage binding: Document store for HR files
- Cache layer: Hybrid caching for org charts
- Intent routing: Workflow routing by department
- Client cache: Offline policy document viewing

### Example 07: Property Management

**Industry**: Real Estate  
**Complexity**: Medium  
**Key Features**: Spatial data, IoT devices, asset lifecycle, temporal access

**Architecture Highlights**:
- Property listings with geospatial search
- IoT smart locks with temporal access codes
- Maintenance tickets with scheduling
- Lease agreements with expiring assets
- Tenant portal with payment workflows
- Property inspection photos with metadata
- Utility monitoring from IoT sensors
- Common area booking system

**v1.6 Enhancements**:
- Storage binding: Geospatial database for properties
- Cache layer: Local caching for property photos
- Intent routing: Zone-based maintenance routing
- Resource requirements: Seasonal scaling for booking system
- Client cache: Offline maintenance checklist

### Example 08: Learning Management System

**Industry**: Education  
**Complexity**: Medium  
**Key Features**: Rich content, asset management, collaboration, offline support

**Architecture Highlights**:
- Course content with structured documents
- Video assets with streaming and transcoding
- Student submissions with grading workflows
- Discussion forums with real-time updates
- Collaborative document editing for groups
- Progress tracking with materialized views
- Offline content delivery for mobile
- Accessibility support (WCAG compliance)

**v1.6 Enhancements**:
- Storage binding: Video CDN for course content
- Cache layer: Hybrid caching for course materials
- Intent routing: Load-balanced submission processing
- Resource requirements: Auto-scale video transcoders
- Client cache: Offline course viewing with watermarking

### Example 09: Social Networking Platform

**Industry**: Consumer  
**Complexity**: High  
**Key Features**: Social graph, activity streams, ML recommendations, content moderation

**Architecture Highlights**:
- User profiles with privacy controls
- Friend/follower graph with traversal queries
- Activity feed with streaming updates
- Content recommendations using ML
- Image/video posts with transformations
- Real-time messaging with presence
- Content moderation workflows
- GDPR data export and deletion

**Data Governance**:
- User consent for data collection
- Retention policies for deleted content
- Right-to-access export functionality
- Anonymization for ML training data

**v1.6 Enhancements**:
- Storage binding: Graph database for social connections
- Cache layer: Distributed caching for feeds
- Intent routing: Fan-out routing for post creation
- Resource requirements: Auto-scale feed generators
- Client cache: Offline post composition with queue

### Example 10: Smart Factory / Industrial IoT

**Industry**: Manufacturing  
**Complexity**: High  
**Key Features**: IoT sensors, streaming analytics, ML predictive maintenance, safety compliance

**Architecture Highlights**:
- Production line sensors with high-frequency data
- Temporal windows for streaming aggregation
- Predictive maintenance using ML inference
- Quality control with computer vision
- Safety incident workflows with escalation
- Equipment status dashboards
- Energy monitoring and optimization
- Compliance reporting for regulations

**v1.6 Enhancements**:
- Storage binding: Time-series database for sensor data
- Cache layer: Local caching for edge gateways
- Intent routing: Priority routing for safety alerts
- Resource requirements: Scale analytics workers with load
- Client cache: Offline safety checklist execution

### Example 11: Legal Case Management

**Industry**: Legal  
**Complexity**: High  
**Key Features**: Durable workflows, legal holds, delegation, full-text search

**Architecture Highlights**:
- Case files with role-based access
- Document search with full-text and metadata
- Workflow automation for case stages
- Client delegation with time-bounded access
- Legal hold implementation for litigation
- Billing and time tracking
- Court date scheduling with notifications
- Confidential communication channels

**Data Governance**:
- Attorney-client privilege enforcement
- Legal hold preventing deletion
- Audit trail for chain of custody
- Retention policies by case status
- Encryption for sensitive documents

**v1.6 Enhancements**:
- Storage binding: Secure document store with encryption
- Cache layer: Distributed caching for search results
- Intent routing: Case-type-based workflow routing
- Client cache: Offline case file viewing

### Example 12: Voice-First Smart Home

**Industry**: Consumer IoT  
**Complexity**: Medium  
**Key Features**: Voice interface, IoT device control, device groups, temporal access

**Architecture Highlights**:
- Device registry with capability metadata
- Voice command routing to intents
- Device groups for room/scene control
- Temporal access for guests
- Automation rules with scheduled triggers
- Energy usage tracking
- Security monitoring with alerts
- Integration with external services

**v1.6 Enhancements**:
- Storage binding: KV for device state
- Cache layer: Local caching for quick responses
- Intent routing: Voice command classification
- Resource requirements: Scale voice processing
- Client cache: Offline device control fallback

### Example 13: Subscription Commerce

**Industry**: SaaS  
**Complexity**: Medium  
**Key Features**: External API integration, webhooks, scheduled billing, usage metering

**Architecture Highlights**:
- Subscription plans with pricing tiers
- Scheduled billing with recurring triggers
- Usage metering for consumption-based pricing
- Payment gateway integration with retry
- Dunning workflows for failed payments
- Webhook delivery for integrations
- Customer portal with self-service
- Revenue recognition reporting

**v1.6 Enhancements**:
- Storage binding: Time-series for usage metrics
- Cache layer: Distributed caching for pricing
- Intent routing: Retry routing for failed payments
- Resource requirements: Scale billing processors monthly
- Client cache: Offline usage dashboard viewing

### Example 14: Collaborative Document Editor

**Industry**: Productivity  
**Complexity**: High  
**Key Features**: Real-time collaboration, rich content, operational transform, offline support

**Architecture Highlights**:
- Structured document content
- Real-time collaboration with presence
- Operational transform for conflict resolution
- Version history with snapshots
- Comments and suggestions
- Permission-based editing
- Offline editing with sync
- Export to multiple formats

**v1.6 Enhancements**:
- Storage binding: Document database with versioning
- Cache layer: Hybrid caching for active documents
- Intent routing: Load-balanced edit operations
- Resource requirements: Scale collaboration servers
- Client cache: Offline editing with OT-based sync

### Example 15: Government Permitting System

**Industry**: Government  
**Complexity**: High  
**Key Features**: Durable workflows, spatial queries, accessibility, public transparency

**Architecture Highlights**:
- Permit applications with multi-stage workflows
- Property spatial queries for zoning
- Human review tasks with SLA tracking
- Public comment periods with notifications
- Document submission with validation
- Fee calculation and payment
- Accessibility compliance (WCAG, Section 508)
- Public transparency dashboard

**Data Governance**:
- Public records retention policies
- Audit trail for all decisions
- FOIA compliance for document requests
- Anonymization for statistical reporting

**v1.6 Enhancements**:
- Storage binding: Geospatial database for parcels
- Cache layer: Distributed caching for public data
- Intent routing: Priority routing by permit type
- Resource requirements: Scale for public comment periods
- Client cache: Offline application form completion

---

**Note**: These examples are provided as architectural references. For complete working implementations, see [Part II: Complete Banking Example](#part-ii-complete-banking-example-all-features) which demonstrates end-to-end patterns suitable for production deployment.

---

**Version**: 1.6  
**Last Updated**: 2026-02-04  
**Status**: Production Ready
