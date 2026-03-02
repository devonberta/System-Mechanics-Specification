# Appendix G: Modeling Playbook (v1.3)

## Overview

This playbook provides step-by-step guidance for designing data models that work well with SMS's invariant-oriented approach. Use this when starting a new model or refactoring an existing one.

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

- [ ] All invariants identified and classified
- [ ] Cross-entity invariants in views
- [ ] No derived state in entities
- [ ] Relationships explicit with cardinality
- [ ] Authority scope appropriate
- [ ] Migration policy defined
- [ ] Violation handling specified
- [ ] Views are rebuildable
- [ ] Model passes independence test

---

**Version**: 1.3
**Status**: Normative Guidance
**Last Updated**: 2026-01-03
