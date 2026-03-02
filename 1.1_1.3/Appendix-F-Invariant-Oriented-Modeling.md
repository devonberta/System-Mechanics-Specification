# Appendix F: Invariant-Oriented & Relational Data Modeling (v1.3)

## Overview

This appendix provides guidance on decomposing complex business invariants across entities, relationships, and views rather than embedding them in single entities. This approach enables SMS to handle sophisticated correctness requirements without requiring distributed transactions or global coordination.

---

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

## Invariant Placement Table

| Invariant Type | Placement | Enforcement | Example |
|----------------|-----------|-------------|---------|
| Single-field constraint | Entity | Write-time | `age >= 0` |
| Cross-field constraint | Entity | Write-time | `end > start` |
| Cross-entity aggregate | View | Read-time | `balance >= 0` |
| Business rule | Policy | Evaluation-time | `user.verified` |
| Audit requirement | View + Policy | Continuous | Transaction history |

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

## Summary

Invariant-oriented modeling in SMS follows these principles:

1. **Entities are independent** - writable without coordination
2. **Relationships are semantic** - express links, not transactions
3. **Views absorb complexity** - aggregate, validate, derive
4. **Invariants are detected** - not blocked at write time
5. **Authority is entity-scoped** - failure isolation by design

This approach enables SMS to handle complex business requirements while maintaining the system's core guarantees around availability, partition tolerance, and evolution.

---

**Version**: 1.3
**Status**: Normative Guidance
**Last Updated**: 2026-01-03
