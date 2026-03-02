# Appendix G: Modeling Playbook (v1.5)

## Overview

This playbook provides step-by-step guidance for designing data models that work well with SMS's invariant-oriented approach. Use this when starting a new model or refactoring an existing one.

**Version History**:
- **v1.3**: Initial playbook with entity, relationship, and view design checklists
- **v1.5**: Added guidance for assets, search, spatial data, and external integrations

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

## Common Patterns

### Pattern 1: Ledger

**Entities**:
- `Account`: Identity, type, ownership
- `Transaction`: Immutable fact (amount, parties, timestamp)

**Relationships**:
- Transaction → Account (debit)
- Transaction → Account (credit)

**Views**:
- `AccountBalance`: sum(credits) - sum(debits)

**Invariant**: `AccountBalance.result >= 0`

### Pattern 2: Inventory

**Entities**:
- `Product`: Identity, description, SKU
- `StockEvent`: Immutable adjustment (quantity, reason, timestamp)

**Relationships**:
- StockEvent → Product (affects)

**Views**:
- `ProductStock`: sum(adjustments)

**Invariant**: `ProductStock.result >= 0`

### Pattern 3: Scheduling

**Entities**:
- `Resource`: Identity, type, availability rules
- `Booking`: Immutable reservation (start, end, resource)

**Relationships**:
- Booking → Resource (reserves)

**Views**:
- `ResourceSchedule`: time slots with bookings

**Invariant**: No overlapping bookings for same resource

### Pattern 4: Voting

**Entities**:
- `Voter`: Identity, eligibility
- `Vote`: Immutable ballot (choice, timestamp)

**Relationships**:
- Vote → Voter (cast_by)

**Views**:
- `VoterStatus`: has_voted flag
- `TallyView`: count per choice

**Invariant**: At most one vote per voter

---

## Anti-Pattern Recognition

### 🚫 Anti-Pattern: Mutable Aggregate

**Symptom**: Entity contains field that requires reading other entities to compute.

```yaml
# BAD
fields:
  balance: decimal
  total_orders: integer
  last_activity: timestamp
```

**Fix**: Move to view.

### 🚫 Anti-Pattern: Cross-Entity Precondition

**Symptom**: Write requires reading another entity.

```yaml
# BAD
precondition: target_account.is_active == true
```

**Fix**: Accept write, validate in view, handle violations.

### 🚫 Anti-Pattern: Synchronous Invariant

**Symptom**: Invariant must hold immediately after write.

**Fix**: Accept eventual consistency, use views for detection.

### 🚫 Anti-Pattern: Authority Coupling

**Symptom**: Two entities that frequently update together are in different authority scopes.

**Fix**: Either merge entities, or accept eventual consistency.

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

**Version**: 1.5
**Status**: Normative Guidance
**Last Updated**: 2026-01-18
