# SMS v1.5 Reference Examples

## For LLM Learning

This document consolidates modeling guidance, examples, and patterns for learning and applying the System Mechanics Specification v1.5. Use this document to:

- **Understand invariant-oriented design** - Learn the philosophy behind SMS modeling
- **Model new applications** - Follow the playbook for entity, relationship, and view design
- **Find patterns** - Reference common patterns for typical requirements
- **Learn by example** - Study complete worked examples from simple to complex
- **Validate designs** - Use checklists and decision trees to verify your models

**Document Status**: Normative Guidance and Examples  
**Version**: 1.5  
**Last Updated**: 2026-01-18

**Related Documents:**
- **SMS-v1.5-Specification.md**: Normative specification with grammar definitions
- **SMS-v1.5-EBNF-Grammar.md**: Formal grammar for parser/tooling development
- **SMS-v1.5-Implementation-Guide.md**: Detailed implementation patterns
- **SMS-v1.5-Quick-Reference.md**: Rapid-lookup cheat sheets and decision trees

---

## Table of Contents

- [Part I: Modeling Guidance](#part-i-modeling-guidance)
- [Part II: Complete Banking Example (All 28 Addendums)](#part-ii-complete-banking-example-all-features)
- [Part III: Grammar Quick Examples](#part-iii-grammar-quick-examples)
- [Part IV: Comprehensive Production Examples](#part-iv-comprehensive-production-examples)
- [Part V: Common Patterns Library](#part-v-common-patterns-library)

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
model:
  name: Account
  mutability:
    scope: entity
    exclusive: true
```

**Bad**:
```yaml
# Anti-pattern: Entity depends on another entity's state for writes
model:
  name: Transfer
  precondition: source_account.balance >= amount  # Creates dependency
```

### Rule E2: Entities Contain Identity and Constraints, Not Derived State

Entities should store their identity and local constraints, not values that depend on other entities.

**Good**:
```yaml
dataType:
  name: Account
  fields:
    id: uuid
    owner_id: uuid
    created_at: timestamp
    account_type: enum[checking, savings]
```

**Bad**:
```yaml
# Anti-pattern: Balance is derived from transactions
dataType:
  name: Account
  fields:
    id: uuid
    balance: decimal  # Requires coordination to update
```

### Rule E3: Append-Only for Immutable Facts

Facts that never change should be append-only.

```yaml
model:
  name: Transaction
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
model:
  name: <EntityName>
  
  mutability:
    scope: entity | append-only
    exclusive: true
  
  authority:
    scope: entity
    resolution: <expression>
    migration:
      allowed: true | false

dataType:
  name: <EntityName>
  version: v1
  fields:
    id: uuid
    # Identity fields only
    # NO derived or aggregate fields

constraint:
  name: <EntityName>_valid
  appliesTo: <EntityName>
  rules:
    v1: <single-entity constraints>
```

---

## Relationship Design Rules

### Rule R1: Relationships Express Semantic Links

Relationships describe how entities relate, not how they coordinate.

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
relationship:
  name: <verb>
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
view:
  name: AccountBalance
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
view:
  authority_agnostic: true
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
view:
  name: <ViewName>
  
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
authority:
  migration:
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
view:
  authority_agnostic: true  # Tolerates events from multiple regions
```

---

## Common Anti-Patterns

### Anti-Pattern 1: Balance as Entity Field

**Problem**: Storing balance in Account requires coordinated updates.

```yaml
# BAD
dataType:
  name: Account
  fields:
    balance: decimal  # Who updates this?
```

**Solution**: Derive balance in view.

```yaml
# GOOD
view:
  name: AccountBalance
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
dataType:
  name: Account
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
dataType:
  name: Customer
  fields:
    id: uuid
    name: string
    profile_photo: asset
    
asset:
  name: profile_photo
  types: [image/jpeg, image/png]
  max_size: 5MB
  transformations:
    thumbnail: resize(width: 200, height: 200)
  retention:
    duration: account_lifetime
    delete_on_entity_deletion: true
```

**Cross-reference**: Addendum AK (Asset Semantics)

---

### Search (Addendum AL)

When users need to find entities by complex criteria, model search explicitly:

**Decision Point**: Will users search for this entity?
- **YES** → Define search configuration with fields, weights, and ranking

```yaml
# Example: Search customers by name, email, phone
search:
  name: CustomerSearch
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

**Cross-reference**: Addendum AL (Search Semantics)

---

### Spatial Data (Addendum AR)

When modeling location-based entities or constraints:

**Decision Point**: Does location matter for this entity?
- **YES** → Use `geo_point` or `geo_shape` types

```yaml
# Example: Store with service area
dataType:
  name: Store
  fields:
    id: uuid
    name: string
    location: geo_point
    service_area: geo_shape
    
constraint:
  name: ServiceAreaValid
  appliesTo: Store
  rules:
    v1: within(location, service_area)
```

**Cross-reference**: Addendum AR (Spatial Type Semantics)

---

### Scheduled Actions (Addendum AM)

When entities require time-based automation:

**Decision Point**: Does this entity need recurring or scheduled actions?
- **YES** → Define scheduled triggers

```yaml
# Example: Monthly account statement generation
scheduledTrigger:
  name: GenerateMonthlyStatement
  schedule: "0 0 1 * *"  # First day of month at midnight
  intent: GenerateStatement
  scope: per_entity
  entity_filter: account.status == active
```

**Cross-reference**: Addendum AM (Scheduled Trigger Semantics)

---

### External Integration (Addendum AW)

When entities depend on external services:

**Decision Point**: Does this entity interact with third-party systems?
- **YES** → Model external integration with resilience patterns

```yaml
# Example: Credit score from external bureau
externalIntegration:
  name: CreditBureauAPI
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

**Cross-reference**: Addendum AW (External Integration Semantics)

---

### Data Governance (Addendum AX)

For every entity containing sensitive data, define governance:

**Decision Point**: Does this entity contain PII or regulated data?
- **YES** → Define classification, retention, and erasure policy

```yaml
# Example: Customer with PII
dataGovernance:
  model: Customer
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

**Cross-reference**: Addendum AX (Data Governance Semantics)

---

### Structured Content (Addendum AP)

When entities need rich text or formatted content:

**Decision Point**: Does this entity need more than plain text?
- **YES** → Use structured content type

```yaml
# Example: Product with formatted description
dataType:
  name: Product
  fields:
    id: uuid
    name: string
    description: structured_content
    
structuredContent:
  format: markdown
  sanitization: strict
  max_length: 50000
```

**Cross-reference**: Addendum AP (Structured Content Type)

---

### Durable Workflows (Addendum AT)

For long-running processes spanning multiple entities:

**Decision Point**: Does this process take days/weeks with human steps?
- **YES** → Model as durable workflow, not SMS flow

```yaml
# Example: Loan approval process
durableWorkflow:
  name: LoanApproval
  steps:
    - action: creditCheck
      timeout: 5m
    - action: manualReview
      timeout: 3_days
      assigned_to: underwriting_team
    - action: finalDecision
      compensation: rejectLoan
```

**Cross-reference**: Addendum AT (Durable Workflow Semantics)

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

## Summary

Invariant-oriented modeling in SMS follows these principles:

1. **Entities are independent** - writable without coordination
2. **Relationships are semantic** - express links, not transactions
3. **Views absorb complexity** - aggregate, validate, derive
4. **Invariants are detected** - not blocked at write time
5. **Authority is entity-scoped** - failure isolation by design

This approach enables SMS to handle complex business requirements while maintaining the system's core guarantees around availability, partition tolerance, and evolution.

---

# Part II: Complete Banking Example (All Features)

## Overview

This section provides a **complete, production-ready banking system** that demonstrates **ALL 28 v1.5 addendums** in contextually-appropriate banking scenarios. The example covers the full SMS v1.5 specification:

| Layer | Features Demonstrated |
|-------|----------------------|
| **Data Model** | Entities, constraints, assets, spatial types, structured content, inference fields |
| **Relationships** | Graph queries, delegation, causality, cardinality |
| **Views** | Materialization, search indexes, temporal windows, invariant enforcement |
| **Experiences** | Accessibility, conversational UI, collaborative sessions |
| **Workflows** | Durable workflows, offline support, compensation, checkpoints |
| **Integrations** | External APIs, notifications, scheduled triggers |
| **Edge Devices** | ATM terminals, POS devices, self-service kiosks |
| **Governance** | Data classification, retention, consent, right to erasure |

### Feature Coverage Matrix

All 28 v1.5 addendums are demonstrated in this example:

| # | Addendum | Feature | Banking Context |
|---|----------|---------|-----------------|
| X | Form-Intent Binding | Transfer form binding | [Form Binding section](#form-binding-transfer-form-addendum-x) |
| Y | View Materialization | Temporal windows, streaming | [Views section](#views-v13) |
| Z | Intent Delivery | Exactly-once guarantees | Intent definitions throughout |
| AA | Session Contract | Offline support, multi-device | [Offline Support section](#offline-support-mobile-banking-addendum-aa) |
| AB | Indicator Source | Dynamic badges | [UI Views section](#ui-layer-v14) |
| AC | Composition Fetch | Related view loading | [Compositions section](#ui-compositions-v14) |
| AD | Mask Expression | SSN masking | [Admin Views section](#ui-layer-v14) |
| AE | Navigation Semantics | Hierarchical navigation | [Navigation section](#navigation-v14) |
| AF | View Availability | Regional fallback | [Views section](#views-v13) |
| AG | Presentation Hints | Display hints | Data model fields |
| AH | Surface/Container | View grouping | [Compositions section](#ui-compositions-v14) |
| AI | Accessibility | Screen reader, keyboard nav | [Accessibility section](#accessibility-preferences-addendum-ai) |
| AJ | End-to-End Flow | Complete banking flows | Throughout example |
| AK | Asset Semantics | Check images, signatures | [Assets section](#assets-check-images-addendum-ak) |
| AL | Search Semantics | Transaction search | [Search section](#search-transaction-search-addendum-al) |
| AM | Scheduled Triggers | Statements, payments | [Scheduled Triggers section](#scheduled-triggers-monthly-statements-addendum-am) |
| AN | Notification Channel | Multi-channel alerts | [Notifications section](#notifications-transaction-alerts-addendum-an) |
| AO | Collaborative Session | Joint account planning | [Collaborative section](#collaborative-session-joint-account-financial-planning-addendum-ao) |
| AP | Structured Content | Loan documentation | [Structured Content section](#structured-content-loan-documentation-addendum-ap) |
| AQ | Inference/Derived | Fraud scoring, risk | [Inference section](#inferencederived-fields-risk-and-fraud-scoring-addendum-aq) |
| AR | Spatial Types | Branch/ATM locator | [Spatial Types section](#spatial-types-branch-and-atm-locator-addendum-ar) |
| AS | Graph Query | Customer relationships | [Graph Query section](#graph-query-customer-relationship-network-addendum-as) |
| AT | Durable Workflow | Loan approval | [Workflows section](#durable-workflow-loan-approval-addendum-at) |
| AU | Delegation/Consent | Authorized users | [Delegation section](#delegation-authorized-user-addendum-au) |
| AV | Conversational | Banking assistant | [Conversational section](#conversational-ui-banking-assistant-addendum-av) |
| AW | External Integration | Credit bureau API | [External Integration section](#external-integration-credit-bureau-addendum-aw) |
| AX | Data Governance | PII, retention | [Data Governance section](#data-governance-pii-classification-addendum-ax) |
| AY | Edge Device | ATM, POS terminals | [Edge Device section](#edge-device-atm-and-pos-terminals-addendum-ay) |

---

## Problem Statement

Design a system that supports:
1. Creating accounts
2. Processing transactions (debits and credits)
3. Maintaining accurate balances
4. Ensuring balances never go negative
5. Operating across multiple regions
6. **Customer-facing UI** for account holders
7. **Administrator UI** for bank staff

### The Critical Invariant

> **Account balance must never be negative.**

This invariant spans multiple entities (Account + all Transactions) and is the central challenge.

---

## Data Layer (v1.3)

### Entity: Account

```yaml
dataType:
  name: Account
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

model:
  name: Account
  mutability:
    scope: entity
    exclusive: true
  authority:
    scope: entity
    resolution: entity.jurisdiction_region
    migration:
      allowed: false  # Financial data stays in region
```

### Entity: Customer

```yaml
dataType:
  name: Customer
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
```

### Entity: Transaction

```yaml
dataType:
  name: Transaction
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

model:
  name: Transaction
  mutability:
    scope: append-only  # Transactions never change
```

### Entity: Alert

```yaml
dataType:
  name: Alert
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
```

### Entity: AuditLog

```yaml
dataType:
  name: AuditLog
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

---

## Relationships (v1.3)

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

---

## Views (v1.3)

### AccountBalance View

```yaml
view:
  name: AccountBalance
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
```

### CustomerSummary View

```yaml
view:
  name: CustomerSummary
  inputs: [Customer, Account, AccountBalance]
  aggregation:
    account_count: count(accounts)
    total_balance: sum(balances)
    has_alerts: exists(alerts WHERE status == "open")
```

### SystemMetrics View

```yaml
view:
  name: SystemMetrics
  inputs: [Customer, Account, Transaction, Alert]
  aggregation:
    total_customers: count(customers)
    active_accounts: count(accounts WHERE status == "active")
    pending_transactions: count(transactions WHERE status == "pending")
    open_alerts: count(alerts WHERE status == "open")
    critical_alerts: count(alerts WHERE severity == "critical" AND status == "open")
```

---

## SMS Flows (v1.3)

SMS flows orchestrate work units within and across authority boundaries, using atomic groups for transactional semantics and signals for event-driven coordination.

### Transfer Flow

```yaml
smsFlow:
  name: TransferFlow
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
            boundary: account_authority
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
```

### Bill Payment Flow

```yaml
smsFlow:
  name: BillPaymentFlow
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
            boundary: account_authority
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
```

### Loan Origination Flow (Durable with Checkpoints)

```yaml
smsFlow:
  name: LoanOriginationFlow
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
signal:
  name: TransferCompletedSignal
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

signal:
  name: TransferFlaggedSignal
  version: v1
  payload:
    transfer_id: uuid
    fraud_score: decimal
    reasons: array[string]
  subscribers:
    - FraudReviewQueue
    - SecurityAlertService
    
signal:
  name: InsufficientFundsSignal
  version: v1
  payload:
    account_id: uuid
    required: decimal
    available: decimal
  subscribers:
    - NotificationService
    - OverdraftProtectionService

signal:
  name: LoanFundedSignal
  version: v1
  payload:
    application_id: uuid
    amount: decimal
    funded_at: timestamp
  subscribers:
    - NotificationService
    - AccountingService
    - ReportingService

signal:
  name: LoanDeniedSignal
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
workUnit:
  name: DebitAccount
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

workUnit:
  name: CreditAccount
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

workUnit:
  name: CheckFraud
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

---

## Policy Layer

### Subjects and Roles

```yaml
policy:
  subjects:
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

---

## UI Layer (v1.4)

### Presentation Views

#### Customer Views

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
            
    - name: CustomerTransferFormView
      version: v1
      consumes:
        dataState: TransferDraft
        version: v1
      guarantees:
        - capture_transfer_details
        - show_validation
      interactionModel:
        actions:
          - if: form.valid
            action: submit_transfer
            label: "Transfer Funds"
            intent: CreateTransfer
          - if: true
            action: cancel
            label: "Cancel"
            
    - name: CustomerProfileView
      version: v1
      consumes:
        dataState: CustomerRecord
        version: v1
      guarantees:
        - show_profile_info
        - show_security_settings
      interactionModel:
        actions:
          - if: true
            action: edit_profile
            label: "Edit Profile"
          - if: true
            action: change_password
            label: "Change Password"
      fieldPermissions:
        - field: email
          visible: true
        - field: phone
          visible: true
        - field: ssn
          visible: false  # Customers can't see their full SSN
        - field: kyc_status
          visible: true
```

#### Admin Views

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
          
    - name: AdminAccountDetailsView
      version: v1
      consumes:
        dataState: AccountRecord
        version: v1
      guarantees:
        - show_full_account_info
        - show_owner_link
        - show_activity_summary
      interactionModel:
        badges:
          - if: status == "frozen"
            label: "Frozen"
            severity: error
          - if: status == "pending_approval"
            label: "Pending Approval"
            severity: warning
        actions:
          - if: status == "active"
            action: freeze_account
            label: "Freeze Account"
            intent: FreezeAccount
          - if: status == "frozen"
            action: unfreeze_account
            label: "Unfreeze Account"
            intent: UnfreezeAccount
            
    - name: AdminAlertDetailsView
      version: v1
      consumes:
        dataState: AlertRecord
        version: v1
      guarantees:
        - show_alert_details
        - show_entity_context
        - show_resolution_history
      interactionModel:
        badges:
          - if: severity == "critical"
            label: "Critical"
            severity: critical
          - if: severity == "high"
            label: "High Priority"
            severity: warning
        actions:
          - if: status == "open"
            action: acknowledge_alert
            label: "Acknowledge"
            intent: AcknowledgeAlert
          - if: status in ["open", "acknowledged"]
            action: resolve_alert
            label: "Resolve"
            intent: ResolveAlert
          - if: status in ["open", "acknowledged"]
            action: escalate_alert
            label: "Escalate"
            intent: EscalateAlert
```

---

## UI Compositions (v1.4)

### Customer Compositions

```yaml
compositions:
  - name: CustomerDashboard
    version: v1
    description: "Customer home with account overview"
    primary: CustomerDashboardView
    navigation:
      label: "Dashboard"
      purpose: home
      policy_set: CustomerAccess
    related:
      - view: CustomerAccountListView
        relationship: detail
        cardinality: many
        navigation:
          label: "My Accounts"
          scope: within_composition
          
  - name: CustomerAccountWorkspace
    version: v1
    description: "Single account with balance and transactions"
    primary: CustomerAccountDetailsView
    navigation:
      label: "{{account.account_type}} ****{{account.account_number_last4}}"
      purpose: accounts
      parent: CustomerDashboard
      param: account_id
      policy_set: CustomerAccess
      indicator:
        when: account.status == "frozen"
        severity: error
    related:
      - view: CustomerBalanceView
        relationship: derived
        required: true
        navigation:
          label: "Balance"
          scope: within_composition
          
      - view: CustomerTransactionListView
        relationship: detail
        cardinality: many
        basis: [transaction_debits_account, transaction_credits_account]
        navigation:
          label: "Transactions"
          scope: within_composition
          
      - view: CustomerTransferFormView
        relationship: action
        trigger: transfer_funds
        policy_set: CustomerTransferAccess
        navigation:
          label: "Transfer"
          scope: on_action
          
  - name: CustomerProfileWorkspace
    version: v1
    description: "Customer profile management"
    primary: CustomerProfileView
    navigation:
      label: "Profile"
      purpose: settings
      policy_set: CustomerAccess
```

### Admin Compositions

```yaml
  - name: AdminDashboard
    version: v1
    description: "Admin home with system metrics"
    primary: AdminDashboardView
    navigation:
      label: "Dashboard"
      purpose: home
      policy_set: AdminBasicAccess
      indicator:
        when: critical_alerts > 0
        severity: critical
    related:
      - view: AdminAlertListView
        relationship: supplementary
        navigation:
          label: "Recent Alerts"
          scope: within_composition
        data_scope:
          status: "open"
          limit: 5
          
  - name: AdminCustomerWorkspace
    version: v1
    description: "Customer details with accounts and history"
    primary: AdminCustomerDetailsView
    navigation:
      label: "{{customer.full_name}}"
      purpose: customers
      parent: AdminCustomerListWorkspace
      param: customer_id
      policy_set: AdminBasicAccess
      indicator:
        when: customer.kyc_status == "pending"
        severity: warning
    related:
      - view: AdminAccountListView
        relationship: detail
        cardinality: many
        basis: customer_owns_account
        navigation:
          label: "Accounts"
          scope: within_composition
        data_scope:
          owner_id: "{{customer.id}}"
          
      - view: AdminAuditLogListView
        relationship: historical
        cardinality: many
        navigation:
          label: "Activity Log"
          scope: within_composition
        data_scope:
          entity_id: "{{customer.id}}"
          
  - name: AdminAccountWorkspace
    version: v1
    description: "Account details with owner and transactions"
    primary: AdminAccountDetailsView
    navigation:
      label: "{{account.account_number}}"
      purpose: accounts
      parent: AdminAccountListWorkspace
      param: account_id
      policy_set: AdminAccountManagement
      indicator:
        when: account.status == "frozen"
        severity: error
    related:
      - view: CustomerBalanceView
        relationship: derived
        required: true
        navigation:
          label: "Balance"
          scope: within_composition
          
      - view: AdminTransactionListView
        relationship: detail
        cardinality: many
        navigation:
          label: "Transactions"
          scope: within_composition
          
      - view: AdminCustomerDetailsView
        relationship: contextual
        basis: customer_owns_account
        navigation:
          label: "Owner"
          scope: within_composition
          
  - name: AdminAlertWorkspace
    version: v1
    description: "Alert details with context"
    primary: AdminAlertDetailsView
    navigation:
      label: "Alert #{{alert.id_short}}"
      purpose: alerts
      parent: AdminAlertListWorkspace
      param: alert_id
      policy_set: AdminAlertManagement
    related:
      - view: AdminAccountDetailsView
        relationship: contextual
        basis: alert_references_entity
        navigation:
          label: "Related Account"
          scope: within_composition
```

---

## Experiences (v1.4)

### Customer Experience

```yaml
experiences:
  - name: CustomerBanking
    version: v1
    description: "Customer-facing banking experience"
    entry_point: CustomerDashboard
    
    includes:
      - CustomerDashboard
      - CustomerAccountWorkspace
      - CustomerProfileWorkspace
      - HelpWorkspace
      
    policy_set: CustomerAccess
    
    unauthenticated:
      redirect_to: LoginWorkspace
      
    on_unauthorized: conceal
```

### Administrator Experience

```yaml
  - name: BankingAdministration
    version: v1
    description: "Administrative banking experience"
    entry_point: AdminDashboard
    
    includes:
      - AdminDashboard
      - AdminCustomerListWorkspace
      - AdminCustomerWorkspace
      - AdminAccountListWorkspace
      - AdminAccountWorkspace
      - AdminTransactionListWorkspace
      - AdminAlertListWorkspace
      - AdminAlertWorkspace
      - AdminKYCQueueWorkspace
      - AdminAuditLogWorkspace
      - AdminReportsWorkspace
      - HelpWorkspace
      
    policy_set: AdminBasicAccess
    
    unauthenticated:
      redirect_to: AdminLoginWorkspace
      
    on_unauthorized: indicate
```

---

## Navigation (v1.4)

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
      
  scopes:
    - name: within_composition
      description: "Visible when parent composition is active"
    - name: always_visible
      description: "Visible in primary navigation"
    - name: on_action
      description: "Visible when triggered by action"
```

---

## Complete Navigation Hierarchy

### Customer Navigation

```
CustomerBanking Experience
│
├── Dashboard (home) [root]
│   └── My Accounts [within_composition]
│
├── Account Details (accounts)
│   ├── Balance [within_composition]
│   ├── Transactions [within_composition]
│   └── Transfer [on_action]
│
├── Profile (settings)
│
└── Help (help)
```

### Admin Navigation

```
BankingAdministration Experience
│
├── Dashboard (home) [root]
│   └── Recent Alerts [within_composition]
│
├── Customers (customers)
│   └── Customer Details
│       ├── Accounts [within_composition]
│       └── Activity Log [within_composition]
│
├── Accounts (accounts)
│   └── Account Details
│       ├── Balance [within_composition]
│       ├── Transactions [within_composition]
│       └── Owner [within_composition]
│
├── Transactions (transactions)
│
├── Alerts (alerts) [indicator: open_alerts > 0]
│   └── Alert Details
│       └── Related Account [within_composition]
│
├── KYC Queue (compliance) [indicator: pending_kyc > 0]
│
├── Audit Logs (audit)
│
├── Reports (reports)
│
└── Help (help)
```

---

## v1.5 Extensions

### Assets: Check Images (Addendum AK)

Banks need to store check images for regulatory compliance:

```yaml
dataType:
  name: Transaction
  version: v2
  fields:
    id: uuid
    # ... existing fields ...
    check_image: asset  # NEW in v1.5
    
asset:
  name: check_image
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

**Cross-reference**: Addendum AK (Asset Semantics)

---

### Notifications: Transaction Alerts (Addendum AN)

Send multi-channel notifications for significant transactions:

```yaml
notification:
  name: LargeTransactionAlert
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
  tracking:
    log_delivery: true
    log_read: true
```

**Cross-reference**: Addendum AN (Notification Channel Semantics)

---

### Scheduled Triggers: Monthly Statements (Addendum AM)

Generate account statements on a schedule:

```yaml
scheduledTrigger:
  name: GenerateMonthlyStatement
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

**Cross-reference**: Addendum AM (Scheduled Trigger Semantics)

---

### External Integration: Credit Bureau (Addendum AW)

Integrate with external credit scoring service:

```yaml
externalIntegration:
  name: CreditBureauAPI
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
  observability:
    track_latency: true
    track_errors: true
    alert_on_failure_rate: 0.1
```

**Cross-reference**: Addendum AW (External Integration Semantics)

---

### Data Governance: PII Classification (Addendum AX)

Classify and manage customer data for GDPR compliance:

```yaml
dataGovernance:
  model: Customer
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
    sources: [kyc_verification, manual_entry]
  audit:
    log_access: true
    log_modification: true
    retention: 10_years
```

**Cross-reference**: Addendum AX (Data Governance Semantics)

---

### Search: Transaction Search (Addendum AL)

Enable customers to search their transaction history:

```yaml
search:
  name: TransactionSearch
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

**Cross-reference**: Addendum AL (Search Semantics)

---

### Form Binding: Transfer Form (Addendum X)

Bind transfer form to input intent:

```yaml
presentationView:
  name: TransferForm
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
      - view_field: reference
        supplies: reference
        source: input
        required: false
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

**Cross-reference**: Addendum X (Form Intent Binding)

---

### Accessibility Preferences (Addendum AI)

Support customer accessibility needs:

```yaml
accessibilityPreferences:
  name: BankingAccessibility
  scope: session
  preferences:
    - name: high_contrast
      type: boolean
      default: false
      applies_to: [all_views]
    - name: font_scale
      type: enum
      values: [small, medium, large, xlarge]
      default: medium
      applies_to: [all_views]
    - name: reduce_motion
      type: boolean
      default: false
      applies_to: [animations, transitions]
    - name: screen_reader_hints
      type: boolean
      default: false
      applies_to: [all_views]
  keyboard_navigation:
    enabled: true
    shortcuts:
      - key: "Alt+H"
        action: navigate_home
      - key: "Alt+T"
        action: navigate_transactions
      - key: "Alt+N"
        action: new_transfer
```

**Cross-reference**: Addendum AI (Accessibility and User Preferences)

---

### Conversational UI: Banking Assistant (Addendum AV)

Voice/chat interface for common banking tasks:

```yaml
conversationalExperience:
  name: BankingAssistant
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
          validation: account_exists(to_account)
        - name: amount
          type: decimal
          prompt: "How much would you like to transfer?"
          validation: amount > 0 AND amount <= balance
      confirmation:
        required: true
        template: "Transfer {{amount}} to {{to_account}}?"
      action: submit_intent
    - intent: TransactionHistory
      utterances:
        - "Show my transactions"
        - "Recent activity"
        - "What did I spend on [category]"
      response:
        type: list
        data_source: TransactionView
        limit: 10
  fallback:
    strategy: human_handoff
    message: "Let me connect you with a representative"
```

**Cross-reference**: Addendum AV (Conversational Experience Semantics)

---

### Durable Workflow: Loan Approval (Addendum AT)

Multi-day loan approval process with human steps:

```yaml
durableWorkflow:
  name: LoanApproval
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
```

**Cross-reference**: Addendum AT (Durable Workflow Semantics)

---

### Delegation: Authorized User (Addendum AU)

Allow account holder to delegate access to another person:

```yaml
delegation:
  name: AuthorizedUser
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

**Cross-reference**: Addendum AU (Delegation and Consent Semantics)

---

### Spatial Types: Branch and ATM Locator (Addendum AR)

Banks need location-based services for branch and ATM discovery:

```yaml
dataType:
  name: Branch
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
    
dataType:
  name: ATM
  version: v1
  fields:
    id: uuid
    branch_id: uuid
    location: geo_point
    capabilities: array[enum[cash_withdrawal, deposits, check_cashing, transfers]]
    accessibility: array[enum[wheelchair, audio, braille]]
    status: enum[operational, out_of_service, low_cash]
    last_serviced: timestamp

searchIntent:
  name: NearbyBranchesIntent
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
    
  facets:
    include: [services]
    
  result:
    includes: [id, name, address, location, distance, hours_of_operation, services]
    max_results: 20

searchIntent:
  name: NearbyATMsIntent
  version: v1
  description: "Find ATMs with specific capabilities near user"
  
  searches: ATMState
  
  query:
    spatial:
      field: location
      type: within_distance
      within_distance:
        from: "request.user_location"
        max_distance: 5km
      sort:
        by_distance_from: "request.user_location"
        
  filter:
    expression: "atm.status == 'operational' AND 'cash_withdrawal' IN atm.capabilities"
    
  result:
    includes: [id, location, distance, capabilities, accessibility, status]
    max_results: 10

constraint:
  name: ATMWithinServiceArea
  appliesTo: ATM
  rules:
    v1: within(atm.location, branch.service_area)
```

**Cross-reference**: Addendum AR (Spatial Type Semantics)

---

### Graph Query: Customer Relationship Network (Addendum AS)

Banks need to understand customer relationships for referrals, household accounts, and fraud detection:

```yaml
relationship:
  name: Household_Member_Of
  from: CustomerState
  to: HouseholdState
  type: membership
  cardinality: many-to-one
  semantics:
    causal: true
    
relationship:
  name: Referred_By
  from: CustomerState
  to: CustomerState
  type: referral
  cardinality: many-to-one
  semantics:
    track_time: true
    
relationship:
  name: Joint_Account_Holder
  from: CustomerState
  to: AccountState
  type: ownership
  cardinality: many-to-many
  semantics:
    ordered: false

graphQuery:
  name: HouseholdAccountsQuery
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

graphQuery:
  name: ReferralChainQuery
  version: v1
  description: "Find referral chain for bonus calculations"
  
  traverses: Referred_By
  
  start:
    from: CustomerState
    where: "customer.id == request.customer_id"
    
  pattern:
    type: path
    path:
      direction: up
      stop_when: "customer.referred_by == null"
      max_hops: 5
      
  result:
    includes: [customer_id, full_name, signup_date]
    depth_field: referral_level

graphQuery:
  name: FraudNetworkDetection
  version: v1
  description: "Detect suspicious transaction patterns between related accounts"
  
  traverses: [Joint_Account_Holder, Beneficiary_Of]
  
  start:
    from: CustomerState
    where: "customer.id == request.flagged_customer_id"
    
  pattern:
    type: neighbors
    neighbors:
      depth: 2
      direction: both
      filter: "account.status == 'frozen' OR alert.severity == 'critical'"
      
  result:
    includes: [customer_id, account_id, relationship_type, alert_count]
```

**Cross-reference**: Addendum AS (Graph Query Semantics)

---

### Collaborative Session: Joint Account Financial Planning (Addendum AO)

Banks support joint account holders reviewing finances together:

```yaml
collaborativeSession:
  name: FinancialPlanningSession
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

collaborativeSession:
  name: LoanAdvisorSession
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
```

**Cross-reference**: Addendum AO (Collaborative Session Semantics)

---

### Structured Content: Loan Documentation (Addendum AP)

Banks need rich document formats for loan contracts and disclosures:

```yaml
dataType:
  name: LoanDocument
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
    
structuredContent:
  name: loan_document_content
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
    
dataType:
  name: MortgageContract
  version: v1
  fields:
    id: uuid
    property_address: string
    property_location: geo_point
    loan_amount: decimal
    interest_rate: decimal
    term_years: integer
    contract_body: structured_content
    attachments: array[asset]
    
structuredContent:
  name: mortgage_contract_content
  schema: mortgage_contract
  
  blocks:
    allowed:
      - paragraph
      - heading
      - list
      - table
      - property_description
      - escrow_terms
      - insurance_requirements
      - signature_block
      
  custom_blocks:
    - name: property_description
      fields:
        legal_description: string
        parcel_id: string
        boundaries: geo_shape
      presentation:
        include_map: true
        
    - name: escrow_terms
      fields:
        monthly_amount: decimal
        includes: array[enum[taxes, insurance, pmi]]
      presentation:
        display_as: summary_box
```

**Cross-reference**: Addendum AP (Structured Content Type)

---

### Inference/Derived Fields: Risk and Fraud Scoring (Addendum AQ)

Banks use ML models for real-time fraud detection and credit risk assessment:

```yaml
dataType:
  name: Transaction
  version: v2
  fields:
    # ... existing fields ...
    
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

dataType:
  name: Customer
  version: v2
  fields:
    # ... existing fields ...
    
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

materialization:
  name: CustomerInsightsView
  version: v1
  source: [CustomerState, AccountState, TransactionState]
  targetState: CustomerInsightsMaterialized
  
  inference:
    model:
      name: customer_insights_aggregator
      version: v2
    input_fields: [customer_id, account_balances, transaction_history, demographics]
    output_fields:
      - spending_pattern: enum[saver, balanced, spender]
      - recommended_products: array[string]
      - lifetime_value_estimate: decimal
    freshness:
      strategy: lazy
      ttl: 24h
      
  retrieval:
    mode: by_entity
    entity_key: customer_id
```

**Cross-reference**: Addendum AQ (Inference/Derived Fields)

---

### Edge Device: ATM and POS Terminals (Addendum AY)

Banks deploy edge devices for offline-capable ATM and point-of-sale transactions:

```yaml
edgeDevice:
  name: ATMTerminal
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
      - check_scan
      
    hardware:
      display: touchscreen
      input: [touchscreen, keypad]
      output: [display, receipt_printer, cash_dispenser]
      sensors: [card_reader, check_scanner, cash_detector]
      
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
        
  local_state:
    cache:
      customer_data:
        fields: [account_number_hash, daily_limit, available_balance_snapshot]
        encryption: required
        ttl: 4h
      transaction_queue:
        max_size: 100
        persist: true
        
  security:
    encryption:
      pin_entry: hardware_encrypted
      storage: aes_256
    authentication:
      card: chip_preferred
      fallback: magnetic_stripe
    tamper_detection:
      enabled: true
      on_detect: lockdown
      
  intents:
    - name: ATMWithdrawal
      offline_support: limited
      validation: local_then_remote
    - name: ATMDeposit
      offline_support: deferred
    - name: ATMBalanceInquiry
      offline_support: cached

edgeDevice:
  name: POSTerminal
  version: v1
  description: "Point of Sale terminal for merchant payments"
  
  device:
    type: handheld
    runtime: android_embedded
    
    capabilities:
      - card_read
      - nfc_payment
      - pin_entry
      - receipt_print
      - signature_capture
      
  connectivity:
    primary: wifi
    fallback: cellular
    
    offline:
      enabled: true
      max_duration: 2h
      capabilities:
        - payment_authorize_offline
        
      limits:
        max_offline_transaction: 50
        max_total_offline_amount: 1000
        
      sync:
        on_reconnect: immediate
        batch_size: 50
        
  local_state:
    transaction_buffer:
      max_size: 200
      persist: true
      encryption: required
      
  security:
    pci_compliant: true
    encryption:
      card_data: point_to_point
    tokenization:
      enabled: true
      
edgeDevice:
  name: MobileBankingKiosk
  version: v1
  description: "Self-service banking kiosk in retail locations"
  
  device:
    type: kiosk
    runtime: chromeos_kiosk
    
    capabilities:
      - account_opening
      - document_upload
      - video_teller
      - card_issuance
      
  connectivity:
    primary: ethernet
    fallback: cellular_5g
    
  features:
    accessibility:
      screen_reader: true
      height_adjustable: true
      headphone_jack: true
      braille_keypad: available
```

**Cross-reference**: Addendum AY (Edge Device Semantics)

---

### Offline Support: Mobile Banking (Addendum AA)

Mobile banking apps need offline capability for areas with poor connectivity:

```yaml
experience:
  name: CustomerMobileBanking
  version: v1
  description: "Mobile banking app with offline support"
  
  entry_point: MobileDashboard
  
  session:
    required: true
    lifetime:
      mode: sliding
      idle_timeout: 30m
      max_duration: 24h
      
    offline:
      enabled: true
      
      capabilities:
        read: true
        write: queue
        
      cache:
        views: 
          - AccountBalanceView
          - TransactionHistoryView
          - PayeeListView
          - ScheduledPaymentsView
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

**Cross-reference**: Addendum AA (Session Contract - Offline Support)

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

**Cross-reference**: Addendum AI (Accessibility and User Preferences)

---

### Conversational Experience: Banking Assistant (Addendum AV)

Full-featured voice and chat interface for banking tasks:

```yaml
experience:
  name: BankingAssistantExperience
  version: v1
  description: "Conversational banking via voice and chat"
  
  modality:
    type: conversational
    
    channels:
      - type: chat
        platforms: [web, mobile_app, sms]
      - type: voice
        platforms: [phone_ivr, smart_speaker, mobile_app]
        
    voice:
      languages: [en-US, es-US, zh-CN]
      tts:
        voice: neural
        speed: adjustable
      asr:
        model: banking_optimized
        confidence_threshold: 0.7
        
  conversation:
    context:
      ttl: 10m
      carry_forward: [account_context, last_transaction, intent_history]
      
    turn_taking:
      model: mixed
      timeout: 8s
      barge_in: allowed
      
    error_handling:
      no_match: 
        action: clarify
        max_attempts: 3
      no_input:
        action: reprompt
        max_attempts: 2
      low_confidence:
        action: confirm
        threshold: 0.7
        
    disambiguation:
      strategy: ranked_options
      max_options: 3
      
  entry_point: BankingGreetingDialog

presentationComposition:
  name: BankingGreetingDialog
  version: v1
  
  dialog:
    type: question
    
    prompts:
      initial: WelcomePrompt
      reprompt: HowCanIHelpPrompt
      help: BankingHelpPrompt
      
    intents:
      - CheckBalanceIntent
      - TransferMoneyIntent
      - PayBillIntent
      - TransactionHistoryIntent
      - FindBranchIntent
      - SpeakToAgentIntent
      - ReportFraudIntent
      
prompt:
  name: WelcomePrompt
  version: v1
  
  variants:
    - condition: "session.returning_user"
      text: "Welcome back, {{customer.first_name}}. How can I help you today?"
      ssml: "<speak>Welcome back, <say-as interpret-as='name'>{{customer.first_name}}</say-as>. How can I help you today?</speak>"
    - condition: "true"
      text: "Hello! I'm your banking assistant. You can check balances, make transfers, pay bills, or find branches. What would you like to do?"

inputIntent:
  name: CheckBalanceConversational
  version: v1
  
  utterances:
    - "What's my balance"
    - "How much money do I have"
    - "Check my {account_type} account"
    - "What's in my {account_type}"
    - "Account balance"
    
  slots:
    - name: account_type
      type: AccountTypeSlot
      required: false
      prompt: "Which account? Checking or savings?"
      
  fulfillment:
    action: query_view
    view: AccountBalanceView
    response:
      template: "Your {{account_type}} account has a balance of {{balance | currency}}. Your available balance is {{available | currency}}."
      ssml: "<speak>Your {{account_type}} account has a balance of <say-as interpret-as='currency'>{{balance}}</say-as>.</speak>"

inputIntent:
  name: TransferMoneyConversational
  version: v1
  
  utterances:
    - "Transfer money"
    - "Send {amount} to {recipient}"
    - "Move {amount} from {source_account} to {destination}"
    - "Pay {recipient} {amount}"
    
  slots:
    - name: amount
      type: CurrencySlot
      required: true
      prompt: "How much would you like to transfer?"
      validation:
        constraint: amount > 0 AND amount <= available_balance
        error: "The amount must be positive and not exceed your available balance of {{available_balance | currency}}."
        
    - name: source_account
      type: AccountSlot
      required: true
      prompt: "From which account?"
      
    - name: destination
      type: RecipientSlot
      required: true
      prompt: "Who would you like to send money to?"
      
  confirmation:
    required: true
    prompt: "Transfer {{amount | currency}} from {{source_account}} to {{destination}}. Is that correct?"
    
  fulfillment:
    action: submit_intent
    intent: InitiateTransferIntent
    response:
      success: "Done! I've transferred {{amount | currency}} to {{destination}}. Your new balance is {{new_balance | currency}}."
      pending: "Your transfer is being processed. You'll receive a confirmation shortly."
      
  multi_turn:
    enabled: true
    context_retention: full

inputIntent:
  name: ReportFraudConversational
  version: v1
  
  utterances:
    - "Report fraud"
    - "I see a transaction I didn't make"
    - "Someone stole my card"
    - "Unauthorized transaction"
    - "Freeze my account"
    
  priority: high
  
  escalation:
    immediate: true
    to: fraud_specialist
    
  fulfillment:
    action: escalate
    response:
      text: "I'm connecting you with our fraud team right away. Please stay on the line."
      actions:
        - freeze_card
        - create_fraud_alert
        - connect_agent

conversationalFallback:
  name: BankingFallback
  version: v1
  
  strategies:
    - name: clarify
      max_attempts: 2
      prompts:
        - "I'm not sure I understood. Could you rephrase that?"
        - "I can help with balances, transfers, bill pay, or finding branches. Which would you like?"
        
    - name: suggest
      condition: "confidence > 0.4"
      prompt: "Did you mean {{top_suggestion}}?"
      
    - name: human_handoff
      condition: "attempts >= 3 OR user_requests_agent"
      prompt: "Let me connect you with a customer service representative who can help."
      action: transfer_to_agent
      queue: customer_service
      estimated_wait: true
```

**Cross-reference**: Addendum AV (Conversational Experience Semantics)

---

### Enhanced Search: Full-Text Transaction Search (Addendum AL)

Comprehensive search with facets for transaction discovery:

```yaml
search:
  name: TransactionFullTextSearch
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
        factor: 0.1
        
  suggestions:
    enabled: true
    field: counterparty
    max_suggestions: 5
    
  pagination:
    default_size: 25
    max_size: 100
    cursor_based: true
    
  export:
    enabled: true
    formats: [csv, pdf]
    max_records: 10000

search:
  name: StatementFullTextSearch
  version: v1
  target: StatementView
  
  fields:
    - name: content
      weight: 1.0
      analyzers: [standard]
      highlight: true
      snippet_size: 200
      
    - name: statement_date
      range_facet:
        enabled: true
        type: date_range
        
  ranking:
    function: bm25
```

**Cross-reference**: Addendum AL (Search Semantics)

---

### Enhanced Assets: Document Lifecycle (Addendum AK)

Complete asset management for banking documents:

```yaml
asset:
  name: check_image
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

asset:
  name: identity_document
  version: v1
  types: [image/jpeg, image/png, application/pdf]
  max_size: 25MB
  
  transformations:
    thumbnail:
      operation: resize
      params: { width: 200, height: 150 }
    redacted:
      operation: custom
      handler: pii_redaction
      params: { redact: [ssn, date_of_birth] }
      
  lifecycle:
    type: retention_based
    retention: 7_years
    delete_on_entity_deletion: false
    
  verification:
    enabled: true
    handler: document_verification_service
    checks: [authenticity, expiration, face_match]
    
  governance:
    classification: pii
    encryption: required
    access_logging: required

asset:
  name: signature_card
  version: v1
  types: [image/png, image/jpeg]
  max_size: 5MB
  
  transformations:
    normalized:
      operation: custom
      handler: signature_normalizer
      params: { background: white, format: png }
      
  lifecycle:
    type: account_lifetime
    delete_on_account_closure: false
    archive_on_closure: true
    
  verification:
    enabled: true
    handler: signature_verification_service
    
  access:
    policy: ViewSignatureCard
    clearance_required: 3
```

**Cross-reference**: Addendum AK (Asset Semantics)

---

### Enhanced Scheduled Triggers: Banking Automation (Addendum AM)

Comprehensive scheduling for banking operations:

```yaml
scheduledTrigger:
  name: GenerateMonthlyStatement
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

scheduledTrigger:
  name: ProcessRecurringPayments
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
    
  monitoring:
    sla:
      target_completion: 2h
      alert_at: 90%

scheduledTrigger:
  name: DormantAccountCheck
  version: v1
  schedule: "0 2 * * 0"  # Weekly Sunday 2 AM
  timezone: UTC
  
  intent: CheckAccountDormancy
  
  scope: per_entity
  entity_filter: "account.last_activity < now() - 365d AND account.status == 'active'"
  
  notification:
    on_match:
      channel: email
      template: dormant_account_notice
      to: account.owner

scheduledTrigger:
  name: FraudModelRefresh
  version: v1
  schedule: "0 3 * * *"  # Daily 3 AM
  timezone: UTC
  
  intent: RefreshFraudModel
  
  scope: singleton
  
  dependencies:
    requires: [TransactionDataSync]
    
  monitoring:
    track_duration: true
    alert_on_failure: critical
```

**Cross-reference**: Addendum AM (Scheduled Trigger Semantics)

---

## Key Design Decisions

### Intent Over Implementation

| Declared | NOT Declared |
|----------|--------------|
| View relationships | Layout/rendering |
| Navigation purposes | Nav bar styles |
| Field visibility | CSS/colors |
| Indicator conditions | Badge UI |
| On_unauthorized behavior | Error page design |

### Policy Integration

| Level | Policy Source |
|-------|---------------|
| Experience | `experience.policy_set` |
| Composition | `composition.navigation.policy_set` |
| Related View | `related.policy` or `related.policy_set` |
| Field | `fieldPermissions.visible.policy` |

### Hierarchy Rules

1. Roots have no `parent`
2. Hierarchy must be acyclic
3. `purpose` groups for rendering hints
4. `indicator` for dynamic state

---

## Guarantees Achieved

### ✅ Correctness (v1.3)
- Transactions are immutable facts
- Balance is derived, not stored
- Invariant violations are detected

### ✅ Availability (v1.3)
- Entity-scoped authority
- Regional failure isolation
- No global coordination

### ✅ SMS Flows (v1.3)
- Work units with pre/postconditions
- Atomic groups for transactional semantics
- Signals for event-driven coordination
- Checkpoints for durable state recovery
- Compensation for rollback logic
- Task pools for human review assignment

### ✅ UI Composition (v1.4)
- Semantic view grouping
- Policy-controlled visibility
- Field-level masking

### ✅ Navigation (v1.4)
- Hierarchical from parent relationships
- Purpose-based grouping
- Dynamic indicators

### ✅ Security (v1.4)
- Role-based access
- Field masking for PII
- Experience-level policies

### ✅ Assets (v1.5)
- Check images with lifecycle management
- Transformation hints for thumbnails
- 7-year retention for compliance

### ✅ Notifications (v1.5)
- Multi-channel alerts (email, SMS, push)
- Delivery tracking and retry
- User preference respect

### ✅ Scheduled Automation (v1.5)
- Monthly statement generation
- Timezone-aware scheduling
- Idempotent execution

### ✅ External Integration (v1.5)
- Credit bureau with circuit breaker
- Fallback to cached data
- Observability and alerting

### ✅ Data Governance (v1.5)
- PII classification and tracking
- GDPR right to erasure
- 7-year audit retention

### ✅ Search (v1.5)
- Transaction search with facets
- User-scoped security
- Ranking and pagination

### ✅ Accessibility (v1.5)
- High contrast and font scaling
- Screen reader support
- Keyboard navigation

### ✅ Conversational UI (v1.5)
- Voice and chat interfaces
- Multi-turn conversations
- Human handoff fallback

### ✅ Long-Running Processes (v1.5)
- Loan approval workflow
- Human task integration
- State persistence and compensation

### ✅ Delegation (v1.5)
- Authorized user access
- Time-bounded permissions
- Consent capture and audit

### ✅ Spatial Types (v1.5)
- Branch and ATM location discovery
- Geofenced service areas
- Distance-based search and sorting

### ✅ Graph Queries (v1.5)
- Household account relationships
- Referral chain tracking
- Fraud network detection

### ✅ Collaborative Sessions (v1.5)
- Joint account financial planning
- Advisor-client loan review
- Real-time presence and shared state

### ✅ Structured Content (v1.5)
- Rich loan documentation
- Signature blocks and disclosure boxes
- Mortgage contracts with property descriptions

### ✅ Inference/Derived Fields (v1.5)
- Real-time fraud scoring
- Transaction categorization
- Credit risk and churn prediction

### ✅ Edge Devices (v1.5)
- ATM terminals with offline capability
- POS terminals with transaction buffering
- Self-service kiosks with accessibility

### ✅ Offline Support (v1.5)
- Mobile banking in connectivity-limited areas
- Transaction queueing and sync
- Cached balance display with warnings

---

## Summary

This complete Banking example demonstrates **all 28 v1.5 addendums** in production-appropriate contexts. See the [Feature Coverage Matrix](#feature-coverage-matrix) in the Overview section for a complete mapping of addendums to sections.

---

# Part III: Grammar Quick Examples

This section provides concise examples for key grammar elements. For complete specifications demonstrating all v1.5 features, see the [Complete Banking Example](#part-ii-complete-banking-example-all-features) above or the comprehensive examples below.

## Basic Entity with Authority

```yaml
dataType:
  name: Order
  version: v1
  fields:
    id: uuid
    customer_id: uuid
    total: decimal
    status: enum[draft, paid, fulfilled]

model:
  name: Order
  mutability:
    scope: entity
    exclusive: true
  authority:
    scope: entity
    resolution: entity.region
```

## Append-Only Entity (Immutable Facts)

```yaml
dataType:
  name: Transaction
  version: v1
  fields:
    id: uuid
    timestamp: timestamp
    amount: decimal
    type: enum[debit, credit]

model:
  name: Transaction
  mutability:
    scope: append-only
```

## Relationship with Causality

```yaml
relationship:
  name: order_contains_item
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
view:
  name: OrderTotal
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
inputIntent:
  name: CreateOrder
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
              - work_unit: validate_order
              - work_unit: reserve_inventory
        - work_unit: charge_payment
          boundary: billing
      failure:
        policy: compensate
        compensations:
          - on: charge_payment
            do: release_inventory
```

## Policy with RBAC

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

## v1.5: Form Intent Binding

```yaml
presentationView:
  name: CreateOrderForm
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
dataType:
  name: Product
  version: v1
  fields:
    id: uuid
    name: string
    image: asset

asset:
  name: image
  types: [image/jpeg, image/png]
  max_size: 5MB
  transformations:
    thumbnail: resize(width: 200, height: 200)
  retention:
    duration: permanent
```

## v1.5: Scheduled Trigger

```yaml
scheduledTrigger:
  name: DailyReport
  schedule: "0 0 * * *"  # Daily at midnight
  intent: GenerateDailyReport
  scope: singleton
  idempotency:
    key: [date]
```

## v1.5: Notification

```yaml
notification:
  name: OrderShipped
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
search:
  name: ProductSearch
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
dataGovernance:
  model: Customer
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

# Part IV: Comprehensive Production Examples

This section contains complete reference implementations for 15 real-world applications. Each example is production-ready and demonstrates the full SMS v1.5 specification.

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

For a complete demonstration of ALL 28 v1.5 addendums, see [Part II: Complete Banking Example](#part-ii-complete-banking-example-all-features) which provides a fully-worked example suitable for production reference.

---

# Part V: Common Patterns Library

This section catalogs reusable patterns for common requirements.

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
authority:
  scope: entity
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

## Pattern: Real-Time Collaboration

**Use Case**: Document editing, whiteboard, design tools

**Session**:
```yaml
collaborativeSession:
  name: DocumentEditSession
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
durableWorkflow:
  name: MultiStepProcess
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
delegation:
  name: AccountDelegate
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
search:
  name: ProductSearch
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

## Pattern: External API Integration

**Use Case**: Payment processors, credit bureaus, shipping APIs

**Integration**:
```yaml
externalIntegration:
  name: PaymentProcessor
  resilience:
    circuit_breaker:
      threshold: 5
      timeout: 30s
    retry:
      max_attempts: 3
    fallback:
      strategy: use_cached
```

**Key Principle**: Circuit breaker with fallback for resilience.

---

## Pattern: Data Governance

**Use Case**: GDPR compliance, PII management, retention policies

**Governance**:
```yaml
dataGovernance:
  model: Customer
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

---

## Pattern Selection Guide

| Requirement | Pattern |
|-------------|---------|
| Balance/Total tracking | Ledger/Balance |
| Stock management | Inventory |
| Time-based booking | Scheduling |
| One action per entity | Voting |
| Regional data | Multi-Region Authority |
| Live editing | Real-Time Collaboration |
| Multi-day process | Durable Workflows |
| Shared access | Delegation with Consent |
| User search | Search with Facets |
| Third-party service | External API Integration |
| PII compliance | Data Governance |

---

## Conclusion

This document provides:
1. **Modeling philosophy** - Invariant-oriented design principles
2. **Practical guidance** - Checklists, templates, and decision trees
3. **Complete Banking example** - Production-ready reference demonstrating ALL 28 v1.5 addendums
4. **Quick reference** - Grammar examples for common elements
5. **Production examples** - 15 real-world applications across industries
6. **Pattern library** - Reusable solutions for common problems

Use this document to learn SMS v1.5, model new applications, and find proven patterns for your requirements. The Banking example in Part II provides a complete, contextually-appropriate demonstration of every v1.5 feature.

---

**Document Version**: 1.5  
**Status**: Normative Guidance and Examples  
**Last Updated**: 2026-01-18
